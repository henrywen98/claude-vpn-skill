# Cloudflare DNS 策略与双线路设计

什么时候走 CF、什么时候直连、CF 优选 IP 怎么选——本文档解释这些决策背后的"为什么"，让 skill 在不同场景下能做合理推荐。

## 核心模型：GFW 是"单向门卫"

GFW 只管"中国大陆客户端 → 海外目标 IP"这一段流量。它**看不见也管不着**海外 IP 之间的通信。

```
直连场景：
  中国客户端 ──(GFW 在这里检查)──→ VPS 真实 IP
  IP 一旦上 GFW 黑名单 → 客户端连不上

走 CF 场景：
  中国客户端 ──(GFW)──→ CF 边缘节点 ───(海外段，GFW 看不见)───→ VPS 真实 IP
  即使 VPS IP 被封，CF→VPS 这段照样通
```

**结论**：CF 的真正价值不是"加速"，是"备胎"——在 VPS IP 被封时让客户端仍能连上。日常体验直连远胜走 CF。

---

## 双 DNS 记录设计

部署后 CF 上**同时存在两条 A 记录**：

| 记录 | 代理状态 | 作用 |
|---|---|---|
| `{SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}` → VPS IP | 🟠 橙云（Proxied） | CF 兜底入口。客户端用这个域名走 CF |
| `{ROOT_DOMAIN}`（@ 根域）→ VPS IP | ⚪ 灰云（DNS only） | 天然的直连入口。客户端可直接连根域名得到真实 IP |

**为什么用根域而不是新建 `direct.xxx` 子域**：
- 根域名是 zone 自带的，不用新建 DNS 记录
- 通配符证书 `*.example.com` + 根域 `example.com` 都覆盖，无需额外签证书
- 避免污染子域命名空间

**为什么 nginx 不需要为根域单独配置**：
- 客户端实际不会用 host=根域 来连——它们用 host=IP 直连
- nginx 按 SNI 匹配，客户端 SNI 仍发 `{SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}`，命中现有 server block

---

## 客户端配置模式（IP Override）

直连节点的关键技巧：**地址填 IP，但 SNI 和 Host 头保留子域名**。

```
传输层（TCP/TLS）：连 {SERVER_IP}:443，SNI = {SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}
应用层（HTTP/WS）：Host: {SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}, path = /{WS_PATH}
```

这样：
- TCP 握手直奔 VPS（绕过 CF 和 GFW 的 DNS 层污染）
- TLS 握手用域名 SNI（证书校验通过，nginx 按 SNI 命中正确 server block）
- WS Upgrade 时 Host 头正确（nginx location 路由生效）

---

## CF 优选 IP（CF Optimized IPs）

CF 默认 DNS 解析返回的 IP 经常是"垃圾 IP"（高延迟、限速）。**优选 IP** 是手工选过的、对你所在网络较快的 CF 边缘 IP。把它写死在客户端节点里能绕开 CF 的 DNS 轮询。

### 三种获取方法

**方法一：在线工具（最简单）**

- https://www.wetest.vip/page/cloudflare/address_v4.html
- https://www.182682.xyz/

页面会按地理 / ISP 给出实时优选 IP 列表。复制 ping 最低的几个备用。

**方法二：本地测速工具**

[CloudflareSpeedTest](https://github.com/XIU2/CloudflareSpeedTest)：

```bash
./CloudflareST -tll 100 -tl 200 -dn 5
```

按延迟和下载速度自动排序 CF 全段，输出 top N。**最准确**，因为是从你本机测的。

**方法三：社区维护列表**

- https://api.uouin.com/cloudflare.html — 不同地区的优选 IP 列表，按需复制

### 给客户端配几个 CF 优选 IP？

- **1 个就够**：日常切换够用
- **2-3 个更稳**：避免单个 CF IP 临时被 CF 内部下线导致兜底失效

通常推荐：1 个 CF 优选 IP 节点 + 1 个 CF 域名节点（兼顾"快兜底"和"绝对兜底"）。

---

## 应用决策树

skill 根据用户回答（信息收集第 #10/#11 项）走哪条分支：

```
┌─ 月流量 < 500GB?
│   └─ Yes → 提醒用户：重度用易超额。仍配双线路，但建议路由规则收紧（国内分流走 DIRECT）
│
├─ 线路是 CN2 GIA / 联通精品 / 公司主线?
│   └─ Yes → 强推直连为主。CF 兜底配齐，但日常无需切换
│
├─ 线路是廉价 BGP / 易封段?
│   └─ Yes → 直连可用，但建议:
│            - CF 优选 IP 配 2-3 个（防被封批次）
│            - 提醒用户备一笔预算/备用机房，封了能快速换 IP
│
└─ 线路不清楚?
    └─ 默认两条都给齐，让客户端自测延迟选
```

---

## 何时建议**单线路**（不上双线路）？

少数场景双线路是过度设计：

- VPS 的 ToS 不允许暴露真实 IP（如某些 GCP free-tier） → 仅 CF
- 用户明确表示"我只用 CF，不在乎速度" → 仅 CF
- VPS 在国内（仅服务海外用户） → 仅直连，CF 反而拖慢
- 用户用 cloudflared Argo Tunnel → 完全无源 IP 暴露，CF 是唯一路径

**默认情况都应该上双线路**——服务器侧改动为零（同一证书、同一 nginx、同一 Xray 入站），只是客户端多一条节点而已。

---

## 故障切换决策

| 症状 | 应急动作 |
|---|---|
| 直连节点突然断 | 客户端切到 CF 优选 IP 节点 |
| CF 优选 IP 节点也断 | 切到 CF 域名节点（CF DNS 自动选 IP） |
| 三条全断 + 伪装站能访问 | GFW 可能针对域名做 SNI 阻断，换子域名 |
| 三条全断 + 伪装站也打不开 | VPS 真挂或被 ban，去 VPS 控制台看 |

详细排障流程见 `troubleshooting.md`。
