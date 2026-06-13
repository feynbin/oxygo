# Oxygo 系统架构设计

> 这是最新的设计文档，取代之前分散的 `design.md` 系列文档。
>
> 日期：2026-06-13
> 版本：v0.2
> 状态：架构确认阶段

---

## 1. 项目概述

Oxygo 是一套基于 [sing-box](https://github.com/SagerNet/sing-box) 内核构建的多端代理/隧道管理系统。

### 1.1 三端定义

| 端 | 名称 | 部署位置 | 形态 | 用户 |
|---|---|---|---|---|
| **服务端** | oxygo-server | Linux VPS | Go 守护进程 + sing-box 库 + 内嵌 Web 管理页面 | 节点管理员 |
| **管理端** | oxygo-admin | 任意可部署 Web 的服务器 | Web 平台（Go 后端 + 前端 SPA） | 全局管理员 |
| **客户端** | oxygo-client | 终端用户电脑 | Wails3 桌面应用程序（exe） | 普通用户 |

### 1.2 设计原则

1. **服务端是真正的服务提供者**：sing-box 提供所有入站/出站/路由/DNS 能力，oxygo 只负责调度与下发
2. **控制面与数据面分离**：admin ↔ server 之间通过 gRPC + mTLS 双向认证长连接通信，admin 只下发声明式配置
3. **三端各司其职**：服务端提供能力+自检、管理端全局管控、客户端消费服务
4. **零信任默认**：所有跨网络通信加密，通过 CA 体系建立互信

---

## 2. 整体架构

```
                    ┌──────────────────┐
                    │   oxygo-admin    │  ◀── Web 管理平台
                    │   (Web SPA)      │      全局监控、订阅、
                    │                  │      用户管理、流量看板
                    └────────┬─────────┘
                             │ gRPC + mTLS（控制面）
                             │ 双向流：配置下发、状态上报
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ oxygo-server │ │ oxygo-server │ │ oxygo-server │  ◀── 每台 VPS 一个节点
    │   sing-box   │ │   sing-box   │ │   sing-box   │      自带 Web 管理页面
    │  + Web UI   │ │  + Web UI   │ │  + Web UI   │      查看本机健康/状态/流量
    └──────────────┘ └──────────────┘ └──────────────┘
            │                                        │
            └────────── HTTPS 订阅拉取 ──────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  oxygo-client    │  ◀── Wails3 桌面 exe
                    │  (桌面应用程序)   │      选择订阅、连接、
                    │                  │      设置、状态查看
                    └──────────────────┘
```

### 2.1 数据流

| 数据流 | 源 → 目标 | 协议 | 说明 |
|--------|----------|------|------|
| 控制流 | admin ↔ server | gRPC + mTLS | 配置下发、状态上报、命令执行 |
| 订阅流 | server → client | HTTPS | 客户端拉取订阅配置 |
| 管理流 | admin Web → server | gRPC | 管理员操作转发到节点 |
| 代理流 | client ↔ server | VLESS / Hysteria2 | 用户实际业务流量 |

---

## 3. 服务端 (oxygo-server)

### 3.1 角色定位

服务端是**真正的服务提供者**，跑在 VPS 上，每台 VPS 一个节点。

### 3.2 功能清单

| 模块 | 功能 |
|---|---|
| **sing-box 代理** | 提供 VLESS+Reality / Hysteria2 双协议入站 |
| **Web 管理页面** | 内嵌轻量 Web UI，查看本机健康状态、服务运行状况、实时流量、连接数、日志 |
| **Agent 控制** | 启动时反连 admin gRPC，接收配置/重启/健康检查指令 |
| **配置管理** | 本地缓存最新配置（BoltDB），admin 断连时继续服务 |
| **端口跳跃** | Hysteria2 的 UDP 端口跳跃（nftables 自动配置） |
| **证书管理** | 接收 admin 下发的 TLS 证书，自动 reload |
| **指标采集** | 通过 sing-box ClashAPI 采集流量、连接数，上报 admin |

### 3.3 进程模型

```
┌──────────────────────────────────────────┐
│          oxygo-server (Go 进程)            │
│                                          │
│  ┌──────────┐  ┌──────────────┐          │
│  │ Web UI   │  │  Agent       │          │
│  │ (内嵌 HTTP)│  │  gRPC Client  │          │
│  │ :8080    │  │  反连 admin  │          │
│  └──────────┘  └──────┬───────┘          │
│                       │                  │
│  ┌────────────────────▼────────────────┐ │
│  │        Supervisor (sing-box 库)       │ │
│  │  双实例切换式 reload、失败自动回滚    │ │
│  └────────────────┬────────────────────┘ │
│                   │                      │
└───────────────────┼──────────────────────┘
                    │
         TCP:443 (VLESS-Reality)
         UDP:443 (Hysteria2)
```

---

## 4. 管理端 (oxygo-admin)

### 4.1 角色定位

管理端是**全局控制中心**，一个 Web 平台集中管理所有服务端节点。

### 4.2 功能清单

| 模块 | 功能 |
|---|---|
| **节点监控** | 所有服务端在线状态、CPU/内存/带宽、版本、健康度 |
| **用户管理** | CRUD 用户，UUID/密码/到期时间/流量配额 |
| **订阅管理** | 为用户生成订阅链接，支持 sing-box JSON / Clash / base64 多格式 |
| **流量看板** | 按用户/节点/时间维度查看流量曲线，超额告警 |
| **协议模板** | VLESS-Reality、Hysteria2 等模板化配置，可视化表单生成 |
| **证书中心** | 内置 CA，签发 mTLS 证书；ACME DNS-01 自动申请通配符证书 |
| **综合看板** | Dashboard 概览：节点数、在线用户、今日流量、告警事件 |
| **告警** | 节点离线、流量异常、证书即将过期推送（邮件/Webhook） |

### 4.3 技术栈

| 层 | 选型 |
|---|---|
| 后端 | Go（标准库 + grpc-go） |
| 前端 | 任意 SPA 框架（待定） |
| 数据库 | SQLite |
| 通信 | gRPC + mTLS（与 server 控制面） |

---

## 5. 客户端 (oxygo-client)

### 5.1 角色定位

客户端是**终端用户使用的桌面软件**，当前这个 Wails3 项目就是客户端。

### 5.2 功能清单

| 模块 | 功能 |
|---|---|
| **订阅管理** | 录入订阅 token → 拉取节点列表 → 本地解析，定时刷新 |
| **节点选择** | 手动选择 / 自动延迟测速 / 故障自动切换 |
| **连接管理** | 一键连接/断开，显示当前节点、协议、连接时长、累计流量 |
| **代理模式** | 系统代理 / TUN 模式 / 仅规则模式 |
| **路由规则** | 直连/代理/拦截分流，GeoIP+GeoSite，自定义域名/IP 列表 |
| **设置** | 开机自启、自动更新、代理端口、DNS 设置 |
| **诊断** | 内置 ping、curl、DNS 查询、连通性体检 |
| **日志** | 本地日志，可导出问题反馈包 |

### 5.3 技术栈（当前）

| 层 | 选型 |
|---|---|
| 桌面框架 | Wails v3 |
| 前端 | Svelte 5 + TypeScript + Vite |
| VPN 内核 | sing-box 作为 Go 库内嵌（计划中） |
| 本地存储 | SQLite |
| 系统代理 | WinINET / networksetup / gsettings |

---

## 6. 协议设计

### 6.1 代理协议

| 协议 | 传输层 | 端口 | 定位 |
|------|--------|------|------|
| **VLESS + Vision + Reality** | TCP | 443 | 主力，抗审查最强，无需域名 |
| **Hysteria2** | UDP (QUIC) | 443 + 端口跳跃 | 高性能补充，弱网场景 |

### 6.2 控制面协议 (gRPC)

```protobuf
syntax = "proto3";
package oxygo.v1;

// admin <-> server 双向控制面
service NodeAgent {
  // Agent 启动后反连 admin 注册
  rpc Register(RegisterRequest) returns (RegisterResponse);
  // admin 推送配置变更（server-streaming）
  rpc WatchConfig(WatchRequest) returns (stream ConfigRevision);
  // 节点定期上报指标与心跳（client-streaming）
  rpc ReportStats(stream NodeStats) returns (ReportAck);
  // 单次命令（重启、健康检查、日志拉取）
  rpc ExecCommand(CommandRequest) returns (stream CommandChunk);
}
```

### 6.3 订阅协议 (HTTPS)

```
GET /api/v1/subscribe?token=xxx&fmt=singbox|clash|base64
```

由 admin 端提供订阅服务，返回已渲染好的客户端配置（含所有节点的 outbound + urltest）。

---

## 7. 安全设计

| 风险 | 缓解措施 |
|---|---|
| Web 界面暴露 | server Web UI 只监听 localhost 或绑定 mTLS |
| 控制面被攻击 | gRPC + mTLS 双向认证 + NodeSpec 签名校验 |
| 节点被入侵 | 每节点独立证书，无法冒充其他节点 |
| 订阅 token 泄露 | admin 端可一键吊销重新签发 |
| 流量数据 | 仅记录字节数与时间窗，不记录目的域名/IP |

---

## 8. 仓库结构

```
oxygo/
├── cmd/
│   ├── client/          ← Wails3 桌面客户端（当前代码）
│   │   ├── main.go          ← 入口
│   │   ├── app.go
│   │   ├── frontend/        ← Svelte 前端
│   │   └── build/           ← Wails3 构建配置
│   ├── server/          ← 服务端（Go 守护进程 + sing-box）
│   │   └── main.go
│   └── admin/           ← 管理端（Web 平台）
│       └── main.go
├── internal/
│   ├── proto/           ← gRPC 生成代码
│   ├── ca/              ← CA 与证书管理
│   ├── store/           ← 数据持久化层
│   ├── types/           ← 共享数据类型
│   └── transport/       ← gRPC 通信封装
├── proto/               ← .proto 源文件
├── docs/                ← 设计文档
├── frontend/            ← （当前 Wails3 客户端的前端目录）
├── build/               ← （当前 Wails3 客户端的构建配置）
├── main.go              ← （当前 Wails3 入口，属于 cmd/client）
├── go.mod
├── Taskfile.yml
└── README.md
```

> 当前代码（`main.go`、`greetservice.go`、`frontend/`、`build/`）属于客户端，暂留在根目录以保持 Wails3 dev 工具的正常工作。后续可以迁入 `cmd/client/`。

---

## 9. 里程碑

| 阶段 | 范围 | 预估 |
|---|---|---|
| **M0 - 仓库搭建** | 目录结构、Go workspace、proto 骨架、CI 基础 | 已完成 |
| **M1 - 服务端 MVP** | sing-box 库集成，硬编码配置启动 Reality/Hy2，本地 Web UI 查看状态 | 2 周 |
| **M2 - 管理端 MVP** | gRPC 控制面打通，admin 能查看节点状态、下发配置 | 2 周 |
| **M3 - 客户端 MVP** | 订阅拉取、连接/断开、TUN 模式、节点切换 | 2 周 |
| **M4 - 功能完善** | 用户管理、流量统计、订阅生成、协议模板 | 2 周 |
| **M5 - 安全与运维** | CA 体系、证书自动轮换、告警、审计日志 | 1 周 |
| **M6 - 打包发布** | 三端构建签名、自动更新、文档 | 1 周 |

---

## 10. 构建与运行指南

### 10.1 前置要求

| 工具 | 版本 | 用途 |
|------|:----:|------|
| Go | 1.25+ | 三端后端编译 |
| Node.js | 20+ | 前端构建 |
| Bun | 最新 | 前端依赖管理（Wails3 默认） |
| Wails3 CLI | v3.0.0-alpha.100+ | 客户端开发与构建 |
| Task | 最新 | 任务编排（Taskfile.yml） |

安装 Wails3 CLI：

```bash
go install github.com/wailsapp/wails/v3/cmd/wails3@latest
```

### 10.2 客户端 (oxygo-client)

客户端是 Wails3 桌面应用，代码当前在项目根目录。

#### 开发模式（热重载）

```bash
# 在项目根目录执行
wails3 dev
```

- 启动桌面窗口，Go 后端和 Svelte 前端都有热重载
- 默认 Vite 端口 9245
- 前端修改即时生效，Go 修改自动重编译

#### 构建可执行文件

```bash
# 构建当前平台的客户端
wails3 build

# 产物在 bin/ 目录下
# Windows: bin/oxygo.exe
# macOS:   bin/oxygo.app
# Linux:   bin/oxygo
```

#### 跨平台构建

```bash
# Windows (在 Windows 上执行)
wails3 build -platform windows/amd64

# macOS (在 macOS 上执行)
wails3 build -platform darwin/amd64
wails3 build -platform darwin/arm64

# Linux
wails3 build -platform linux/amd64
```

> 注：Wails3 的跨平台构建需要对应的操作系统，目前不支持从单一平台交叉编译出其他平台的 GUI 应用。

---

### 10.3 服务端 (oxygo-server)

服务端是纯 Go 守护进程，跑在 Linux VPS 上，不依赖 Wails3。

#### 本地开发（当前为骨架，暂无实际功能）

```bash
# 直接运行
go run ./cmd/server

# 带 build tags 运行（启用 sing-box 相关功能后）
go run -tags "with_quic,with_clash_api" ./cmd/server
```

#### 构建 Linux 二进制

```bash
# 从开发机交叉编译到 Linux amd64
GOOS=linux GOARCH=amd64 go build -o oxygo-server ./cmd/server

# 带必要的 build tags
GOOS=linux GOARCH=amd64 go build \
  -tags "with_quic,with_clash_api" \
  -o oxygo-server \
  ./cmd/server

# 传输到 VPS
scp oxygo-server root@<vps-ip>:/usr/local/bin/
```

#### VPS 部署（systemd）

将 oxygo-server 注册为系统服务，实现开机自启和崩溃自愈：

```bash
# 创建配置文件目录
sudo mkdir -p /etc/oxygo

# 创建服务文件 /etc/systemd/system/oxygo-server.service
```

```ini
[Unit]
Description=Oxygo VPN Node Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=oxygo
Group=oxygo
ExecStart=/usr/local/bin/oxygo-server --config /etc/oxygo/agent.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/oxygo /var/log/oxygo

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
sudo systemctl daemon-reload
sudo systemctl enable --now oxygo-server
sudo systemctl status oxygo-server

# 查看日志
sudo journalctl -u oxygo-server -f
```

#### Docker 部署

```bash
# 构建 Docker 镜像
docker build -t oxygo-server -f build/docker/Dockerfile .

# 运行容器
docker run -d \
  --name oxygo-server \
  --cap-add=NET_ADMIN \
  --cap-add=NET_BIND_SERVICE \
  -p 443:443 \
  -p 443:443/udp \
  -v /etc/oxygo:/etc/oxygo \
  oxygo-server
```

> 注：Dockerfile 需要服务端代码完成后创建，路径为 `build/docker/Dockerfile`。

---

### 10.4 管理端 (oxygo-admin)

管理端是 Web 平台，Go 后端 + 前端 SPA。

#### 本地开发

```bash
# 启动后端（当前为骨架）
go run ./cmd/admin

# 带配置文件启动
go run ./cmd/admin --config ./admin.yaml
```

#### 构建部署

```bash
# 编译后端
go build -o oxygo-admin ./cmd/admin
```

管理端部署方式灵活（任选其一）：

| 方式 | 命令 | 适用场景 |
|------|------|---------|
| 裸机部署 | `./oxygo-admin --addr :7443` | 直接运行在生产服务器 |
| 反代部署 | Nginx 反代到 `127.0.0.1:7443` | 需要 TLS 终止和域名绑定 |
| Docker 部署 | `docker run -p 7443:7443 oxygo-admin` | 容器化环境 |

前端 SPA 构建产物嵌入 Go 二进制中（通过 `//go:embed`），部署时只需一个二进制文件。

---

### 10.5 Taskfile 自动化

项目提供了 `Taskfile.yml` 统一管理三端的构建任务（当前主要支持客户端，后续扩展）：

```bash
# 查看所有可用任务
task --list

# 构建客户端
task build

# 客户端开发模式
task dev

# 构建服务端（待实现后可用）
task build:server

# Docker 镜像构建
task build:docker
```

### 10.6 构建产物总览

| 端 | 构建命令 | 产物 | 目标平台 |
|----|---------|------|---------|
| 客户端 | `wails3 build` | `bin/oxygo.exe` 等 | Win/macOS/Linux |
| 服务端 | `go build ./cmd/server` | `oxygo-server` 二进制 | Linux amd64/arm64 |
| 管理端 | `go build ./cmd/admin` | `oxygo-admin` 二进制 | 任意支持 Go 的平台 |

> **当前状态**：只有客户端（Wails3）可以正常构建运行，服务端和管理端目前是骨架代码，`go run ./cmd/server` 和 `go run ./cmd/admin` 只会输出启动信息然后退出。等对应端的功能实现后，这里的命令就能真正工作了。

---

## 11. 与旧设计文档的关系

以下旧文档的设计思路仍然有参考价值，但架构定义以本文档为准：

| 旧文档 | 状态 | 说明 |
|--------|:----:|------|
| `design.md` | ❌ 已取代 | 旧的三端定义（admin 误定为桌面端），由本文档取代 |
| `design-server.md` | ⚠️ 参考 | sing-box 库模式集成、Supervisor 设计的细节仍有参考价值 |
| `design-hysteria2.md` | ⚠️ 参考 | Hysteria2 协议参数、ACME 证书、端口跳跃设计可复用 |
| `design-vless-reality.md` | ⚠️ 参考 | Reality 协议细节、dest 管理、X25519 密钥设计可复用 |
| `design-baseline-vs-oxygo.md` | ⚠️ 参考 | 手工 sing-box 与 Oxygo 的能力差异对照表 |
