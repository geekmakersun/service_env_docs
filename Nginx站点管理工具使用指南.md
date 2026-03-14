# Nginx 站点管理工具使用指南

**文档版本**: V1.0  
**适配环境**: Ubuntu 22.04.5 LTS  
**配套脚本**: 4. Nginx站点管理工具.sh  
**用途**: 交互式创建和管理 Nginx 站点配置，支持多种应用类型和 SSL 证书

---

## 目录

1. [功能概述](#功能概述)
2. [前置要求](#前置要求)
3. [快速开始](#快速开始)
4. [功能详解](#功能详解)
5. [支持的PHP版本](#支持的php版本)
6. [应用类型说明](#应用类型说明)
7. [SSL证书配置](#ssl证书配置)
8. [常用操作示例](#常用操作示例)
9. [配置文件结构](#配置文件结构)
10. [常见问题排查](#常见问题排查)

---

## 功能概述

Nginx 站点管理工具是一个交互式脚本，提供以下功能：

| 功能 | 说明 |
|------|------|
| Git导入站点 | 从 Git 仓库克隆代码并自动创建站点配置 |
| 创建新站点 | 手动创建新站点，支持多种应用类型 |
| 删除站点 | 安全删除站点配置和网站文件 |
| 列出所有站点 | 显示所有已配置的站点列表 |
| 查看站点配置 | 查看指定站点的 Nginx 配置 |
| 申请 SSL 证书 | 为站点申请 Let's Encrypt SSL 证书 |
| 检测 PHP 版本 | 自动检测系统上所有可用的 PHP 版本 |
| 查看共用配置 | 查看 Nginx 共用配置片段 |
| 测试并重载 Nginx | 测试配置并平滑重载 Nginx 服务 |

---

## 前置要求

### 系统要求

- Ubuntu 22.04.5 LTS
- Nginx 已安装并运行
- PHP-FPM 已安装（如需运行 PHP 应用）
- root 权限

### 目录结构要求

脚本依赖以下目录结构：

```
/etc/nginx/
├── enabled/          # 启用的站点配置
├── snippets/         # 共用配置片段
└── ssl/             # SSL 证书目录

/var/www/            # 网站根目录
/var/log/nginx/site/ # 站点日志目录
```

### PHP-FPM 检测

脚本会自动检测以下位置的 PHP-FPM socket：
- `/run/php-fpm/php*-fpm.sock`
- `/var/run/php/php*-fpm.sock`

---

## 快速开始

### 启动工具

```bash
# 进入脚本目录
cd /root/服务脚本库/环境脚本

# 执行脚本
./4. Nginx站点管理工具.sh
```

### 主菜单

```
===========================================
  Nginx 站点管理工具
===========================================

  1. Git导入站点
  2. 创建新站点
  3. 删除站点
  4. 列出所有站点
  5. 查看站点配置
  6. 申请 SSL 证书
  7. 检测 PHP 版本
  8. 查看共用配置
  9. 测试并重载 Nginx
  0. 退出
```

---

## 功能详解

### 1. Git导入站点

从 Git 仓库自动导入站点代码并创建配置。

**操作流程：**
1. 输入域名（如：`example.com`）
2. 输入 Git 仓库地址（支持 HTTPS 和 SSH）
3. 选择应用类型
4. 选择 PHP 版本（PHP 应用）
5. 选择是否启用 SSL

**示例：**
```
请输入域名: example.com
请输入Git仓库地址: https://github.com/user/repo.git
请选择应用类型:
  1. 纯静态网站
  2. WordPress
  3. ThinkPHP
  ...
```

### 2. 创建新站点

手动创建新的站点配置。

**操作流程：**
1. 输入域名
2. 选择应用类型
3. 选择 PHP 版本（PHP 应用）
4. 选择是否启用 SSL
5. 确认创建

### 3. 删除站点

安全删除站点配置和相关文件。

**注意：**
- 会删除 `/etc/nginx/enabled/` 下的配置文件
- 会删除 `/var/www/` 下的网站文件
- 会删除 `/var/log/nginx/site/` 下的日志文件
- **操作前会要求确认**

### 4. 列出所有站点

显示所有已配置的站点：
- 站点域名
- 配置文件路径
- 网站根目录

### 5. 查看站点配置

查看指定站点的完整 Nginx 配置。

### 6. 申请 SSL 证书

为已有站点申请 Let's Encrypt SSL 证书。

**要求：**
- 域名必须已解析到服务器
- 80 端口必须可访问
- 站点配置必须存在

### 7. 检测 PHP 版本

自动检测系统上所有可用的 PHP-FPM 版本。

### 8. 查看共用配置

查看 Nginx snippets 目录下的共用配置片段。

### 9. 测试并重载 Nginx

测试 Nginx 配置语法并平滑重载服务。

---

## 支持的PHP版本

脚本会自动检测系统中安装的 PHP 版本。支持的 PHP 版本包括：

- PHP 7.4
- PHP 8.0
- PHP 8.1
- PHP 8.2
- PHP 8.3
- PHP 8.4

**PHP-FPM Socket 路径：**
```
/run/php-fpm/php84-fpm.sock
/run/php-fpm/php82-fpm.sock
/var/run/php/php8.4-fpm.sock
```

---

## 应用类型说明

| 类型 | 说明 | 伪静态规则 |
|------|------|-----------|
| 纯静态网站 | HTML/CSS/JS 静态页面 | 无 |
| WordPress | WordPress 博客/CMS | `try_files $uri $uri/ /index.php?$args;` |
| ThinkPHP | ThinkPHP 框架 | `rewrite ^(.*)$ /index.php?s=$1 last;` |
| Laravel | Laravel 框架 | `try_files $uri $uri/ /index.php?$query_string;` |
| Vue/React SPA | 单页应用 | `try_files $uri $uri/ /index.html;` |
| Typecho | Typecho 博客 | 自动判断 index.html/index.php |
| Discuz | Discuz 论坛 | Discuz 专用伪静态规则 |
| 迅睿CMS | 迅睿CMS系统 | 使用 public 目录作为根目录 |
| 自定义PHP | 通用PHP应用 | 标准 PHP 伪静态 |

### WordPress 安全设置

选择 WordPress 类型时，会自动包含以下安全配置：
```nginx
include snippets/wordpress-security.conf;
```

包含的安全规则：
- 保护 wp-config.php
- 禁止访问敏感文件
- 防止目录遍历
- 限制 XML-RPC 访问

### 迅睿CMS 特殊处理

迅睿CMS 使用 `public` 目录作为 Web 根目录：
```nginx
root /var/www/example.com/public;
```

---

## SSL证书配置

### 证书申请流程

1. 选择 "申请 SSL 证书" 功能
2. 输入域名
3. 脚本会自动：
   - 验证域名解析
   - 使用 Certbot 申请证书
   - 更新 Nginx 配置
   - 重载 Nginx 服务

### 证书存储位置

```
/etc/letsencrypt/live/example.com/
├── fullchain.pem    # 完整证书链
└── privkey.pem      # 私钥
```

### 自动续期

Let's Encrypt 证书有效期为 90 天，建议设置自动续期：

```bash
# 测试续期
certbot renew --dry-run

# 设置定时任务
echo "0 3 * * * /usr/bin/certbot renew --quiet" | crontab -
```

---

## 常用操作示例

### 示例1：创建 WordPress 站点

```bash
# 启动工具
./4. Nginx站点管理工具.sh

# 选择: 2. 创建新站点
# 输入域名: blog.example.com
# 选择应用类型: 2. WordPress
# 选择PHP版本: php84
# 启用SSL: 否（后续可申请）
```

### 示例2：从 Git 导入 Laravel 项目

```bash
# 启动工具
./4. Nginx站点管理工具.sh

# 选择: 1. Git导入站点
# 输入域名: app.example.com
# Git仓库: git@github.com:user/laravel-app.git
# 选择应用类型: 4. Laravel
# 选择PHP版本: php84
# 启用SSL: 是
```

### 示例3：为已有站点添加 SSL

```bash
# 启动工具
./4. Nginx站点管理工具.sh

# 选择: 6. 申请 SSL 证书
# 输入域名: example.com
# 脚本自动申请并配置证书
```

---

## 配置文件结构

### 生成的站点配置示例

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /var/www/example.com;
    index index.php index.html index.htm;

    access_log /var/log/nginx/site/example.com.access.log main;
    error_log /var/log/nginx/site/example.com.error.log warn;

    # 错误页面处理
    include snippets/error-pages.conf;

    # 伪静态规则（根据应用类型自动生成）
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP 处理
    include snippets/php84.conf;

    # 安全设置
    include snippets/security.conf;
}
```

### SSL 配置示例

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # ... 其他配置
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

---

## 常见问题排查

### 问题1：PHP-FPM 版本未检测到

**现象：** 脚本提示 "未检测到 PHP-FPM"

**解决：**
```bash
# 检查 PHP-FPM 是否运行
systemctl status php84-fpm

# 检查 socket 文件是否存在
ls -la /run/php-fpm/

# 如果没有运行，启动服务
systemctl start php84-fpm
systemctl enable php84-fpm
```

### 问题2：Nginx 配置测试失败

**现象：** 创建站点后提示配置错误

**解决：**
```bash
# 手动测试配置
nginx -t

# 查看详细错误信息
cat /var/log/nginx/error.log
```

### 问题3：SSL 证书申请失败

**现象：** Certbot 报错

**可能原因：**
- 域名未解析到服务器
- 80 端口被防火墙阻挡
- 已有其他进程占用 80 端口

**解决：**
```bash
# 检查域名解析
dig example.com

# 检查端口
netstat -tlnp | grep :80

# 手动申请证书（调试用）
certbot --nginx -d example.com --dry-run
```

### 问题4：站点无法访问

**排查步骤：**

1. 检查 Nginx 是否运行：
```bash
systemctl status nginx
```

2. 检查站点配置是否存在：
```bash
ls -la /etc/nginx/enabled/
```

3. 检查网站文件是否存在：
```bash
ls -la /var/www/example.com/
```

4. 查看错误日志：
```bash
tail -f /var/log/nginx/site/example.com.error.log
```

### 问题5：权限错误

**现象：** 403 Forbidden 错误

**解决：**
```bash
# 修复网站目录权限
chown -R www-data:www-data /var/www/example.com
chmod -R 755 /var/www/example.com

# 如果是 PHP 应用，确保 PHP-FPM 用户正确
# 编辑 /usr/local/php84/etc/php-fpm.d/www.conf
# user = www-data
# group = www-data
```

---

## 相关文档

- [Nginx-ModSecurity-部署指南.md](./Nginx-ModSecurity-部署指南.md)
- [PHP-8.4-FPM-编译安装指南.md](./PHP-8.4-FPM-编译安装指南.md)
- [MariaDB-APT安装指南.md](./MariaDB-APT安装指南.md)
- [单域名证书申请指南.md](./单域名证书申请指南.md)
