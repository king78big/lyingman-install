#!/usr/bin/env bash

set -Eeuo pipefail

APP_DIR="/root/LyingMan"
SERVICE_NAME="lyingman"
NODE_VERSION="4.9.1"
WEB_PORT="81"
WS_PORT="8080"
SOURCE_REPO="https://github.com/linchuming/LyingMan.git"

echo "========================================"
echo "      LyingMan 狼人杀一键安装脚本"
echo "========================================"
echo

if [ "$(id -u)" -ne 0 ]; then
    echo "错误：请使用 root 用户运行此脚本。"
    exit 1
fi

echo "[1/9] 更新系统并安装基础组件..."

export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get install -y \
    git \
    curl \
    wget \
    ca-certificates \
    build-essential \
    nginx \
    openssl \
    python3

echo
echo "[2/9] 安装 NVM..."

export NVM_DIR="/root/.nvm"

if [ ! -s "$NVM_DIR/nvm.sh" ]; then
    curl -fsSL \
        https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh \
        | bash
fi

if [ -s "$NVM_DIR/nvm.sh" ]; then
    . "$NVM_DIR/nvm.sh"
else
    echo "错误：NVM 安装失败。"
    exit 1
fi

echo
echo "[3/9] 安装 Node.js ${NODE_VERSION}..."

if ! nvm ls "$NODE_VERSION" >/dev/null 2>&1; then
    nvm install "$NODE_VERSION"
fi

nvm alias default "$NODE_VERSION"
nvm use "$NODE_VERSION"

NODE_BIN="$(command -v node)"
NPM_BIN="$(command -v npm)"

echo
echo "Node.js: $($NODE_BIN -v)"
echo "npm:     $($NPM_BIN -v)"

echo
echo "[4/9] 下载 LyingMan..."

if [ -d "$APP_DIR" ]; then
    BACKUP_DIR="${APP_DIR}.backup.$(date +%Y%m%d_%H%M%S)"
    echo "发现旧目录，备份到：$BACKUP_DIR"
    mv "$APP_DIR" "$BACKUP_DIR"
fi

git clone "$SOURCE_REPO" "$APP_DIR"

if [ ! -f "$APP_DIR/server/server.js" ]; then
    echo "错误：没有找到 $APP_DIR/server/server.js"
    exit 1
fi

echo
echo "[5/9] 安装项目依赖..."

cd "$APP_DIR"

if [ -f "$APP_DIR/package.json" ]; then
    npm install --unsafe-perm
elif [ -f "$APP_DIR/server/package.json" ]; then
    cd "$APP_DIR/server"
    npm install --unsafe-perm
else
    echo "未发现 package.json，跳过 npm install。"
fi

echo
echo "[6/9] 创建 systemd 服务..."

cat > "/etc/systemd/system/${SERVICE_NAME}.service" <<EOF
[Unit]
Description=LyingMan Werewolf Game Server
After=network.target

[Service]
Type=simple
WorkingDirectory=${APP_DIR}
Environment=PATH=/root/.nvm/versions/node/v${NODE_VERSION}/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/root/.nvm/versions/node/v${NODE_VERSION}/bin/node ${APP_DIR}/server/server.js
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable "$SERVICE_NAME"
systemctl restart "$SERVICE_NAME"

sleep 3

echo
echo "检查 LyingMan 服务..."

if systemctl is-active --quiet "$SERVICE_NAME"; then
    echo "LyingMan 服务启动成功。"
else
    echo "警告：LyingMan 服务可能没有正常启动。"
    systemctl --no-pager -l status "$SERVICE_NAME" || true
fi

echo
echo "检查端口..."

if ss -lntp 2>/dev/null | grep -q ":${WEB_PORT} "; then
    echo "Web 服务端口 ${WEB_PORT} 正常。"
else
    echo "警告：没有检测到 ${WEB_PORT} 端口。"
fi

if ss -lntp 2>/dev/null | grep -q ":${WS_PORT} "; then
    echo "WebSocket 端口 ${WS_PORT} 正常。"
else
    echo "警告：没有检测到 ${WS_PORT} 端口。"
fi

echo
echo "========================================"
echo "请选择访问方式"
echo "========================================"
echo
echo "绑定域名：输入你的域名，例如："
echo "game.example.com"
echo
echo "不绑定域名：直接按 Enter"
echo

read -r -p "请输入绑定域名（直接回车使用 IP 模式）: " DOMAIN

echo

if [ -n "$DOMAIN" ]; then

    echo "========================================"
    echo "域名模式"
    echo "域名：$DOMAIN"
    echo "========================================"
    echo

    SERVER_IP="$(curl -4 -fsS https://api.ipify.org || true)"

    if [ -n "$SERVER_IP" ]; then
        echo "检测到服务器 IP：$SERVER_IP"
        echo
        echo "请确保："
        echo "$DOMAIN 已经解析到 $SERVER_IP"
        echo
    fi

    echo "[7/9] 配置 Nginx..."

    rm -f /etc/nginx/sites-enabled/default

    cat > /etc/nginx/sites-available/lyingman <<EOF
server {
    listen 80;
    listen [::]:80;

    server_name ${DOMAIN};

    location /ws/ {
        proxy_pass http://127.0.0.1:${WS_PORT}/;

        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    location / {
        proxy_pass http://127.0.0.1:${WEB_PORT};

        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

    ln -sf \
        /etc/nginx/sites-available/lyingman \
        /etc/nginx/sites-enabled/lyingman

    nginx -t
    systemctl enable nginx
    systemctl restart nginx

    echo
    echo "[8/9] 安装 HTTPS..."

    apt-get install -y certbot python3-certbot-nginx

    echo
    echo "正在申请 Let's Encrypt SSL 证书..."
    echo "如果域名 DNS 没有正确解析到本服务器，证书申请可能失败。"
    echo

    if certbot --nginx \
        -d "$DOMAIN" \
        --non-interactive \
        --agree-tos \
        --register-unsafely-without-email \
        --redirect; then

        echo
        echo "SSL 证书申请成功。"

    else

        echo
        echo "========================================"
        echo "SSL 证书申请失败"
        echo "========================================"
        echo
        echo "通常是因为："
        echo "1. 域名没有解析到本服务器"
        echo "2. DNS 还没有生效"
        echo "3. 80 端口无法从公网访问"
        echo
        echo "网站仍然可以通过 HTTP 访问："
        echo "http://${DOMAIN}"
        echo
    fi

    echo
    echo "[9/9] 最终检查 Nginx..."

    nginx -t
    systemctl restart nginx

    echo
    echo "========================================"
    echo "       LyingMan 安装完成"
    echo "========================================"
    echo

    if [ -f "/etc/letsencrypt/live/${DOMAIN}/fullchain.pem" ]; then
        echo "网站地址："
        echo "https://${DOMAIN}"
        echo
        echo "WebSocket："
        echo "wss://${DOMAIN}/ws/"
    else
        echo "网站地址："
        echo "http://${DOMAIN}"
        echo
        echo "WebSocket："
        echo "ws://${DOMAIN}/ws/"
    fi

else

    echo "========================================"
    echo "IP 模式"
    echo "========================================"

    echo
    echo "[7/9] 配置 Nginx..."

    rm -f /etc/nginx/sites-enabled/default

    cat > /etc/nginx/sites-available/lyingman <<EOF
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location /ws/ {
        proxy_pass http://127.0.0.1:${WS_PORT}/;

        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    location / {
        proxy_pass http://127.0.0.1:${WEB_PORT};

        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

    ln -sf \
        /etc/nginx/sites-available/lyingman \
        /etc/nginx/sites-enabled/lyingman

    nginx -t
    systemctl enable nginx
    systemctl restart nginx

    SERVER_IP="$(curl -4 -fsS https://api.ipify.org || true)"

    echo
    echo "[8/9] IP 模式不申请 SSL。"

    echo
    echo "[9/9] 最终检查..."

    nginx -t
    systemctl restart nginx

    echo
    echo "========================================"
    echo "       LyingMan 安装完成"
    echo "========================================"
    echo

    if [ -n "$SERVER_IP" ]; then
        echo "网站地址："
        echo "http://${SERVER_IP}"
        echo
        echo "WebSocket："
        echo "ws://${SERVER_IP}/ws/"
    else
        echo "请使用服务器公网 IP 访问。"
    fi

fi

echo
echo "========================================"
echo "常用管理命令"
echo "========================================"
echo
echo "查看状态："
echo "systemctl status ${SERVICE_NAME}"
echo
echo "重启游戏："
echo "systemctl restart ${SERVICE_NAME}"
echo
echo "查看日志："
echo "journalctl -u ${SERVICE_NAME} -f"
echo
echo "重启 Nginx："
echo "systemctl restart nginx"
echo
echo "========================================"
