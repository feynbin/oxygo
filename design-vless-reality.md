# VLESS + Vision + Reality 实施设计

> 版本：v0.1 ｜ 日期：2026-05-27 ｜ 上游：[design-server.md](./design-server.md)、[design-hysteria2.md](./design-hysteria2.md)
> 范围：服务端 VLESS-Reality 入站的完整设计，含 GFW 威胁对策、dest 站点选择、X25519 密钥与 ShortID 管理、Vision 流控、客户端配套要求。
> 定位：与 Hysteria2 并行的第二主力协议，承担 TCP 通道与"无自有域名场景"的兜底。

---

## 1. 在双协议体系中的定位

| 维度 | Hysteria2 | **VLESS-Reality** | 互补关系 |
|---|---|---|---|
| 传输层 | UDP (QUIC) | TCP | UDP 被 QoS 时 TCP 兜底 |
| 拥塞控制 | Brutal（主动抢带宽） | TCP BBR/Cubic（守规矩） | 高峰期 Hy2 抢流量、稳态 Reality 节能 |
| 证书 | 自有通配符证书（DNS-01） | **借用 dest 站点的证书** | Reality 不需要任何自有证书 |
| 域名依赖 | 必须自有域名 | **不需要自有域名** | 节点 IP 暴露即可入站 |
| 抗主动探测 | masquerade 反代 | dest 转发（更彻底） | Reality 强于 Hy2 |
| 客户端兼容 | sing-box / Hiddify / Hy2 官方 | sing-box / Xray / NekoBox | 二者交集即可 |
| 失败时退路 | 切到 Reality | 切到 Hy2 | 客户端自动 fallback |

**两个协议的部署位置在 oxygo 中是等价的**：同一个节点上 Hy2 监听 `udp/:443`、Reality 监听 `tcp/:443`，端口不冲突；订阅自动下发两个 outbound，客户端按延迟/可达性选择。

---

## 2. Reality 工作原理（理解之后再谈 GFW）

Reality 是 XTLS 团队在 2023 年提出的方案，核心思路是**借用一个真实运行的 HTTPS 站点的 TLS 握手与证书来作为伪装**，DPI 看到的是合法的 TLS 1.3 + H2 流量，根本不存在"代理特征"。

**握手流程**（关键点用 ⭐ 标注）：

```
[客户端]                                          [oxygo-server (Reality)]
   │                                                       │
   │  ① TLS ClientHello                                    │
   │     SNI = www.microsoft.com  ⭐ 写真实 dest 域名       │
   │     携带 authKey（X25519(client_pub, server_priv)     │
   │              + shortID 派生）                          │
   │ ──────────────────────────────────────────────────▶   │
   │                                                       │
   │                                  ② 服务端校验 authKey  │
   │                                  ┌────────────────┐  │
   │                                  │ 合法 oxygo 客户端│  │
   │                                  │  → 走 VLESS    │  │
   │                                  │ 非法 / 探测     │  │
   │                                  │  → 完全转发到   │  │
   │                                  │    dest 真实站点│  │
   │                                  └────────────────┘  │
   │                                                       │
   │  ③ 合法路径：客户端"挪用"了 dest 的证书完成 TLS 握手   │
   │     ⭐ DPI 看到的是 dest 的真证书，CT 日志、证书链都对得上│
   │                                                       │
   │  ④ TLS 1.3 内部跑 VLESS 协议（Vision 流控加持）        │
```

**Reality 与传统 TLS 代理的根本区别**：
- 不需要自己申请证书：客户端在 TLS 握手中"挪用" dest 的证书
- 错误的客户端不被拒绝，而是被透明转发到真实 dest，**主动探测者看到的是 dest 网站本身**
- DPI 把流量识别为"客户端在访问 dest"，因为 SNI、证书、ALPN 全都对得上

---

## 3. GFW 威胁建模（针对 Reality）

复用 design-hysteria2.md §2.1 的威胁编号体系（T1~T10），对 Reality 的影响重新评估：

| # | 威胁 | 对 Reality 的影响 | 本协议对策 |
|---|---|---|---|
| T1 | UDP QoS | **不适用**（Reality 走 TCP） | — |
| T2 | 主动探测 | **几乎完全免疫**：非法握手会被转发到 dest，探测者看到真站点 | dest 转发是 Reality 原生能力，零配置 |
| T3 | SNI 黑名单 | **无关**：SNI 是 dest 域名（如 microsoft.com），属于白名单 | dest 选大站即可 |
| T4 | 证书 CT 扫描 | **无关**：不签自己的证书，CT 里没有任何 oxygo 相关条目 | Reality 的核心优势 |
| T5 | TLS 指纹 (JA3/JA4) | 中等：非主流客户端指纹仍可识别 | 客户端 sing-box 启用 **uTLS Chrome 指纹** |
| T6 | QUIC 指纹 | **不适用**（TCP） | — |
| T7 | 流量分析 | 低：Vision 流控掩盖 TLS-in-TLS 熵特征 | 必启 `flow: xtls-rprx-vision` |
| T8 | IP 整段封锁 | 与所有协议一样，节点冗余 | admin 多节点 + 健康监控 |
| T9 | 端口扫描 | 低：端口 443 与正常 HTTPS 站点一致；任何握手都会被 dest 转发，扫描者看到正常网站 | 单端口 443，无端口跳跃需求 |
| T10 | DNS 污染 | **无关**：节点 IP 可直接配置在客户端，无需域名 | 订阅写入 IP；可选配 SNI=dest 域名 |

**威胁优先级（Reality）**：

- **P0**：T5（TLS 指纹）、T7（流量分析）
- **P1**：T8（IP 封锁）
- **不存在**：T1、T3、T4、T6、T9、T10

**与 Hy2 对比**：Reality 把 T1/T3/T4/T6/T9/T10 几乎全部消解，是**抗审查上限最高**的方案。代价是性能上限低于 Hy2（TCP vs QUIC + Brutal）。

---

## 4. dest 站点选择（Reality 的核心决策）

**dest（也叫 target）是 Reality 协议借用的真实 HTTPS 站点**。选错 dest 是 Reality 部署最常见的失败原因。

### 4.1 硬性要求

dest 站点必须**全部**满足以下条件：

| 条件 | 检测方法 | 不满足的后果 |
|---|---|---|
| 支持 TLS 1.3 | `openssl s_client -tls1_3 -connect dest:443` | Reality 握手失败 |
| 支持 H2 (HTTP/2) | ALPN 协商出 `h2` | sing-box 校验失败 |
| 证书域名匹配 SNI | `openssl s_client -servername` 验证 | 证书校验失败 |
| 无 HTTP→HTTPS 强制跳转到子域名 | curl 跟随重定向 | 流量模式异常 |
| 站点本身**未被 GFW 封锁** | 国内多机房 `curl` 测试 | 国内客户端连不上 |
| 是大型站点（非个人/小众） | Alexa/Tranco rank | 流量模式异常易被聚类识别 |
| 服务器响应延迟稳定 | 多次 ping/curl 测延迟 | 节点性能波动 |

### 4.2 推荐 dest 候选清单（v1 内置）

按节点地理位置分类（GFW 主要拦截"反向跨境"流量，dest 与节点同区域更自然）：

**全球通用（任何地区节点都可用）**：

| dest | TLS 1.3 | H2 | GFW 状态 | 评估 |
|---|---|---|---|---|
| `www.microsoft.com:443` | ✅ | ✅ | 未封锁 | 主选；流量极大，零特征 |
| `learn.microsoft.com:443` | ✅ | ✅ | 未封锁 | 备选；文档站，长连接合理 |
| `www.apple.com:443` | ✅ | ✅ | 未封锁 | 备选 |
| `swdist.apple.com:443` | ✅ | ✅ | 未封锁 | 软件分发，大流量合理 |

**亚太节点优选**：

| dest | 节点位置 | 评估 |
|---|---|---|
| `www.tesla.com:443` | 香港/日本/新加坡 | 合理 |
| `dl.google.com:443` | ⚠️ 不推荐 | Google 被封 |
| `s0.awsstatic.com:443` | 任何 AWS 区域附近 | AWS CDN |

**欧美节点优选**：

| dest | 评估 |
|---|---|
| `www.cloudflare.com:443` | ⚠️ 谨慎：CF 自身 TLS 配置可能有特殊指纹 |
| `www.amazon.com:443` | 可用 |
| `www.yahoo.com:443` | 可用 |

**绝对不要选**：

- 任何 Google 域名（被封锁，反向流量异常）
- 自己控制的小站（流量模式会暴露）
- 已被其他代理项目滥用的 dest（被关联识别风险）
- 短链/重定向站、CDN 边缘节点

### 4.3 admin 端的 dest 管理

设计上 admin 维护一份"dest 候选注册表"：

```go
type DestCandidate struct {
    Host       string    // "www.microsoft.com"
    Port       uint16    // 443
    Regions    []string  // ["global", "apac", "us"]
    Verified   bool      // 上一次健康检查通过
    LastProbe  time.Time
    TLS13      bool
    H2         bool
    LatencyMs  map[string]int  // 各节点测得的延迟
    Notes      string
}
```

**自动健康检查**（admin 定期跑）：
- 每 6 小时对所有 dest 候选做一次探测
- 任一节点对 dest 的连通性测试失败 → 在 UI 标红，告警管理员
- 添加新节点时自动测一遍所有候选 dest 的延迟，按延迟排序推荐

### 4.4 节点 ↔ dest 绑定策略

| 策略 | 描述 | 评估 |
|---|---|---|
| **全节点同 dest** | 所有节点都借用 `www.microsoft.com` | 简单；同 IP 段多节点流量模式相同，被聚类风险微升 |
| **每节点独立 dest** | 每个节点轮换/分配不同 dest | 更分散；管理成本高 |
| **按区域分配** | apac 节点用 microsoft，us 节点用 apple | **v1 推荐**：兼顾分散与管理 |

---

## 5. X25519 密钥与 ShortID 管理

### 5.1 X25519 密钥对

Reality 使用 X25519 ECDH 派生握手认证密钥。**每个 inbound 一对密钥**：

| 字段 | 持有者 | 用途 |
|---|---|---|
| `private_key` | 服务端（节点） | sing-box inbound 配置 |
| `public_key` | 客户端 | 订阅中下发，用于派生 authKey |

**生成**：
- admin 端使用 `golang.org/x/crypto/curve25519` 生成密钥对（也可调用 sing-box 自带 `sing-box generate reality-keypair` 工具）
- 私钥下发到节点（gRPC + mTLS 信道）
- 公钥嵌入用户订阅

**轮换**：
- 默认 365 天轮换一次（手动触发；自动轮换需所有客户端订阅同步刷新，谨慎）
- 旧密钥保留 7 天宽限期（sing-box 支持多 private_key？目前不支持，需双 inbound 并行——v1 不实现，留待 v2）

### 5.2 ShortID

ShortID 是 Reality 增加的二次认证因子，**只有携带正确 ShortID 的客户端才会进入 VLESS 通道**，其余全部被转发到 dest。

| 属性 | 取值 |
|---|---|
| 长度 | 0~16 hex 字符（即 0~8 字节） |
| **空字符串 `""`** | ⚠️ 表示"匹配任意 shortID"，**严禁配置**，会丧失二次认证 |
| 推荐长度 | **8 hex 字符（4 字节）** |
| 每 inbound 允许的数量 | 多个，最多 8 个 |

### 5.3 ShortID 分配策略

| 策略 | 描述 | 评估 |
|---|---|---|
| 单 shortID，全用户共享 | 每个 inbound 配 1 个 shortID，所有用户都用它 | **v1 推荐**：简单，足够安全（用户隔离靠 VLESS UUID） |
| 每用户组一个 shortID | 按用户组（如 VIP/普通）分配 | v2 考虑 |
| 每用户一个 shortID | 8 个 shortID 上限不够，且 sing-box 性能下降 | 不采用 |

**v1 默认**：单 shortID，由 admin 在创建 inbound 时随机生成，存入 `InboundSpec.Params["short_id"]`。

### 5.4 SpiderX（v1.10+ 可选）

Reality 还支持 `spider_x` 参数模拟客户端访问 dest 的爬虫路径，进一步逼真。v1 不启用（默认值 `/` 已足够），文档保留位置以备 v2。

---

## 6. Vision 流控（必启）

Vision（`xtls-rprx-vision`）是 XTLS 的客户端→服务端流控扩展，**只对 VLESS 入站生效**。它解决了 TLS-in-TLS 代理的字节熵特征问题。

**机制**：客户端识别出 inner TLS 握手报文后，**直接转发原始 TLS 包**而非再加一层加密，使外层流量与正常 HTTPS 访问的字节熵分布一致。

**配置**：
- 服务端 `users[].flow: "xtls-rprx-vision"`
- 客户端 outbound `flow: "xtls-rprx-vision"`
- **必须双端一致**，否则连接失败

**风险**：Vision 只在客户端访问 HTTPS 站点（即代理目标也是 TLS）时生效；访问明文 HTTP 没有 inner TLS，Vision 不工作但也不影响连接。

---

## 7. Reality 服务端配置设计

### 7.1 推荐参数集（v1 默认）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `listen` | `[::]:443` | 双栈，复用 443/TCP |
| `tls.enabled` | `true` | |
| `tls.server_name` | dest 域名（如 `www.microsoft.com`） | **必须**等于 dest，客户端 SNI 与之一致 |
| `tls.reality.enabled` | `true` | |
| `tls.reality.handshake.server` | dest 域名 | 服务端连到这个真实站点做转发 |
| `tls.reality.handshake.server_port` | `443` | |
| `tls.reality.private_key` | 32 字节 X25519 | 节点独立 |
| `tls.reality.short_id` | `[<8 hex>]` | 至少一个、不可为空字符串 |
| `tls.reality.max_time_difference` | `1m` | 抗重放，宽容时钟漂移 |
| `users[].uuid` | UUID v4 | 每用户独立 |
| `users[].flow` | `xtls-rprx-vision` | 必启 |
| `transport` | 不配置 | Reality 直接走 TLS 即可，不需要 WS/gRPC |

### 7.2 渲染后的 sing-box inbound 示例

```json
{
  "type": "vless",
  "tag": "vless-reality-in",
  "listen": "::",
  "listen_port": 443,
  "users": [
    { "name": "u-001", "uuid": "8ffd1a8b-...", "flow": "xtls-rprx-vision" },
    { "name": "u-002", "uuid": "ec7b2a4f-...", "flow": "xtls-rprx-vision" }
  ],
  "tls": {
    "enabled": true,
    "server_name": "www.microsoft.com",
    "reality": {
      "enabled": true,
      "handshake": {
        "server": "www.microsoft.com",
        "server_port": 443
      },
      "private_key": "<base64 32 bytes>",
      "short_id": ["a1b2c3d4"],
      "max_time_difference": "1m0s"
    }
  }
}
```

### 7.3 `NodeSpec` → Reality inbound 渲染映射

```go
type RealityParams struct {
    DestHost   string   // "www.microsoft.com"
    DestPort   uint16   // 443
    PrivateKey string   // base64 32B
    ShortIDs   []string // 8-hex 串，至少 1 个
    // PublicKey 不在节点配置中（只在客户端订阅里）
}
```

`Users[].UUID` 复用 design-server.md §1.4 中的用户 UUID。

---

## 8. 端口共用（与 Hy2 同节点 443）

Reality 走 **TCP 443**，Hy2 走 **UDP 443**，Linux 内核层面两者**互不冲突**（不同协议族），sing-box 也支持同一进程内两个 inbound 分别监听。

`singboxgen` 渲染时需要保证：
- Hy2 inbound `listen_port = 443`，listen 接 UDP（sing-box 自动按协议选）
- Reality inbound `listen_port = 443`，listen 接 TCP（同上）
- 两个 inbound `tag` 不同，便于统计区分

**端口冲突检测**：渲染器在合并多个 inbound 时按 `(protocol, port)` 二元组判重，避免人工配错。

---

## 9. 客户端配套要求

| 配置项 | 取值 | 由谁负责 |
|---|---|---|
| outbound type | `vless` | admin 订阅生成 |
| `server` | 节点 IP（或独立的 `nodeN.example.com`） | 同上 |
| `server_port` | 443 | 同上 |
| `uuid` | 用户 UUID | 同上 |
| `flow` | `xtls-rprx-vision` | 同上 |
| `tls.enabled` | `true` | 同上 |
| `tls.server_name` | **dest 域名**（不是节点域名！） | 同上 |
| `tls.utls.enabled` | `true` | 同上 |
| `tls.utls.fingerprint` | `chrome`（或随机轮换） | 同上 |
| `tls.reality.enabled` | `true` | 同上 |
| `tls.reality.public_key` | base64 32B | 同上 |
| `tls.reality.short_id` | 一个 admin 下发的 shortID | 同上 |

**关键陷阱**：
- 客户端 SNI（`server_name`）必须等于 dest 域名，**不是节点 IP/域名**
- 客户端 `server`（连接目标）应该用**节点 IP**，避免被 DNS 污染（T10），无需任何节点域名

---

## 10. 实施路线图（Reality）

承接 design-hysteria2.md §9 的进度（M1.x 完成时 Hy2 已可用），Reality 排在 M3：

| 里程碑 | 任务 | 验收标准 |
|---|---|---|
| **M3.1** | `singboxgen` 增加 Reality 渲染，X25519 生成工具集成到 admin | `oxygo-admin keypair gen` 输出合法 X25519 对 |
| **M3.2** | admin 端 dest 候选注册表 + 自动健康检查 | UI 显示每个 dest 的 TLS 1.3 / H2 / 延迟 |
| **M3.3** | 同节点 Reality + Hy2 双 inbound 共存联调 | 单节点同时承载两个协议，sing-box `clash_api` 区分统计 |
| **M3.4** | 客户端订阅同时下发两个 outbound | 客户端按延迟自动选择，故障自动切换 |
| **M3.5** | uTLS 指纹随机化（可选） | 客户端订阅中 fingerprint 字段随机 chrome/firefox/safari |

**预计周期**：M3 整体 2 周（含联调与文档）。

---

## 11. 测试矩阵

| 测试类型 | 用例 |
|---|---|
| 单元 | `singboxgen.RenderReality` 输入边界：空 shortID（必须拒绝）、非法 X25519 长度、空 dest |
| 单元 | dest 健康检查函数：TLS 1.3 探测、H2 ALPN 探测、超时处理 |
| 集成 | 容器化 sing-box 客户端连接 → curl `https://example.com` 走代理 → 成功 |
| 集成 | 错误 shortID 连接 → 应看到 dest 真实站点响应（curl `https://node:443` 返回 microsoft 页面） |
| 集成 | 错误 UUID + 正确 shortID → 同样转发到 dest（防止精确探测） |
| 集成 | 双协议共存：同一节点 Hy2 + Reality 同时工作，互不影响 |
| 主动探测 | 用 `openssl s_client -servername www.microsoft.com -connect node:443` 应看到 dest 完整证书链 |
| 流量分析 | tcpdump 抓 100MB 走 Reality 的流量，与直接访问 dest 的流量做包大小直方图对比，应高度相似 |
| 安全 | 私钥泄露场景：撤销节点 → admin 一键吊销并重生密钥对 |

---

## 12. 与上游设计的修订

| 文档 | 章节 | 修订 |
|---|---|---|
| design-server.md | §1.3 推荐方案 | "VLESS-Reality 作为主力"中"伪装目标：www.microsoft.com / www.cloudflare.com" → **删除 Cloudflare**，原因见本文 §4.2 |
| design-server.md | §2.3 InboundSpec | `Params` 增加 `ShortIDs []string`、`DestHost`、`DestPort`、`PrivateKey` |
| design-server.md | §3.6 证书与机密 | Reality 部分明确：**节点不持有任何 TLS 证书**（与 Hy2 不同） |
| design.md | §3.1 admin 功能 | "协议模板"中增加"dest 候选管理"子功能 |
| design.md | §6.3 证书与信任链 | 注明 Reality 不使用 Oxygo CA 体系 |

---

## 13. 待决策项（Reality）

1. **dest 候选清单是否打包内置**：v1 建议内置 5~10 个经过验证的 dest，admin UI 允许用户增删。
2. **同节点是否强制双协议**：用户可能希望只开 Reality 或只开 Hy2。v1 倾向**默认双开**，UI 允许单开。
3. **uTLS 指纹是否随机轮换**：固定 `chrome` 已足够，随机化反而可能引入异常。v1 **固定 chrome**。
4. **是否允许客户端走节点域名而非 IP**：用域名解析便于 IP 切换，但引入 T10 风险。v1 **默认 IP**，admin 可在订阅生成时选 IP 或域名。
5. **X25519 密钥轮换是否自动化**：自动会导致旧订阅失效，v1 **手动触发**。

---

## 14. 双协议协同：客户端订阅设计

最终客户端收到的订阅（sing-box 格式）应包含**两个 outbound**，按 admin 配置的策略组合：

```json
{
  "outbounds": [
    { "type": "selector", "tag": "auto",
      "outbounds": ["hy2-node1", "reality-node1", "hy2-node2", "..."],
      "default": "hy2-node1",
      "interrupt_exist_connections": false
    },
    { "type": "urltest", "tag": "auto-fast",
      "outbounds": ["hy2-node1", "reality-node1", "hy2-node2", "..."],
      "url": "https://www.gstatic.com/generate_204",
      "interval": "3m",
      "tolerance": 50
    },
    { "type": "hysteria2", "tag": "hy2-node1", "server": "node1.proxy.example.com", "server_port": 443, "...": "..." },
    { "type": "vless", "tag": "reality-node1", "server": "1.2.3.4", "server_port": 443, "flow": "xtls-rprx-vision", "tls": { "...": "..." } }
  ]
}
```

**默认策略**：
- `urltest` 自动按延迟选择，Hy2 通常更快 → 高带宽时段自动用 Hy2
- 用户手动选择 `selector` 可强制走 Reality（UDP 被限速时的兜底）
- 客户端 UI 给出"Auto / 强制 Hy2 / 强制 Reality"三个一键开关

---

## 15. 给开发者的下一步

1. **本周**：与你确认 §13 待决策项 1～5。
2. **M1~M2 期间**：暂不动 Reality，集中精力把 Hy2 链路打通。
3. **M3.1 启动时**：先做 X25519 工具与 dest 注册表，**这两个不依赖 sing-box，可与 Hy2 收尾并行开发**。
4. **M3.3 是关键节点**：双 inbound 共存通常一次跑通；若失败基本是 `listen` 字段配置错误，先查 `sing-box check` 输出。
