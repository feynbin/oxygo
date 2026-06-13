# 手工安装 sing-box vs Oxygo 增量

> 版本：v0.1 ｜ 日期：2026-05-28
> 目的：先把"在一台 Ubuntu VPS 上不依赖 oxygo、纯手工把 sing-box 跑起来提供代理能力"的完整流程列清楚；再据此对比 oxygo 在 sing-box 之上补齐了哪些能力。这份文档既是排错时回退到原生 sing-box 的速查手册，也是产品价值清单。

---

## Part 1：在 Ubuntu 上手工跑起 sing-box

下文以 **Ubuntu 22.04 / 24.04 LTS** 为例，目标：一台 VPS 同时提供 **Hysteria2 + VLESS-Reality** 双入站。

### 1.1 前置条件

| 项 | 要求 |
|---|---|
| 操作系统 | Ubuntu 22.04 LTS 或 24.04 LTS（x86_64 / arm64） |
| 域名 | 已托管在支持 API 的 DNS 服务商（Cloudflare 推荐）—— Hy2 用 |
| 端口 | 入站 UDP/443、TCP/443，以及端口跳跃 UDP 40000-50000 |
| 权限 | 拥有 sudo 的非 root 用户 |
| 时钟 | 时间同步开启（Reality 对时钟敏感，差异 >1 分钟会失败） |

```bash
# 时钟同步检查
timedatectl status     # NTP service: active
sudo timedatectl set-ntp true
```

### 1.2 安装 sing-box

**方式 A：官方 APT 仓库（推荐）**

```bash
sudo curl -fsSL https://sing-box.app/gpg.key \
    -o /etc/apt/keyrings/sagernet.asc
sudo chmod a+r /etc/apt/keyrings/sagernet.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/sagernet.asc] \
    https://deb.sagernet.org/ * *" \
    | sudo tee /etc/apt/sources.list.d/sagernet.list

sudo apt update
sudo apt install -y sing-box

sing-box version       # 验证
```

**方式 B：直接下载 release 二进制**

适合不希望依赖第三方 APT 源的场景：

```bash
VER=$(curl -s https://api.github.com/repos/SagerNet/sing-box/releases/latest \
    | grep tag_name | cut -d '"' -f 4 | sed 's/^v//')
ARCH=$(dpkg --print-architecture)   # amd64 / arm64

curl -fsSL "https://github.com/SagerNet/sing-box/releases/download/v${VER}/sing-box-${VER}-linux-${ARCH}.tar.gz" \
    | sudo tar xz -C /tmp
sudo install /tmp/sing-box-${VER}-linux-${ARCH}/sing-box /usr/local/bin/sing-box
```

**方式 C：从源码编译（启用全部 tag）**

仅在 release 不含你需要的功能时使用：

```bash
sudo apt install -y golang-go
go install -v -tags \
    with_quic,with_grpc,with_dhcp,with_wireguard,with_utls,with_clash_api \
    github.com/sagernet/sing-box/cmd/sing-box@latest
sudo install ~/go/bin/sing-box /usr/local/bin/sing-box
```

### 1.3 准备工作目录与用户

```bash
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/sing-box sing-box || true
sudo mkdir -p /etc/sing-box /var/lib/sing-box /var/log/sing-box
sudo chown -R sing-box:sing-box /var/lib/sing-box /var/log/sing-box
sudo chmod 0750 /etc/sing-box
```

让非 root 进程也能监听 443：

```bash
sudo setcap cap_net_bind_service=+ep /usr/local/bin/sing-box
```

### 1.4 生成 Reality 密钥与 ShortID

```bash
# X25519 keypair
sing-box generate reality-keypair
# 输出：
# PrivateKey: <base64-32B>
# PublicKey:  <base64-32B>

# ShortID（用 openssl 取 4 字节随机十六进制）
openssl rand -hex 4
# 输出：a1b2c3d4
```

留存这三样：私钥、公钥、ShortID（后两者给客户端用）。

### 1.5 申请 Hy2 通配符证书

以 Cloudflare DNS 为例，使用 `lego`（acme 客户端）：

```bash
sudo apt install -y lego

export CLOUDFLARE_DNS_API_TOKEN="<token>"   # 仅 Zone.DNS edit 权限即可
sudo -E lego \
    --email you@example.com \
    --dns cloudflare \
    --domains "*.proxy.example.com" \
    --domains "proxy.example.com" \
    --path /etc/sing-box/certs \
    --accept-tos \
    run

# 证书路径：
#   /etc/sing-box/certs/certificates/_.proxy.example.com.crt
#   /etc/sing-box/certs/certificates/_.proxy.example.com.key
```

把 DNS A/AAAA 记录指到本机 IP：

```
node1.proxy.example.com  A   <your-vps-ipv4>
```

### 1.6 编写 sing-box 配置

把以下内容写到 `/etc/sing-box/config.json`（替换占位符）：

```json
{
  "log": { "level": "warn", "output": "/var/log/sing-box/box.log", "timestamp": true },
  "experimental": {
    "clash_api": { "external_controller": "127.0.0.1:9090" }
  },
  "inbounds": [
    {
      "type": "hysteria2",
      "tag": "hy2-in",
      "listen": "::",
      "listen_port": 443,
      "up_mbps": 900,
      "down_mbps": 900,
      "obfs": {
        "type": "salamander",
        "password": "<32-byte-random>"
      },
      "users": [
        { "name": "u-alice", "password": "<per-user-pwd-1>" },
        { "name": "u-bob",   "password": "<per-user-pwd-2>" }
      ],
      "tls": {
        "enabled": true,
        "server_name": "node1.proxy.example.com",
        "alpn": ["h3"],
        "min_version": "1.3",
        "certificate_path": "/etc/sing-box/certs/certificates/_.proxy.example.com.crt",
        "key_path":         "/etc/sing-box/certs/certificates/_.proxy.example.com.key"
      },
      "masquerade": "https://www.bing.com/"
    },
    {
      "type": "vless",
      "tag": "vless-reality-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        { "name": "u-alice", "uuid": "<uuid-1>", "flow": "xtls-rprx-vision" },
        { "name": "u-bob",   "uuid": "<uuid-2>", "flow": "xtls-rprx-vision" }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": { "server": "www.microsoft.com", "server_port": 443 },
          "private_key": "<x25519-priv>",
          "short_id": ["a1b2c3d4"]
        }
      }
    }
  ],
  "outbounds": [
    { "type": "direct", "tag": "direct" }
  ]
}
```

**校验配置**：

```bash
sudo sing-box check -c /etc/sing-box/config.json
```

### 1.7 编写 systemd unit

APT 安装方式已自带 `sing-box.service`；二进制方式需自己写：

```ini
# /etc/systemd/system/sing-box.service
[Unit]
Description=sing-box service
After=network-online.target
Wants=network-online.target

[Service]
User=sing-box
Group=sing-box
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/local/bin/sing-box run -c /etc/sing-box/config.json
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sing-box
sudo systemctl status sing-box      # 检查
sudo journalctl -u sing-box -f       # 看日志
```

### 1.8 配置端口跳跃（仅 Hy2）

```bash
# 开放 UDP 端口段（云厂商安全组也要放行）
sudo apt install -y nftables
sudo nft add table inet sb
sudo nft add chain inet sb prerouting '{ type nat hook prerouting priority -100; }'
sudo nft add rule inet sb prerouting iif "eth0" udp dport 40000-50000 redirect to :443

# 持久化
sudo nft list ruleset | sudo tee /etc/nftables.conf
sudo systemctl enable --now nftables
```

`eth0` 按 `ip route show default` 实际网卡名替换。

### 1.9 配置 ufw / 云安全组

```bash
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw allow 40000:50000/udp
sudo ufw reload
```

云厂商（AWS/阿里云/Vultr）的安全组也要同样放行。

### 1.10 客户端连接测试

在另一台机器装 sing-box 客户端，写一份最小 outbound 配置：

```jsonc
{
  "outbounds": [
    { "type": "hysteria2",
      "tag": "hy2-out",
      "server": "node1.proxy.example.com",
      "server_port": 443,
      "server_ports": ["40000:50000"],
      "up_mbps": 50, "down_mbps": 200,
      "password": "<per-user-pwd-1>",
      "obfs": { "type": "salamander", "password": "<32-byte-random>" },
      "tls": { "enabled": true, "server_name": "node1.proxy.example.com", "alpn": ["h3"] }
    },
    { "type": "vless",
      "tag": "reality-out",
      "server": "<vps-ip>",
      "server_port": 443,
      "uuid": "<uuid-1>",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": { "enabled": true, "fingerprint": "chrome" },
        "reality": { "enabled": true, "public_key": "<x25519-pub>", "short_id": "a1b2c3d4" }
      }
    }
  ]
}
```

```bash
curl --proxy socks5://127.0.0.1:1080 https://api.ipify.org    # 看到 VPS IP 即成功
```

### 1.11 续期与运维（手工方式）

- **证书续期**：lego 不带定时器，需要自己写 cron：
  ```cron
  0 3 * * 1  root  lego --email you@example.com --dns cloudflare \
            --domains "*.proxy.example.com" --path /etc/sing-box/certs \
            renew --days 30 && systemctl reload sing-box
  ```
- **改用户/密码**：编辑 `config.json` → `sing-box check` → `systemctl restart sing-box`（连接全断）
- **看流量**：`curl -s http://127.0.0.1:9090/traffic` 或 yacd 仪表盘
- **多节点**：每台 VPS 都重做 §1.2~§1.10，密钥/UUID 自己保管同步
- **新增用户**：每台节点 `config.json` 的 `users[]` 各加一条，restart sing-box

至此，**一台**节点的 sing-box 就能独立提供代理能力了。

---

## Part 2：Oxygo 在 sing-box 之上补齐的能力

把 Part 1 整套流程做完，你得到的是"一台能用的节点"。要长期稳定运营，仍然欠缺很多东西。Oxygo 的存在就是把这些缺口补齐。

### 2.1 能力差异总览

| 能力 | 手工 sing-box | Oxygo 提供 |
|---|---|---|
| 单节点跑起来 | ✅ | ✅ |
| 多节点统一管理 | ❌ 每台机器分别 SSH | ✅ admin 一处下发 |
| 用户生命周期 | ⚠️ 仅 `users[]` 静态数组 | ✅ CRUD、过期、配额、批量导入 |
| 流量统计 | ⚠️ 仅 ClashAPI 实时值 | ✅ 历史曲线、按用户/节点聚合、超额告警 |
| 订阅服务 | ❌ 自己写脚本/Nginx | ✅ HTTPS endpoint + token + 多格式 |
| 证书自动化 | ⚠️ 手工 cron + lego | ✅ admin 集中 ACME、续期、批量下发 |
| Reality 密钥/dest 管理 | ⚠️ 手工生成、写配置 | ✅ 一键生成、dest 健康检查注册表 |
| 端口跳跃 | ⚠️ 手写 nftables | ✅ Agent 自动配置、回收 |
| 配置变更 | ⚠️ 编辑 JSON + restart（断连） | ✅ 双实例切换式 reload、失败回滚 |
| 离线运行 | ⚠️ 改完忘了重启 = 不生效 | ✅ Agent 本地 BoltDB 缓存最新 revision |
| 控制面安全 | ❌ SSH 暴露面大 | ✅ gRPC + mTLS + 签名校验 |
| 日志/审计 | ⚠️ 本地日志文件 | ✅ 流式回传、审计表、告警 webhook |
| 故障切换 | ❌ 客户端写死单节点 | ✅ 订阅含 urltest，多节点自动切换 |
| 客户端体验 | ❌ 用户手工导入配置 | ✅ Wails3 桌面，一键连接 / TUN 模式 |
| 升级 | ⚠️ apt upgrade，每台重做 | ✅ admin 控制面升级 + 节点自更新 |
| 版本基线 | ⚠️ 各节点版本可能漂移 | ✅ admin 强制版本一致性 |

### 2.2 按场景分组的详细价值

#### A. 多节点运维（最直接的痛点）

**手工方式的痛**：3 个 VPS、5 个用户。每加一个用户，要 SSH 进 3 台机器，分别改 `config.json`，分别 restart，期间用户的连接全断。

**Oxygo 的解法**：
- admin UI 上点"新增用户" → 一次 NodeSpec 下发 → 3 台节点同时应用 → 双实例切换式 reload，断连窗口 1~3s
- 节点上下线、协议参数变更、流量配额调整，全部走同一个 UI

#### B. 用户与订阅

**手工的缺口**：
- sing-box 的 `users[]` 没有"过期时间""配额"概念，到期不会自停
- 用户拿不到订阅链接，要管理员手工把 JSON 发过去
- 用户换设备，订阅没法吊销

**Oxygo 的补齐**：
- 数据模型有 `users(expire_at, traffic_quota, traffic_used)`，admin 周期性轮询，到期/超额时下发空 `users[]`（即吊销）
- 订阅服务：`GET /api/v1/subscribe?token=xxx&fmt=singbox|clash|base64` → 返回**已渲染好的客户端配置**，含所有节点的 Hy2+Reality outbound 与 `urltest`
- token 吊销后，下次刷新即失败；token-设备绑定（可选）防多人共用

#### C. 流量统计与审计

**手工的缺口**：
- ClashAPI 只暴露**当下**的流量速率与连接列表，进程重启即丢失
- 想看"用户 Alice 这个月用了多少流量"必须自己写采集器

**Oxygo 的补齐**：
- Agent 30s 周期采样 ClashAPI 的 `/connections`（按 user 标签），增量写本地 BoltDB
- 每 1min 通过 gRPC stream 上报到 admin
- admin 落库 `traffic_logs(user_id, node_id, up_bytes, down_bytes, window)`
- UI 提供按用户/节点/时间维度的曲线、TopN、超额告警

#### D. 证书自动化

**手工的缺口**：
- lego 不带 daemon，需要 cron 跑续期；忘了写 cron 证书就过期
- 多节点要么共享同一份证书（手工 scp）要么每节点各申请（CT 日志风险大）

**Oxygo 的补齐**：
- admin 内置 ACME 客户端（基于 `lego` 库），托管 DNS Provider 凭据
- 通配符证书统一签发，到期前 30 天自动续期
- 续期成功 → gRPC 推送到所有节点 → supervisor 触发 sing-box 双实例切换式 reload

#### E. Reality 的 dest 与密钥治理

**手工的缺口**：
- dest 选错（如选了 Cloudflare，TLS 配置有特殊指纹）就埋坑
- 不知道 dest 现在还能不能用（被封锁了？TLS 配置变了？）
- 密钥/ShortID 写在配置里，轮换意味着所有客户端订阅作废

**Oxygo 的补齐**：
- dest 候选注册表，每 6h 自动健康检查（TLS 1.3、H2、延迟）
- 按节点区域推荐 dest，UI 标红失效项
- 密钥轮换 → 自动重生客户端订阅 → token 不变，订阅刷新即生效

#### F. 配置变更的中断窗口

**手工的痛**：
- 改 `users[]` 加一个用户 → `systemctl restart sing-box` → **所有现存连接全断**（即使加的是别人）
- 配置 typo → restart 后服务直接挂

**Oxygo 的补齐**：
- 双实例切换式 reload：先 close 旧、再 start 新，断连窗口 1~3s（vs systemd restart 的不确定窗口）
- 配置先用 `option.Validate` 校验，校验失败拒绝下发
- 启动失败自动回滚到上一个 revision（BoltDB 缓存）

#### G. 控制面安全

**手工方式的攻击面**：
- 改配置靠 SSH，SSH 端口要开
- 多节点需要在管理机存所有节点 SSH 密钥
- 没有审计：谁什么时候改了什么

**Oxygo 的补齐**：
- 节点 SSH 端口可以只对管理 IP 开放（首次部署后），日常控制走 gRPC
- gRPC + mTLS + NodeSpec 签名，三重校验
- 所有 admin 操作进 `audit_logs` 表

#### H. 客户端体验

**手工方式**：
- 用户拿到的是一段 JSON，自己装 sing-box CLI 或 NekoBox 导入
- 改密码、换节点都得手工再发一次

**Oxygo 的补齐**：
- oxygo-client（Wails3 桌面应用）：录入订阅 token → 自动拉取、自动连接、TUN/系统代理一键切换
- 故障自动切换：当前节点连不上 3 次 → 切到 `urltest` 选的次优节点
- 内置诊断：ping、curl、DoH 解析

### 2.3 概念映射：手工 → Oxygo

把手工流程的每一步对应到 Oxygo 的功能模块，方便理解：

| 手工步骤（Part 1） | Oxygo 对应模块 |
|---|---|
| §1.2 装 sing-box | Agent 内嵌 sing-box 库，无需单独安装 |
| §1.3 创建用户/目录 | Agent 安装脚本自动完成 |
| §1.4 生成 Reality 密钥 | admin 端 `crypto/x25519` 一键生成、入库 |
| §1.5 申请证书 | admin 证书中心 ACME DNS-01 |
| §1.6 编 config.json | admin UI 表单 → NodeSpec → singboxgen 渲染 |
| §1.7 systemd unit | oxygo-server 自带 unit，部署脚本写入 |
| §1.8 nftables 端口跳跃 | Agent 启动时自动下发，退出时清理 |
| §1.9 ufw 防火墙 | **仍需用户手工**（云厂商安全组也是）—— Oxygo 不接管主机防火墙 |
| §1.10 客户端测试 | oxygo-client 内置连通性体检 |
| §1.11 续期/改用户/重启 | admin UI 操作，节点自动收敛 |

### 2.4 Oxygo **不接管**的能力（边界）

明确划清边界，避免误解：

| 能力 | 处理方 | 说明 |
|---|---|---|
| 主机 OS 加固 | 用户/运维 | unattended-upgrades、SSH 密钥登录、fail2ban |
| 主机防火墙（ufw/云安全组） | 用户/运维 | Oxygo 只管 nftables 中 sing-box 相关规则 |
| DNS 记录维护 | 用户 | 节点域名 A/AAAA 由用户在 DNS 服务商配 |
| DDoS 防护 | 用户 / 云厂商 | 超出 Oxygo 范围 |
| 内核 BBR 等性能调优 | 用户/运维 | 推荐但不强制 |
| 节点 IP 切换 | 用户 | 节点 IP 变更后 admin UI 改 host 字段 |

---

## Part 3：何时回退到手工 sing-box

虽然 Oxygo 把绝大多数运维自动化了，仍然有几个场景**推荐先在手工 sing-box 上验证**，再放回 Oxygo：

1. **新协议/新参数评估**：sing-box 新版加了某协议，先用手工 config.json 跑通，验证可行后再加进 `singboxgen` 渲染器
2. **GFW 异常排查**：怀疑某 dest 被封 / Hy2 被 QoS，手工配置最小复现环境，排除 Oxygo 自身因素
3. **客户端兼容性**：客户端跑 sing-box / NekoBox / Hiddify 行为不一致时，手工配置剥离 Oxygo 订阅逻辑

把 Part 1 的步骤当作回退/排错的"黄金路径"。

---

## 附：关键命令速查

```bash
# 状态
systemctl status sing-box
journalctl -u sing-box -n 200 --no-pager

# 重载（手工方式：会断连）
sing-box check -c /etc/sing-box/config.json && \
    systemctl restart sing-box

# 实时统计（需启用 clash_api）
curl -s http://127.0.0.1:9090/traffic
curl -s http://127.0.0.1:9090/connections | jq

# Reality 密钥
sing-box generate reality-keypair

# 证书续期
lego --email ... --dns cloudflare --domains "*.proxy.example.com" \
     --path /etc/sing-box/certs renew --days 30

# nftables
nft list ruleset
nft flush table inet sb
```
