---
title: PPanel Server 二进制安装教程
published: 2025-08-31
description: 'PPanel 面板 服务端的二进制安装教程'
image: 'https://raw.githubusercontent.com/perfect-panel/ppanel-web/main/apps/user/public/pwa-maskable-192x192.png
'
tags: [PPanel]
category: '教程'
draft: false 
lang: 'zh_CN'
---
#### 前置系统要求

- Debian 12

#### 省流版：一键脚本
```bash
wget -N https://raw.githubusercontent.com/Ember-Moth/ppanel-install-script/refs/heads/main/install.sh && bash install.sh
````


#### 安装前的准备
```bash
# 更新系统
apt update && apt upgrade -y

# 安装基础工具
apt install -y curl wget git unzip software-properties-common \
  apt-transport-https ca-certificates gnupg lsb-release

# 设置时区
timedatectl set-timezone Asia/Shanghai

```
## 步骤 1：安装 Nginx
```bash
# 添加 Nginx 官方仓库
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
  http://nginx.org/packages/mainline/debian bookworm nginx" \
  | tee /etc/apt/sources.list.d/nginx.list

# 设置仓库优先级
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
  | tee /etc/apt/preferences.d/99nginx

# 安装 Nginx
apt update && apt install -y nginx

# 修改用户为 www-data
sed -i 's/^user.*/user www-data;/' /etc/nginx/nginx.conf

# 启动服务
systemctl start nginx && systemctl enable nginx
```

## 步骤 2：安装 Redis

```bash
# 添加 Redis 仓库
curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] \
  https://packages.redis.io/deb bookworm main" | tee /etc/apt/sources.list.d/redis.list

# 安装 Redis
apt update && apt install -y redis

# 配置 Redis
sed -i 's/^# bind 127.0.0.1 ::1/bind 127.0.0.1 ::1/' /etc/redis/redis.conf
sed -i 's/^# maxmemory <bytes>/maxmemory 256mb/' /etc/redis/redis.conf
sed -i 's/^# maxmemory-policy noeviction/maxmemory-policy allkeys-lru/' /etc/redis/redis.conf

# 启动服务
systemctl restart redis-server && systemctl enable redis-server
```

## 步骤 3：安装 MariaDB

```bash
# 添加 MariaDB 仓库
mkdir -p /etc/apt/keyrings
curl -o /etc/apt/keyrings/mariadb-keyring.pgp \
  'https://mariadb.org/mariadb_release_signing_key.pgp'

cat > /etc/apt/sources.list.d/mariadb.sources <<EOF
X-RepoLib-Name: MariaDB
Types: deb
URIs: https://deb.mariadb.org/11.8/debian
Suites: bookworm
Components: main
Signed-By: /etc/apt/keyrings/mariadb-keyring.pgp
EOF

# 安装 MariaDB
export DEBIAN_FRONTEND=noninteractive
apt update && apt install -y mariadb-server mariadb-client

# 启动服务
systemctl start mariadb && systemctl enable mariadb

# 安全初始化
mariadb-secure-installation <<EOF
\n
n
n
y
y
y
y
EOF
```

### 创建数据库

```bash
# 生成安全密码
DB_PASSWORD=$(openssl rand -base64 16)
echo "请牢记数据库密码！！！ 数据库密码：$DB_PASSWORD"

# 创建数据库和用户（使用 mariadb 命令，避免弃用警告）
mariadb -u root <<EOF
CREATE DATABASE ppanel_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER USER 'root'@'localhost' IDENTIFIED BY '$DB_PASSWORD';
FLUSH PRIVILEGES;
EOF
```

## 步骤 4：配置 Nginx
```bash
# 创建配置目录
mkdir -p /etc/nginx/conf.d

# 创建站点配置
cat > /etc/nginx/conf.d/ppanel.conf <<EOF
server {
    listen 80;
    listen [::]:80;
    server_name $domain;

    location / {
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header REMOTE-HOST \$remote_addr;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_http_version 1.1;

        add_header X-Cache \$upstream_cache_status;

        proxy_pass http://127.0.0.1:8080;
    }

    location ~* \.(gif|png|jpg|css|js|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control public;
    }
}
EOF

# 测试配置
nginx -t

# 重载 Nginx
systemctl reload nginx
```
为了保护用户数据安全，强烈建议配置 HTTPS：

```bash
# 安装 Certbot
apt install -y certbot python3-certbot-nginx

# 获取证书
certbot --nginx -d $domain --non-interactive --agree-tos -m admin@$domain
```

## 步骤 5：安装ppanel-server

确定系统架构，并下载对应的二进制文件

下载地址：`https://github.com/perfect-panel/server/releases`

示例说明：系统：Debian amd64，用户：root，当前目录：/root

- 下载

```shell
wget -O ppanel-server-linux-amd64.tar.gz \
  https://github.com/perfect-panel/server/releases/latest/download/ppanel-server-linux-amd64.tar.gz

```

- 解压

```shell
tar -zxvf ppanel-server-linux-amd64.tar.gz
```

- 移动

```shell
sudo mv ppanel-server /usr/local/bin/ppanel-server
sudo mkdir -p /usr/local/etc/ppanel
sudo mv ./etc/ppanel.yaml /usr/local/etc/ppanel/
```

- 赋予二进制文件执行权限

```shell
sudo chmod +x /usr/local/bin/ppanel-server
```

- 修改 ppanel.yaml 配置文件
```shell
AccessSecret=$(openssl rand -base64 16)
cat > /usr/local/etc/ppanel/ppanel.yaml <<EOF
Host: 127.0.0.1
Port: 8080
Debug: false

JwtAuth:
  AccessSecret: $AccessSecret
  AccessExpire: 604800

Logger:
  FilePath: /var/log/ppanel/ppanel.log
  MaxSize: 50
  MaxBackup: 3
  MaxAge: 30
  Compress: true
  Level: info

MySQL:
  Addr: 127.0.0.1:3306
  Username: root
  Password: $DB_PASSWORD
  Dbname: ppanel_db
  Config: charset=utf8mb4&parseTime=true&loc=Asia%2FShanghai
  MaxIdleConns: 10
  MaxOpenConns: 100
  LogMode: info
  LogZap: true
  SlowThreshold: 1000

Redis:
  Host: 127.0.0.1:6379
  Pass: ''
  DB: 0

Administrator:
  Email: admin@ppanel.dev
  Password: password
EOF
```

- 创建 systemd 服务文件

```shell
cat > /etc/systemd/system/ppanel.service <<EOF
[Unit]
Description=PPANEL Server
After=network.target

[Service]
ExecStart=/usr/local/bin/ppanel-server run --config /usr/local/etc/ppanel/ppanel.yaml
Restart=always
User=root
WorkingDirectory=/usr/local/bin

[Install]
WantedBy=multi-user.target
EOF
```

- 重新加载 systemd 服务

```shell
systemctl daemon-reload
```

- 启动服务

```shell
systemctl start ppanel
```

##### 其他说明

1. 安装路径：二进制文件最终移动到 /usr/local/bin 目录下
2. systemd 服务：
    - 服务名称：ppanel
    - 服务配置文件：/etc/systemd/system/ppanel.service
    - 服务启动命令：systemctl start ppanel
    - 服务停止命令：systemctl stop ppanel
    - 服务重启命令：systemctl restart ppanel
    - 服务状态命令：systemctl status ppanel
    - 服务开机自启：systemctl enable ppanel
3. 设置开机自启可通过以下命令开机自启

   ```shell
   systemctl enable ppanel
   ```

4. 服务日志：服务日志默认输出到 /usr/local/bin/ppanel.log 文件中
5. 可通过 `journalctl -u ppanel -f` 查看服务日志
6. 当配置文件为空或者不存在的情况下，服务会使用默认配置启动，配置文件路径为：`./etc/ppanel.yaml`，
   请通过`http://服务器地址:8080/init` 初始化系统配置