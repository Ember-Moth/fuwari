---
title: Unishop 安装教程
published: 2025-09-29
description: 'UniShop 发卡面板的安装教程'
image: ''
tags: [UniShop]
category: '教程'
draft: false 
lang: 'zh_CN'
---

#### 前置系统要求

- Debian 12

#### 安装前的准备
```bash
# 更新系统
apt update && apt upgrade -y

# 安装基础工具
apt install -y curl wget git unzip software-properties-common \
  apt-transport-https ca-certificates gnupg lsb-release

# 设置时区
timedatectl set-timezone Asia/Shanghai

# 禁用防火墙（如果启用）
ufw disable
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


```bash
# 生成安全密码
DB_PASSWORD=$(openssl rand -base64 16)
echo "请牢记数据库密码！！！ 数据库密码：$DB_PASSWORD"

# 创建数据库和用户（使用 mariadb 命令，避免弃用警告）
mariadb -u root -p <<EOF
CREATE DATABASE unishop_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER USER 'root'@'localhost' IDENTIFIED BY '$DB_PASSWORD';
FLUSH PRIVILEGES;
EOF
```

## 步骤 4：安装 PHP
```bash
# 添加 PHP 仓库
curl -sSLo /tmp/php.gpg https://packages.sury.org/php/apt.gpg
gpg --dearmor < /tmp/php.gpg > /usr/share/keyrings/php-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/php-archive-keyring.gpg] \
  https://packages.sury.org/php/ bookworm main" > /etc/apt/sources.list.d/php.list

# 安装 PHP 7.4 及必需扩展
apt update && apt install -y php7.4-{bcmath,bz2,cli,common,curl,fpm,gd,gmp,igbinary,intl,mbstring,mysql,opcache,readline,redis,soap,xml,yaml,zip}



# 配置 PHP
sed -i 's/^max_execution_time.*/max_execution_time = 300/' /etc/php/7.4/fpm/php.ini
sed -i 's/^memory_limit.*/memory_limit = 256M/' /etc/php/7.4/fpm/php.ini
sed -i 's/^post_max_size.*/post_max_size = 50M/' /etc/php/7.4/fpm/php.ini
sed -i 's/^upload_max_filesize.*/upload_max_filesize = 50M/' /etc/php/7.4/fpm/php.ini
sed -i 's/^;date.timezone.*/date.timezone = Asia\/Shanghai/' /etc/php/7.4/fpm/php.ini

# 配置 PHP-FPM
sed -i 's/^;listen.owner.*/listen.owner = www-data/' /etc/php/7.4/fpm/pool.d/www.conf
sed -i 's/^;listen.group.*/listen.group = www-data/' /etc/php/7.4/fpm/pool.d/www.conf
sed -i 's/^;listen.mode.*/listen.mode = 0660/' /etc/php/7.4/fpm/pool.d/www.conf

# 启动服务
systemctl restart php7.4-fpm && systemctl enable php7.4-fpm
```

## 步骤 5：安装 Composer
```bash
# 下载并安装 Composer
curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php
HASH=$(curl -sS https://composer.github.io/installer.sig)
php -r "if (hash_file('SHA384', '/tmp/composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('/tmp/composer-setup.php'); } echo PHP_EOL;"
php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm /tmp/composer-setup.php

# 验证安装
composer --version
```

## 步骤 6：配置 Nginx
```bash
# 创建配置目录
mkdir -p /etc/nginx/conf.d

# 创建站点配置
cat > /etc/nginx/conf.d/unishop.conf <<'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;  # 修改为你的域名
    
    root /var/www/unishop/public;
    index index.php index.html;
    
    location / {
        try_files $uri /index.php$is_args$args;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    location ~ /\.(?!well-known).* {
        deny all;
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
certbot --nginx -d your-domain.com
```

## 步骤 7：部署UniShop
```bash
# 创建网站目录
mkdir -p /var/www/unishop
cd /var/www

# 克隆项目
git clone https://github.com/Surge-Lab/UniShop.git unishop

cd unishop

# 配置 Git 安全目录（必需）
# 因为后续会将目录权限改为 www-data，需要预先配置避免 Git 报错
git config --global --add safe.directory /var/www/unishop


# 安装依赖
echo "开始安装 Composer 依赖..."
composer install

# 验证安装是否成功
if [ ! -f vendor/autoload.php ]; then
    echo "错误：Composer 依赖安装失败"
    echo "请检查错误信息并重新运行 composer install"
    exit 1
fi

# 环境配置
cp .env.example .env
php artisan key:generate
```

### 导入数据库

```bash
# 假如你的数据库密码为 12345678，则 命令应该为 mariadb -u root -p123456789 < database/sql/install.sql 
mariadb -u root -pyour_password < database/sql/install.sql
```
### 设置目录权限
```bash
chown -R www-data:www-data /var/www/unishop
chmod -R 755 /var/www/unishop/storage
chmod -R 755 /var/www/unishop/bootstrap/cache
```

### 设置守护进程
```bash
cat > /etc/systemd/system/unishop.service <<EOF
[Unit]
Description=UniShop Queue
After=network.target

[Service]
User=www-data
Group=www-data
ExecStart=/usr/bin/php /var/www/unishop/artisan queue:work
Restart=always
WorkingDirectory=/var/www/unishop

[Install]
WantedBy=multi-user.target
EOF
```

- 重新加载 systemd 服务

```bash
systemctl daemon-reload
```

- 启动服务

```bash
systemctl start unishop
```

- 开机自启

```bash
systemctl enable unishop
```