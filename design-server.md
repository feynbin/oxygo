# Oxygo-Server 详细设计文档

> 版本：v0.1 草案 ｜ 日期：2026-05-27 ｜ 上游文档：[design.md](./design.md)

本文聚焦 **oxygo-server**（节点 Agent）的内部架构，回答两个核心问题：

1. **协议如何选择**：用什么协议提供代理能力，依据是什么。
2. **库模式如何集成**：如何把 sing-box 作为 Go 库嵌入到 Agent 进程内并安全管理其生命周期。

---

## 1. 协议选型

### 1.1 选型坐标系

评估任何代理协议都围绕以下 5 个维度：

| 维度 | 含义 |
|---|---|
| **抗审查 (Censorship Resistance)** | 在 GFW/DPI 环境下能否长期稳定 |
| **性能 (Throughput / Latency)** | 高带宽、低延迟、丢包恢复能力 |
| **部署门槛** | 是否需要域名、CA 证书、CDN、特定端口 |
| **协议成熟度** | sing-box 官方支持等级、客户端生态、CVE 历史 |
| **运维复杂度** | 故障定位、流量统计、用户隔离的难度 |

### 1.2 候选协议横评（基于 sing-box 1.10+ 能力）

| 协议组合 | 抗审查 | 性能 | 部署 | 成熟度 | 推荐位 |
|---|---|---|---|---|---|
| **VLESS + Vision + Reality** | ★★★★★ | ★★★★☆ | ★★★★★ 无需域名 | ★★★★★ | **主力** |
| **Hysteria2 (QUIC/UDP)** | ★★★★☆ | ★★★★★ 拥塞控制 Brutal | ★★★★☆ 用户已有域名 + DNS-01 自动化 | ★★★★☆ | **主力（弱网/高带宽）** |
| **AnyTLS** | ★★★★☆ | ★★★★☆ | ★★★★☆ | ★★★☆☆ 较新 | 备选 |
| **TUIC v5 (QUIC)** | ★★★★☆ | ★★★★★ 0-RTT | ★★★☆☆ 需证书 | ★★★★☆ | 备选 |
| **ShadowTLS v3 + Shadowsocks-2022** | ★★★★★ TLS-in-TLS | ★★★★☆ | ★★★★☆ 需真实 TLS 站点 | ★★★★☆ | 备选（高敏感场景） |
| **Trojan + TLS** | ★★★☆☆ | ★★★☆☆ | ★★★☆☆ 需域名+证书 | ★★★★★ | 备选（兼容性） |
| **VMess (TCP/WS/gRPC)** | ★★☆☆☆ | ★★★☆☆ | ★★★★☆ | ★★★★★ | **不推荐**（特征明显） |
| **原版 Shadowsocks (无插件)** | ★☆☆☆☆ | ★★★★☆ | ★★★★★ | ★★★★★ | **不推荐** |
| **WireGuard / OpenVPN** | ★☆☆☆☆ | ★★★★★ | ★★★★☆ | ★★★★★ | 不在本项目范围 |

### 1.3 推荐方案

**默认启用双协议并行（用户可在客户端切换）**：

```
┌──────────────────────────────────────────────────────────────┐
│ Inbound A : VLESS + Vision + Reality                         │
│   - 端口 443/TCP                                              │
│   - dest 伪装目标：www.microsoft.com（默认）/ apple.com 等    │
│     ⚠️ Cloudflare 因 TLS 配置存在特殊指纹，不推荐为 dest      │
│   - 用途：日常浏览，最高抗审查                                │
│   - 详见 design-vless-reality.md                              │
├──────────────────────────────────────────────────────────────┤
│ Inbound B : Hysteria2                                        │
│   - 端口 443/UDP + 端口跳跃 40000-50000/UDP                  │
│   - 通配符证书（admin ACME DNS-01 签发后下发）                │
│   - 用途：视频、下载、弱网场景                                │
│   - 详见 design-hysteria2.md                                  │
└──────────────────────────────────────────────────────────────┘
```

**选型理由**：

1. **VLESS-Reality 作为主力**
   - 无需购买域名和申请证书，把节点 IP 直接当入口
   - Reality 借用真实站点的 TLS 握手特征，DPI 看到的是合法 TLS 流量
   - sing-box 与 Xray-core 都原生支持，客户端兼容性最佳
   - Vision 流控（XTLS Vision）规避 TLS-in-TLS 的字节熵特征

2. **Hysteria2 作为高性能补充**
   - QUIC over UDP，丢包恢复优于 TCP 多倍
   - 自带 Brutal 拥塞控制，可在带宽限速场景跑满
   - 缺点：UDP 在部分网络（公司 NAT、酒店 Wi-Fi）会被限制，所以与 Reality 互补

3. **不内置 VMess 与裸 SS**
   - 已出现成熟的主动探测攻击，作为新项目没有兼容包袱

### 1.4 协议参数标准化

| 参数 | VLESS-Reality | Hysteria2 |
|---|---|---|
| 端口 | TCP 443 | UDP 443 |
| 用户标识 | UUID | password (per-user) |
| 加密握手 | Reality (X25519) | TLS 1.3 |
| 流控 | xtls-rprx-vision | 无（QUIC 内部） |
| 伪装目标 | dest = `www.microsoft.com:443` | masquerade = `https://download.microsoft.com` |
| 客户端必需字段 | publicKey, shortId, serverName | password, sni, insecure=false |

**特别约定**：管理端为每个用户生成的 UUID/Password 在两种协议下复用，但 Reality 使用 UUID 认证，Hysteria2 使用 Password；服务端配置时按用户分别落入 `users[]` 数组。

---

## 2. oxygo-server 程序架构

### 2.1 进程模型

oxygo-server 是单一 Go 进程，内部包含三个核心运行时层：

```
┌─────────────────────────────────────────────────────────────────┐
│                       oxygo-server (Go 进程)                     │
│                                                                  │
│  ┌────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  Control Plane │   │  Box Supervisor  │   │ Stats/Logs    │  │
│  │  (gRPC client) │──▶│  (sing-box 实例) │──▶│ Exporter      │  │
│  │  反连 admin    │   │  库模式托管       │   │ 流量/连接/日志│  │
│  └────────────────┘   └──────────────────┘   └──────────────┘  │
│           │                    │                      │         │
│           └──────────┬─────────┴──────────────────────┘         │
│                      ▼                                          │
│           ┌──────────────────────┐                              │
│           │  Local State Store   │                              │
│           │  (BoltDB, /var/lib)  │                              │
│           └──────────────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
                      │
                      │ libc / syscall
                      ▼
        TCP:443 (VLESS-Reality)   UDP:443 (Hysteria2)
```

### 2.2 模块划分

| 模块 | 路径（建议） | 职责 |
|---|---|---|
| `agent` | `internal/server/agent` | 进程入口、信号处理、模块编排 |
| `control` | `internal/server/control` | gRPC client、反连 admin、配置订阅、命令处理 |
| `supervisor` | `internal/server/supervisor` | sing-box 实例生命周期管理（核心） |
| `singboxgen` | `internal/server/singboxgen` | 把管理端下发的 `NodeSpec` 渲染为 `option.Options` |
| `metrics` | `internal/server/metrics` | 从 sing-box 抓取流量与连接，按用户聚合 |
| `logsink` | `internal/server/logsink` | 接管 sing-box 的 log factory，环形缓冲 + 流式上报 |
| `store` | `internal/server/store` | BoltDB 封装：缓存最近 revision、本地证书、流量增量 |
| `health` | `internal/server/health` | 自检：端口可达性、CPU/内存/磁盘 |

### 2.3 核心数据结构

```go
// 由管理端下发，是 sing-box 配置的"上层抽象"
type NodeSpec struct {
    Revision   uint64               // 单调递增
    Inbounds   []InboundSpec        // 当前节点要开的入站
    Users      []UserSpec           // 所有用户（按订阅过滤后）
    Routing    RoutingSpec          // 路由规则（可选）
    Cert       *CertBundle          // 域名证书（Hysteria2/Trojan 需要）
    Signature  []byte               // admin CA 对 Revision+SHA 的签名
}

type InboundSpec struct {
    Tag      string                 // "vless-reality" / "hy2"
    Kind     InboundKind            // 枚举：VLESS_REALITY / HYSTERIA2 / ...
    Listen   string                 // "::"
    Port     uint16                 // 443
    // 协议特定参数（按 Kind 填充）：
    //   Hy2:     PortRange, ObfsKey, UpMbps, DownMbps, MasqueradeURL
    //   Reality: DestHost, DestPort, PrivateKey, ShortIDs []string
    Params   map[string]any
}

type UserSpec struct {
    ID       string    // 系统内 ID
    UUID     string    // VLESS 用
    Password string    // Hysteria2/Trojan 用
    Quota    int64     // 字节，0 表无限
}
```

> **关键设计**：Agent 不直接接受原生 sing-box JSON，而是接受 `NodeSpec` 抽象。原因有三：
> 1. **协议替换零成本**：将来 Reality 被破解切换到 AnyTLS，只需改 `singboxgen` 模块，不动协议
> 2. **多版本兼容**：sing-box 升级 option schema 不会冲击 admin↔server 协议
> 3. **签名稳定**：admin 签名 NodeSpec 而非渲染后的 JSON，避免 JSON 序列化差异破坏签名

---

## 3. sing-box 库模式集成（核心实现）

### 3.1 依赖引入

```bash
go get github.com/sagernet/sing-box@latest
# 视构建需要启用相关 tags：
#   with_quic         - Hysteria2/TUIC（必需）
#   with_grpc         - 如需 gRPC 传输
#   with_clash_api    - 启用 Clash API 抓统计（推荐）
#   with_wireguard    - 暂不需要
# 服务器端不需要 with_gvisor / with_tun
```

构建命令示例：

```bash
go build -tags "with_quic,with_clash_api" -o oxygo-server ./cmd/server
```

### 3.2 实例生命周期

sing-box 库本身**不支持热替换 inbound**，因此采用**双实例切换**策略：

```
当前激活实例（active）        待激活实例（pending）
       │                              │
       │  收到新 NodeSpec             │
       │ ───────────────────────────▶│ 构造 options
       │                              │ box.New(ctx)
       │                              │ box.Start()
       │                              │ 健康检查（监听端口可读）
       │                              │
       │ ◀────── 流量切换 ───────────│  swap pointer (atomic)
       │ box.Close()                  │
       │ (优雅等待最长 30s)           │
       ▼                              ▼
     销毁                           成为新 active
```

**注意事项**：
- 因为 VLESS 和 Hysteria2 监听端口在新旧实例间冲突，需要在 `box.Close()` 旧实例**之后**才能 `box.Start()` 新实例
- 折中方案：先 Close 再 Start，期间有 1~3s 的连接断开窗口，可接受
- 替代方案（高级）：用户接入采用 SO_REUSEPORT 多进程，但库模式下复杂度过高，**v1 不采用**

> **决策**：v1 采用**先 Close 再 Start**（停机式 reload），代价是每次配置变更约 2s 中断；后续版本通过端口复用 + 多 instance 改造为零停机。这与 design.md 第 4.3 节"零停机重载"目标的差距需在路线图说明。

### 3.3 关键代码骨架

> 以下仅示意接口形状，具体字段以 sing-box 实际 API 为准（每次升级需校对）。

```go
package supervisor

import (
    "context"
    "sync/atomic"
    "time"

    box "github.com/sagernet/sing-box"
    "github.com/sagernet/sing-box/option"
)

type Supervisor struct {
    active     atomic.Pointer[box.Box]
    activeRev  atomic.Uint64
    gen        Generator          // singboxgen 接口
    logFactory log.Factory        // 注入到 sing-box
    ctx        context.Context
}

// Apply 把新 NodeSpec 落地。线程安全，串行调用。
func (s *Supervisor) Apply(spec *NodeSpec) error {
    if spec.Revision <= s.activeRev.Load() {
        return ErrStaleRevision        // 防回放
    }

    opts, err := s.gen.Render(spec)    // NodeSpec -> option.Options
    if err != nil {
        return fmt.Errorf("render: %w", err)
    }
    if err := opts.Validate(); err != nil {  // sing-box 内置校验
        return fmt.Errorf("invalid: %w", err)
    }

    // 1. 优雅停旧
    if old := s.active.Load(); old != nil {
        ctx, cancel := context.WithTimeout(s.ctx, 30*time.Second)
        defer cancel()
        _ = old.Close()                // sing-box.Close 是同步阻塞的
    }

    // 2. 启动新
    inst, err := box.New(box.Options{
        Context: s.ctx,
        Options: opts,
        // 注入自定义 log factory，用于日志回传 admin
        // 注入自定义 platform interface（服务器端为空实现）
    })
    if err != nil {
        return fmt.Errorf("new: %w", err)
    }
    if err := inst.Start(); err != nil {
        _ = inst.Close()
        return fmt.Errorf("start: %w", err)
    }

    // 3. 自检：监听端口已就绪
    if err := s.probe(spec, 3*time.Second); err != nil {
        _ = inst.Close()
        return fmt.Errorf("probe: %w", err)
    }

    s.active.Store(inst)
    s.activeRev.Store(spec.Revision)
    return nil
}
```

**配套要点**：

- **失败回滚**：若新实例启动失败，从 BoltDB 读上一份成功的 NodeSpec 重新 Apply。
- **首启**：进程启动时先从 BoltDB 加载最近一次 revision 启动（admin 不可达时仍提供服务），随后再连 admin 同步最新。
- **panic 隔离**：`box.New` 内部的 panic 用 recover 兜底；连续 3 次失败暂停 Apply，进入"维护模式"等管理员介入。

### 3.4 sing-box 配置渲染（`singboxgen`）

VLESS-Reality 渲染示例（伪代码，仅示意核心字段）：

```go
func renderVlessReality(in InboundSpec, users []UserSpec) option.Inbound {
    var u []option.VLESSUser
    for _, x := range users {
        u = append(u, option.VLESSUser{
            Name: x.ID, UUID: x.UUID, Flow: "xtls-rprx-vision",
        })
    }
    return option.Inbound{
        Type: C.TypeVLESS,
        Tag:  in.Tag,
        VLESSOptions: option.VLESSInboundOptions{
            ListenOptions: option.ListenOptions{
                Listen: option.NewListenAddress(in.Listen),
                ListenPort: in.Port,
            },
            Users: u,
            InboundTLSOptionsContainer: option.InboundTLSOptionsContainer{
                TLS: &option.InboundTLSOptions{
                    Enabled: true,
                    ServerName: in.Param("dest_sni"),
                    Reality: &option.InboundRealityOptions{
                        Enabled: true,
                        Handshake: option.InboundRealityHandshakeOptions{
                            ServerOptions: option.ServerOptions{
                                Server: in.Param("dest_host"),
                                ServerPort: 443,
                            },
                        },
                        PrivateKey: in.Param("priv_key"),
                        ShortID:    []string{in.Param("short_id")},
                    },
                },
            },
        },
    }
}
```

> Hysteria2 渲染类似，差异在于 `option.Hysteria2InboundOptions` 的 `Up/Down Mbps`、`Masquerade`、`Obfs`、`Users[].Password`。

### 3.5 统计与日志接入

**统计**：启用 sing-box 的 `experimental.clash_api` 暴露 `127.0.0.1:9090`（仅本机），Agent 通过 HTTP 拉取：
- `GET /connections` → 当前连接列表（按 inbound 标签 + 用户名聚合）
- `GET /traffic` → 实时上下行速率
- `GET /memory` → 内存占用

按 30s 周期采样，差量写入 BoltDB，每 1min 通过 gRPC stream 上报 admin。

**日志**：实现 sing-box 的 `log.Factory` 接口，把日志同时输出到：
1. 本地 ring buffer（最多 10MB，崩溃前快照）
2. gRPC stream（实时回传 admin，按级别过滤，默认 WARN+）

### 3.6 证书与机密

| 物料 | 来源 | 存储 | 说明 |
|---|---|---|---|
| Reality privateKey/publicKey | admin 生成 X25519 对，私钥下发节点 | BoltDB（机器绑定密钥加密） | **节点本身不持有任何 TLS 证书**（Reality 借用 dest 站点证书） |
| Reality shortID | admin 下发 | 同上 | 严禁空字符串，长度 8 hex |
| Hysteria2 TLS 证书 | admin 通过 ACME **DNS-01** 签发**通配符**证书并下发 PEM | 同上 | 所有节点共用同一张 `*.proxy.example.com` |
| 用户 UUID/Password | admin 生成，作为 NodeSpec.Users 下发 | 内存为主，重启从 BoltDB 恢复 | UUID 给 VLESS，Password 给 Hy2 |
| Agent 自身 mTLS 证书 | 首次 SSH 部署注入，后续 admin 轮换 | `/etc/oxygo/agent.crt+key`，0600 | 仅用于控制面 |

**最小权限**：
- Agent 进程以 `oxygo` 系统用户运行，**不需要 root**
- 监听 443 端口通过 `setcap cap_net_bind_service=+ep` 授予二进制
- `/etc/oxygo/` 目录所有者 `oxygo:oxygo`，权限 0700

---

## 4. 错误与可观测性

### 4.1 错误分级

| 级别 | 例子 | 处理 |
|---|---|---|
| **致命** | sing-box 启动失败且回滚也失败、BoltDB 损坏 | 退出进程，systemd 自动重启 |
| **可恢复** | 单次 Apply 失败、admin 断连 | 重试（指数退避）+ 上报 |
| **业务** | 用户认证失败、流量超限 | sing-box 自身处理，Agent 仅统计 |

### 4.2 指标（暴露给 admin，也可挂 Prometheus）

- `oxygo_singbox_up` (gauge, 0/1)
- `oxygo_singbox_revision` (gauge)
- `oxygo_traffic_bytes_total{user, inbound, direction}` (counter)
- `oxygo_connections_active{inbound}` (gauge)
- `oxygo_apply_duration_seconds` (histogram)
- `oxygo_apply_errors_total{reason}` (counter)

### 4.3 健康检查端点

Agent 本地暴露 `127.0.0.1:7777`（默认）：
- `GET /healthz` → 200 表示 supervisor 有 active 实例
- `GET /readyz` → 200 表示已连接 admin 且最新 revision 已应用

---

## 5. 部署形态

**目标平台**：Linux x86_64 + arm64，glibc 2.31+。不支持 Alpine（musl）首版。

**systemd unit**（`/etc/systemd/system/oxygo-server.service`）：

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

**agent.yaml** 最小示例：

```yaml
admin:
  endpoint: "admin.example.com:7443"   # admin gRPC 地址
  ca_file: "/etc/oxygo/admin-ca.pem"
  cert_file: "/etc/oxygo/agent.crt"
  key_file: "/etc/oxygo/agent.key"

local:
  data_dir: "/var/lib/oxygo"
  log_dir: "/var/log/oxygo"
  health_listen: "127.0.0.1:7777"

singbox:
  clash_api_listen: "127.0.0.1:9090"
  log_level: "warn"
```

---

## 6. 与上游设计的差异说明

| design.md 中的描述 | 本文调整 | 原因 |
|---|---|---|
| "sing-box 子进程模式（推荐）" | 改为**库模式** | 用户决策；库模式减少 IPC、统一日志、便于测试 |
| "sing-box 1.10+ 支持 inbound listener 热替换" | 实际不支持；v1 采用**停机式 reload** | 库模式下无法跨实例共享监听套接字 |
| "Agent 通过 sing-box ClashAPI 或 V2Ray 兼容 API 抓取统计" | 收敛到仅用 **ClashAPI** | V2Ray stats API 已边缘化，统一一种降低维护 |

---

## 7. 待决策项（针对服务端）

1. **是否第一版就支持 Hysteria2**：若只做 Reality 可显著减少 ACME/证书复杂度，先单协议 MVP。
2. **是否引入 Prometheus 端点**：admin 已收集指标，再加 Prometheus 是否冗余。
3. **BoltDB vs Pebble vs 纯文件**：当前数据规模小（<10MB），BoltDB 足够；除非未来流量审计本地化才考虑 Pebble。
4. **配置加密**：BoltDB 内的密钥是否需要 OS 密钥环加密；当前文件权限 0600 是否足够。
5. **跨架构构建**：是否需要 musl/Alpine 支持；目前 sing-box QUIC 相关在 musl 上有兼容性历史问题。

---

## 8. 下一步行动（建议）

1. **本周**：确认协议组合（Reality+Hy2 vs 仅 Reality）以及待决策项 1～3。
2. **下周（M0 后半）**：搭建 `cmd/server` 与 `internal/server/supervisor` 骨架，能加载一份硬编码的 NodeSpec 启动 sing-box。
3. **M1**：完成 `singboxgen` 渲染器 + Apply 闭环 + 本地 BoltDB 缓存，离线场景可独立工作。
4. **M2**：接入 gRPC 控制面，与 admin 联调。
