---
title: '新VPS到手后必做的安全加固与初始化配置指南'
description: '如果你拿到了一个新的VPS不知道要先做什么，或者你有闲置的小鸡不知道干什么，这篇文章或许会给你不错的建议。从登录到基础加固，手把手带你完成新VPS的初始化设置。'
pubDate: '2025-05-04'
tags: ['VPS', 'Linux', '安全', 'SSH', 'Docker', '运维']
categories: ['运维']
---

> 拿到新的小鸡（VPS）后，你应该先做什么？
> 恭喜你拥有了一台属于自己的云服务器（VPS）！这是一片充满无限可能的数字天地。但在你开始部署网站、搭建应用之前，务必先花上15-30分钟，完成一些至关重要的基础设置。

本文将以一台全新的、基于 Ubuntu 系统的 VPS 为例，手把手带你完成从登录到基础加固的全过程。

---

## 准备工作

在开始之前，你需要从 VPS 提供商那里获取三个关键信息：

- 服务器的IP地址（例如：`192.0.2.1`）
- 初始用户名（通常是 `root`）
- 初始密码

还需要一个 SSH 客户端工具：
- macOS / Linux：使用系统自带的"终端"(Terminal)
- Windows：使用 Windows Terminal 或 PuTTY

---

## 首次登录与更新系统

```bash
ssh root@你的服务器IP地址
```

登录后的首要任务是更新系统：

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 创建新用户并授予管理员权限

一直使用 root 用户操作服务器是一个非常危险的习惯。

```bash
# 创建新用户
adduser your_username

# 授予sudo权限
usermod -aG sudo your_username
```

**验证新用户**：
```bash
# 退出root登录
exit

# 用新用户名登录
ssh your_username@你的服务器IP地址

# 验证权限
sudo apt update
```

---

## 加固 SSH 服务

SSH 是我们远程管理服务器的唯一入口，保护好它至关重要。

### 1. 禁用 root 用户远程登录

```bash
sudo nano /etc/ssh/sshd_config
```

找到 `PermitRootLogin` 这一行，修改为 `no`。

### 2. 更改默认 SSH 端口

找到 `#Port 22`，修改为一个不常用的端口号（范围建议在 1024-65535）：

```bash
Port 2255
```

### 3. 应用更改

```bash
sudo systemctl restart sshd
```

**重要**：先打开一个新的终端窗口测试，确保新连接正常后再关闭旧连接。

```bash
ssh -p 2255 your_username@你的服务器IP地址
```

---

## 配置基础防火墙

UFW (Uncomplicated Firewall) 是一个非常易于使用的防火墙管理工具。

```bash
# 安装UFW
sudo apt install ufw

# 设置默认规则：禁止所有传入，允许所有传出
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许必要连接
sudo ufw allow 2255/tcp  # SSH（你改过的端口）
sudo ufw allow http      # 80端口
sudo ufw allow https    # 443端口

# 启用防火墙
sudo ufw enable

# 查看状态
sudo ufw status verbose
```

---

## 安装 Fail2ban 防御工具

Fail2ban 通过分析系统日志中的异常行为，自动封禁可疑 IP 地址。

```bash
sudo apt update && sudo apt install fail2ban
```

### 配置 Fail2ban

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

常见配置参数：

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8  # 白名单IP
bantime = 3600          # 封禁时长（秒）
findtime = 600          # 检测时间段（秒）
maxretry = 5            # 最大尝试次数

[sshd]
enabled = true
```

### 启动服务

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

---

## 设置系统时间

```bash
sudo timedatectl set-timezone Asia/Shanghai

# 验证
timedatectl
# 或
date
```

---

## 使用 SSH 密钥登录

### 1. 在本地电脑上生成密钥对

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- `-t ed25519`：指定密钥类型
- `-C "your_email@example.com"`：添加注释标识

### 2. 将公钥复制到服务器

**方法1：使用 ssh-copy-id（推荐）**

```bash
ssh-copy-id your_username@server_ip

# 如果修改了SSH端口
ssh-copy-id -p 2255 your_username@server_ip
```

**方法2：手动复制**

```bash
# 本地显示公钥
cat ~/.ssh/id_ed25519.pub

# 服务器上写入
mkdir -p ~/.ssh
echo "在这里粘贴你的公钥内容" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 3. 禁用密码登录

确认密钥登录成功后，编辑 SSH 配置：

```bash
sudo nano /etc/ssh/sshd_config
```

找到 `PasswordAuthentication`，修改为 `no`：

```ini
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
sudo systemctl restart sshd
```

---

## 安装 Docker 和 Docker Compose

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 添加当前用户到 docker 用户组
sudo usermod -aG docker $USER

# 验证版本
docker --version
docker compose version
```

---

## 安装 Nginx Proxy Manager

通过 Nginx 反向代理，所有服务流量统一转发，提升专业性、安全性与灵活性。

```bash
# 创建安装目录
mkdir ~/docker/npm && cd ~/docker/npm

# 创建 compose.yml
nano compose.yml
```

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

```bash
docker compose up -d
```

访问 `公网IP:81` 进入后台管理页面。

---

## Docker 容器网络配置

让 NPM 通过访问不同容器的内部 IP 来进行代理，docker 容器不需要暴露到公网。

```bash
# 创建容器时加入 npm_default 网络
services:
  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer_data:/data
    networks:
      - npm_default

networks:
  npm_default:
    external: true
```

```bash
# 查看容器IP
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 容器名

# 查看网络信息
docker network inspect npm_default
```

然后在 NPM 中配置代理，使用域名访问服务。

---

## 总结

| 步骤 | 操作 | 目的 |
|:---|:---|:---|
| 1 | 更新系统 | 确保安全补丁 |
| 2 | 创建新用户 | 避免使用root |
| 3 | 加固SSH | 禁用root+改端口 |
| 4 | 配置防火墙 | UFW基础防护 |
| 5 | 安装Fail2ban | 防暴力破解 |
| 6 | 设置时区 | 北京时间 |
| 7 | SSH密钥登录 | 免密更安全 |
| 8 | 安装Docker | 容器化部署 |
| 9 | 安装NPM | 反向代理 |