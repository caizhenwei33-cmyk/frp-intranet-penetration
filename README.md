# FRP 内网穿透完整教程

> **FRP (Fast Reverse Proxy)** — 一款高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 协议，用于将内网服务通过公网服务器暴露到互联网。

---

## 📖 目录 / Table of Contents

- [原理讲解 / How It Works](#-原理讲解--how-it-works)
- [架构概览 / Architecture Overview](#-架构概览--architecture-overview)
- [快速开始 / Quick Start](#-快速开始--quick-start)
- [Docker 部署 / Docker Deployment](#-docker-部署--docker-deployment)
- [原生部署 / Native Deployment](#-原生部署--native-deployment)
- [配置详解 / Configuration Reference](#-配置详解--configuration-reference)
- [常用命令 / Common Commands](#-常用命令--common-commands)
- [高级用法 / Advanced Usage](#-高级用法--advanced-usage)
- [常见问题 / FAQ](#-常见问题--faq)
- [安全建议 / Security Best Practices](#-安全建议--security-best-practices)
- [☕ 支持 / Support](#-支持--support)

---

## 🔬 原理讲解 / How It Works

### 中文说明

内网穿透（Intranet Penetration）是指将处于内网（NAT / 防火墙后方）的服务，通过一台具有公网 IP 的服务器暴露到公网的技术。

FRP 采用 **C/S 架构**（Client/Server）：

1. **服务端 (frps)**：部署在公网服务器上，监听一个控制端口和一个或多个代理端口。
2. **客户端 (frpc)**：部署在内网目标机器上，主动连接到服务端，并注册需要暴露的服务。
3. 客户端与服务端之间建立**长连接**（基于 TCP），公网用户访问服务端的代理端口时，流量通过这个长连接隧道转发到内网服务。

**工作流程：**

```
公网用户 ──→ frps (公网服务器:7000) ──→ frpc (内网:192.168.1.100:80)
```

### English Description

FRP (Fast Reverse Proxy) is a high-performance reverse proxy application that supports TCP, UDP, HTTP, and HTTPS protocols. It is designed to expose services behind NAT or firewalls to the public internet through a server with a public IP address.

FRP uses a **Client/Server architecture**:

1. **Server (frps)**: Runs on a public server, listens on a control port and proxy ports.
2. **Client (frpc)**: Runs on the internal machine, actively connects to the server, and registers services to be exposed.
3. A **persistent TCP tunnel** is established between client and server. Public users access services through this tunnel.

**Data Flow:**

```
Internet User ──→ frps (Public Server:7000) ──→ frpc (Internal:192.168.1.100:80)
```

---

## 🏗 架构概览 / Architecture Overview

### 组件 / Components

| 组件 Component | 部署位置 Location | 端口 Port | 作用 Purpose |
|---------------|-------------------|-----------|-------------|
| frps          | 公网服务器         | 7000 (控制) / 自定义 | 接收 frpc 连接，转发公网请求 |
| frpc          | 内网机器           | 无（主动连接） | 注册内网服务，维护隧道 |
| 公网用户       | 互联网             | —          | 通过 frps 的代理端口访问内网服务 |

### 支持的协议 / Supported Protocols

| 协议 Protocol | 用途 Use Case | 示例 Example |
|--------------|---------------|-------------|
| TCP          | 任意 TCP 服务 | SSH、数据库、RDP |
| UDP          | DNS、游戏服务 | 53 端口转发 |
| HTTP         | Web 服务      | Nginx、Tomcat |
| HTTPS        | 安全 Web 服务 | 带证书的 Web 应用 |
| STCP         | 加密 TCP     | 安全的点对点通信 |

---

## ⚡ 快速开始 / Quick Start

### 前提条件 / Prerequisites

- 一台具有公网 IP 的 Linux 服务器（VPS）
- 一台内网机器（可以是树莓派、NAS 或普通 PC）
- 两者均可访问 GitHub Releases 下载 FRP 二进制文件

### 下载 / Download

```bash
# 查看最新版本：https://github.com/fatedier/frp/releases
export FRP_VERSION=0.61.2

# Linux amd64
wget https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_amd64.tar.gz
tar -xzf frp_${FRP_VERSION}_linux_amd64.tar.gz
cd frp_${FRP_VERSION}_linux_amd64

# Linux arm64 (树莓派、Oracle ARM)
wget https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_arm64.tar.gz
tar -xzf frp_${FRP_VERSION}_linux_arm64.tar.gz
cd frp_${FRP_VERSION}_linux_arm64
```

### 文件结构 / File Structure

```
frp_${VERSION}_linux_amd64/
├── frps           # 服务端二进制
├── frpc           # 客户端二进制
├── frps.toml      # 服务端配置
├── frpc.toml      # 客户端配置
├── LICENSE
└── README.md
```

---

## 🐳 Docker 部署 / Docker Deployment

### 服务端 (frps) — Docker Compose

创建 `docker-compose-frps.yml`：

```yaml
version: "3.8"

services:
  frps:
    image: snowdreamtech/frps:latest
    container_name: frps
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frps.toml:/etc/frp/frps.toml:ro
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

创建 `frps.toml`：

```toml
[common]
bind_port = 7000
bind_addr = "0.0.0.0"

# 控制面板（可选）
web_server.addr = "0.0.0.0"
web_server.port = 7500
web_server.user = "admin"
web_server.password = "your_strong_password"

# 身份验证
auth.token = "your_secure_token_here"

# 日志
log.to = "./frps.log"
log.level = "info"
log.max_days = 7
```

启动：

```bash
docker compose -f docker-compose-frps.yml up -d
```

### 服务端 (frps) — Docker CLI

```bash
docker run -d --name frps --restart unless-stopped --network host \
  -v /opt/frp/frps.toml:/etc/frp/frps.toml:ro \
  snowdreamtech/frps:latest
```

### 客户端 (frpc) — Docker Compose

创建 `docker-compose-frpc.yml`：

```yaml
version: "3.8"

services:
  frpc:
    image: snowdreamtech/frpc:latest
    container_name: frpc
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

创建 `frpc.toml`：

```toml
[common]
server_addr = "your-server-ip-or-domain"
server_port = 7000
auth.token = "your_secure_token_here"

# 暴露 SSH 服务（TCP 代理）
[ssh]
type = "tcp"
local_ip = "127.0.0.1"
local_port = 22
remote_port = 6000

# 暴露 Web 服务（HTTP 代理）
[web]
type = "http"
local_ip = "127.0.0.1"
local_port = 80
custom_domains = "your-domain.com"
```

启动：

```bash
docker compose -f docker-compose-frpc.yml up -d
```

### 客户端 (frpc) — Docker CLI

```bash
docker run -d --name frpc --restart unless-stopped --network host \
  -v /opt/frp/frpc.toml:/etc/frp/frpc.toml:ro \
  snowdreamtech/frpc:latest
```

---

## 📦 原生部署 / Native Deployment

### 服务端部署 / Server Setup

```bash
# 解压并安装
tar -xzf frp_*_linux_*.tar.gz
cd frp_*_linux_*

# 修改配置
vim frps.toml

# 启动
./frps -c frps.toml

# 后台运行（使用 systemd）
sudo cp frps /usr/local/bin/
sudo mkdir -p /etc/frp
sudo cp frps.toml /etc/frp/

# 创建 systemd 服务
sudo tee /etc/systemd/system/frps.service <<EOF
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
User=nobody
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
sudo systemctl status frps
```

### 客户端部署 / Client Setup

```bash
# 同样解压，复制
tar -xzf frp_*_linux_*.tar.gz
cd frp_*_linux_*

vim frpc.toml

# 测试运行
./frpc -c frpc.toml

# systemd 服务
sudo cp frpc /usr/local/bin/
sudo mkdir -p /etc/frp
sudo cp frpc.toml /etc/frp/

sudo tee /etc/systemd/system/frpc.service <<EOF
[Unit]
Description=FRP Client
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=nobody
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable frpc
sudo systemctl start frpc
sudo systemctl status frpc
```

---

## 📝 配置详解 / Configuration Reference

### 完整服务端配置 / Full Server Config (frps.toml)

```toml
[common]
# 基础配置
bind_addr = "0.0.0.0"                    # 监听地址
bind_port = 7000                          # 控制端口（frpc 连接端口）

# KCP 配置（可选，UDP 加速）
kcp_bind_port = 7000                      # KCP 绑定端口（通常用同一端口）
kcp_connect_timeout = 30                  # KCP 连接超时（秒）

# QUIC 配置（v0.52.0+）
quic_bind_port = 7001                     # QUIC 监听端口
quic_keepalive_period = 10                # QUIC 保活间隔
quic_max_idle_timeout = 30                # QUIC 最大空闲超时
quic_initial_rtt = 0.02                   # QUIC 初始 RTT

# HTTP 代理配置
vhost_http_port = 8080                    # HTTP 代理端口
vhost_https_port = 443                    # HTTPS 代理端口

# Web 控制面板
web_server.addr = "0.0.0.0"
web_server.port = 7500
web_server.user = "admin"
web_server.password = "change_me_please"

# 身份验证
auth.token = "your_very_secure_token_abcdef123456"

# TLS 加密
tls.force = false                         # 是否强制 TLS
tls.cert_file = "/etc/frp/server.crt"
tls.key_file = "/etc/frp/server.key"

# 日志配置
log.to = "/var/log/frps.log"
log.level = "info"                        # trace, debug, info, warn, error
log.max_days = 7

# 性能限制
max_pool_count = 50                       # 最大连接池数
max_ports_per_client = 0                  # 每个客户端最大端口数（0=无限制）
allow_ports = "2000-3000,3001,3002,4000-50000"
```

### 完整客户端配置 / Full Client Config (frpc.toml)

```toml
[common]
server_addr = "frp.example.com"
server_port = 7000
auth.token = "your_very_secure_token_abcdef123456"

# 协议（支持 tcp, kcp, websocket, quic）
protocol = "tcp"

# TLS 配置
tls.enable = true
tls.trusted_ca_file = "/etc/frp/ca.crt"
tls.server_name = "frp.example.com"

# 日志
log.to = "/var/log/frpc.log"
log.level = "info"
log.max_days = 3

# SSH 转发
[ssh]
type = "tcp"
local_ip = "127.0.0.1"
local_port = 22
remote_port = 6000
use_compression = true                    # 启用压缩
use_encryption = true                     # 启用加密

# HTTP Web 服务
[web]
type = "http"
local_ip = "127.0.0.1"
local_port = 8080
custom_domains = ["web.example.com", "app.example.com"]
host_header_rewrite = "127.0.0.1"         # 重写 Host 头
use_compression = true

# HTTPS Web 服务
[web-https]
type = "https"
local_ip = "127.0.0.1"
local_port = 443
custom_domains = "secure.example.com"
plugin = "https2http"                     # HTTPS 卸载到 HTTP
plugin_local_addr = "127.0.0.1:80"
plugin_crt_path = "/etc/frp/server.crt"
plugin_key_path = "/etc/frp/server.key"

# RDP（Windows 远程桌面）
[rdp]
type = "tcp"
local_ip = "192.168.1.50"
local_port = 3389
remote_port = 3389

# Samba 文件共享
[samba]
type = "tcp"
local_ip = "192.168.1.100"
local_port = 445
remote_port = 4450

# UDP 服务（DNS 转发示例）
[dns]
type = "udp"
local_ip = "192.168.1.1"
local_port = 53
remote_port = 5300
use_encryption = true
use_compression = true

# 访问控制（白名单 IP）
[restricted-service]
type = "tcp"
local_ip = "127.0.0.1"
local_port = 9000
remote_port = 9000
allow_users = ["trusted_user1", "trusted_user2"]
```

---

## 🔧 常用命令 / Common Commands

### 服务端 / Server

```bash
# 启动（前台）
./frps -c frps.toml

# 检查配置
./frps verify -c frps.toml

# 查看帮助
./frps --help

# 查看版本
./frps --version

# 重载配置（SIGHUP）
kill -HUP $(pgrep frps)

# 查看日志
tail -f /var/log/frps.log

# 查看连接状态
ss -tnp | grep frps
```

### 客户端 / Client

```bash
# 启动（前台，观察日志）
./frpc -c frpc.toml

# 检查配置
./frpc verify -c frpc.toml

# 仅显示代理状态（不启动）
./frpc status -c frpc.toml

# 重载配置（热更新，仅新增/删除代理）
./frpc reload -c frpc.toml

# 查看帮助
./frpc --help

# 查看日志
tail -f /var/log/frpc.log

# 测试连通性
nc -zv your-server-ip 7000
```

### Docker 管理 / Docker Management

```bash
# 查看服务端日志
docker logs -f frps

# 查看客户端日志
docker logs -f frpc

# 重启客户端
docker restart frpc

# 更新镜像
docker compose pull frpc
docker compose up -d frpc
```

---

## 🚀 高级用法 / Advanced Usage

### 1. HTTP 域名复用 / HTTP Virtual Hosting

通过 frps 的 HTTP 代理功能，多个域名共用同一端口 (8080)：

```toml
# frpc — 服务 A
[web-app-a]
type = "http"
local_port = 8080
custom_domains = "app-a.example.com"

# frpc — 服务 B
[web-app-b]
type = "http"
local_port = 3000
custom_domains = "app-b.example.com"
```

### 2. STCP — 安全点对点隧道 / Secure P2P Tunnel

STCP 提供端到端加密，只有持有正确 `secret_key` 的客户端才能连接：

```toml
# 服务端（暴露者）
[secret_ssh]
type = "stcp"
sk = "my_secret_key_123"
local_ip = "127.0.0.1"
local_port = 22

# 访问者（消费者）
[secret_ssh_visitor]
type = "stcp"
role = "visitor"
server_name = "secret_ssh"
sk = "my_secret_key_123"
bind_addr = "127.0.0.1"
bind_port = 2222
```

访问方连接 `127.0.0.1:2222` 即可 SSH 到暴露方的机器。

### 3. XTCP — 点对点直连 / P2P Direct Connection

XTCP 尝试 NAT 穿透建立直连（类似 P2P），不经过服务端转发流量：

```toml
# 暴露者
[p2p_ssh]
type = "xtcp"
sk = "p2p_secret"
local_ip = "127.0.0.1"
local_port = 22

# 访问者
[p2p_ssh_visitor]
type = "xtcp"
role = "visitor"
server_name = "p2p_ssh"
sk = "p2p_secret"
bind_addr = "127.0.0.1"
bind_port = 2222
```

### 4. 多客户端/多服务 / Multiple Clients & Services

```toml
# frpc — 家庭 NAS
[nas-ssh]
type = "tcp"
local_ip = "192.168.1.100"
local_port = 22
remote_port = 6100

[nas-web]
type = "http"
local_ip = "192.168.1.100"
local_port = 5000
custom_domains = "nas.example.com"

# frpc — 办公室电脑
[office-rdp]
type = "tcp"
local_ip = "10.0.0.50"
local_port = 3389
remote_port = 6200
```

### 5. 负载均衡 / Load Balancing

```toml
[web-group]
type = "tcp"
local_ip = "127.0.0.1"
local_port = 8080
remote_port = 8080
group = "web-group"
group_key = "group_key_123"
# 第二个 frpc 实例配置同样的 group 和 group_key
# frps 会自动在它们之间分配请求
```

### 6. 使用 KCP 协议加速 / KCP Protocol Acceleration

在 UDP 不稳定环境下，KCP 可以显著提升隧道性能：

```toml
[common]
# 客户端配置
protocol = "kcp"
```

### 7. 多级代理 / Multi-level Proxy

```toml
# 中间节点（跳板机）作为 frpc 连接 frps
# 再在中间节点上运行第二个 frpc 连接内网
# 适用于多层 NAT 环境
```

### 8. 使用 Dashboard 监控 / Dashboard Monitoring

访问 `http://your-server-ip:7500` 查看：

| 指标 Metric | 说明 Description |
|-------------|-----------------|
| 代理数量     | 当前在线代理数 |
| 流量统计     | 入站/出站流量 |
| 客户端列表   | 已连接客户端 |
| 代理状态     | 在线/离线 |
| 错误日志     | 最近错误信息 |

---

## ❓ 常见问题 / FAQ

### Q1: 客户端无法连接到服务端

**可能原因：**
- 服务端防火墙未放行端口
- 认证 Token 不匹配
- 网络不通（ping 测试）

**解决方案：**
```bash
# 服务端放行端口
sudo firewall-cmd --add-port=7000/tcp --permanent
sudo firewall-cmd --reload

# 或者 iptables
sudo iptables -A INPUT -p tcp --dport 7000 -j ACCEPT

# 测试连通性
telnet your-server-ip 7000
```

### Q2: 连接成功但无法访问内网服务

**可能原因：**
- 本地服务未启动
- local_ip/local_port 配置错误
- 远程端口被占用

**排查：**
```bash
# 检查本地服务是否监听
ss -tlnp | grep 80

# 检查 frps 日志
tail -f /var/log/frps.log

# 检查 frpc 日志
tail -f /var/log/frpc.log
```

### Q3: 性能问题 / 延迟高

**优化建议：**
```toml
# 启用压缩
use_compression = true

# 使用 KCP 协议（UDP 环境更优）
protocol = "kcp"

# 增加连接池
pool_count = 10
```

### Q4: HTTP 域名配置不生效

**检查：**
```toml
# 确保 custom_domains 已配置 DNS A 记录指向你的服务器 IP
# 确保 frps 的 vhost_http_port 已配置
[common]
vhost_http_port = 8080

# 如果使用 HTTPS，还需要配置 vhost_https_port 和证书
```

### Q5: 如何更新 FRP 版本？

```bash
# 1. 下载新版
wget https://github.com/fatedier/frp/releases/download/vNEW_VERSION/frp_NEW_VERSION_linux_amd64.tar.gz
tar -xzf frp_NEW_VERSION_linux_amd64.tar.gz

# 2. 替换二进制
sudo systemctl stop frps
sudo cp frp_NEW_VERSION_linux_amd64/frps /usr/local/bin/
sudo systemctl start frps

# 3. 验证
/usr/local/bin/frps --version
```

### Q6: Docker 容器内无法访问本地端口？

```yaml
# 使用 network_mode: host 让容器直接使用宿主机网络
services:
  frpc:
    network_mode: host
```

### Q7: 如何配置多个客户端共享同一个 Token？

```toml
# 每个客户端使用相同的 auth.token
# 服务端在 [common] 段配置一个 token
# 客户端在各自的 [common] 段使用相同的 token
# 通过不同的代理名称区分
```

### Q8: 日志提示 "login to server failed: EOF"

这通常表示：
- 服务端 frps 未启动或崩溃
- 防火墙/NAT 丢弃了连接
- 端口不匹配

---

## 🔒 安全建议 / Security Best Practices

| 建议 Suggestion | 说明 Description |
|----------------|-----------------|
| 使用强 Token | 至少 32 位随机字符 |
| 启用 TLS 加密 | 防止中间人攻击 |
| 使用 STCP | 敏感服务使用安全隧道 |
| 绑定白名单 IP | `allow_ports` 限制暴露端口 |
| 定期更新 | 关注 CVE 和安全公告 |
| 最小权限运行 | systemd 使用 `User=nobody` |
| 防火墙限制 | 仅放行必要端口 |
| 监控日志 | 配置日志告警 |
| 使用 SSH Key | 避免密码认证 |
| 内网服务加固 | 即使穿透也应确保服务安全 |

```bash
# 生成 TLS 证书（自签名示例）
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/frp/server.key \
  -out /etc/frp/server.crt \
  -days 3650 \
  -subj "/CN=frp.example.com"

# 生成强随机 Token
openssl rand -base64 32

# 设置严格权限
chmod 600 /etc/frp/frps.toml
chmod 600 /etc/frp/server.key
chown nobody:nogroup /etc/frp/frps.toml
```

---

## 🔗 参考链接 / References

- [FRP 官方文档](https://github.com/fatedier/frp)
- [FRP Releases](https://github.com/fatedier/frp/releases)
- [FRP Docker 镜像](https://hub.docker.com/r/snowdreamtech/frps)
- [FRP 配置示例](https://github.com/fatedier/frp/tree/master/conf)

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
