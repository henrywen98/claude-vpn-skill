---
name: x-ui-deploy
version: "3.0"
description: |
  Deploy, configure, or maintain a VLESS + XHTTP + TLS + Cloudflare CDN VPN service
  using the 3X-UI panel on a VPS. Use this skill when the user mentions: deploying a
  VPN server, setting up a proxy, installing x-ui/3x-ui, configuring VLESS/XHTTP,
  科学上网、翻墙、搭建代理、VPN 节点部署、Cloudflare CDN 代理配置。
  Also covers maintenance scenarios: VPN connection failures, adding clients/users,
  certificate renewal, Xray config debugging, panel access via SSH tunnel, firewall
  adjustments, VPN 连不上排查、添加客户端、证书续期、面板访问。
---

# X-UI VPN 部署

部署 VLESS + XHTTP + TLS + Cloudflare CDN 架构的 VPN 服务。

## 架构

```
客户端 → Cloudflare CDN (443) → Nginx (TLS/反代) → Xray (127.0.0.1:10000) → Internet
```

| 组件 | 技术 |
|------|------|
| VPN 协议 | VLESS |
| 传输方式 | XHTTP over TLS（替代 WebSocket，抗检测更强、弱网更稳） |
| 管理面板 | 3X-UI (MHSanaei) |
| 反向代理 | Nginx |
| SSL 证书 | acme.sh + Cloudflare DNS 验证 |
| CDN | Cloudflare（隐藏真实 IP） |

---

## 工作流程

整个部署分三步：**收集信息** → **SSH 逐步执行** → **Cloudflare 配置**。

### 第一步：收集前置信息

在做任何事情之前，向用户收集以下全部信息。逐轮询问，拿到所有信息后再进入第二步。

#### 必须收集的信息

| # | 信息 | 说明 | 示例 |
|---|------|------|------|
| 1 | VPS 服务器 IP | 服务器公网 IPv4 | `203.0.113.50` |
| 2 | SSH 用户名 | 默认 root | `root` |
| 3 | SSH 端口 | 默认 22 | `22` |
| 4 | SSH 认证方式 | 密码 or 密钥（密钥需路径） | `~/.ssh/id_rsa` |
| 5 | 根域名 | 已添加到 Cloudflare 的域名 | `example.com` |
| 6 | 子域名前缀 | VPN 节点名称，默认 `node1` | `node1` |
| 7 | 邮箱 | 用于 SSL 证书注册 | `you@email.com` |
| 8 | CF 认证方式 | Global API Key 或 API Token | 二选一 |
| 9 | CF 凭据 | Key+Email 或 Token 值 | — |

#### 收集时的引导话术

按以下顺序提问，每轮最多问 2-3 个相关项：

**第一轮**（服务器信息）：
> 请提供你的 VPS 信息：
> 1. 服务器公网 IP
> 2. SSH 用户名（默认 root）
> 3. SSH 端口（默认 22）
> 4. SSH 认证方式：密码登录还是密钥？如果是密钥，本地密钥路径是什么？

**第二轮**（域名信息）：
> 请提供域名相关信息：
> 1. 根域名（需要已经添加到 Cloudflare）
> 2. 子域名前缀（用于 VPN 节点，比如 `node1`，最终域名就是 `node1.example.com`）
> 3. 邮箱（用于 SSL 证书注册通知）

**第三轮**（Cloudflare 认证）：
> 最后需要 Cloudflare API 凭据来自动申请 SSL 证书：
>
> **方式一（推荐）**：Global API Key + 邮箱
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 查看 Global API Key
>
> **方式二**：API Token
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 创建令牌
> - 需要权限：Zone DNS Edit, Zone SSL Edit, Zone Settings Edit, Zone Page Rules Edit
>
> 你选哪种方式？请提供对应的凭据。

#### 信息验证

收集完所有信息后，汇总确认：

```
请确认以下部署信息：
  服务器:    root@203.0.113.50:22 (密钥认证)
  域名:      node1.example.com
  邮箱:      you@email.com
  CF 认证:   Global API Key

确认无误后开始部署。
```

用户确认后才进入第二步。

---

### 第二步：SSH 连接并逐步部署

#### 操作方式

1. **先完整读取 `references/manual-deploy.md`**，了解全部 15 个步骤
2. SSH 连接到服务器（根据认证方式选择命令）：
   ```bash
   # 密钥认证
   ssh -p {SSH_PORT} -i {KEY_PATH} {USER}@{SERVER_IP}
   # 密码认证
   ssh -p {SSH_PORT} {USER}@{SERVER_IP}
   ```
3. 先执行**步骤 0（定义变量）**，将用户提供的值填入变量定义块
4. 然后按步骤 1-15 顺序执行，命令中的 `$变量` 会自动展开为实际值

#### 执行原则

- **保持 SSH 会话连续**：所有步骤必须在同一个 SSH 会话中执行。如果会话断开重连，必须重新执行步骤 0（定义变量）和步骤 3 的变量读取部分
- **每步检查输出**：确认成功再继续下一步
- **失败时排障**：读取 `references/troubleshooting.md` 中对应的诊断命令，尝试修复后重试
- **不要跳步**：每一步都有依赖关系
- **步骤 10（安装 3X-UI）**是交互式的：安装器会要求输入用户名/密码/端口，随意输入即可，步骤 11 会通过数据库覆盖

#### 部署完成后输出

部署成功后，向用户展示以下信息：
- VLESS 客户端链接（完整一行）
- X-UI 面板的 SSH 隧道命令 + 用户名/密码
- 需要在 Cloudflare 控制台完成的配置（第三步）

---

### 第三步：Cloudflare 配置（部署后必做）

提醒用户在 Cloudflare 控制台完成以下配置：

| # | 配置项 | 操作 |
|---|--------|------|
| 1 | DNS A 记录 | `{SUBDOMAIN_PREFIX}` → 服务器 IP，启用橙色云（Proxied） |
| 2 | SSL/TLS 模式 | 设为 `Full (strict)` |
| 3 | 最低 TLS 版本 | 设为 `1.2` |
| 4 | WebSocket | 开启（Network 选项卡，XHTTP 经过 CF 时仍需此选项） |

---

## 客户端配置

### VLESS 链接格式

```
vless://{UUID}@{DOMAIN}:443?encryption=none&security=tls&sni={DOMAIN}&type=xhttp&host={DOMAIN}&path=%2F{WS_PATH}&fp=chrome#备注名
```

### 关键注意事项

1. **链接必须完整一行**，复制时不能有换行或空格——这是最常见的"连不上"原因
2. **端口必须是 443**，不是内部端口 10000
3. **必须有 `security=tls`、`sni` 和 `fp=chrome`**（fp 是 uTLS 指纹伪装，让 TLS 握手看起来像 Chrome 浏览器）
4. **面板导出的链接不能直接用**——端口、TLS、传输类型参数都是错的，必须按上面的格式自己拼

### 推荐客户端

| 平台 | 客户端 |
|------|--------|
| iOS | Shadowrocket, V2Box |
| Android | v2rayNG |
| Windows | v2rayN, Clash Verge |
| macOS | V2RayXS, Clash Verge |

---

## 运维场景

当用户的问题不是新部署，而是运维相关时，根据场景读取对应的参考文件：

| 场景 | 参考文件 |
|------|----------|
| 连不上、502、SSL 错误等 | 读取 `references/troubleshooting.md` |
| 添加客户端、证书续期、服务管理 | 读取 `references/maintenance.md` |
| 想了解手动部署的详细步骤 | 读取 `references/manual-deploy.md` |

---

## 核心踩坑提醒

部署和排障时始终牢记以下要点（详细说明见 `references/troubleshooting.md`）：

1. **xrayTemplateConfig 必须完整**——缺少 `outbounds` 会导致连上但无法上网
2. **3x-ui 默认暴露订阅端口**——必须手动禁用
3. **面板导出的 VLESS 链接参数不对**——必须手动拼链接
4. **添加用户用「添加客户端」**——不要新建入站
5. **改完面板配置要重启 x-ui**——config.json 可能不同步
6. **客户端链接复制最容易断行**——排障第一件事检查链接完整性
