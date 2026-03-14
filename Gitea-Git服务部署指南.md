# Gitea Git 服务部署指南

**文档版本**: V2.0  
**适配环境**: Ubuntu 22.04.5 LTS  
**核心版本**: Gitea 1.25.x  
**最后更新**: 2026-03-11

---

## 目录

1. [方案选型说明](#方案选型说明)
2. [快速部署（推荐）](#快速部署推荐)
3. [系统环境检查](#系统环境检查)
4. [数据库准备](#数据库准备)
5. [创建Git用户](#创建git用户)
6. [安装Gitea](#安装gitea)
7. [配置Systemd服务](#配置systemd服务)
8. [Nginx反向代理配置](#nginx反向代理配置)
9. [HTTPS配置说明](#ssl证书配置)
10. [初始化配置](#初始化配置)
11. [功能验证](#功能验证)
12. [常用命令速查](#常用命令速查)
13. [常见问题排查](#常见问题排查)
14. [修复和重新安装](#修复和重新安装)

---

## 方案选型说明

### 为什么选择 Gitea

| 方案 | 内存占用 | 特点 | 推荐场景 |
|------|---------|------|---------|
| **Gitea** | ~100MB | 轻量、Go编写、安装简单 | 小型团队、个人项目 |
| Gogs | ~50MB | 更轻量，但更新较慢 | 极低资源环境 |
| GitLab CE | ~4GB+ | 功能全面、企业级 | 大型团队、CI/CD需求 |
| GitLab EE | ~4GB+ | 企业功能、付费 | 企业级应用 |

**Gitea 是轻量级 Git 服务的理想选择，适合资源有限的环境。**

### 架构说明

```
用户请求 → Nginx (反向代理) → Gitea (3000端口) → MariaDB (数据库)
```

> **注意**: 当前配置为 HTTP 模式，后续可按需启用 HTTPS。

---

## 快速部署（推荐）

### 使用自动化脚本部署

我们提供了完善的自动化部署脚本，包含所有修复和优化：

```bash
# 进入部署脚本目录
cd /root/服务脚本库/部署脚本

# 运行 Gitea 部署脚本
./2..Gitea部署脚本.sh
```

脚本会自动完成：
- ✅ 系统环境检查
- ✅ Git 用户创建和目录权限设置
- ✅ Gitea 下载和安装
- ✅ Systemd 服务配置
- ✅ Nginx 反向代理配置
- ✅ 完整的配置文件生成

### 修复和重新安装

如果现有 Gitea 出现问题，可以使用修复脚本：

```bash
# 运行修复和重新安装脚本
cd /root/服务脚本库/部署脚本
./4.Gitea-修复和重新安装脚本.sh
```

此脚本会：
- 🔧 备份现有配置和数据
- 🧹 完全卸载现有 Gitea
- 🚀 重新安装并恢复数据
- 🔒 修复所有权限问题

---

## 系统环境检查

### 验证现有服务

```bash
# 检查 Nginx 状态
systemctl status nginx --no-pager

# 检查 MariaDB 状态
systemctl status mariadb --no-pager

# 检查 Git 版本
git --version

# 检查内存
free -h
```

### 安装依赖

```bash
apt update
apt install -y git curl wget
```

---

## 数据库准备

### 创建 Gitea 数据库和用户

```bash
# 登录 MariaDB
mysql -u root -p
```

执行以下 SQL：

```sql
-- 创建数据库（使用 utf8mb4 支持表情符号）
CREATE DATABASE gitea CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';

-- 创建专用用户
CREATE USER 'gitea'@'localhost' IDENTIFIED BY '你的强密码';

-- 授权
GRANT ALL PRIVILEGES ON gitea.* TO 'gitea'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;

-- 退出
EXIT;
```

### 验证数据库连接

```bash
mysql -u gitea -p -e "SHOW DATABASES;"
```

---

## 创建Git用户

### 创建专用系统用户

```bash
# 创建 git 用户（用于运行 Gitea 服务）
adduser --system --shell /bin/bash --gecos 'Git Version Control' \
    --group --disabled-password --home /home/git git

# 创建必要目录（修复：添加所有必要的子目录）
mkdir -p /home/git/{data,custom/conf,log}
mkdir -p /var/lib/gitea/{custom,data,log,cache,sessions,queue,indexers}
mkdir -p /var/lib/gitea/data/{repositories,lfs,attachments,avatars}
mkdir -p /var/lib/gitea/custom/{conf,public,templates}
mkdir -p /etc/gitea

# 设置正确的权限（修复：确保所有权限正确）
chown -R git:git /home/git
chmod -R 755 /home/git
chown -R git:git /var/lib/gitea
chmod -R 750 /var/lib/gitea
chmod 755 /var/lib/gitea/custom
chmod 755 /var/lib/gitea/custom/conf
chmod 755 /var/lib/gitea/custom/public
chmod 755 /var/lib/gitea/custom/templates
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

### 目录结构说明

| 目录路径 | 功能说明 | 权限 |
|---------|---------|------|
| `/home/git` | Git 用户主目录 | 755 |
| `/var/lib/gitea` | Gitea 工作目录 | 750 |
| `/var/lib/gitea/data` | 数据目录（仓库、附件等） | 750 |
| `/var/lib/gitea/custom` | 自定义配置和静态文件 | 755 |
| `/var/lib/gitea/custom/public` | 静态文件目录 | 755 |
| `/var/lib/gitea/custom/templates` | 模板文件目录 | 755 |
| `/etc/gitea` | 配置文件目录 | 770 |

---

## 安装Gitea

### 下载最新版本

```bash
# 创建安装目录
mkdir -p /service/gitea
cd /service/gitea

# 获取最新版本号（或指定版本）
GITEA_VERSION="1.25.4"

# 下载二进制文件（根据系统架构选择）
# x86_64 架构
wget -O gitea https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64

# 或使用 ARM64 架构
# wget -O gitea https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-arm64

# 设置权限
chmod +x gitea
chown git:git gitea

# 验证安装
./gitea --version
```

---

## 配置Systemd服务

### 创建服务文件（修复：添加环境变量）

```bash
cat > /etc/systemd/system/gitea.service << 'EOF'
[Unit]
Description=Gitea (Git with a cup of tea)
After=network.target

[Service]
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/service/gitea/gitea web -c /etc/gitea/app.ini
Restart=always
RestartSec=3

# 安全加固
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

# 资源限制
LimitNOFILE=524288:524288
LimitNPROC=512:512

# 环境变量（修复：添加必要的环境变量）
Environment=GITEA_WORK_DIR=/var/lib/gitea
Environment=GITEA_CUSTOM=/var/lib/gitea/custom
Environment=HOME=/home/git
Environment=USER=git

[Install]
WantedBy=multi-user.target
EOF
```

### 启用并启动服务

```bash
# 重载 systemd
systemctl daemon-reload

# 启用开机自启
systemctl enable gitea

# 启动服务
systemctl start gitea

# 检查状态
systemctl status gitea --no-pager
```

---

## Nginx反向代理配置

> **说明**: 当前仅配置 HTTP，HTTPS 证书可使用 `Nginx站点管理工具.sh` 完成。

### 创建站点配置（HTTP）

```bash
cat > /etc/nginx/sites-available/git.example.com.conf << 'EOF'
# Gitea Git 服务反向代理配置
# 域名: git.example.com

upstream gitea_backend {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    listen 80;
    listen [::]:80;
    server_name git.example.com;

    access_log /var/log/nginx/site/git.example.com.access.log;
    error_log /var/log/nginx/site/git.example.com.error.log;

    client_max_body_size 100M;

    location / {
        proxy_pass http://gitea_backend;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        proxy_buffering off;
        proxy_request_buffering off;
    }
}
EOF
```

### 创建日志目录

```bash
mkdir -p /var/log/nginx/site
touch /var/log/nginx/site/git.example.com.{access,error}.log
chown -R www-data:www-data /var/log/nginx/site
```

### 启用站点并测试

```bash
# 创建软链接
ln -s /etc/nginx/sites-available/git.example.com.conf /etc/nginx/sites-enabled/

# 测试配置
nginx -t

# 重载配置
systemctl reload nginx
```

---

## HTTPS配置说明

### 使用 Nginx 站点管理工具申请 SSL 证书

> **说明**: 推荐使用 `/root/服务脚本库/部署脚本/1. Nginx站点管理工具.sh` 完成 SSL 证书申请和配置。

### 步骤 1: 运行站点管理工具

```bash
# 运行 Nginx 站点管理工具
cd /root/服务脚本库/部署脚本
./1. Nginx站点管理工具.sh
```

### 步骤 2: 选择 "申请 SSL 证书" 选项

1. 在菜单中选择 **6. 申请 SSL 证书**
2. 选择您的 Gitea 站点域名（如 `git.example.com`）
3. 选择验证方式：
   - **1. 标准 HTTP 验证**：需要 80 端口可从外网访问
   - **2. 阿里云 DNS 验证**：推荐，无需开放端口（需要阿里云 API 密钥）
   - **3. Cloudflare DNS 验证**：推荐，无需开放端口（需要 Cloudflare API Token）

### 步骤 3: 等待证书申请完成

- 工具会自动安装所需依赖（如 certbot）
- 自动申请并配置 SSL 证书
- 自动更新 Nginx 配置以启用 HTTPS

### 步骤 4: 修改 Gitea 配置

配置 SSL 证书后，需要修改 Gitea 配置：

```bash
# 编辑配置文件
vi /etc/gitea/app.ini

# 修改以下配置
[server]
ROOT_URL = https://git.example.com/

# 重启服务
systemctl restart gitea
```

### 步骤 5: 验证 HTTPS 访问

- 浏览器访问 `https://git.example.com`
- 确认 SSL 证书状态为 "安全"
- 测试 Git 操作是否正常

---

## 初始化配置

### 首次访问 Web 界面

1. 浏览器访问 `http://git.example.com`
2. 首次访问会显示配置页面

### 数据库配置

| 配置项 | 值 |
|-------|-----|
| 数据库类型 | MySQL/MariaDB |
| 主机 | 127.0.0.1:3306 |
| 用户名 | gitea |
| 密码 | 你的数据库密码 |
| 数据库名称 | gitea |

### 基础配置

| 配置项 | 推荐值 |
|-------|-------|
| 域名 | git.example.com |
| SSH 端口 | 2222（或自定义） |
| HTTP 端口 | 3000 |
| 应用 URL | http://git.example.com/ |
| 日志路径 | /var/lib/gitea/log |

### 管理员账户

首次安装时创建管理员账户：
- 用户名：admin（或自定义）
- 密码：强密码
- 邮箱：管理员邮箱

### 可选配置

| 功能 | 建议 |
|------|------|
| 禁用用户注册 | 生产环境建议禁用 |
| 邮件服务 | 按需配置 SMTP |
| 第三方登录 | 按需配置 OAuth |

---

## 功能验证

### 检查服务状态

```bash
# 检查 Gitea 服务
systemctl status gitea --no-pager

# 检查端口监听
ss -tlnp | grep 3000

# 检查日志
tail -f /var/lib/gitea/log/gitea.log
```

### 测试 Git 操作

```bash
# 创建测试仓库目录
mkdir -p /tmp/test-repo && cd /tmp/test-repo

# 初始化仓库
git init

# 创建测试文件
echo "# Test Repository" > README.md
git add .
git commit -m "Initial commit"

# 添加远程仓库（替换为你的用户名）
git remote add origin http://git.example.com/username/test-repo.git

# 推送（会提示输入用户名密码）
git push -u origin main
```

### 测试 SSH 克隆

```bash
# SSH 克隆（需先配置 SSH 密钥）
# 如果使用非标准 SSH 端口（如 2222），需要指定端口：
git clone ssh://git@git.example.com:2222/username/test-repo.git

# 或在 ~/.ssh/config 中配置：
# Host git.example.com
#     Port 2222
#     User git
```

### SSH 配置文件（推荐）

```bash
# 编辑 SSH 配置文件
vi ~/.ssh/config

# 添加以下内容（根据实际端口调整）
Host git.example.com
    Port 2222
    User git
    IdentityFile ~/.ssh/id_rsa
```

---

## 常用命令速查

### 服务管理

```bash
# 启动服务
systemctl start gitea

# 停止服务
systemctl stop gitea

# 重启服务
systemctl restart gitea

# 查看状态
systemctl status gitea

# 查看日志
journalctl -u gitea -f
```

### 权限修复

```bash
# 修复目录权限（解决后台显示不全等问题）
chown -R git:git /var/lib/gitea
chmod -R 750 /var/lib/gitea
chmod 755 /var/lib/gitea/custom
chmod 755 /var/lib/gitea/custom/conf
chmod 755 /var/lib/gitea/custom/public
chmod 755 /var/lib/gitea/custom/templates
chown git:git /etc/gitea/app.ini
chmod 640 /etc/gitea/app.ini

# 重启服务
systemctl restart gitea
```

### 备份与恢复

```bash
# 备份（Gitea 内置命令）
su - git -c "/service/gitea/gitea dump -c /etc/gitea/app.ini"

# 备份文件位置
ls -la /var/lib/gitea/

# 恢复
# 1. 停止服务
# 2. 解压备份文件
# 3. 恢复数据目录和数据库
# 4. 启动服务
```

### 配置文件

```bash
# 主配置文件
/etc/gitea/app.ini

# 编辑配置后重启服务
systemctl restart gitea
```

---

## 常见问题排查

### 问题1：后台管理页面显示不全

**症状**：
- 后台管理页面样式错乱
- 部分功能无法显示
- 静态文件加载失败

**原因**：
- 目录权限不正确
- 静态文件目录不可访问

**解决方案**：

```bash
# 修复目录权限
chown -R git:git /var/lib/gitea
chmod -R 750 /var/lib/gitea
chmod 755 /var/lib/gitea/custom
chmod 755 /var/lib/gitea/custom/public
chmod 755 /var/lib/gitea/custom/templates

# 清除浏览器缓存后重试
systemctl restart gitea
```

### 问题2：迁移仓库时出现 404 错误

**症状**：
- 从 GitHub 迁移仓库时提示 404 Not Found
- 页面显示 "您正尝试访问的页面不存在"

**原因**：
- 权限不足
- CSRF 保护问题
- 会话配置不正确

**解决方案**：

1. **检查是否已登录**：
   - 确保使用管理员账户登录
   - 检查会话是否过期

2. **检查权限配置**：
   ```bash
   # 编辑配置文件
   vi /etc/gitea/app.ini
   
   # 确保以下配置正确
   [service]
   DISABLE_REGISTRATION = false
   REQUIRE_SIGNIN_VIEW = false
   ```

3. **检查会话配置**：
   ```bash
   # 确保会话目录存在且权限正确
   mkdir -p /var/lib/gitea/sessions
   chown -R git:git /var/lib/gitea/sessions
   ```

4. **重启服务**：
   ```bash
   systemctl restart gitea
   ```

### 问题3：无法访问 Web 界面

**排查步骤**：
```bash
# 检查 Gitea 服务状态
systemctl status gitea

# 检查端口监听
ss -tlnp | grep 3000

# 检查防火墙
ufw status

# 检查 Nginx 错误日志
tail -f /var/log/nginx/site/git.example.com.error.log
```

### 问题4：Git 推送失败

**可能原因**：
- 仓库权限问题
- 磁盘空间不足
- 客户端请求体大小限制

**解决方案**：
```bash
# 检查磁盘空间
df -h /home/git

# 检查仓库权限
ls -la /var/lib/gitea/data/repositories/

# 调整 Nginx 请求体大小（已在配置中设置 client_max_body_size）
```

### 问题5：SSH 连接失败

**排查步骤**：
```bash
# 检查 SSH 服务
systemctl status sshd

# 检查 Git 用户 SSH 配置
ls -la /home/git/.ssh/

# 测试 SSH 连接
ssh -vT git@git.example.com
```

### 问题6：数据库连接失败

**排查步骤**：
```bash
# 测试数据库连接
mysql -u gitea -p -h localhost gitea

# 检查 MariaDB 服务
systemctl status mariadb

# 检查 Gitea 日志
grep -i "database\|mysql\|error" /var/lib/gitea/log/gitea.log
```

---

## 修复和重新安装

### 何时需要重新安装

- Gitea 出现严重权限问题
- 静态文件损坏或缺失
- 配置文件混乱无法修复
- 需要清理并重新开始

### 使用修复脚本

我们提供了专门的修复和重新安装脚本：

```bash
# 进入部署脚本目录
cd /root/服务脚本库/部署脚本

# 运行修复脚本
./4.Gitea-修复和重新安装脚本.sh
```

脚本会执行以下操作：

1. **备份现有数据**：
   - 配置文件
   - 数据库
   - 仓库数据
   - 自定义文件

2. **完全卸载**：
   - 停止服务
   - 删除服务文件
   - 删除安装目录
   - 删除配置和数据目录

3. **重新安装**：
   - 创建 Git 用户
   - 创建完整的目录结构
   - 设置正确的权限
   - 下载并安装 Gitea
   - 配置 Systemd 服务
   - 恢复数据（可选）

4. **验证安装**：
   - 检查服务状态
   - 检查端口监听
   - 显示完成信息

### 手动修复步骤

如果无法使用脚本，可以手动修复：

```bash
# 1. 备份数据
cp -r /etc/gitea /root/gitea_backup_$(date +%Y%m%d)
cp -r /var/lib/gitea /root/gitea_data_backup_$(date +%Y%m%d)

# 2. 停止服务
systemctl stop gitea

# 3. 修复权限
chown -R git:git /var/lib/gitea
chmod -R 750 /var/lib/gitea
chmod 755 /var/lib/gitea/custom
chmod 755 /var/lib/gitea/custom/public
chmod 755 /var/lib/gitea/custom/templates
chown git:git /etc/gitea/app.ini
chmod 640 /etc/gitea/app.ini

# 4. 重启服务
systemctl start gitea

# 5. 检查状态
systemctl status gitea
```

---

## 附录：完整配置文件参考

### 修复后的 app.ini 配置示例

```ini
APP_NAME = Gitea: Git with a cup of tea
RUN_MODE = prod
WORK_PATH = /var/lib/gitea

[server]
PROTOCOL = http
DOMAIN = git.example.com
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3000
ROOT_URL = https://git.example.com/
DISABLE_SSH = false
SSH_PORT = 2222
START_SSH_SERVER = true
LFS_START_SERVER = true
OFFLINE_MODE = false
APP_DATA_PATH = /var/lib/gitea/data

[database]
DB_TYPE = mysql
HOST = 127.0.0.1:3306
NAME = gitea
USER = gitea
PASSWD = 你的密码
SCHEMA = 
SSL_MODE = disable
CHARSET = utf8mb4
PATH = 
LOG_SQL = false

[repository]
ROOT = /var/lib/gitea/data/repositories

[session]
PROVIDER = file
PROVIDER_CONFIG = /var/lib/gitea/sessions

[cache]
ADAPTER = file
PATH = /var/lib/gitea/cache

[queue]
TYPE = file
PATH = /var/lib/gitea/queue

[indexer]
ISSUE_INDEXER_PATH = /var/lib/gitea/indexers/issues.bleve
REPO_INDEXER_ENABLED = false

[log]
MODE = console, file
LEVEL = info
ROOT_PATH = /var/lib/gitea/log

[security]
INSTALL_LOCK = true
SECRET_KEY = 随机生成的密钥
INTERNAL_TOKEN = 随机生成的令牌
PASSWORD_HASH_ALGO = pbkdf2
MIN_PASSWORD_LENGTH = 8
PASSWORD_COMPLEXITY = off
SUCCESSFUL_TOKENS_CACHE_SIZE = 20

[service]
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = false
DISABLE_REGISTRATION = false
ALLOW_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
REQUIRE_SIGNIN_VIEW = false
DEFAULT_KEEP_EMAIL_PRIVATE = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true
DEFAULT_ENABLE_TIMETRACKING = true
NO_REPLY_ADDRESS = noreply.local

[mailer]
ENABLED = false

[openid]
ENABLE_OPENID_SIGNIN = false
ENABLE_OPENID_SIGNUP = false

[oauth2]
JWT_SECRET = 随机生成的密钥

[picture]
DISABLE_GRAVATAR = false
ENABLE_FEDERATED_AVATAR = false

[attachment]
ENABLED = true
PATH = /var/lib/gitea/data/attachments
ALLOWED_TYPES = image/jpeg|image/png|application/zip|application/gzip
MAX_SIZE = 4
MAX_FILES = 5
```

---

**文档编写**: Trae AI  
**最后更新**: 2026-03-11  
**版本**: V2.0（包含所有修复和优化）
