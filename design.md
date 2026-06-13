# Oxygo VPN 系统设计文档

> 版本：v0.1 草案 ｜ 日期：2026-05-27 ｜ 状态：设计评审阶段

## 1. 项目概述

Oxygo 是一套基于 [sing-box](https://github.com/SagerNet/sing-box) 内核构建的多端代理/隧道管理系统，由三个独立子系统组成：

| 子系统 | 角色 | 形态 | 部署位置 |
|---|---|---|---|
| **oxygo-admin** | 管理端（控制面） | Wails3 桌面应用 | 运维/管理员工作机 |
| **oxygo-server** | 服务节点（数据面 Agent） | Go 守护进程 + sing-box | Linux VPS / 云主机 |
| **oxygo-client** | 客户端（终端用户） | Wails3 桌面应用 | 终端用户 PC/笔记本 |

### 1.1 设计原则

1. **协议栈完全收敛到 sing-box**：不自造轮子，所有入站/出站/路由/DNS 能力由 sing-box 提供，本项目只负责"调度与下发"。
2. **控制面与数据面分离**：gRPC + mTLS 双向认证的长连接通道，管理端只下发"声明式配置"，节点本地协调 sing-box 进程。
3. **零信任默认**：所有跨进程/跨网络通信加密；客户端、节点、管理端三方互信通过 CA 体系建立。
4. **声明式优于命令式**：管理端持有期望状态（desired state），节点定期上报实际状态（actual state），由 reconciler 推进收敛。

---

## 2. 整体架构

```
┌─────────────────────┐                ┌──────────────────────┐
│   oxygo-admin       │                │   oxygo-client       │
│   (Wails3 桌面)     │                │   (Wails3 桌面)      │
│   - 节点管理        │                │   - 订阅/连接        │
│   - 用户/订阅管理   │                │   - 内嵌 sing-box    │
│   - 配置编排        │                │   - TUN/系统代理     │
│   - 流量审计        │                └─────────┬────────────┘
└────────┬────────────┘                          │
         │ gRPC + mTLS                           │ HTTPS 订阅
         │ (Server Streaming)                    │ + 流量数据
         ▼                                       ▼
┌─────────────────────────────────────────────────────────────┐
│   oxygo-server  (Linux 守护进程，多节点)                     │
│   ┌────────────────────┐    ┌────────────────────────────┐  │
│   │  Agent (Go)        │◀──▶│  sing-box (子进程/库)       │  │
│   │  - 配置 reconcile  │    │  - VLESS/Trojan/Hysteria2  │  │
│   │  - 流量/连接统计   │    │  - TLS / Reality / uTLS    │  │
│   │  - 健康上报        │    │  - 路由/DNS                │  │
│   └────────────────────┘    └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 数据流

- **控制流**：admin → server（配置下发）、server → admin（状态/流量上报），双向 gRPC stream。
- **订阅流**：client → admin（HTTPS 拉取订阅 token 对应的 sing-box JSON 配置）。
- **代理流**：client ↔ server（用户实际业务流量，走 sing-box 选定的协议）。

---

## 3. oxygo-admin（管理端）

### 3.1 功能清单

| 模块 | 功能 |
|---|---|
| **节点管理** | 添加/删除/分组节点；查看节点在线状态、版本、CPU/内存/带宽；远程触发重启 sing-box；查看节点 sing-box 实时日志（流式） |
| **协议模板** | 内置 VLESS-Reality、Trojan-TLS、Hysteria2、ShadowTLS 等模板；可视化表单生成对应 sing-box `inbounds` 片段；**dest 候选注册表**（Reality 用）：维护、健康检查、按区域推荐 |
| **用户管理** | CRUD 用户；每用户绑定 UUID/密码/到期时间/流量配额；批量导入导出 |
| **订阅管理** | 为用户生成订阅链接（含 token）；支持 sing-box 原生 JSON、Clash、通用 base64 节点列表三种格式 |
| **路由/规则** | 全局路由策略（GeoIP/GeoSite/自定义规则集）编辑与下发 |
| **流量审计** | 按用户/节点/时间维度查看流量；超额告警；连接数实时面板 |
| **证书中心** | 内置 CA，签发 admin/server/client 的 mTLS 证书；通过 ACME **DNS-01** 申请通配符证书供 Hysteria2 入站使用；**DNS Provider 凭据托管**（Cloudflare 首版） |
| **告警** | 节点离线、流量异常、证书即将过期、sing-box 进程崩溃等事件推送（邮件/Webhook） |

### 3.2 技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 桌面框架 | Wails v3 (`v3.0.0-alpha.96`) | 已选定；Go 后端 + Webview 前端，无 Electron 体积 |
| 前端 | Svelte 5 + TypeScript + Vite 8 | 已选定；Runes 模式响应式 |
| UI 组件 | **Skeleton UI** 或 **shadcn-svelte**（待定） | 现代深色风格 + Tailwind |
| 状态管理 | Svelte 5 `$state` runes | 内置即可，无需 Pinia/Redux |
| 本地存储 | **SQLite**（`modernc.org/sqlite` 纯 Go 实现） | 单机管理元数据，无 CGO 依赖 |
| ORM | **Ent** 或 **sqlc** | Ent 适合 schema 演进；sqlc 类型安全且零运行时反射 |
| gRPC | `google.golang.org/grpc` + `protoc-gen-go` | 控制面唯一通信协议 |
| 证书 | `crypto/x509` + `cloudflare/cfssl`（可选） | 内置 CA，避免外部依赖 |
| 配置生成 | `github.com/sagernet/sing-box` 作为**库** 引用 | 直接复用其 schema 定义与校验逻辑，避免 JSON 漂移 |

### 3.3 数据模型（核心表）

```
nodes(id, name, group, host, grpc_port, public_addr, region, status, last_seen, cert_id, version)
users(id, username, uuid, password_hash, traffic_quota, traffic_used, expire_at, enabled)
inbounds(id, node_id, protocol, port, transport, tls_config_json, tag)
subscriptions(id, user_id, token, format, node_filter, created_at, revoked)
traffic_logs(id, user_id, node_id, up_bytes, down_bytes, window_start, window_end)
audit_logs(id, actor, action, target, payload, created_at)
certs(id, kind, subject, not_before, not_after, pem, key_encrypted)
```

### 3.4 关键交互

- **添加节点向导**：管理员录入节点 SSH/IP → admin 通过 SSH（一次性）推送 oxygo-server 二进制 + 客户端证书 → 启动 systemd 服务 → 节点回连 gRPC → 入库激活。
- **配置下发**：管理员保存变更 → admin 推送 `ApplyConfig(ConfigRevision)` → server 端校验 → 调用 sing-box `reload` → 上报应用结果与新 revision。

---

## 4. oxygo-server（服务节点）

### 4.1 功能清单

| 模块 | 功能 |
|---|---|
| **Agent 控制** | 启动时携带客户端证书反连 admin 的 gRPC；接收配置/重启/健康检查指令 |
| **sing-box 生命周期** | 启动、双实例切换式 reload、优雅停止、崩溃自愈（指数退避重启）、版本自检与升级 |
| **配置管理** | 本地 `/var/lib/oxygo`（admin 下发的 NodeSpec + 渲染缓存）、`/etc/oxygo/agent.yaml`（Agent 自身配置：admin 地址、证书路径） |
| **指标采集** | 通过 sing-box ClashAPI 抓取连接数、上下行字节；按用户标签聚合 |
| **端口跳跃** | Hy2 inbound 启用时，自动下发/清理 nftables（回退 iptables）UDP 端口段 redirect 规则 |
| **日志聚合** | 接管 sing-box log factory，按等级过滤，流式回传 admin |
| **本地防护** | 限制 Agent 可写路径；关键文件 SELinux/AppArmor 友好；fail2ban 集成可选 |

### 4.2 技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 语言 | Go 1.25 | 与 admin 共用 protobuf 定义 |
| 进程模型 | 单二进制 + systemd unit | 部署简单，无 Docker 强依赖 |
| sing-box 集成 | **库模式**（`github.com/sagernet/sing-box` 直接 import） | 减少 IPC、统一日志、便于单元测试；详见 [design-server.md](./design-server.md) §3 |
| sing-box API | 启用 `experimental.clash_api`（仅 127.0.0.1） | 统一从 ClashAPI 抓统计与连接 |
| 配置下发 | gRPC stream `Watch(ConfigSpec) stream Patch` | 增量下发减少带宽 |
| 持久化 | 本地 BoltDB（`go.etcd.io/bbolt`）缓存最近一次有效配置 | 即使 admin 不可达也能继续服务 |

### 4.3 关键设计

- **Reconcile 循环**：Agent 维护 `desired ↔ actual` 状态机，每次收到新 revision 重新渲染 `option.Options` 并触发**双实例切换式 reload**；失败回滚到上一个 revision。
- **mTLS 启动证书**：首次部署时 admin 通过 SSH 注入证书；后续证书轮换通过当前 gRPC 通道下发新证书，旧证书延后失效（重叠期 24h）。
- **停机式 reload**：sing-box 库模式下监听端口无法跨实例共享，v1 采用先 `box.Close()` 再 `box.New()+Start()` 的方式，每次配置变更约 1~3s 连接中断。零停机改造留待 v2（端口复用 + 多实例）。

---

## 5. oxygo-client（客户端）

### 5.1 功能清单

| 模块 | 功能 |
|---|---|
| **订阅** | 录入订阅链接 → 拉取节点列表 → 本地解析；定时刷新（间隔可配） |
| **节点选择** | 手动选择 / 自动延迟测速（TCP+ICMP+HTTP）/ 故障自动切换 |
| **连接管理** | 一键连接/断开；显示当前节点、协议、连接时长、累计流量 |
| **代理模式** | 系统代理（HTTP/SOCKS5 自动设置）/ TUN 模式（全局接管）/ 仅规则模式 |
| **路由规则** | 直连/代理/拦截分流；GeoIP+GeoSite；自定义域名/IP 列表 |
| **诊断** | 内置 ping、curl、DNS 查询、连通性体检 |
| **日志** | 本地环形缓冲日志，可选上传问题反馈包 |
| **自启动 & 自动更新** | 开机启动；版本检查；增量更新（仅二进制差量） |

### 5.2 技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 桌面框架 | Wails v3（同 admin） | 跨平台 Win/macOS/Linux |
| 前端 | Svelte 5 + TS（同 admin） | 共享部分 UI 组件库 |
| VPN 内核 | **sing-box 作为 Go 库内嵌** | 客户端需要 TUN，库模式更易管理虚拟网卡生命周期 |
| TUN 驱动 | Windows: **Wintun**；macOS: utun；Linux: tun.ko | sing-box 已封装跨平台 TUN |
| 权限提升 | Windows: UAC manifest；macOS: Helper Tool (SMJobBless)；Linux: polkit / setcap | TUN 与系统路由表需要管理员权限 |
| 系统代理设置 | Win: WinINET API；macOS: `networksetup`；Linux: gsettings/env | 已有成熟方案 |
| 订阅拉取 | 标准库 `net/http` + 用户证书 mTLS（可选） | 订阅服务由 admin 提供 |
| 本地存储 | SQLite（同 admin 技术栈） | 节点列表、规则、流量统计 |
| 自动更新 | `github.com/wailsapp/wails/v3` 的 update 模块 或 Sparkle 协议 | 待评估 |

### 5.3 关键设计

- **权限隔离**：UI 进程以普通用户权限运行；TUN/路由操作通过本地 IPC 调用一个常驻特权 Helper（Windows Service / launchd / systemd --user 提权服务）。降低被 webview 漏洞利用的攻击面。
- **配置生成**：客户端从订阅获得 sing-box 配置片段（仅 outbounds + 路由），与本地的 inbound（TUN/Mixed 入站）合并后启动。
- **故障切换**：内置健康检查（每 30s 测当前节点 HTTP 200），失败 3 次自动切换到延迟最低的备用节点。

---

## 6. 跨子系统协议设计

### 6.1 gRPC 服务定义（草案）

```protobuf
syntax = "proto3";
package oxygo.v1;

// admin <-> server
service NodeAgent {
  // Agent 启动后调用，建立长连接
  rpc Register(RegisterRequest) returns (RegisterResponse);
  // 服务端推送配置变更（admin 是 server，节点是 client）
  rpc WatchConfig(WatchRequest) returns (stream ConfigRevision);
  // 节点定期上报指标 & 心跳
  rpc ReportStats(stream NodeStats) returns (ReportAck);
  // 单次命令（重启、健康检查、日志拉取）
  rpc ExecCommand(CommandRequest) returns (stream CommandChunk);
}

// client <-> admin（HTTPS REST，非 gRPC，便于通过 CDN）
// GET /api/v1/subscribe?token=xxx&fmt=singbox|clash|base64
// POST /api/v1/report-traffic  （可选，客户端侧统计）
```

### 6.2 配置同步策略

- admin 下发的是**上层抽象 `NodeSpec`**（含 inbounds、users、可选通配符证书 `Cert.WildcardDomain`），节点本地 `singboxgen` 渲染为 sing-box `option.Options`。详见 [design-server.md](./design-server.md) §2.3。
- 每个 revision 含 `revision_id`（单调递增）、`sha256`、`signature`（admin CA 对 NodeSpec 签名，而非渲染后 JSON）。
- 节点校验签名后落盘 BoltDB，reload 成功才回 ACK；admin 据此渲染节点同步状态。

### 6.3 证书与信任链

```
Oxygo Root CA  (admin 持有，离线/受保护)
 ├── admin-server-cert   (gRPC server 用)
 ├── node-cert-<nodeID>  (每节点一证一密钥)
 └── client-cert-<userID> (可选，用于强认证订阅)
```

- 证书有效期：节点 90d，客户端 365d，CA 10y。
- 撤销：本地 CRL（小规模）+ OCSP（后期可加）。

> **注**：上述 CA 体系仅用于 admin↔server↔client 的**控制面 mTLS**。两类协议入站使用的 TLS 证书与本 CA 体系**无关**：
> - **Hysteria2** 使用 admin 通过 ACME DNS-01 申请的**通配符证书**（如 `*.proxy.example.com`），由 admin 集中签发并下发到节点。
> - **VLESS-Reality** **不使用任何自有证书**——客户端 TLS 握手挪用 dest 真实站点的证书（详见 [design-vless-reality.md](./design-vless-reality.md) §2）。

---

## 7. 安全设计

| 风险 | 缓解措施 |
|---|---|
| Webview RCE 经管理端获取节点控制权 | UI 进程与控制面 RPC 隔离；敏感操作（删节点、改 CA）需二次确认 + 审计 |
| 管理端单点失陷 | CA 私钥使用 OS 密钥环（Windows DPAPI / macOS Keychain / libsecret）加密；导出需密码 |
| 节点服务器被入侵 | Agent 只持有节点自身证书，无法假冒其他节点；配置签名校验防中间人篡改 |
| 客户端订阅令牌泄露 | token 与设备指纹绑定（可选）；管理端可一键吊销重新签发 |
| 流量日志包含 PII | 仅记录字节数与时间窗，不记录目的域名/IP；遵循最小化原则 |
| sing-box CVE | Agent 周期性比对官方版本，重大 CVE 自动升级（管理员可关闭） |
| 供应链 | `go mod verify` + 依赖锁定；CI 中跑 `govulncheck` + `npm audit` |

**最小权限要求**：
- admin：本机文件读写（数据库 + CA）、出站 TLS（连接节点 gRPC）、入站 HTTPS（订阅服务，可选只监听 localhost + 反代）。
- server：监听 gRPC 端口、读写 `/etc/oxygo`、启停 sing-box 子进程、调用 `iptables/nft`（如启用透明代理）。
- client：TUN 设备、路由表、系统代理设置。

---

## 8. 部署与运维

### 8.1 仓库结构（建议演进）

```
oxygo/
├── cmd/
│   ├── admin/          # Wails3 入口（替换当前 main.go）
│   ├── server/         # Agent 入口
│   └── client/         # Wails3 客户端入口
├── internal/
│   ├── proto/          # protobuf 生成代码
│   ├── ca/             # CA 与证书管理
│   ├── singbox/        # sing-box 配置编排 & 子进程/库适配层
│   ├── store/          # SQLite/Bolt 数据访问层
│   ├── transport/      # gRPC server/client 封装
│   └── ui/             # 共享 UI 后端服务
├── frontend/
│   ├── admin/          # 管理端 Svelte 应用
│   └── client/         # 客户端 Svelte 应用
├── proto/              # *.proto 源文件
├── build/              # Wails 资源
└── Taskfile.yml
```

> 当前仓库只有单一 Wails app，需要重构为多入口；建议第一阶段先拆 `cmd/`。

### 8.2 构建与发布

| 构建产物 | 工具链 | 平台 |
|---|---|---|
| oxygo-admin | `wails3 build` | Windows x64、macOS arm64/x64 |
| oxygo-server | `go build` + 静态链接 | Linux x86_64、Linux arm64 |
| oxygo-client | `wails3 build` | Windows x64、macOS arm64/x64、Linux x86_64 |

- CI：GitHub Actions，矩阵构建 + 签名（Windows EV、macOS notarization）。
- 发布：GitHub Releases + 自建分发（客户端走 admin 提供的更新 endpoint）。

---

## 9. 里程碑（建议）

| 阶段 | 范围 | 估算 |
|---|---|---|
| **M0 - 项目重构** | 拆分多入口、引入 proto、CA 骨架 | 1 周 |
| **M1 - 管理端 MVP** | 节点 CRUD、手动录入 sing-box JSON 下发 | 2 周 |
| **M2 - Server Agent** | gRPC 长连、sing-box 子进程托管、状态上报 | 2 周 |
| **M3 - 客户端 MVP** | 订阅拉取、TUN 模式连接、节点切换 | 2 周 |
| **M4 - 协议模板与可视化** | 协议表单化、路由规则编辑器 | 2 周 |
| **M5 - 流量审计与告警** | 流量统计、配额、邮件/Webhook 告警 | 1 周 |
| **M6 - 安全加固** | CA 密钥保护、CRL、自动升级、二次确认 | 1 周 |
| **M7 - 打包发布** | 三端签名、自动更新、文档 | 1 周 |

---

## 10. 待决策项

1. **UI 组件库**：Skeleton vs shadcn-svelte vs 自研。
2. **sing-box 集成方式**：admin/server 端使用子进程还是库；客户端默认库模式但需评估 Wails3 与 sing-box 静态链接的体积。
3. **客户端是否支持移动端**：当前 Wails3 仅桌面；若需要 iOS/Android 需基于 sing-box-for-android / Apple 工程独立实现。
4. **多管理员协作**：当前设计为单管理员单机；多人协作需引入后端服务（admin 拆分为 server + desktop UI）。
5. **国际化**：是否一开始就接入 i18n（建议是）。

---

## 11. 验证方法

- **单元测试**：proto 编解码、CA 签发/校验、配置 diff、流量聚合算法。
- **集成测试**：docker-compose 起一个 admin + 三个 server 容器，跑订阅 → 客户端连接 → 测速 → 节点切换的端到端脚本。
- **安全测试**：使用 `nuclei` 扫管理端 HTTPS endpoint；`golangci-lint` + `govulncheck` 入 CI 门禁；gRPC fuzzing（`go-fuzz`）。
- **性能基准**：单节点 1000 并发连接吞吐；admin 同时管理 100 节点的内存/CPU 占用。
