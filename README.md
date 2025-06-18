#!/bin/bash

# ==========================
# WireGuard VPN 安装脚本
# 适用于 Ubuntu 20.04+/Debian
# ==========================

set -e

# 设置服务端信息
SERVER_PORT=51820
SERVER_IP=$(curl -s ifconfig.me)  # 或手动指定IP
WG_INTERFACE=wg0
WG_CONF="/etc/wireguard/${WG_INTERFACE}.conf"

# 生成密钥对
mkdir -p /etc/wireguard/keys
cd /etc/wireguard/keys

umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

SERVER_PRIV_KEY=$(<server_private.key)
SERVER_PUB_KEY=$(<server_public.key)
CLIENT_PRIV_KEY=$(<client_private.key)
CLIENT_PUB_KEY=$(<client_public.key)

# 安装 WireGuard
apt update
apt install -y wireguard iproute2 resolvconf

# 启用 IP 转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf
sysctl -p

# 创建 WireGuard 配置
cat > $WG_CONF <<EOF
[Interface]
PrivateKey = $SERVER_PRIV_KEY
Address = 10.0.0.1/24
ListenPort = $SERVER_PORT
SaveConfig = true

PostUp = ufw route allow in on $WG_INTERFACE out on eth0; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = ufw route delete allow in on $WG_INTERFACE out on eth0; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = $CLIENT_PUB_KEY
AllowedIPs = 10.0.0.2/32
EOF

chmod 600 $WG_CONF

# 启动服务
systemctl enable wg-quick@$WG_INTERFACE
systemctl start wg-quick@$WG_INTERFACE

# 输出客户端配置
cat > ~/client.conf <<EOF
[Interface]
PrivateKey = $CLIENT_PRIV_KEY
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = $SERVER_PUB_KEY
Endpoint = $SERVER_IP:$SERVER_PORT
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

echo -e "\n✅ WireGuard VPN 安装完成"
echo "📁 客户端配置文件位于: ~/client.conf"