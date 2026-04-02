# 部署执行蓝图

Claude 按此文件的步骤逐条在服务器上执行命令。

## 步骤 0：定义变量

在服务器上执行任何操作前，先定义以下变量。用户提供的值在第一步收集阶段已获取。

```bash
# === 用户提供的值（替换为实际值）===
ROOT_DOMAIN="example.com"
SUBDOMAIN_PREFIX="node1"
EMAIL="you@email.com"
SSH_PORT="22"

# Cloudflare 认证（二选一）
# 方式一：Global API Key
CF_Key="your_global_api_key"
CF_Email="your_cf_email"
# 方式二：API Token
# CF_Token="your_api_token"

# === 自动拼接 ===
DOMAIN="${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}"
XUI_PORT="54321"
```

后续所有命令直接使用这些变量，无需额外替换。

---

## 步骤 1：检查系统环境

```bash
whoami  # 必须是 root
cat /etc/os-release | head -3  # 必须是 Debian/Ubuntu
ss -tlnp | grep -E ':80 |:443 '  # 检查端口占用
```

如果 80/443 已被占用，提醒用户现有服务可能被覆盖，确认后再继续。

## 步骤 2：安装依赖

```bash
apt update && apt install -y nginx ufw fail2ban curl wget openssl sqlite3 socat cron python3-bcrypt
```

> `python3-bcrypt` 用于后续生成面板密码哈希。如果包不存在，后续会 fallback 到 `pip3 install bcrypt`。

## 步骤 3：生成安全参数

```bash
mkdir -p /root/.secrets && chmod 700 /root/.secrets

# XHTTP 路径（16 字符 hex，短路径容易被扫描发现）
openssl rand -hex 8 > /root/.secrets/ws_path.txt

# 客户端 UUID
cat /proc/sys/kernel/random/uuid > /root/.secrets/vless_uuid.txt

# X-UI 面板凭据
echo "admin_$(openssl rand -hex 4)" > /root/.secrets/xui_username.txt
openssl rand -base64 18 > /root/.secrets/xui_password.txt

chmod 600 /root/.secrets/*.txt
```

读取生成的值，后续步骤需要：

```bash
WS_PATH=$(cat /root/.secrets/ws_path.txt)
UUID=$(cat /root/.secrets/vless_uuid.txt)
XUI_USER=$(cat /root/.secrets/xui_username.txt)
XUI_PASS=$(cat /root/.secrets/xui_password.txt)

echo "WS_PATH: $WS_PATH"
echo "UUID: $UUID"
echo "XUI_USER: $XUI_USER"
echo "XUI_PASS: $XUI_PASS"
```

## 步骤 4：安装 acme.sh 并申请 SSL 证书

```bash
curl https://get.acme.sh | sh -s email="$EMAIL"
```

设置 Cloudflare API（根据用户选择的认证方式，变量在步骤 0 已定义）：

```bash
# 方式一已通过 CF_Key 和 CF_Email 导出
export CF_Key CF_Email
# 方式二取消注释：export CF_Token
```

申请并安装证书：

```bash
~/.acme.sh/acme.sh --issue -d "$ROOT_DOMAIN" -d "*.$ROOT_DOMAIN" --dns dns_cf --keylength ec-256

mkdir -p /root/cert
~/.acme.sh/acme.sh --install-cert -d "$ROOT_DOMAIN" \
  --cert-file "/root/cert/$ROOT_DOMAIN.cer" \
  --key-file "/root/cert/$ROOT_DOMAIN.key" \
  --fullchain-file /root/cert/fullchain.cer \
  --ca-file /root/cert/ca.cer \
  --reloadcmd "systemctl reload nginx"

chmod 600 /root/cert/*.key
```

> 如果证书申请失败，通常是 Cloudflare API 凭据不对或权限不足。让用户重新检查。

## 步骤 5：创建伪装站点

```bash
mkdir -p "/var/www/$DOMAIN"
cat > "/var/www/$DOMAIN/index.html" << 'SITEEOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CloudPeak Systems</title>
    <meta name="description" content="Professional digital solutions and consulting services.">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; color: #1a1a2e; line-height: 1.6; }
        .nav { background: #fff; padding: 1rem 2rem; border-bottom: 1px solid #e8e8e8; display: flex; justify-content: space-between; align-items: center; }
        .nav-brand { font-size: 1.3rem; font-weight: 600; color: #2563eb; text-decoration: none; }
        .nav-links { display: flex; gap: 2rem; list-style: none; }
        .nav-links a { text-decoration: none; color: #555; font-size: 0.95rem; }
        .hero { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 5rem 2rem; text-align: center; }
        .hero h1 { font-size: 2.5rem; font-weight: 700; margin-bottom: 1rem; }
        .hero p { font-size: 1.2rem; opacity: 0.9; max-width: 600px; margin: 0 auto 2rem; }
        .btn { display: inline-block; background: #fff; color: #764ba2; padding: 0.8rem 2rem; border-radius: 6px; text-decoration: none; font-weight: 600; }
        .features { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 2rem; padding: 4rem 2rem; max-width: 1100px; margin: 0 auto; }
        .feature { text-align: center; padding: 2rem; }
        .feature h3 { margin: 1rem 0 0.5rem; font-size: 1.2rem; }
        .feature p { color: #666; font-size: 0.95rem; }
        .icon { font-size: 2.5rem; }
        .footer { background: #f8f9fa; padding: 2rem; text-align: center; color: #888; font-size: 0.85rem; border-top: 1px solid #e8e8e8; }
    </style>
</head>
<body>
    <nav class="nav">
        <a href="#" class="nav-brand">CloudPeak Systems</a>
        <ul class="nav-links">
            <li><a href="#">Services</a></li>
            <li><a href="#">About</a></li>
            <li><a href="#">Contact</a></li>
        </ul>
    </nav>
    <section class="hero">
        <h1>Digital Solutions for Modern Business</h1>
        <p>We help companies transform their operations with cutting-edge technology and data-driven strategies.</p>
        <a href="#" class="btn">Learn More</a>
    </section>
    <section class="features">
        <div class="feature"><div class="icon">&#9729;</div><h3>Cloud Infrastructure</h3><p>Scalable and secure cloud solutions tailored to your business needs.</p></div>
        <div class="feature"><div class="icon">&#9881;</div><h3>Process Automation</h3><p>Streamline workflows and reduce operational costs with smart automation.</p></div>
        <div class="feature"><div class="icon">&#128200;</div><h3>Data Analytics</h3><p>Turn your data into actionable insights with advanced analytics platforms.</p></div>
    </section>
    <footer class="footer">&copy; 2026 CloudPeak Systems. All rights reserved.</footer>
</body>
</html>
SITEEOF
```

## 步骤 6：加固 Nginx 全局配置

```bash
sed -i 's/ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;/ssl_protocols TLSv1.2 TLSv1.3;/' /etc/nginx/nginx.conf
sed -i 's/# server_tokens off;/server_tokens off;/' /etc/nginx/nginx.conf
```

## 步骤 7：配置 Nginx 站点

```bash
cat > "/etc/nginx/sites-available/$DOMAIN" << NGINXEOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;
    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate /root/cert/fullchain.cer;
    ssl_certificate_key /root/cert/$ROOT_DOMAIN.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    server_tokens off;
    root /var/www/$DOMAIN;
    index index.html;

    # XHTTP 代理 → Xray
    location /$WS_PATH {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;

        # XHTTP mode=auto 时也需要支持 WebSocket upgrade
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        # 禁用缓冲，确保流式传输
        proxy_buffering off;
        proxy_cache off;
    }

    location / { try_files \$uri \$uri/ =404; }
    location ~ /\. { deny all; }
}
NGINXEOF
```

> heredoc 不带引号，`$DOMAIN`、`$ROOT_DOMAIN`、`$WS_PATH` 会被 bash 展开为实际值。Nginx 变量（`$server_name`、`$host` 等）用 `\$` 转义以保留。

启用配置：

```bash
ln -sf "/etc/nginx/sites-available/$DOMAIN" /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

## 步骤 8：配置防火墙

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow "$SSH_PORT/tcp"
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny "$XUI_PORT/tcp"
echo "y" | ufw enable
```

## 步骤 9：配置 Fail2Ban

```bash
cat > /etc/fail2ban/jail.local << JAILEOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port = $SSH_PORT
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200
JAILEOF

systemctl enable fail2ban && systemctl restart fail2ban
```

## 步骤 10：安装 3X-UI

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

> **交互处理**：安装器会依次要求输入：
> 1. 是否继续安装 → 输入 `y`
> 2. 用户名 → 随意输入（如 `temp`），后续会通过数据库覆盖
> 3. 密码 → 随意输入（如 `temp`），后续会通过数据库覆盖
> 4. 端口 → 输入 `54321`
>
> 使用 **MHSanaei/3x-ui**，不要用已停维的 vaxilu/x-ui。

等待安装完成，确认数据库文件存在：

```bash
ls -la /etc/x-ui/x-ui.db
```

## 步骤 11：配置 3X-UI 面板

```bash
systemctl stop x-ui

# 面板仅 localhost 访问
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webListen', '127.0.0.1');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webPort', '$XUI_PORT');"

# 更新面板凭据（bcrypt 密码哈希）
# 确保 bcrypt 模块可用
python3 -c "import bcrypt" 2>/dev/null || pip3 install bcrypt -q
# 注意：openssl rand -base64 生成的密码仅含 A-Za-z0-9+/= 字符，不含单引号，此处安全
HASHED=$(python3 -c "
import bcrypt
password = '$XUI_PASS'
print(bcrypt.hashpw(password.encode(), bcrypt.gensalt(10)).decode())
")
sqlite3 /etc/x-ui/x-ui.db "UPDATE users SET username='$XUI_USER', password='$HASHED' WHERE id=1;"

# 生成新 secret
NEW_SECRET=$(openssl rand -hex 16)
sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='$NEW_SECRET' WHERE key='secret';"

# 禁用订阅端口（默认会在公网暴露！）
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subEnable', 'false');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subListen', '127.0.0.1');"
```

## 步骤 12：设置 Xray 模板

模板必须完整，否则 outbounds 为 null 导致流量无法出站：

```bash
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('xrayTemplateConfig', '{
  \"log\": {\"access\": \"none\", \"dnsLog\": false, \"loglevel\": \"warning\"},
  \"dns\": {\"servers\": [\"8.8.8.8\", \"1.1.1.1\"]},
  \"routing\": {
    \"domainStrategy\": \"IPIfNonMatch\",
    \"rules\": [
      {\"type\": \"field\", \"inboundTag\": [\"api\"], \"outboundTag\": \"api\"},
      {\"type\": \"field\", \"outboundTag\": \"blocked\", \"protocol\": [\"bittorrent\"]}
    ]
  },
  \"outbounds\": [
    {\"tag\": \"direct\", \"protocol\": \"freedom\", \"settings\": {}},
    {\"tag\": \"blocked\", \"protocol\": \"blackhole\", \"settings\": {}}
  ],
  \"policy\": {\"levels\": {\"0\": {\"statsUserDownlink\": true, \"statsUserUplink\": true}}, \"system\": {\"statsInboundDownlink\": true, \"statsInboundUplink\": true}},
  \"api\": {\"tag\": \"api\", \"services\": [\"HandlerService\", \"LoggerService\", \"StatsService\"]},
  \"stats\": {}
}');"
```

## 步骤 13：创建 VLESS 入站

```bash
SETTINGS="{\"clients\":[{\"id\":\"$UUID\",\"flow\":\"\",\"email\":\"default-user\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"\",\"reset\":0}],\"decryption\":\"none\",\"fallbacks\":[]}"
STREAM="{\"network\":\"xhttp\",\"security\":\"none\",\"xhttpSettings\":{\"path\":\"/$WS_PATH\",\"host\":\"$DOMAIN\",\"mode\":\"auto\"}}"
SNIFFING="{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\"],\"metadataOnly\":false,\"routeOnly\":true}"

sqlite3 /etc/x-ui/x-ui.db "INSERT INTO inbounds (user_id, up, down, total, all_time, remark, enable, expiry_time, listen, port, protocol, settings, stream_settings, tag, sniffing) VALUES (1, 0, 0, 0, 0, 'VLESS-XHTTP-TLS-CF', 1, 0, '127.0.0.1', 10000, 'vless', '$SETTINGS', '$STREAM', 'inbound-10000', '$SNIFFING');"

systemctl start x-ui
sleep 3
```

## 步骤 14：健康检查

逐项检查，全部通过才算部署成功：

```bash
# 服务状态
systemctl is-active nginx x-ui fail2ban

# 端口监听
ss -tlnp | grep -E ':80 |:443 |:10000 |:54321 '

# Xray 监听在 localhost
ss -tlnp | grep '127.0.0.1:10000'

# X-UI 面板仅 localhost
ss -tlnp | grep "127.0.0.1:$XUI_PORT"

# Xray outbounds 不为 null
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
"

# 防火墙状态
ufw status

# SSL 证书有效期
openssl x509 -in /root/cert/fullchain.cer -noout -enddate
```

如果任一项不通过，参考 `troubleshooting.md` 对应的诊断和修复方法。

## 步骤 15：保存配置并输出结果

```bash
SERVER_IP=$(curl -s --max-time 10 ifconfig.me)

VLESS_LINK="vless://${UUID}@${DOMAIN}:443?encryption=none&security=tls&sni=${DOMAIN}&type=xhttp&host=${DOMAIN}&path=%2F${WS_PATH}&fp=chrome#VLESS-XHTTP-TLS-CF"

cat > /root/vpn-config.txt << CFGEOF
========================================
  VPN Configuration - $DOMAIN
  Generated: $(date -u '+%Y-%m-%d %H:%M:%S UTC')
========================================

## Architecture
Client → Cloudflare CDN (443) → Nginx (TLS) → Xray (127.0.0.1:10000) → Internet

## Domain
  Domain:     $DOMAIN
  Server IP:  $SERVER_IP (hidden behind Cloudflare)

## VLESS Client Link (copy as ONE LINE, no line breaks!)
$VLESS_LINK

## Manual Client Config
  Protocol:   VLESS
  Address:    $DOMAIN
  Port:       443 (NOT 10000)
  UUID:       $UUID
  Encryption: none
  Transport:  XHTTP
  Path:       /$WS_PATH
  Host:       $DOMAIN
  TLS:        Enabled (MUST be on)
  SNI:        $DOMAIN (MUST be set)
  Fingerprint: chrome

## X-UI Panel Access (SSH tunnel ONLY)
  SSH Tunnel: ssh -L $XUI_PORT:localhost:$XUI_PORT root@$SERVER_IP
  URL:        http://localhost:$XUI_PORT
  Username:   $XUI_USER
  Password:   $XUI_PASS

## Adding New Clients
  Do NOT create new inbounds! Add clients to existing inbound:
  Panel → Inbound list → VLESS-XHTTP-TLS-CF → "+" → Add Client
  Then generate link using this template:
  vless://<UUID>@$DOMAIN:443?encryption=none&security=tls&sni=$DOMAIN&type=xhttp&host=$DOMAIN&path=%2F$WS_PATH&fp=chrome#<NAME>
  WARNING: Panel-exported links have WRONG port, TLS, and transport settings!

## Security
  - UFW: only $SSH_PORT/80/443 open
  - X-UI: localhost only (127.0.0.1:$XUI_PORT)
  - Xray: localhost only (127.0.0.1:10000)
  - Fail2Ban: SSH $SSH_PORT, 3 attempts → 2hr ban
  - Subscription port: disabled

## Key Files
  - Secrets:     /root/.secrets/
  - Nginx Site:  /etc/nginx/sites-available/$DOMAIN
  - Fail2Ban:    /etc/fail2ban/jail.local
  - X-UI DB:     /etc/x-ui/x-ui.db
  - Xray Config: /usr/local/x-ui/bin/config.json (auto-generated, don't edit)
  - Certs:       /root/cert/
CFGEOF

chmod 600 /root/vpn-config.txt
cat /root/vpn-config.txt
```

将输出中的 VLESS 链接、面板凭据、SSH 隧道命令整理后展示给用户。
