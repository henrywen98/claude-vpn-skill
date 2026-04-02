# x-ui-deploy

一个 Claude Code Skill，通过 SSH 在 VPS 上自动部署 VLESS + XHTTP + TLS + Cloudflare CDN 架构的 VPN 服务。

## 架构

```
客户端 → Cloudflare CDN (443) → Nginx (TLS/反代) → Xray (127.0.0.1:10000) → Internet
```

| 组件 | 技术 |
|------|------|
| VPN 协议 | VLESS |
| 传输方式 | XHTTP over TLS |
| 管理面板 | 3X-UI (MHSanaei) |
| 反向代理 | Nginx |
| SSL 证书 | acme.sh + Cloudflare DNS 验证 |
| CDN | Cloudflare（隐藏真实 IP） |

## 为什么用 XHTTP 而不是 WebSocket

- XHTTP 是 Xray 官方推荐替代 WebSocket 的传输协议
- 流量分片成多个短 HTTP 请求，更像正常网页浏览，抗 GFW 检测更强
- 弱网环境下更稳定（某个请求断了不影响整体）
- 支持 HTTP/3 (QUIC)
- 客户端链接包含 `fp=chrome`（uTLS 指纹伪装），TLS 握手看起来像 Chrome 浏览器

## 使用前准备

| 项目 | 说明 |
|------|------|
| VPS 服务器 | Debian 11/12 或 Ubuntu 20.04+ |
| 域名 | 已添加到 Cloudflare |
| Cloudflare API | Global API Key 或 API Token |
| 邮箱 | 用于 SSL 证书注册 |

## 使用方式

### 安装 Skill

将 `x-ui-vpn-skill/skills/x-ui-deploy/` 目录复制到你的 Claude Code skills 路径下。

### 触发

对 Claude 说以下任意一种：

- "帮我部署一个 VPN"
- "搭建 x-ui 代理"
- "在服务器上配置 VLESS"
- "VPN 连不上了"（运维场景）
- "帮我加一个 VPN 用户"（运维场景）

### 工作流程

1. **收集信息** — Claude 分 3 轮提问，收集服务器 IP、域名、Cloudflare 凭据等
2. **SSH 部署** — Claude 连接到你的 VPS，按 15 个步骤逐条执行命令完成部署
3. **Cloudflare 配置** — Claude 提醒你在 Cloudflare 控制台完成 DNS 和 SSL 设置

部署完成后，你会得到：
- 可直接导入客户端的 VLESS 链接
- X-UI 管理面板的 SSH 隧道命令和登录凭据

## 文件结构

```
skills/x-ui-deploy/
├── SKILL.md                      # 核心工作流（Claude 的行为指令）
└── references/
    ├── manual-deploy.md           # 部署执行蓝图（15 步完整命令）
    ├── troubleshooting.md         # 故障排查 + 踩坑经验
    └── maintenance.md             # 日常运维 + 安全加固
```

## 安全特性

- UFW 防火墙仅开放 SSH/80/443
- X-UI 面板仅 localhost 访问（通过 SSH 隧道使用）
- 3X-UI 默认订阅端口已禁用
- Fail2Ban 防暴力破解（SSH 3 次失败封禁 2 小时）
- Nginx 隐藏版本号 + HSTS + 安全头部
- TLSv1/TLSv1.1 已禁用
- Cloudflare 隐藏真实服务器 IP
- 伪装站点（看起来像正规企业官网）

## 推荐客户端

| 平台 | 客户端 |
|------|--------|
| iOS | Shadowrocket, V2Box |
| Android | v2rayNG |
| Windows | v2rayN, Clash Verge |
| macOS | V2RayXS, Clash Verge |

## 适用系统

- Debian 11/12
- Ubuntu 20.04+

## 许可证

MIT
