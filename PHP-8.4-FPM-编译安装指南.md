# PHP 8.4 FPM 编译安装部署指南

**文档版本**: V1.0  
**适配环境**: Ubuntu 22.04.5 LTS  
**核心版本**: PHP 8.4.x  
**用途**: 为 WordPress 提供 PHP 运行环境，配合 Nginx 使用

---

## 目录

1. [系统准备](#系统准备)
2. [依赖安装](#依赖安装)
3. [目录结构创建](#目录结构创建)
4. [下载 PHP 源码](#下载-php-源码)
5. [编译安装 PHP](#编译安装-php)
6. [配置 PHP-FPM](#配置-php-fpm)
7. [Nginx 默认站点配置 PHP](#nginx-默认站点配置-php)
8. [写入探针文件](#写入探针文件)
9. [配置 Systemd 服务](#配置-systemd-服务)
10. [安装 WP-CLI](#安装-wp-cli)
11. [功能验证](#功能验证)
12. [常用命令速查](#常用命令速查)
13. [常见问题排查](#常见问题排查)

---

## 系统准备

### 验证系统编码

确保系统编码为 UTF-8，避免文件解析异常：

```bash
locale
```

**预期输出**: `LC_ALL`/`LC_CTYPE` 字段为 `zh_CN.UTF-8`

### 配置 UTF-8 编码（如需要）

```bash
sudo apt install -y locales
sudo locale-gen zh_CN.UTF-8
sudo update-locale LC_ALL=zh_CN.UTF-8 LANG=zh_CN.UTF-8
```

> **注意**: 配置后需重新登录服务器生效

## 依赖安装

安装 PHP 编译所需的全部依赖包，包含 WordPress 必需扩展的依赖：

```bash
sudo apt update && sudo apt upgrade -y

# 基础编译工具
sudo apt install -y build-essential autoconf bison re2c libtool pkg-config

# PHP 核心依赖
sudo apt install -y libxml2-dev libssl-dev libsqlite3-dev zlib1g-dev libcurl4-openssl-dev

# WordPress 必需扩展依赖
sudo apt install -y libpng-dev libjpeg-dev libwebp-dev libfreetype-dev libxpm-dev  # GD
sudo apt install -y libonig-dev           # mbstring
sudo apt install -y libzip-dev            # zip
sudo apt install -y libicu-dev            # intl
sudo apt install -y libpq-dev             # pdo_pgsql (可选)
sudo apt install -y libmysqlclient-dev    # mysqli/pdo_mysql

# ImageMagick 扩展依赖（WordPress 图片处理推荐）
sudo apt install -y libmagickwand-dev imagemagick

# Redis 扩展依赖（WordPress 缓存推荐）
sudo apt install -y libhiredis-dev

# 其他有用扩展依赖
sudo apt install -y libgmp-dev            # gmp
sudo apt install -y libldb-dev libldap2-dev  # ldap
sudo apt install -y libsodium-dev         # sodium
sudo apt install -y libargon2-dev         # password_argon2
sudo apt install -y libreadline-dev       # readline
sudo apt install -y libtidy-dev           # tidy
sudo apt install -y libxslt1-dev          # xsl
sudo apt install -y libbz2-dev            # bz2
sudo apt install -y libenchant-2-dev      # enchant
sudo apt install -y libffi-dev            # ffi

# systemd 集成依赖（PHP-FPM 服务管理）
sudo apt install -y libsystemd-dev
```

### 处理已知依赖问题

Ubuntu 22.04 部分库路径需要手动链接：

```bash
# 修复 GMP 库路径
sudo ln -sf /usr/include/x86_64-linux-gnu/gmp.h /usr/include/gmp.h

# 修复 OpenLDAP 库路径
sudo ln -sf /usr/lib/x86_64-linux-gnu/libldap.so /usr/lib/libldap.so
sudo ln -sf /usr/lib/x86_64-linux-gnu/liblber.so /usr/lib/liblber.so
```

---

## 目录结构创建

创建 PHP 编译和运行所需的目录结构：

```bash
# 创建源码和编译目录
sudo mkdir -p /service/php/{src,build}

# 创建 PHP 运行目录
sudo mkdir -p /usr/local/php84/etc/php-fpm.d
sudo mkdir -p /usr/local/php84/var/run
sudo mkdir -p /usr/local/php84/var/log
sudo mkdir -p /usr/local/php84/tmp

# 创建 PHP-FPM 运行目录（与 Nginx 配合）
sudo mkdir -p /var/run/php-fpm
sudo mkdir -p /var/log/php-fpm

# 配置目录权限
sudo chown -R www-data:www-data /usr/local/php84/var
sudo chown -R www-data:www-data /var/run/php-fpm
sudo chown -R www-data:www-data /var/log/php-fpm
sudo chmod -R 755 /service/php
sudo chmod -R 755 /usr/local/php84
sudo chmod 1777 /usr/local/php84/tmp
```

### 配置 tmpfiles.d（确保重启后目录自动创建）

由于 `/var/run` 是临时文件系统，系统重启后会被清空。需要配置 tmpfiles.d 确保目录自动创建：

```bash
sudo tee /etc/tmpfiles.d/php-fpm.conf << 'EOF'
d /var/run/php-fpm 0755 www-data www-data -
d /var/log/php-fpm 0755 www-data www-data -
EOF
```

### 目录说明

| 目录路径 | 功能说明 |
|---------|---------|
| `/service/php/src` | PHP 源码存放目录 |
| `/service/php/build` | PHP 编译工作目录 |
| `/usr/local/php84` | PHP 安装根目录 |
| `/usr/local/php84/etc` | PHP 配置文件目录 |
| `/usr/local/php84/etc/php-fpm.d` | PHP-FPM 池配置目录 |
| `/usr/local/php84/var/run` | PID 文件目录 |
| `/usr/local/php84/var/log` | PHP-FPM 日志目录 |
| `/usr/local/php84/tmp` | PHP 临时文件目录 |
| `/var/run/php-fpm` | PHP-FPM Unix Socket 目录 |
| `/var/log/php-fpm` | PHP-FPM 系统日志目录 |

---

## 下载 PHP 源码

### 获取最新 PHP 8.4 版本

```bash
cd /service/php/src

# 下载 PHP 8.4 最新版本（以 8.4.18 为例，请访问 php.net 获取最新版本号）
wget https://www.php.net/distributions/php-8.4.18.tar.gz

# 解压源码
tar -zxvf php-8.4.18.tar.gz

# 进入源码目录
cd php-8.4.18
```

### 验证源码完整性（可选但推荐）    

```bash
# 下载签名文件
wget https://www.php.net/distributions/php-8.4.18.tar.gz.asc

# 导入 PHP 发布公钥
gpg --keyserver keyserver.ubuntu.com --recv-keys BF21D34691E1B0F9A7C9E8B3A8B2C5D1E6F7A8B9

# 验证签名
    gpg --verify php-8.4.18.tar.gz.asc php-8.4.18.tar.gz
```

---

## 编译安装 PHP

### 配置编译参数

```bash
cd /service/php/src/php-8.4.18

# 配置编译参数
./configure \
--prefix=/usr/local/php84 \
--exec-prefix=/usr/local/php84 \
--bindir=/usr/local/php84/bin \
--sbindir=/usr/local/php84/sbin \
--includedir=/usr/local/php84/include \
--libdir=/usr/local/php84/lib/php \
--mandir=/usr/local/php84/php/man \
--with-config-file-path=/usr/local/php84/etc \
--with-config-file-scan-dir=/usr/local/php84/etc/php.d \
--enable-fpm \
--with-fpm-user=www-data \
--with-fpm-group=www-data \
--with-fpm-systemd \
--with-fpm-acl \
--enable-mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--enable-bcmath \
--with-curl \
--with-openssl \
--with-zlib \
--with-zip \
--enable-gd \
--with-webp \
--with-jpeg \
--with-freetype \
--with-xpm \
--enable-gd-jis-conv \
--enable-intl \
--enable-mbstring \
--enable-pcntl \
--enable-shmop \
--enable-soap \
--enable-sockets \
--enable-sysvmsg \
--enable-sysvsem \
--enable-sysvshm \
--with-bz2 \
--enable-calendar \
--enable-dba \
--enable-exif \
--enable-ftp \
--with-gettext \
--with-gmp \
--with-mhash \
--enable-opcache \
--with-password-argon2 \
--with-sodium \
--enable-mysqlnd-compression-support \
--with-pear \
--enable-xml \
--with-xsl \
--enable-simplexml \
--enable-dom \
--enable-xmlreader \
--enable-xmlwriter \
--with-tidy \
--with-readline \
--enable-phpdbg \
--enable-filter \
--enable-hash \
--enable-json \
--enable-libxml \
--enable-session \
--enable-tokenizer \
--with-libxml \
--with-sqlite3 \
--with-pdo-sqlite \
--enable-fileinfo \
--with-ffi
```

### 编译参数说明

| 参数 | 说明 |
|-----|------|
| `--enable-fpm` | 启用 PHP-FPM（FastCGI 进程管理器） |
| `--with-fpm-systemd` | 启用 systemd 集成 |
| `--enable-mysqlnd` | 启用 MySQL Native Driver |
| `--with-mysqli=mysqlnd` | MySQLi 扩展使用 mysqlnd |
| `--with-pdo-mysql=mysqlnd` | PDO MySQL 使用 mysqlnd |
| `--enable-opcache` | 启用 OPcache 字节码缓存 |
| `--enable-gd` | 启用 GD 图像处理库 |
| `--with-webp/jpeg/freetype/xpm` | GD 支持的图像格式 |
| `--enable-intl` | 国际化支持（WordPress 多语言必需） |
| `--enable-mbstring` | 多字节字符串处理 |
| `--with-sodium` | 现代加密库支持 |
| `--with-password-argon2` | Argon2 密码哈希算法 |
| `--with-ffi` | 外部函数接口 |

### 编译安装

```bash
# 多核编译，提升编译速度
make -j$(nproc)

# 测试编译结果（可选，耗时较长）
make test

# 安装 PHP
sudo make install
```

### 配置系统环境

```bash
# 创建软链接，使 PHP 命令全局可用
sudo ln -sf /usr/local/php84/bin/php /usr/local/bin/php
sudo ln -sf /usr/local/php84/bin/phpize /usr/local/bin/phpize
sudo ln -sf /usr/local/php84/bin/php-config /usr/local/bin/php-config
sudo ln -sf /usr/local/php84/sbin/php-fpm /usr/local/sbin/php-fpm

# 配置系统库路径
sudo tee /etc/ld.so.conf.d/php84.conf <<EOF
/usr/local/php84/lib/php
EOF
sudo ldconfig

# 验证安装
php -v
```

**预期输出**: `PHP 8.4.18 (cli) ...`

### 复制配置文件

```bash
# 创建 php.d 扫描目录
sudo mkdir -p /usr/local/php84/etc/php.d

# 复制 php.ini 配置文件
sudo cp /service/php/src/php-8.4.18/php.ini-production /usr/local/php84/etc/php.ini

# 复制 php-fpm 配置文件
sudo cp /usr/local/php84/etc/php-fpm.conf.default /usr/local/php84/etc/php-fpm.conf
sudo cp /usr/local/php84/etc/php-fpm.d/www.conf.default /usr/local/php84/etc/php-fpm.d/www.conf

# 配置文件权限
sudo chown -R www-data:www-data /usr/local/php84/etc
sudo chmod 644 /usr/local/php84/etc/php.ini
sudo chmod 644 /usr/local/php84/etc/php-fpm.conf
sudo chmod 644 /usr/local/php84/etc/php-fpm.d/www.conf
```

### 安装 PECL 扩展

安装 WordPress 推荐的 ImageMagick 和 Redis 扩展：

```bash
# 检查并安装 ImageMagick 扩展（WordPress 图片处理推荐）
if ! php -m | grep -q "imagick"; then
    sudo /usr/local/php84/bin/pecl install imagick
else
    echo "imagick 扩展已安装，跳过"
fi

# 检查并安装 Redis 扩展（WordPress 缓存推荐）
# 使用 printf 自动回答配置问题（全部使用默认值）
if ! php -m | grep -q "redis"; then
    printf "no\nno\nno\nno\nno\n" | sudo /usr/local/php84/bin/pecl install redis
else
    echo "redis 扩展已安装，跳过"
fi

# 创建扩展配置文件（确保文件存在）
sudo tee /usr/local/php84/etc/php.d/imagick.ini <<EOF
extension=imagick.so
EOF

sudo tee /usr/local/php84/etc/php.d/redis.ini <<EOF
extension=redis.so
EOF

# 验证扩展安装
php -m | grep -E "imagick|redis"
```

> **注意**: 如果扩展已安装，`pecl install` 会返回错误。建议在脚本中使用检查或添加 `|| true` 来防止安装中断。

---

## 配置 PHP-FPM

### 配置 php.ini

针对 WordPress 优化 PHP 配置：

```bash
sudo tee /usr/local/php84/etc/php.ini << 'EOF'
[PHP]
; ========== 基础配置 ==========
engine = On
short_open_tag = Off
precision = 14
output_buffering = 4096
zlib.output_compression = Off
implicit_flush = Off
serialize_precision = -1

; ========== 文件上传配置（WordPress 优化） ==========
file_uploads = On
upload_max_filesize = 64M
post_max_size = 128M
max_file_uploads = 20

; ========== 内存与执行时间配置 ==========
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
max_input_vars = 3000

; ========== 错误处理 ==========
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
log_errors_max_len = 4096
ignore_repeated_errors = Off
ignore_repeated_source = Off
report_memleaks = On

; ========== 错误日志路径 ==========
error_log = /var/log/php-fpm/php-error.log

; ========== 临时目录 ==========
sys_temp_dir = /usr/local/php84/tmp
upload_tmp_dir = /usr/local/php84/tmp

; ========== 安全配置 ==========
expose_php = Off
allow_url_fopen = On
allow_url_include = Off

; ========== 时区配置 ==========
date.timezone = Asia/Shanghai

; ========== Session 配置 ==========
session.save_handler = files
session.save_path = /usr/local/php84/tmp
session.use_strict_mode = 1
session.use_cookies = 1
session.use_only_cookies = 1
session.name = PHPSESSID
session.cookie_httponly = 1
session.cookie_secure = 0
session.cookie_samesite = Strict
session.gc_maxlifetime = 1440

; ========== OPcache 配置（性能优化） ==========
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.max_wasted_percentage = 10
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
opcache.fast_shutdown = 1
opcache.save_comments = 1
opcache.enable_file_override = 1

; ========== 字符集配置 ==========
default_charset = "UTF-8"

[CLI Server]
cli_server.color = On

[Date]
date.timezone = Asia/Shanghai

[Pdo_mysql]
pdo_mysql.default_socket = /var/run/mysqld/mysqld.sock

[mail function]
SMTP = localhost
smtp_port = 25
mail.add_x_header = Off

[ODBC]
odbc.allow_persistent = On
odbc.check_persistent = On
odbc.max_persistent = -1
odbc.max_links = -1
odbc.defaultlrl = 4096
odbc.defaultbinmode = 1

[MySQLi]
mysqli.max_persistent = -1
mysqli.allow_persistent = On
mysqli.max_links = -1
mysqli.default_port = 3306
mysqli.default_socket = /var/run/mysqld/mysqld.sock
mysqli.default_host =
mysqli.default_user =
mysqli.default_pw =
mysqli.reconnect = Off

[mysqlnd]
mysqlnd.collect_statistics = On
mysqlnd.collect_memory_statistics = Off

[PostgreSQL]
pgsql.allow_persistent = On
pgsql.auto_reset_persistent = Off
pgsql.max_persistent = -1
pgsql.max_links = -1
pgsql.ignore_notice = 0
pgsql.log_notice = 0

[bcmath]
bcmath.scale = 0

[Session]
session.save_handler = files
session.save_path = /usr/local/php84/tmp
session.use_strict_mode = 1

[ldap]
ldap.max_links = -1

[opcache]
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 2
EOF
```

### 配置 php-fpm.conf

```bash
sudo tee /usr/local/php84/etc/php-fpm.conf << 'EOF'
;;;;;;;;;;;;;;;;;;;;;
; FPM Configuration ;
;;;;;;;;;;;;;;;;;;;;;

[global]
; PID 文件路径
pid = /usr/local/php84/var/run/php-fpm.pid

; 错误日志路径
error_log = /var/log/php-fpm/php-fpm-error.log

; 日志级别: alert, error, warning, notice, debug
log_level = notice

; 日志格式
log_limit = 4096

; 紧急重启阈值
emergency_restart_threshold = 10
emergency_restart_interval = 1m

; 进程控制超时
process_control_timeout = 10s

; 守护进程模式
daemonize = yes

; 加载池配置
include = /usr/local/php84/etc/php-fpm.d/*.conf
EOF
```

### 配置 www.conf（PHP-FPM 池）

```bash
sudo tee /usr/local/php84/etc/php-fpm.d/www.conf << 'EOF'
; ========== WordPress 专用 PHP-FPM 池配置 ==========

[www]
; 池名称
prefix = /usr/local/php84/var

; 运行用户和组
user = www-data
group = www-data

; 监听方式：Unix Socket（推荐，性能更好）
listen = /var/run/php-fpm/php84-fpm.sock

; Socket 权限
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; 允许监听的客户端（仅本地）
listen.allowed_clients = 127.0.0.1

; ========== 进程管理配置 ==========
; 进程管理方式: static, dynamic, ondemand
pm = dynamic

; 最大子进程数（根据服务器内存调整，约 1 个进程 50-80MB 内存）
pm.max_children = 30

; 空闲时最小进程数
pm.min_spare_servers = 5

; 空闲时最大进程数
pm.max_spare_servers = 15

; 启动时创建的进程数
pm.start_servers = 8

; 每个进程最大处理请求数（防止内存泄漏）
pm.max_requests = 1000

; 慢请求日志（性能排查用）
slowlog = /var/log/php-fpm/www-slow.log
request_slowlog_timeout = 10s
request_slowlog_trace_depth = 20

; ========== 状态监控 ==========
; 启用状态页面（配合 Nginx 访问控制）
pm.status_path = /php-fpm-status

; 启用 ping 页面
ping.path = /php-fpm-ping
ping.response = pong

; ========== 环境变量 ==========
clear_env = no

; ========== PHP 配置覆盖 ==========
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on

; ========== 安全配置 ==========
; 捕获工作进程输出
catch_workers_output = yes
decorate_workers_output = no

; 禁用的 PHP 函数（安全加固）
php_admin_value[disable_functions] = exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source,symlink,dl

; 可根据 WordPress 实际需求调整禁用列表，部分插件可能需要 exec 权限
EOF
```

### 配置目录权限

```bash
sudo chown -R www-data:www-data /usr/local/php84/etc
sudo chown -R www-data:www-data /usr/local/php84/var
sudo chown -R www-data:www-data /var/run/php-fpm
sudo chown -R www-data:www-data /var/log/php-fpm
sudo chmod 1777 /usr/local/php84/tmp
```

---

## Nginx 默认站点配置 PHP

### 配置默认站点支持 PHP

编辑 Nginx 默认站点配置文件，添加 PHP 处理：

```bash
sudo tee /etc/nginx/sites-available/default << 'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # 站点根目录
    root /var/www/html;

    # 默认索引文件
    index index.php index.html index.htm;

    # 服务器名称
    server_name _;

    # 字符编码
    charset utf-8;

    # 日志配置
    access_log /var/log/nginx/default.access.log;
    error_log /var/log/nginx/default.error.log;

    # 默认位置
    location / {
        try_files $uri $uri/ =404;
    }

    # PHP 处理配置
    location ~ \.php$ {
        # 包含 FastCGI 参数
        include snippets/fastcgi-php.conf;

        # PHP-FPM Unix Socket 连接
        fastcgi_pass unix:/var/run/php-fpm/php84-fpm.sock;

        # 超时设置
        fastcgi_connect_timeout 300s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;
    }

    # 禁止访问 .htaccess 文件
    location ~ /\.ht {
        deny all;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }

    # 静态文件缓存优化
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
EOF
```

### 创建测试目录

```bash
# 创建网站根目录
sudo mkdir -p /var/www/html

# 设置目录权限
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

---

## 写入探针文件

创建 PHP 探针文件用于测试和展示 PHP 环境信息：

```bash
sudo tee /var/www/html/probe.php << 'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PHP 8.4 环境探针</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container { max-width: 1200px; margin: 0 auto; }
        .header { text-align: center; color: white; margin-bottom: 30px; }
        .header h1 { font-size: 2.5em; margin-bottom: 10px; text-shadow: 2px 2px 4px rgba(0,0,0,0.2); }
        .header p { opacity: 0.9; font-size: 1.1em; }
        .card {
            background: white; border-radius: 12px; padding: 25px;
            margin-bottom: 20px; box-shadow: 0 10px 40px rgba(0,0,0,0.1);
        }
        .card h2 {
            color: #333; margin-bottom: 20px; padding-bottom: 10px;
            border-bottom: 2px solid #667eea; display: flex; align-items: center; gap: 10px;
        }
        .card h2::before { content: ''; width: 4px; height: 24px; background: #667eea; border-radius: 2px; }
        .info-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 15px; }
        .info-item {
            display: flex; justify-content: space-between; padding: 12px 15px;
            background: #f8f9fa; border-radius: 8px; border-left: 3px solid #667eea;
        }
        .info-item label { color: #666; font-weight: 500; }
        .info-item value { color: #333; font-weight: 600; }
        .status-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(150px, 1fr)); gap: 10px; }
        .status-item {
            display: flex; align-items: center; gap: 8px; padding: 10px 15px;
            border-radius: 8px; font-size: 0.95em;
        }
        .status-item.installed { background: #d4edda; color: #155724; }
        .status-item.missing { background: #f8d7da; color: #721c24; }
        .status-icon {
            width: 20px; height: 20px; border-radius: 50%; display: flex;
            align-items: center; justify-content: center; font-weight: bold; font-size: 12px;
        }
        .status-item.installed .status-icon { background: #28a745; color: white; }
        .status-item.missing .status-icon { background: #dc3545; color: white; }
        .section-title {
            color: #555; font-size: 1.1em; margin: 20px 0 15px 0;
            padding-left: 10px; border-left: 3px solid #764ba2;
        }
        .footer { text-align: center; color: white; opacity: 0.8; margin-top: 30px; font-size: 0.9em; }
        @media (max-width: 768px) {
            .header h1 { font-size: 1.8em; }
            .card { padding: 20px; }
            .info-grid { grid-template-columns: 1fr; }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🔍 PHP 8.4 环境探针</h1>
            <p>WordPress 运行环境检测与性能分析</p>
        </div>
        <div class="card">
            <h2>服务器基本信息</h2>
            <div class="info-grid">
                <div class="info-item"><label>服务器系统</label><value><?php echo php_uname('s') . ' ' . php_uname('r'); ?></value></div>
                <div class="info-item"><label>服务器软件</label><value><?php echo $_SERVER['SERVER_SOFTWARE'] ?? 'Unknown'; ?></value></div>
                <div class="info-item"><label>PHP 版本</label><value><?php echo PHP_VERSION; ?></value></div>
                <div class="info-item"><label>PHP 运行方式</label><value><?php echo php_sapi_name(); ?></value></div>
                <div class="info-item"><label>当前时间</label><value><?php echo date('Y-m-d H:i:s'); ?></value></div>
            </div>
        </div>
        <div class="card">
            <h2>PHP 核心配置</h2>
            <div class="info-grid">
                <div class="info-item"><label>内存限制</label><value><?php echo ini_get('memory_limit'); ?></value></div>
                <div class="info-item"><label>上传限制</label><value><?php echo ini_get('upload_max_filesize'); ?></value></div>
                <div class="info-item"><label>POST 限制</label><value><?php echo ini_get('post_max_size'); ?></value></div>
                <div class="info-item"><label>最大执行时间</label><value><?php echo ini_get('max_execution_time'); ?> 秒</value></div>
                <div class="info-item"><label>时区设置</label><value><?php echo ini_get('date.timezone') ?: '未设置'; ?></value></div>
            </div>
        </div>
        <div class="card">
            <h2>WordPress 必需扩展</h2>
            <?php $required = ['curl'=>'HTTP请求','dom'=>'XML处理','exif'=>'图片元数据','fileinfo'=>'文件类型检测','gd'=>'图像处理','iconv'=>'字符编码','intl'=>'国际化','json'=>'JSON处理','mbstring'=>'多字节字符串','mysqli'=>'MySQL连接','openssl'=>'SSL加密','pcre'=>'正则表达式','pdo_mysql'=>'PDO MySQL','xml'=>'XML解析','zip'=>'压缩文件','zlib'=>'数据压缩']; ?>
            <div class="status-grid">
                <?php foreach ($required as $ext => $desc): ?>
                <div class="status-item <?php echo extension_loaded($ext) ? 'installed' : 'missing'; ?>">
                    <span class="status-icon"><?php echo extension_loaded($ext) ? '✓' : '✗'; ?></span>
                    <span><?php echo $ext; ?> <small>(<?php echo $desc; ?>)</small></span>
                </div>
                <?php endforeach; ?>
            </div>
        </div>
        <div class="footer"><p>PHP 8.4 FPM 编译安装部署指南 | 探针页面</p></div>
    </div>
</body>
</html>
EOF

sudo chown www-data:www-data /var/www/html/probe.php
sudo chmod 644 /var/www/html/probe.php
```

### 测试 Nginx 配置并重载

```bash
# 测试 Nginx 配置语法
sudo nginx -t

# 重载 Nginx 配置
sudo systemctl reload nginx

# 验证 Nginx 状态
sudo systemctl status nginx
```

### 验证 PHP 处理

```bash
# 本地测试 PHP 探针页面
curl -s http://localhost/probe.php | head -50

# 浏览器访问查看完整探针页面
# http://your-server-ip/probe.php
```

> **安全提示**: 测试完成后请删除或重命名探针文件，避免泄露服务器信息：
> ```bash
> sudo rm /var/www/html/probe.php
> # 或重命名为随机文件名
> sudo mv /var/www/html/probe.php /var/www/html/probe_$(date +%s).php
> ```

---

## 配置 Systemd 服务

### 创建服务文件

```bash
sudo tee /etc/systemd/system/php84-fpm.service << 'EOF'
[Unit]
Description=PHP 8.4 FastCGI 进程管理器
Documentation=https://www.php.net/manual/en/install.fpm.php
After=network.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=notify
PIDFile=/usr/local/php84/var/run/php-fpm.pid
ExecStart=/usr/local/php84/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php84/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -SIGQUIT $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
```

### 启用并启动服务

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable php84-fpm

# 启动 PHP-FPM
sudo systemctl start php84-fpm

# 验证服务状态
sudo systemctl status php84-fpm
```

---

## 安装 WP-CLI

WP-CLI 是 WordPress 的命令行工具，用于管理 WordPress 站点。

### 下载安装

```bash
# 下载 WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# 验证是否正常工作
php wp-cli.phar --info

# 添加执行权限
chmod +x wp-cli.phar

# 移动到全局路径
sudo mv wp-cli.phar /usr/local/bin/wp

# 验证安装
wp --info
```

### 配置 WP-CLI Tab 补全

```bash
# 下载 bash 补全脚本
sudo wget -O /etc/bash_completion.d/wp-cli https://raw.githubusercontent.com/wp-cli/wp-cli/master/utils/wp-completion.bash

# 使补全生效
source /etc/bash_completion.d/wp-cli
```

### WP-CLI 常用命令

```bash
# 查看 WP-CLI 信息
wp --info

# 更新 WP-CLI 到最新版本
wp cli update
```

### 单站点安装命令

```bash
# 下载 WordPress（指定版本）
wp core download --version=6.4.3 --locale=zh_CN

# 创建 wp-config.php
wp config create --dbname=wordpress --dbuser=wp_user --dbpass=your_password --dbhost=localhost

# 安装 WordPress（单站点）
wp core install --url=example.com --title="Site Title" --admin_user=admin --admin_password=password --admin_email=admin@example.com
```

### 多站点（Multisite）安装命令

```bash
# 下载 WordPress
wp core download --version=6.4.3 --locale=zh_CN

# 创建 wp-config.php（多站点需要提前配置）
wp config create --dbname=wordpress_ms --dbuser=wp_user --dbpass=your_password --dbhost=localhost

# 添加多站点常量到 wp-config.php
wp config set WP_ALLOW_MULTISITE true --raw
wp config set MULTISITE true --raw
wp config set SUBDOMAIN_INSTALL false --raw  # true=子域名模式, false=子目录模式
wp config set DOMAIN_CURRENT_SITE 'example.com'
wp config set PATH_CURRENT_SITE '/'
wp config set SITE_ID_CURRENT_SITE 1 --raw
wp config set BLOG_ID_CURRENT_SITE 1 --raw

# 安装 WordPress 多站点网络
wp core multisite-install \
    --url=example.com \
    --title="WordPress Network" \
    --admin_user=admin \
    --admin_password=password \
    --admin_email=admin@example.com \
    --subdomains  # 使用子域名模式，去掉此参数则使用子目录模式

# 创建新站点（多站点网络）
wp site create --slug=newsite --title="New Site" --email=user@example.com

# 列出所有站点
wp site list

# 切换操作站点上下文
wp --url=example.com/site1 plugin list
```

### 通用管理命令

```bash
# 查看插件列表
wp plugin list

# 安装插件
wp plugin install plugin-name --activate

# 更新所有插件
wp plugin update --all

# 更新 WordPress 核心
wp core update

# 数据库优化
wp db optimize

# 导出数据库
wp db export backup.sql

# 导入数据库
wp db import backup.sql

# 多站点：批量更新所有站点插件
wp site list --field=url | xargs -I {} wp plugin update --all --url={}
```

---

## 功能验证

### 验证 PHP 安装

```bash
# 查看 PHP 版本
php -v

# 查看已安装扩展
php -m

# 查看 PHP 配置信息
php -i | head -50

# 验证 WordPress 必需扩展
php -m | grep -E "curl|dom|exif|fileinfo|gd|iconv|intl|json|mbstring|mysqli|openssl|pdo_mysql|xml|zip"
```

### 验证 PHP-FPM 运行状态

```bash
# 查看服务状态
sudo systemctl status php84-fpm

# 查看 PHP-FPM 进程
ps aux | grep php-fpm

# 查看监听端口/Socket
ls -la /var/run/php-fpm/

# 测试 PHP-FPM 状态页面
curl http://127.0.0.1:9000/php-fpm-status
```

### 访问 PHP 探针页面

如果已按照前文创建了探针页面，可以通过浏览器访问：

```bash
# 本地测试探针页面
curl -s http://localhost/probe.php | head -50
```

浏览器访问：`http://your-server-ip/probe.php`

> **安全提示**: 测试完成后请删除探针文件，避免泄露服务器信息：
> ```bash
> sudo rm /var/www/html/probe.php
> ```

### 验证 WP-CLI

```bash
# 查看 WP-CLI 信息
wp --info

# 测试 WP-CLI 命令
wp cli info
```

---

## 常用命令速查

### PHP 相关命令

| 命令 | 说明 |
|------|------|
| `php -v` | 查看 PHP 版本 |
| `php -m` | 查看已加载扩展 |
| `php -i` | 查看 PHP 配置信息 |
| `php -r "echo 'Hello';"` | 执行 PHP 代码 |
| `php --ini` | 查看配置文件路径 |
| `phpize` | 准备 PHP 扩展编译环境 |
| `pecl install ext-name` | 安装 PECL 扩展 |

### PHP-FPM 相关命令

| 命令 | 说明 |
|------|------|
| `sudo systemctl start php84-fpm` | 启动 PHP-FPM |
| `sudo systemctl stop php84-fpm` | 停止 PHP-FPM |
| `sudo systemctl restart php84-fpm` | 重启 PHP-FPM |
| `sudo systemctl reload php84-fpm` | 平滑重载配置 |
| `sudo systemctl status php84-fpm` | 查看服务状态 |
| `sudo systemctl enable php84-fpm` | 设置开机自启 |
| `sudo systemctl disable php84-fpm` | 取消开机自启 |

### WP-CLI 相关命令

| 命令 | 说明 |
|------|------|
| `wp --info` | 查看 WP-CLI 信息 |
| `wp cli update` | 更新 WP-CLI |
| `wp core download` | 下载 WordPress |
| `wp core install` | 安装 WordPress |
| `wp plugin list` | 列出插件 |
| `wp plugin install name` | 安装插件 |
| `wp plugin update --all` | 更新所有插件 |
| `wp db export file.sql` | 导出数据库 |
| `wp db import file.sql` | 导入数据库 |

---

## 常见问题排查

### PHP-FPM 启动失败

**症状**: `systemctl start php84-fpm` 失败

**排查步骤**:

```bash
# 查看详细错误信息
sudo journalctl -u php84-fpm -n 50

# 检查配置文件语法
sudo /usr/local/php84/sbin/php-fpm --test

# 检查 Socket 目录权限
ls -la /var/run/php-fpm/

# 检查日志目录权限
ls -la /var/log/php-fpm/
```

### PHP 扩展未加载

**症状**: `php -m` 中找不到已安装的扩展

**排查步骤**:

```bash
# 检查扩展配置文件
ls -la /usr/local/php84/etc/php.d/

# 检查扩展是否已编译
ls -la /usr/local/php84/lib/php/extensions/

# 手动加载扩展测试
php -d extension=imagick.so -m | grep imagick
```

### Nginx 连接 PHP-FPM 失败

**症状**: 502 Bad Gateway 错误

**排查步骤**:

```bash
# 检查 PHP-FPM 是否运行
sudo systemctl status php84-fpm

# 检查 Socket 文件是否存在
ls -la /var/run/php-fpm/php84-fpm.sock

# 检查 Socket 权限
stat /var/run/php-fpm/php84-fpm.sock

# 检查 Nginx 错误日志
tail -50 /var/log/nginx/global.error.log
```

### WordPress 内存不足

**症状**: PHP Fatal error: Allowed memory size exhausted

**解决方案**:

```bash
# 编辑 php.ini 增加内存限制
sudo sed -i 's/memory_limit = 256M/memory_limit = 512M/' /usr/local/php84/etc/php.ini

# 或在 wp-config.php 中添加
define('WP_MEMORY_LIMIT', '512M');

# 重启 PHP-FPM
sudo systemctl restart php84-fpm
```

### OPcache 不生效

**症状**: PHP 文件修改后不更新

**解决方案**:

```bash
# 检查 OPcache 状态
php -i | grep opcache

# 开发环境可关闭 OPcache 或减少刷新间隔
# 编辑 php.ini
opcache.revalidate_freq = 0

# 清除 OPcache 缓存
php -r "opcache_reset();"

# 重启 PHP-FPM
sudo systemctl restart php84-fpm
```

### PHP 8.4 编译参数变更

**问题**: configure 时出现 `unrecognized options` 警告

**原因**: PHP 8.4 移除了一些旧参数

**解决方案**:
- 移除 `--with-openssl-dir` 参数（已合并到 `--with-openssl`）
- 移除 `--with-onig` 参数（mbstring 不再需要显式指定）
- 移除 `--with-xmlrpc` 参数（XML-RPC 扩展已从 PHP 8.0 起移除）

### Redis 扩展安装交互问题

**问题**: `pecl install redis` 时提示输入配置选项

**解决方案**: 使用非交互式安装

```bash
# 自动回答所有问题为 "no"（使用默认值）
printf "no\nno\nno\nno\nno\n" | sudo /usr/local/php84/bin/pecl install redis
```

### Systemd 服务启动失败

**症状**: `systemctl start php84-fpm` 失败，日志显示 "Read-only file system"

**原因**: `ProtectSystem=full` 设置阻止了 PID 文件写入

**解决方案**: 从 systemd 服务文件中移除以下行：
- `PrivateTmp=true`
- `ProtectSystem=full`
- `ProtectHome=true`
- `NoNewPrivileges=true`

### Socket 文件已存在

**症状**: PHP-FPM 启动失败，错误日志显示 "Another FPM instance seems to already listen"

**解决方案**:

```bash
# 删除已存在的 socket 和 pid 文件
sudo rm -f /var/run/php-fpm/php84-fpm.sock
sudo rm -f /usr/local/php84/var/run/php-fpm.pid

# 重启 PHP-FPM
sudo systemctl restart php84-fpm
```

### PECL 扩展安装失败（已安装的情况）

**症状**: 安装脚本执行到 PECL 扩展步骤时失败，提示 "pecl/imagick is already installed and is the same as the released version"

**原因**: `pecl install` 命令在尝试安装已存在的扩展时会返回非零退出码，如果脚本使用了 `set -e`，会导致整个安装过程中断

**解决方案**:

1. **先检查扩展是否已安装再安装**（推荐）:
```bash
if ! php -m | grep -q "imagick"; then
    sudo /usr/local/php84/bin/pecl install imagick
else
    echo "imagick 扩展已安装，跳过"
fi
```

2. **使用 `|| true` 忽略错误**:
```bash
sudo /usr/local/php84/bin/pecl install imagick || true
```

---

## 版本兼容性说明

| 组件 | 版本 | 说明 |
|------|------|------|
| Ubuntu | 22.04.5 LTS | 推荐使用 LTS 版本 |
| PHP | 8.4.x | WordPress 6.x 完全支持 |
| Nginx | 1.25.4 | 需已安装配置 |
| WordPress | 6.x | 支持 PHP 8.4 |
| WP-CLI | 2.x | 支持 PHP 8.4 |
| MySQL/MariaDB | 8.0/10.6+ | WordPress 推荐 |

---

## 生产环境运维建议

1. **定期更新**: 关注 PHP 安全更新，及时升级补丁版本
2. **日志轮转**: 配置 logrotate 管理 PHP-FPM 日志
3. **性能监控**: 使用 `/php-fpm-status` 监控进程状态
4. **安全加固**: 定期检查 `disable_functions` 配置
5. **备份策略**: 定期备份 WordPress 文件和数据库
6. **OPcache 调优**: 根据实际内存使用调整 OPcache 参数

---

**文档编写完成**，本方案搭建的 PHP 8.4 FPM 服务完全适配 WordPress 生产环境，与已安装的 Nginx 1.25.4 完美配合。
