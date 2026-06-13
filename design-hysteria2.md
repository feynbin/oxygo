# Hysteria2 优先实施设计

> 版本：v0.1 ｜ 日期：2026-05-27 ｜ 上游：[design-server.md](./design-server.md)
> 范围：服务端 Hysteria2 入站的完整设计，含 GFW 威胁对策、域名/证书、端口跳跃、混淆与限速、客户端配套要求。
> 目标：作为 oxygo-server 第一个落地协议，M2 阶段先把 Hysteria2 链路打通。

---

## 1. 为什么 Hysteria2 优先

| 维度 | 评估 |
|---|---|
| 协议成熟度 | sing-box 1.10+ 一类支持，官方 Hysteria 团队持续维护 |
| 性能 | 基于 QUIC，自带 Brutal 拥塞控制，可打满共享出口带宽 |
| 部署门槛 | 单端口/UDP + 一张 TLS 证书；用户已具备域名，门槛已扫清 |
| 抗 DPI | 内置 Salamander 混淆 + masquerade，QUIC 流量伪装为正常 HTTPS |
| 客户端生态 | sing-box / Hiddify / NekoBox / 官方 Hysteria 客户端互通 |
| 与 Reality 互补 | UDP 协议补 TCP，Brutal 补 BBR，QUIC 补 TCP 队头阻塞 |

**风险已知**：Hy2 最大软肋是 GFW 对 UDP 的 QoS 限速；解决方案是**端口跳跃 + 混淆 + 必要时降级到 Reality（TCP）**。

---

## 2. GFW 威胁建模

GFW 不是单一系统，而是多种检测手段的组合。下面把已知的、对 Hysteria2 直接相关的威胁列清楚，并给出对应缓解。

### 2.1 威胁矩阵

| # | 威胁 | 触发场景 | 对 Hy2 的影响 | 本项目对策 |
|---|---|---|---|---|
| T1 | **UDP QoS / 限速** | 敏感时期或大流量持续 | UDP 丢包率飙升、长连接中断 | **端口跳跃** + **Brutal 重传** + 客户端自动重连 |
| T2 | **主动探测 (Active Probing)** | GFW 向可疑 IP:port 发送 HTTPS/QUIC 探测 | 若服务端响应异常即被标记 | **Masquerade** 把非法握手代理到真实 HTTPS 站点 |
| T3 | **SNI 黑名单 / SNI RST** | TLS ClientHello 的 SNI 命中黑名单 | TLS 握手被切断 | **使用自有域名**，避免共享/公共 SNI；DNS 解析私有化 |
| T4 | **证书透明度 (CT Log) 扫描** | Let's Encrypt 等签发证书入 CT 日志，被批量扫描 | 节点 IP 被关联到代理域名后封锁 IP | **通配符证书**（DNS-01）+ **不公开节点 IP**（节点域名独立） |
| T5 | **TLS 指纹 (JA3/JA4)** | 非主流 TLS 客户端指纹被识别 | 客户端被标记 | 客户端 sing-box 内置 **uTLS** 模拟 Chrome；服务端用标准 Go TLS 即可 |
| T6 | **QUIC 协议指纹** | Initial 包的 Token、Version、TLS 扩展顺序异常 | QUIC 流量整体可识别 | **启用 Salamander obfs**，把 QUIC 包伪装成随机字节流 |
| T7 | **流量分析（包长/时序）** | 长连接稳态流量被聚类分析 | 中长期可能被识别 | Hy2 自身的随机填充 + 不刻意降低或拉满带宽 |
| T8 | **IP 整段封锁** | 单 IP 被封后波及同段 | 节点失效 | 多节点冗余 + admin 端节点健康监控 + 一键下线 |
| T9 | **端口扫描封锁** | 暴露的 UDP 端口被识别 | 端口被封 | 端口跳跃随机化 + 不暴露 22/443 等常见端口 |
| T10 | **DNS 污染** | 节点域名解析被污染 | 客户端连不上 | 客户端使用 **DoH/DoT**（如 1.1.1.1）解析节点域名 |

### 2.2 威胁优先级

按"发生概率 × 影响"排序，本项目必须解决的 **P0** 威胁：

- **T1（UDP QoS）**、**T2（主动探测）**、**T6（QUIC 指纹）**、**T10（DNS 污染）**

**P1**（应解决但可二期）：T3、T4、T8。
**P2**（运维层面解决）：T5、T7、T9。

---

## 3. 域名与证书策略

### 3.1 域名规划（用户已有域名前提）

**推荐拓扑**：

```
example.com                          ← 主域名（用户用途）
├── (留空，不指向 oxygo 任何资源)
│
├── *.proxy.example.com              ← 节点域名通配符（DNS-01 通配证书）
│   ├── node1.proxy.example.com  →  A   1.2.3.4
│   ├── node2.proxy.example.com  →  AAAA 2001:db8::1
│   └── ...
│
├── api.example.com                  ← admin 订阅 / API（HTTPS）
└── mask.example.com                 ← Masquerade 默认目标（静态站，可选）
```

**关键约束**：
- 节点域名 **不要**暴露在公开页面（不要在网站、GitHub README 留链接），减少被动扫描发现概率
- 节点域名解析 TTL 设短（120~300s），便于切换 IP
- 不要把节点 IP 直接 A 到根域名 `example.com`，根域名通常已被关联到用户身份

### 3.2 证书策略

**强烈推荐：通配符证书 + DNS-01 验证**

| 选项 | 推荐 | 说明 |
|---|---|---|
| 通配符证书 `*.proxy.example.com` | ✅ | 一证覆盖所有节点，CT 日志只暴露通配符，不暴露具体节点子域 |
| 单证书每节点 `nodeN.proxy.example.com` | ❌ | 每签发一次都进 CT 日志，相当于公开节点列表 |
| 自签证书 | ❌ | 客户端必须设 `insecure=true`，违背 T2/T5 缓解原则 |

**ACME CA 选择**：

| CA | CT 强制 | DNS-01 | 通配符 | 评估 |
|---|---|---|---|---|
| Let's Encrypt | 是 | ✅ | ✅ | 主选，生态最全 |
| ZeroSSL | 是 | ✅ | ✅ | 备选，免费额度足够 |
| Buypass Go | 是 | ✅ | ✅ | 备选，180 天有效期 |
| Google Trust Services | 是 | ✅ | ✅ | 备选 |

> CT 都是强制的，所以核心是**用通配符把节点身份隐藏到通配符之下**，而不是去找不上 CT 的 CA。

**DNS-01 验证流程**（admin 集中签发）：

```
[admin]                                         [DNS Provider API]
   │  用户提供 DNS API Token（Cloudflare/阿里云/Route53）
   │  admin 用 lego 库或 certmagic 申请
   │ ──────── DNS-01 challenge ──────────────────▶
   │                                                  TXT _acme-challenge.proxy
   │ ◀───────── 证书 + 私钥 ─────────────────────────
   │
   │  存入 admin 数据库（CA 表）
   │  ↓
   │  下发到各节点（gRPC 推送，包含在 NodeSpec.Cert）
   ▼
[oxygo-server N 个节点] —— 全部使用同一张通配符证书
```

**续期策略**：
- admin 在证书到期前 30 天自动续期
- 续期成功后通过 gRPC 推送新证书；节点 supervisor 触发 sing-box 实例 reload
- 失败告警走 design.md 第 3.1 节定义的告警通道

---

## 4. Hysteria2 服务端配置设计

### 4.1 推荐参数集（v1 默认）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `listen` | `[::]:443` | 双栈，标准 HTTPS 端口减小特征 |
| `listen_port_range` | `40000-50000` | **端口跳跃范围**（见 §5） |
| `tls.server_name` | `nodeN.proxy.example.com` | 客户端 SNI 必须一致 |
| `tls.alpn` | `["h3"]` | HTTP/3 ALPN，与 masquerade 一致 |
| `tls.min_version` | `1.3` | 强制 TLS 1.3 |
| `obfs.type` | `salamander` | **必启**，对抗 T6 |
| `obfs.password` | 32 字节随机 | 每节点独立，与用户密码无关 |
| `up_mbps` | 节点出口带宽 × 0.9 | 给 Brutal 用，留 10% 余量 |
| `down_mbps` | 节点出口带宽 × 0.9 | 同上 |
| `ignore_client_bandwidth` | `false` | 信任客户端声明的速率 |
| `masquerade.type` | `proxy` | 反代到真实 HTTPS 站点（见 §6） |
| `masquerade.proxy.url` | `https://www.bing.com/` 等 | 必须是合法可访问的 HTTPS |
| `masquerade.proxy.rewrite_host` | `true` | 重写 Host 头匹配上游 |

### 4.2 渲染后的 sing-box inbound 示例

```json
{
  "type": "hysteria2",
  "tag": "hy2-in",
  "listen": "::",
  "listen_port": 443,
  "up_mbps": 900,
  "down_mbps": 900,
  "obfs": {
    "type": "salamander",
    "password": "<32-byte-random-base64>"
  },
  "users": [
    { "name": "u-001", "password": "<per-user-password>" },
    { "name": "u-002", "password": "<per-user-password>" }
  ],
  "tls": {
    "enabled": true,
    "server_name": "node1.proxy.example.com",
    "alpn": ["h3"],
    "min_version": "1.3",
    "certificate_path": "/etc/oxygo/certs/wildcard.crt",
    "key_path": "/etc/oxygo/certs/wildcard.key"
  },
  "masquerade": "https://www.bing.com/"
}
```

> 注：sing-box 的 `masquerade` 字段支持简写为 URL（等价于 `{type:"proxy",proxy:{url:...}}`）。文件路径或字符串两种形态都可。

### 4.3 `NodeSpec` → Hy2 inbound 渲染映射

承接 design-server.md §2.3 定义的 `NodeSpec` 抽象，Hy2 在 `InboundSpec.Params` 中需要的字段：

```go
type Hy2Params struct {
    PortRange   string  // "40000-50000"，可选；空则单端口
    PrimaryPort uint16  // 443
    ObfsKey     string  // 服务端独立随机
    UpMbps      int     // 上行限速
    DownMbps    int     // 下行限速
    MasqueradeURL string // 反代目标
    // SNI / cert 由 NodeSpec.Cert 注入
}
```

`Users` 中 `Password` 字段直接复用 design-server.md §1.4 中"管理端为每个用户生成的 password"。

---

## 5. 端口跳跃（应对 T1 UDP QoS）

### 5.1 原理

Hy2 客户端在连接建立后**周期性切换源/目的端口**，使 GFW 难以基于五元组施加 QoS。技术实现上：服务端在内核层面把整个端口范围的 UDP 流量都转发到 sing-box 实际监听的单一端口，sing-box 不感知端口跳跃。

### 5.2 服务端 nftables 规则（推荐）

```bash
# 创建表与链（仅首次）
nft add table inet oxygo
nft add chain inet oxygo prerouting { type nat hook prerouting priority -100 \; }

# 把 40000-50000 的 UDP 流量重定向到 443
nft add rule inet oxygo prerouting \
    iif "eth0" udp dport 40000-50000 redirect to :443
```

**封装为 Agent 能力**：
- Agent 启动时若 `Hy2Params.PortRange` 非空，自动下发上述 nft 规则
- 退出时清理（trap SIGTERM）
- 需要 `CAP_NET_ADMIN`，systemd unit 中显式声明

> 替代方案：iptables `REDIRECT` 也可（兼容老内核），但 nftables 是现代 Linux 默认且更高效。Agent 优先尝试 nftables，回退到 iptables。

### 5.3 客户端配置约定

```yaml
server: "node1.proxy.example.com:443,40000-50000"   # sing-box / Hy2 标准语法
```

逗号后是跳跃范围；sing-box 客户端会自动启用 port hopping。

### 5.4 风险与边界

- 端口范围过大（如 1024-65535）会触发 GFW 端口扫描告警；推荐限制到 1 万个端口以内
- 与防火墙/云厂商安全组冲突：必须放开该 UDP 端口段（部分云厂商 UI 限制端口段大小，需用 API 批量开）
- IPv6 同样适用，规则需在 `ip6` 族再写一份

---

## 6. Masquerade 设计（应对 T2 主动探测）

### 6.1 工作机制

当任意客户端发起 QUIC 连接但**没有提供正确的 `password` 或 `obfs`**，sing-box 不再直接拒绝，而是把这个连接当作普通 HTTP/3 请求转发到 `masquerade.url`，返回上游真实站点的响应。GFW 主动探测看到的是一个**正常的 HTTPS 网站**。

### 6.2 Masquerade URL 选择原则

| 选择 | 评估 |
|---|---|
| 用户自己控制的静态站（如 `mask.example.com` 上托管的简单页面） | ✅ 最可控；可放真实内容增加可信度 |
| 知名公开 HTTPS 站点（如 `https://www.bing.com/`） | ⚠️ 简单但可能违反对方 ToS；Bing/微软等大流量站不敏感，但小站会拒绝 |
| Hy2 内置 `file` 模式（直接返回静态文件） | ⚠️ 行为太"干净"，主动探测拿到的是单文件，易被识别 |

**v1 默认**：
- admin 端提供模板，让用户自部署一个 mask 站点（推荐放在 `mask.example.com`，Nginx + 几个静态页）
- 也允许用户配置外部 URL（如 Bing），但 UI 上明确风险提示

### 6.3 mask 站点最小实现

```nginx
server {
    listen 443 ssl http2;
    server_name mask.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/mask;
    index index.html;

    # 给探测者一份正常的网页
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

页面内容随便放一份合法的静态页（公司主页、个人博客、文档站皆可）。

---

## 7. Brutal 拥塞控制与限速

### 7.1 Brutal 工作机制

Hy2 协议层支持 **Brutal CC**：客户端在握手时声明自己期望的上下行速率，服务端按声明速率发送（即使会丢包），通过强重传保证可靠性。优点是能在共享出口、被 BBR 抑制的链路上"抢"到带宽。

### 7.2 参数选择

| 节点出口带宽 | 推荐 `up_mbps` / `down_mbps` | 备注 |
|---|---|---|
| 1 Gbps 独享 | 900 / 900 | 留 10% 给系统流量 |
| 1 Gbps 共享 | 200 / 200 | 看 VPS 限速策略，过高会触发母机限速 |
| 100 Mbps | 90 / 90 | |
| 50 Mbps | 40 / 40 | |

### 7.3 多用户场景的限速

**重要**：sing-box 的 Hy2 限速是 inbound 级别，不是用户级别。多用户场景的隔离要在路由层做：

- v1：不做用户级限速，依赖 `Quota` 字节配额 + admin 监控告警
- v2：通过 sing-box 的 `route.rules` + `tag` 把不同用户路由到不同 outbound，再对 outbound 限速

> 这是一个**已知限制**，需在 admin UI 上明确告知。

---

## 8. 客户端配套要求

虽然本文聚焦服务端，但有些 GFW 缓解只在服务端做不够，必须客户端配合：

| 缓解项 | 客户端要求 |
|---|---|
| T1 端口跳跃 | sing-box 客户端 server 字段使用 `host:port,range` 语法 |
| T5 TLS 指纹 | sing-box 客户端启用 `tls.utls.enabled=true`, `utls.fingerprint="chrome"` |
| T6 QUIC 混淆 | 客户端 `obfs.type=salamander` + 同密钥 |
| T10 DNS 污染 | 客户端 DNS 入站强制 DoH（`https://1.1.1.1/dns-query`），节点域名走加密 DNS |

这些字段由 admin 在生成订阅时**自动写入**，用户不需要手动配置。

---

## 9. 实施路线图（Hy2 优先）

按 design.md §9 的 M0~M7 排期，把 Hy2 拆到具体周次：

| 里程碑 | 任务 | 验收标准 |
|---|---|---|
| **M0.5** | 项目结构拆分完毕，`cmd/server` 可启动空进程 | `go run ./cmd/server` 输出 banner，连不到 admin 即退出 |
| **M1.1** | `singboxgen` 完成 Hy2 字段渲染，本地用硬编码 NodeSpec 启动 sing-box | 用 sing-box 官方客户端能连上、能访问外网 |
| **M1.2** | 通配符证书 ACME 流程打通（admin 端，先单测覆盖） | DNS-01 challenge 成功签发并落库 |
| **M1.3** | 端口跳跃 nftables 集成 | `nft list ruleset` 看到 redirect 规则；客户端端口跳跃生效 |
| **M1.4** | Masquerade 联调 | 用 `curl https://node` 看到 mask 站点响应；用 Hy2 客户端能正常代理 |
| **M2.1** | gRPC 控制面：Hy2 NodeSpec 从 admin 下发 → 节点 Apply | admin UI 改密码，节点 2 秒内生效 |
| **M2.2** | ClashAPI 流量采集 + 上报 | admin 看到实时流量曲线 |
| **M2.3** | 失败回滚 + BoltDB 缓存 | 模拟 admin 断网，节点重启仍能恢复服务 |

**M1.x 全部完成时，Hy2 已是一个可独立使用的代理服务**；M2.x 把它接入 oxygo 整套控制面。

---

## 10. 测试矩阵

| 测试类型 | 用例 |
|---|---|
| 单元 | `singboxgen.RenderHy2` 输入边界（空 users、超长 SNI、非法端口范围） |
| 单元 | nftables 规则生成函数：端口段、IPv6、回滚清理 |
| 集成 | docker-compose 起 server 容器（含 nft）+ sing-box 客户端容器，验证连通性 |
| 集成 | ACME DNS-01 用 Pebble + challtestsrv 模拟，验证签发与续期 |
| 弱网 | `tc qdisc` 加 30% 丢包 + 100ms 抖动，验证 Brutal 下吞吐衰减 |
| 主动探测 | `curl -v https://node:443` / `openssl s_client` 应看到 mask 站点正常响应；客户端密码错误时同样看到 mask |
| 端口跳跃 | tcpdump 抓 UDP，确认源端口在 40000-50000 范围内变化 |
| 安全 | 用错误 obfs key 连接，应**完全无响应**而非显式拒绝 |

---

## 11. 与上游设计的修订

| 文档 | 章节 | 修订 |
|---|---|---|
| design-server.md | §1.2 协议横评 | Hy2 部署门槛由 "★★★☆☆ 需域名+证书" 上调至 "★★★★☆"（用户已有域名 + DNS-01 自动化） |
| design-server.md | §1.3 推荐方案 | 默认配置加入端口跳跃 `40000-50000` |
| design-server.md | §3.6 证书与机密 | "Hysteria2 TLS 证书"明确为通配符证书，签发方式 DNS-01 |
| design.md | §3.1 admin 功能 | "证书中心"增加 DNS Provider 凭据托管 |
| design.md | §6.2 配置同步 | NodeSpec 增加 `Cert.WildcardDomain` 字段 |

---

## 12. 待决策项

1. **mask 站点是否由 admin 自动部署**：用户已有域名，admin 是否要在节点上自动起一个 nginx + 静态页（增加复杂度）？v1 倾向**只提供模板和文档，由用户手工部署**。
2. **DNS Provider 抽象层**：v1 只支持 Cloudflare 还是先支持多家（阿里、腾讯、Route53）？建议先 Cloudflare 一家，后续基于 lego 库扩展。
3. **端口跳跃是否默认开启**：开启后云厂商安全组配置变重；v1 倾向**默认开启**但 admin UI 提供一键关闭。
4. **客户端订阅是否暴露通配符证书 SNI**：必须暴露给客户端做 SNI 校验，无法隐藏；这是 CT 模型下的合理代价。
5. **是否支持 IPv6-only 节点**：技术上可行，但 GFW 对 IPv6 检测较弱反而是优势；v1 测试矩阵需覆盖。

---

## 13. 给开发者的下一步

1. **本周**：与你确认 §12 待决策项 1～3。
2. **下周（M0.5）**：建好 `cmd/server` 与 `internal/server/{supervisor,singboxgen}`，提交一个能加载 YAML 配置启动 Hy2 的 demo。
3. **后续**：按 §9 路线图推进。
