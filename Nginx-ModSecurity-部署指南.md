# Nginx + ModSecurity 编译安装部署指南

**文档版本**: V6.0  
**适配环境**: Ubuntu 22.04.5 LTS  
**核心版本**: Nginx 1.25.4、ModSecurity 3.0.12  
**核心调整**: 拆分为独立的 Nginx 安装脚本和 ModSecurity 安装脚本，使用自定义安全规则（不依赖 OWASP CRS），配置自定义错误页面，支持不同攻击类型显示对应错误页面**

---

## 目录

1. [系统准备](#系统准备)
2. [安装方式选择](#安装方式选择)
3. [单独安装 Nginx](#单独安装-nginx)
4. [单独安装 ModSecurity](#单独安装-modsecurity)
5. [Nginx 集成 ModSecurity](#nginx-集成-modsecurity)
6. [集成 OWASP CRS 规则集](#集成-owasp-crs-规则集)
7. [配置 ModSecurity](#配置-modsecurity)
8. [配置 Nginx](#配置-nginx)
9. [配置自定义错误页面](#配置自定义错误页面)
10. [根据攻击类型显示不同错误页面](#根据攻击类型显示不同错误页面)
11. [配置 Systemd 服务](#配置-systemd-服务)
12. [创建默认站点](#创建默认站点)
13. [功能验证](#功能验证)
14. [多站点管理](#多站点管理)
15. [HTTPS 部署](#https-部署)
16. [常见问题排查](#常见问题排查)
17. [版本兼容性说明](#版本兼容性说明)
18. [日志配置与管理](#日志配置与管理)
19. [常用命令速查](#常用命令速查)
20. [生产环境运维建议](#生产环境运维建议)

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

---

## 安装方式选择

本指南提供两种安装方式：

### 方式一：单独安装 Nginx

使用独立的 Nginx 安装脚本，仅安装 Nginx 服务，不包含 ModSecurity。

**适用场景**：
- 仅需要 Nginx 作为 Web 服务器
- 后续可能需要单独安装 ModSecurity
- 对系统资源有较高要求

### 方式二：单独安装 ModSecurity

使用独立的 ModSecurity 安装脚本，仅安装 ModSecurity 库和规则，不包含 Nginx。

**适用场景**：
- 已安装 Nginx，需要添加 WAF 功能
- 调试 ModSecurity 规则
- 与其他 Web 服务器配合使用

### 方式三：Nginx 集成 ModSecurity

先安装 ModSecurity，然后重新编译 Nginx 并集成 ModSecurity 模块。

**适用场景**：
- 需要完整的 WAF 保护
- 生产环境部署
- 对安全性有较高要求

---

## 单独安装 Nginx

使用独立的 Nginx 安装脚本进行安装：

```bash
# 运行 Nginx 安装脚本
sudo bash /root/服务脚本库/执行脚本/Nginx安装脚本.sh
```

### 安装流程

1. 检查系统环境和权限
2. 安装编译依赖
3. 创建目录结构
4. 下载 Nginx 源码
5. 编译安装 Nginx
6. 配置 Nginx
7. 安装错误页面
8. 创建 Systemd 服务
9. 配置日志轮转
10. 验证安装并启动服务

### 验证安装

```bash
# 检查 Nginx 版本
nginx -v

# 检查服务状态
systemctl status nginx

# 测试访问
curl http://localhost/
```

---

## 单独安装 ModSecurity

使用独立的 ModSecurity 安装脚本进行安装：

```bash
# 运行 ModSecurity 安装脚本
sudo bash /root/服务脚本库/执行脚本/ModSecurity安装脚本.sh
```

### 安装流程

1. 检查系统环境和权限
2. 安装编译依赖
3. 创建目录结构
4. 下载 ModSecurity 源码和 Nginx 连接器
5. 编译安装 ModSecurity
6. 下载 CRS 规则集
7. 配置 ModSecurity
8. 验证安装

### 验证安装

```bash
# 检查 ModSecurity 库是否加载
ldconfig -p | grep modsecurity

# 检查配置文件是否存在
ls -la /etc/nginx/modsecurity/
```

---

## Nginx 集成 ModSecurity

### 步骤 1：安装 ModSecurity

首先按照上述步骤安装 ModSecurity。

### 步骤 2：重新编译 Nginx

需要重新编译 Nginx 并添加 ModSecurity 模块：

```bash
# 进入 Nginx 源码目录
cd /usr/local/src/nginx-1.25.4

# 清理之前的编译
make clean

# 配置编译参数，添加 ModSecurity 模块
auto/configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --modules-path=/usr/lib/nginx/modules \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --http-client-body-temp-path=/var/cache/nginx/client_temp \
    --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
    --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
    --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
    --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
    --user=www-data \
    --group=www-data \
    --with-compat \
    --with-file-aio \
    --with-threads \
    --with-http_addition_module \
    --with-http_auth_request_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_mp4_module \
    --with-http_random_index_module \
    --with-http_realip_module \
    --with-http_secure_link_module \
    --with-http_slice_module \
    --with-http_ssl_module \
    --with-http_stub_status_module \
    --with-http_sub_module \
    --with-http_v2_module \
    --with-http_v3_module \
    --with-mail \
    --with-mail_ssl_module \
    --with-stream \
    --with-stream_realip_module \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --add-module=/usr/local/src/modsecurity-nginx-v1.0.3 \
    --with-cc-opt="-I/usr/local/modsecurity/include -O2 -fstack-protector-strong -Wformat -Werror=format-security" \
    --with-ld-opt="-L/usr/local/modsecurity/lib -Wl,-rpath,/usr/local/modsecurity/lib"

# 编译安装
make -j$(nproc)
sudo make install
```

### 步骤 3：在 Nginx 配置中启用 ModSecurity

编辑 Nginx 主配置文件：

```bash
sudo vim /etc/nginx/nginx.conf
```

在 `http` 块中添加以下配置：

```nginx
# ModSecurity WAF 配置
modsecurity on;
modsecurity_rules_file /etc/nginx/modsecurity/modsec.conf;
```

### 步骤 4：验证集成

```bash
# 检查 Nginx 版本和模块
nginx -V 2>&1 | grep -i modsecurity

# 测试配置语法
nginx -t

# 重启 Nginx 服务
sudo systemctl restart nginx

# 测试 ModSecurity 是否生效
curl -s -o /dev/null -w "%{http_code}" "http://localhost/?test=<script>alert(1)</script>"
```

---

## 集成 OWASP CRS 规则集

**说明**：本方案使用自定义安全规则，不依赖 OWASP CRS。以下内容仅供参考，如需使用 OWASP CRS 可自行安装。

OWASP CRS 为通用 Web 应用防护规则集，是 ModSecurity 实现 XSS、SQL 注入拦截的核心。

```bash
cd /etc/nginx/modsecurity

# 拉取 3.3.4 版本规则集
git clone --depth 1 --branch v3.3.4 https://github.com/coreruleset/coreruleset.git rules
```

### 克隆失败处理：手动下载压缩包方式

```bash
cd /etc/nginx/modsecurity
wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v3.3.4.tar.gz -O crs-3.3.4.tar.gz
tar -zxvf crs-3.3.4.tar.gz
mv coreruleset-3.3.4 rules
```

---

## 配置 ModSecurity

### 创建 ModSecurity 主配置文件

删除废弃指令，关闭调试日志，开启主动拦截模式：

```bash
sudo tee /etc/nginx/modsecurity/modsec.conf << 'EOF'
# ModSecurity 3.0.12 核心配置
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecResponseBodyMimeType text/plain text/html text/xml application/json

# 临时文件目录（必配，确保 www-data 有读写权限）
SecDataDir /etc/nginx/modsecurity/tmp
SecTmpDir /etc/nginx/modsecurity/tmp

# 日志配置（3.x 兼容，删除废弃 SecLogDir）
SecDebugLog /var/log/nginx/modsecurity/debug.log
SecDebugLogLevel 0
SecAuditLog /var/log/nginx/modsecurity/audit.log
SecAuditLogFormat JSON
SecAuditLogType Serial

# 加载自定义规则
Include /etc/nginx/modsecurity/custom.conf

# 可选：添加误拦截放行规则（根据业务需求调整）
# SecRule REQUEST_URI "@beginsWith /api/health" "id:100,phase:1,allow,nolog"
EOF
```

### 创建自定义安全规则

创建自定义安全规则文件，包含 XSS、SQL 注入、命令注入、文件包含、路径遍历等攻击检测：

```bash
sudo tee /etc/nginx/modsecurity/custom.conf << 'EOF'
# ===========================================
# ModSecurity 自定义安全规则
# 规则 ID 范围: 100000-199999
# ===========================================

# ----------
# XSS 攻击检测 (规则 ID: 100001-100099)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx <script[\s\S]*?>|javascript:|on\w+\s*=" \
    "id:100001,phase:2,deny,status:403,log,msg:'XSS Attack Detected',setenv:MODSEC_ATTACK_TYPE=xss"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx <iframe|<object|<embed|<svg[\s\S]*?onload" \
    "id:100002,phase:2,deny,status:403,log,msg:'XSS Attack Detected (iframe/object/embed)',setenv:MODSEC_ATTACK_TYPE=xss"

# ----------
# SQL 注入检测 (规则 ID: 100100-100199)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(union\s+select|select\s+from|insert\s+into|delete\s+from|drop\s+table|update\s+\w+\s+set)" \
    "id:100100,phase:2,deny,status:403,log,msg:'SQL Injection Detected',setenv:MODSEC_ATTACK_TYPE=sql_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(or\s+1\s*=\s*1|and\s+1\s*=\s*1|'\s*or\s+'|\"\s*or\s+\")" \
    "id:100101,phase:2,deny,status:403,log,msg:'SQL Injection Detected (Boolean-based)',setenv:MODSEC_ATTACK_TYPE=sql_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(exec\s*\(|execute\s*\(|xp_cmdshell|sp_executesql)" \
    "id:100102,phase:2,deny,status:403,log,msg:'SQL Injection Detected (Command execution)',setenv:MODSEC_ATTACK_TYPE=sql_injection"

# ----------
# 命令注入检测 (规则 ID: 100200-100299)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(;|\||`|\$\(|\$\{)\s*(ls|cat|pwd|whoami|id|uname|wget|curl|nc|bash|sh|python|perl|ruby|php)" \
    "id:100200,phase:2,deny,status:403,log,msg:'Command Injection Detected',setenv:MODSEC_ATTACK_TYPE=command_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(\|\|.*\||&&.*&|\$\(|`.*`)" \
    "id:100201,phase:2,deny,status:403,log,msg:'Command Injection Detected (Shell metacharacters)',setenv:MODSEC_ATTACK_TYPE=command_injection"

# ----------
# 文件包含攻击检测 (规则 ID: 100300-100399)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(php://|file://|expect://|data://|zip://|phar://)" \
    "id:100300,phase:2,deny,status:403,log,msg:'File Inclusion Detected (Protocol wrapper)',setenv:MODSEC_ATTACK_TYPE=file_inclusion"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(include\s*\(|require\s*\(|include_once\s*\(|require_once\s*\()" \
    "id:100301,phase:2,deny,status:403,log,msg:'File Inclusion Detected (PHP functions)',setenv:MODSEC_ATTACK_TYPE=file_inclusion"

# ----------
# 路径遍历攻击检测 (规则 ID: 100400-100499)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx \.\./|\.\.\\" \
    "id:100400,phase:2,deny,status:403,log,msg:'Path Traversal Detected',setenv:MODSEC_ATTACK_TYPE=path_traversal"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(/etc/passwd|/etc/shadow|/etc/hosts|/proc/self|/var/log)" \
    "id:100401,phase:2,deny,status:403,log,msg:'Path Traversal Detected (Sensitive file access)',setenv:MODSEC_ATTACK_TYPE=path_traversal"

# ----------
# 敏感文件访问检测 (规则 ID: 100500-100599)
# ----------
SecRule REQUEST_URI "@rx (?i)\.(env|git|svn|bak|backup|sql|conf|config|ini|log|sh|py|pl|rb)$" \
    "id:100500,phase:2,deny,status:403,log,msg:'Sensitive File Access Detected',setenv:MODSEC_ATTACK_TYPE=sensitive_file"

SecRule REQUEST_URI "@rx (?i)(\.htaccess|\.htpasswd|web\.config|\.DS_Store)" \
    "id:100501,phase:2,deny,status:403,log,msg:'Sensitive File Access Detected (Config files)',setenv:MODSEC_ATTACK_TYPE=sensitive_file"

# ----------
# 敏感路径访问检测 (规则 ID: 100600-100699)
# ----------
SecRule REQUEST_URI "@rx (?i)^/(admin|manager|phpmyadmin|mysql|backup|config|test|tmp|debug)" \
    "id:100600,phase:2,deny,status:403,log,msg:'Sensitive Path Access Detected',setenv:MODSEC_ATTACK_TYPE=sensitive_path"

# ----------
# 恶意编码检测 (规则 ID: 100700-100799)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(%3Cscript|%3C/script|%3Ciframe|%3Cobject|%3Cembed|%253C)" \
    "id:100700,phase:2,deny,status:403,log,msg:'Malicious Encoding Detected',setenv:MODSEC_ATTACK_TYPE=malicious_encoding"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(0x[0-9a-f]+|\\x[0-9a-f]{2}|\\u[0-9a-f]{4})" \
    "id:100701,phase:2,deny,status:403,log,msg:'Malicious Encoding Detected (Hex/Unicode)',setenv:MODSEC_ATTACK_TYPE=malicious_encoding"

# ----------
# 默认规则 (规则 ID: 199999)
# ----------
# 如果以上规则都没有匹配，但请求被拦截，使用通用错误页面
SecRule REQUEST_URI "@rx .*" \
    "id:199999,phase:5,pass,nolog,setenv:MODSEC_ATTACK_TYPE=general"
EOF
```

### 配置文件权限

确保 www-data 用户对 ModSecurity 配置文件有读写权限：

```bash
sudo chown www-data:www-data /etc/nginx/modsecurity/modsec.conf
sudo chmod 640 /etc/nginx/modsecurity/modsec.conf

sudo chown www-data:www-data /etc/nginx/modsecurity/custom.conf
sudo chmod 640 /etc/nginx/modsecurity/custom.conf
```

---

## 配置 Nginx

集成 ModSecurity，实现多站点管理，预留 SSL/HTTPS 配置，自定义错误页面：

```bash
sudo tee /etc/nginx/nginx.conf << EOF
# Nginx 主配置文件
# 运行用户，www-data 是 Debian/Ubuntu 系统默认的 Web 服务用户
user  www-data;

# 工作进程数，auto 表示自动检测 CPU 核心数
worker_processes  auto;

# 错误日志路径和级别：debug/info/notice/warn/error/crit
error_log  /var/log/nginx/global.error.log warn;

# 主进程 PID 文件位置
pid        /var/run/nginx.pid;

# 事件模块配置
events {
    # 单个工作进程的最大连接数
    worker_connections  1024;
    # 使用 epoll 事件模型（Linux 高性能 I/O 多路复用）
    use epoll;
    # 允许一个进程同时接受多个新连接
    multi_accept on;
}

# HTTP 服务配置
http {
    # 加载 MIME 类型映射表
    include       /etc/nginx/mime.types;
    # 默认文件类型（二进制流，浏览器会下载而非显示）
    default_type  application/octet-stream;

    # 自定义访问日志格式
    # $remote_addr: 客户端 IP
    # $remote_user: 客户端用户名
    # $time_local: 访问时间
    # $request: 请求方法和路径
    # $status: 响应状态码
    # $body_bytes_sent: 响应体大小
    # $http_referer: 来源页面
    # $http_user_agent: 客户端浏览器信息
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # 访问日志路径，使用 main 格式
    access_log  /var/log/nginx/global.access.log  main;

    # ========== 性能优化配置 ==========
    # 启用 sendfile，零拷贝方式传输文件，提高性能
    sendfile        on;
    # 在 sendfile 开启时，合并数据包一次性发送
    tcp_nopush      on;
    # 禁用 Nagle 算法，减少网络延迟
    tcp_nodelay     on;
    # 长连接超时时间（秒）
    keepalive_timeout  65;
    # MIME 类型哈希表最大大小
    types_hash_max_size 2048;
    # 请求体最大大小（上传文件限制）
    client_max_body_size 100M;
    # 隐藏 Nginx 版本号，增强安全性
    server_tokens off;

    # ========== ModSecurity WAF 配置 ==========
    # 开启 ModSecurity Web 应用防火墙
    modsecurity on;
    # ModSecurity 规则配置文件路径
    modsecurity_rules_file /etc/nginx/modsecurity/modsec.conf;

    # ========== 根据攻击类型映射错误页面 ==========
    # ModSecurity 通过环境变量 MODSEC_ATTACK_TYPE 传递攻击类型
    # map 指令根据攻击类型返回对应的错误页面路径
    map $modsec_attack_type $error_page_path {
        default                /error-html/403_通用安全威胁.html;
        xss                    /error-html/403_XSS攻击.html;
        sql_injection          /error-html/403_SQL注入攻击.html;
        command_injection      /error-html/403_命令注入攻击.html;
        file_inclusion         /error-html/403_文件包含攻击.html;
        path_traversal         /error-html/403_路径遍历攻击.html;
        sensitive_file         /error-html/403_敏感文件访问.html;
        sensitive_path         /error-html/403_敏感路径访问.html;
        malicious_encoding     /error-html/403_恶意编码攻击.html;
        general                /error-html/403_通用安全威胁.html;
    }

    # ========== 自定义错误页面 ==========
    # 400 错误请求
    error_page 400 /error-html/400_错误请求.html;
    # 401 未授权
    error_page 401 /error-html/401_未授权.html;
    # 403 安全威胁拦截（基于攻击类型显示不同页面）
    error_page 403 @custom_403;
    # 404 资源未找到
    error_page 404 /error-html/404_资源未找到.html;
    # 500 服务器内部错误
    error_page 500 /error-html/500_服务器错误.html;
    # 502 错误网关
    error_page 502 /error-html/502_错误网关.html;
    # 503 服务不可用
    error_page 503 /error-html/503_服务不可用.html;
    # 504 网关超时
    error_page 504 /error-html/504_网关超时.html;

    # ========== 错误页面处理（所有站点共用） ==========
    # 捕获 ModSecurity 设置的环境变量
    map $MODSEC_ATTACK_TYPE $modsec_attack_type {
        default $MODSEC_ATTACK_TYPE;
    }

    # ========== 多站点管理 ==========
    # 仅加载 enabled 目录下的站点配置（软链接方式管理站点）
    include /etc/nginx/sites-enabled/*.conf;

    # ========== SSL/TLS 基础配置（预留，站点可直接使用） ==========
    # 支持的 TLS 协议版本，仅启用安全的 TLS 1.2 和 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    # 优先使用服务器端的加密套件顺序
    ssl_prefer_server_ciphers on;
    # 推荐的安全加密套件（支持前向保密）
    # ECDHE/DHE: 密钥交换算法（支持前向保密）
    # AES256-GCM: 对称加密算法（高强度）
    # SHA384/SHA512: 消息认证码算法
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    # SSL 会话超时时间
    ssl_session_timeout 1d;
    # SSL 会话缓存，shared:SSL 表示所有工作进程共享，10m 缓存大小
    ssl_session_cache shared:SSL:10m;
    # 禁用 SSL 会话票据，增强前向保密安全性
    ssl_session_tickets off;
}
EOF
```

### 验证配置语法

```bash
nginx -t
```

**预期输出**: `syntax is ok`

---

## （可选）集成 OWASP CRS 规则集

**说明**：本方案使用自定义安全规则，不依赖 OWASP CRS。以下内容仅供参考，如需使用 OWASP CRS 可自行安装。

OWASP CRS 为通用 Web 应用防护规则集，是 ModSecurity 实现 XSS、SQL 注入拦截的核心。

```bash
cd /etc/nginx/modsecurity

# 拉取 3.3.4 版本规则集
git clone --depth 1 --branch v3.3.4 https://github.com/coreruleset/coreruleset.git rules
```

### 克隆失败处理：手动下载压缩包方式

```bash
cd /etc/nginx/modsecurity
wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v3.3.4.tar.gz -O crs-3.3.4.tar.gz
tar -zxvf crs-3.3.4.tar.gz
mv coreruleset-3.3.4 rules
```

---

## 配置 ModSecurity

### 创建 ModSecurity 主配置文件

删除废弃指令，关闭调试日志，开启主动拦截模式：

```bash
sudo tee /etc/nginx/modsecurity/modsec.conf << 'EOF'
# ModSecurity 3.0.10 核心配置
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecResponseBodyMimeType text/plain text/html text/xml application/json

# 临时文件目录（必配，确保 www-data 有读写权限）
SecDataDir /etc/nginx/modsecurity/tmp
SecTmpDir /etc/nginx/modsecurity/tmp

# 日志配置（3.x 兼容，删除废弃 SecLogDir）
SecDebugLog /var/log/nginx/modsecurity/debug.log
SecDebugLogLevel 0
SecAuditLog /var/log/nginx/modsecurity/audit.log
SecAuditLogFormat JSON
SecAuditLogType Serial

# 加载自定义规则
Include /etc/nginx/modsecurity/custom.conf

# 可选：添加误拦截放行规则（根据业务需求调整）
# SecRule REQUEST_URI "@beginsWith /api/health" "id:100,phase:1,allow,nolog"
EOF
```

### 创建自定义安全规则

创建自定义安全规则文件，包含 XSS、SQL 注入、命令注入、文件包含、路径遍历等攻击检测：

```bash
sudo tee /etc/nginx/modsecurity/custom.conf << 'EOF'
# ============================================
# ModSecurity 自定义安全规则
# 规则 ID 范围: 100000-199999
# ============================================

# ----------
# XSS 攻击检测 (规则 ID: 100001-100099)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx <script[\s\S]*?>|javascript:|on\w+\s*=" \
    "id:100001,phase:2,deny,status:403,log,msg:'XSS Attack Detected',setenv:MODSEC_ATTACK_TYPE=xss"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx <iframe|<object|<embed|<svg[\s\S]*?onload" \
    "id:100002,phase:2,deny,status:403,log,msg:'XSS Attack Detected (iframe/object/embed)',setenv:MODSEC_ATTACK_TYPE=xss"

# ----------
# SQL 注入检测 (规则 ID: 100100-100199)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(union\s+select|select\s+from|insert\s+into|delete\s+from|drop\s+table|update\s+\w+\s+set)" \
    "id:100100,phase:2,deny,status:403,log,msg:'SQL Injection Detected',setenv:MODSEC_ATTACK_TYPE=sql_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(or\s+1\s*=\s*1|and\s+1\s*=\s*1|'\s*or\s+'|\"\s*or\s+\")" \
    "id:100101,phase:2,deny,status:403,log,msg:'SQL Injection Detected (Boolean-based)',setenv:MODSEC_ATTACK_TYPE=sql_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(exec\s*\(|execute\s*\(|xp_cmdshell|sp_executesql)" \
    "id:100102,phase:2,deny,status:403,log,msg:'SQL Injection Detected (Command execution)',setenv:MODSEC_ATTACK_TYPE=sql_injection"

# ----------
# 命令注入检测 (规则 ID: 100200-100299)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(;|\||`|\$\(|\$\{)\s*(ls|cat|pwd|whoami|id|uname|wget|curl|nc|bash|sh|python|perl|ruby|php)" \
    "id:100200,phase:2,deny,status:403,log,msg:'Command Injection Detected',setenv:MODSEC_ATTACK_TYPE=command_injection"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(\|\|.*\||&&.*&|\$\(|`.*`)" \
    "id:100201,phase:2,deny,status:403,log,msg:'Command Injection Detected (Shell metacharacters)',setenv:MODSEC_ATTACK_TYPE=command_injection"

# ----------
# 文件包含攻击检测 (规则 ID: 100300-100399)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(php://|file://|expect://|data://|zip://|phar://)" \
    "id:100300,phase:2,deny,status:403,log,msg:'File Inclusion Detected (Protocol wrapper)',setenv:MODSEC_ATTACK_TYPE=file_inclusion"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(include\s*\(|require\s*\(|include_once\s*\(|require_once\s*\()" \
    "id:100301,phase:2,deny,status:403,log,msg:'File Inclusion Detected (PHP functions)',setenv:MODSEC_ATTACK_TYPE=file_inclusion"

# ----------
# 路径遍历攻击检测 (规则 ID: 100400-100499)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx \.\./|\.\.\\\\" \
    "id:100400,phase:2,deny,status:403,log,msg:'Path Traversal Detected',setenv:MODSEC_ATTACK_TYPE=path_traversal"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(/etc/passwd|/etc/shadow|/etc/hosts|/proc/self|/var/log)" \
    "id:100401,phase:2,deny,status:403,log,msg:'Path Traversal Detected (Sensitive file access)',setenv:MODSEC_ATTACK_TYPE=path_traversal"

# ----------
# 敏感文件访问检测 (规则 ID: 100500-100599)
# ----------
SecRule REQUEST_URI "@rx (?i)\.(env|git|svn|bak|backup|sql|conf|config|ini|log|sh|py|pl|rb)$" \
    "id:100500,phase:2,deny,status:403,log,msg:'Sensitive File Access Detected',setenv:MODSEC_ATTACK_TYPE=sensitive_file"

SecRule REQUEST_URI "@rx (?i)(\.htaccess|\.htpasswd|web\.config|\.DS_Store)" \
    "id:100501,phase:2,deny,status:403,log,msg:'Sensitive File Access Detected (Config files)',setenv:MODSEC_ATTACK_TYPE=sensitive_file"

# ----------
# 敏感路径访问检测 (规则 ID: 100600-100699)
# ----------
SecRule REQUEST_URI "@rx (?i)^/(admin|manager|phpmyadmin|mysql|backup|config|test|tmp|debug)" \
    "id:100600,phase:2,deny,status:403,log,msg:'Sensitive Path Access Detected',setenv:MODSEC_ATTACK_TYPE=sensitive_path"

# ----------
# 恶意编码检测 (规则 ID: 100700-100799)
# ----------
SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(%3Cscript|%3C/script|%3Ciframe|%3Cobject|%3Cembed|%253C)" \
    "id:100700,phase:2,deny,status:403,log,msg:'Malicious Encoding Detected',setenv:MODSEC_ATTACK_TYPE=malicious_encoding"

SecRule REQUEST_URI|REQUEST_BODY|ARGS "@rx (?i)(0x[0-9a-f]+|\\x[0-9a-f]{2}|\\u[0-9a-f]{4})" \
    "id:100701,phase:2,deny,status:403,log,msg:'Malicious Encoding Detected (Hex/Unicode)',setenv:MODSEC_ATTACK_TYPE=malicious_encoding"

# ----------
# 默认规则 (规则 ID: 199999)
# ----------
# 如果以上规则都没有匹配，但请求被拦截，使用通用错误页面
SecRule REQUEST_URI "@rx .*" \
    "id:199999,phase:5,pass,nolog,setenv:MODSEC_ATTACK_TYPE=general"
EOF
```

### 配置文件权限

确保 www-data 用户对 ModSecurity 配置文件有读写权限：

```bash
sudo chown www-data:www-data /etc/nginx/modsecurity/modsec.conf
sudo chmod 640 /etc/nginx/modsecurity/modsec.conf

sudo chown www-data:www-data /etc/nginx/modsecurity/custom.conf
sudo chmod 640 /etc/nginx/modsecurity/custom.conf
```

---

## 配置 Nginx

集成 ModSecurity，实现多站点管理，预留 SSL/HTTPS 配置，自定义错误页面：

```bash
sudo tee /etc/nginx/nginx.conf << EOF
# Nginx 主配置文件
# 运行用户，www-data 是 Debian/Ubuntu 系统默认的 Web 服务用户
user  www-data;

# 工作进程数，auto 表示自动检测 CPU 核心数
worker_processes  auto;

# 错误日志路径和级别：debug/info/notice/warn/error/crit
error_log  /var/log/nginx/global.error.log warn;

# 主进程 PID 文件位置
pid        /var/run/nginx.pid;

# 事件模块配置
events {
    # 单个工作进程的最大连接数
    worker_connections  1024;
    # 使用 epoll 事件模型（Linux 高性能 I/O 多路复用）
    use epoll;
    # 允许一个进程同时接受多个新连接
    multi_accept on;
}

# HTTP 服务配置
http {
    # 加载 MIME 类型映射表
    include       /etc/nginx/mime.types;
    # 默认文件类型（二进制流，浏览器会下载而非显示）
    default_type  application/octet-stream;

    # 自定义访问日志格式
    # $remote_addr: 客户端 IP
    # $remote_user: 客户端用户名
    # $time_local: 访问时间
    # $request: 请求方法和路径
    # $status: 响应状态码
    # $body_bytes_sent: 响应体大小
    # $http_referer: 来源页面
    # $http_user_agent: 客户端浏览器信息
    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status \$body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    # 访问日志路径，使用 main 格式
    access_log  /var/log/nginx/global.access.log  main;

    # ========== 性能优化配置 ==========
    # 启用 sendfile，零拷贝方式传输文件，提高性能
    sendfile        on;
    # 在 sendfile 开启时，合并数据包一次性发送
    tcp_nopush      on;
    # 禁用 Nagle 算法，减少网络延迟
    tcp_nodelay     on;
    # 长连接超时时间（秒）
    keepalive_timeout  65;
    # MIME 类型哈希表最大大小
    types_hash_max_size 2048;
    # 请求体最大大小（上传文件限制）
    client_max_body_size 100M;
    # 隐藏 Nginx 版本号，增强安全性
    server_tokens off;

    # ========== ModSecurity WAF 配置 ==========
    # 开启 ModSecurity Web 应用防火墙
    modsecurity on;
    # ModSecurity 规则配置文件路径
    modsecurity_rules_file /etc/nginx/modsecurity/modsec.conf;

    # ========== 根据攻击类型映射错误页面 ==========
    # ModSecurity 通过环境变量 MODSEC_ATTACK_TYPE 传递攻击类型
    # map 指令根据攻击类型返回对应的错误页面路径
    map $modsec_attack_type $error_page_path {
        default                /error-html/403_通用安全威胁.html;
        xss                    /error-html/403_XSS攻击.html;
        sql_injection          /error-html/403_SQL注入攻击.html;
        command_injection      /error-html/403_命令注入攻击.html;
        file_inclusion         /error-html/403_文件包含攻击.html;
        path_traversal         /error-html/403_路径遍历攻击.html;
        sensitive_file         /error-html/403_敏感文件访问.html;
        sensitive_path         /error-html/403_敏感路径访问.html;
        malicious_encoding     /error-html/403_恶意编码攻击.html;
        general                /error-html/403_通用安全威胁.html;
    }

    # ========== 自定义错误页面 ==========
    # 400 错误请求
    error_page 400 /error-html/400_错误请求.html;
    # 401 未授权
    error_page 401 /error-html/401_未授权.html;
    # 403 安全威胁拦截（基于攻击类型显示不同页面）
    error_page 403 @custom_403;
    # 404 资源未找到
    error_page 404 /error-html/404_资源未找到.html;
    # 500 服务器内部错误
    error_page 500 /error-html/500_服务器错误.html;
    # 502 错误网关
    error_page 502 /error-html/502_错误网关.html;
    # 503 服务不可用
    error_page 503 /error-html/503_服务不可用.html;
    # 504 网关超时
    error_page 504 /error-html/504_网关超时.html;

    # ========== 错误页面处理（所有站点共用） ==========
    # 捕获 ModSecurity 设置的环境变量
    map $MODSEC_ATTACK_TYPE $modsec_attack_type {
        default $MODSEC_ATTACK_TYPE;
    }

    # ========== 多站点管理 ==========
    # 仅加载 enabled 目录下的站点配置（软链接方式管理站点）
    include /etc/nginx/enabled/*.conf;

    # ========== SSL/TLS 基础配置（预留，站点可直接使用） ==========
    # 支持的 TLS 协议版本，仅启用安全的 TLS 1.2 和 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    # 优先使用服务器端的加密套件顺序
    ssl_prefer_server_ciphers on;
    # 推荐的安全加密套件（支持前向保密）
    # ECDHE/DHE: 密钥交换算法（支持前向保密）
    # AES256-GCM: 对称加密算法（高强度）
    # SHA384/SHA512: 消息认证码算法
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    # SSL 会话超时时间
    ssl_session_timeout 1d;
    # SSL 会话缓存，shared:SSL 表示所有工作进程共享，10m 缓存大小
    ssl_session_cache shared:SSL:10m;
    # 禁用 SSL 会话票据，增强前向保密安全性
    ssl_session_tickets off;
}
EOF
```

### 验证配置语法

```bash
nginx -t
```

**预期输出**: `syntax is ok`

---

## 配置自定义错误页面

创建美观的自定义错误页面目录，并配置权限：

```bash
# 复制自定义错误页面
sudo mkdir -p /etc/nginx/error-html
sudo chown -R www-data:www-data /etc/nginx/error-html
sudo chmod -R 755 /etc/nginx/error-html
```

### 复制错误页面文件

错误页面文件已预置在 `/root/error` 目录中，直接复制到 Nginx 目录：

```bash
sudo cp /root/error/*.html /etc/nginx/error-html/
sudo chown -R www-data:www-data /etc/nginx/error-html
sudo chmod -R 644 /etc/nginx/error-html/*.html
```

**错误页面文件列表**：

| 文件名 | 说明 |
|--------|------|
| `400_错误请求.html` | 400 错误请求 |
| `401_未授权.html` | 401 未授权访问 |
| `403_通用安全威胁.html` | 通用安全威胁拦截页面 |
| `403_XSS攻击.html` | XSS 攻击拦截页面 |
| `403_SQL注入攻击.html` | SQL 注入攻击拦截页面 |
| `403_命令注入攻击.html` | 命令注入攻击拦截页面 |
| `403_文件包含攻击.html` | 文件包含攻击拦截页面 |
| `403_路径遍历攻击.html` | 路径遍历攻击拦截页面 |
| `403_敏感文件访问.html` | 敏感文件访问拦截页面 |
| `403_敏感路径访问.html` | 敏感路径访问拦截页面 |
| `403_恶意编码攻击.html` | 恶意编码攻击拦截页面 |
| `404_资源未找到.html` | 404 资源未找到 |
| `500_服务器错误.html` | 500 服务器内部错误 |
| `502_错误网关.html` | 502 错误网关 |
| `503_服务不可用.html` | 503 服务不可用 |
| `504_网关超时.html` | 504 网关超时 |

### 测试错误页面

可以通过以下方式测试自定义错误页面：

**HTTP 状态码测试**：

| 状态码 | 测试方法 | 说明 |
|--------|----------|------|
| 400 | `curl -X POST "http://localhost" -H "Content-Length: abc"` | 发送无效请求头 |
| 401 | 配置 auth_basic 后访问受保护资源 | 需要认证的页面 |
| 403 | 访问被 ModSecurity 拦截的请求 | 见下方攻击测试 URL |
| 404 | `curl http://localhost/不存在的路径` | 访问不存在的资源 |
| 500 | 后端程序抛出异常 | 需要后端服务支持 |
| 502 | 停止上游服务器后访问代理 | 需要反向代理配置 |
| 503 | 服务器过载或维护模式 | 需要特定配置 |
| 504 | 上游服务器响应超时 | 需要反向代理配置 |

**ModSecurity 攻击拦截测试 URL**（触发 403 错误页面）：

| 攻击类型 | 测试 URL | 触发页面 |
|----------|----------|----------|
| XSS 攻击 | `http://localhost/?q=<script>alert(1)</script>` | `403_XSS攻击.html` |
| SQL 注入 | `http://localhost/?id=1' OR '1'='1` | `403_SQL注入攻击.html` |
| 命令注入 | `http://localhost/?cmd=;ls -la` | `403_命令注入攻击.html` |
| 路径遍历 | `http://localhost/?file=../../../etc/passwd` | `403_路径遍历攻击.html` |
| 文件包含 | `http://localhost/?page=php://filter/` | `403_文件包含攻击.html` |
| 敏感文件 | `http://localhost/?file=/etc/shadow` | `403_敏感文件访问.html` |
| 敏感路径 | `http://localhost/admin/config.php.bak` | `403_敏感路径访问.html` |
| 恶意编码 | `http://localhost/?q=%3Cscript%3E` | `403_恶意编码攻击.html` |
| 其他攻击 | `http://localhost/?q=' UNION SELECT` | `403_通用安全威胁.html` |

**快速测试命令**：

```bash
# 测试 404 页面
curl -i http://localhost/不存在的页面

# 测试 XSS 拦截
curl -i "http://localhost/?q=<script>alert(1)</script>"

# 测试 SQL 注入拦截
curl -i "http://localhost/?id=1'%20OR%20'1'='1"

# 测试命令注入拦截
curl -i "http://localhost/?cmd=;cat%20/etc/passwd"

# 测试路径遍历拦截
curl -i "http://localhost/?file=../../../etc/passwd"
```

---

## 根据攻击类型显示不同错误页面

本功能可以根据 ModSecurity 检测到的不同攻击类型，显示对应的自定义错误页面，提升用户体验和安全提示。

**说明**：由于我们使用的是自定义安全规则（不依赖 OWASP CRS），攻击类型检测和拦截规则已经集成在 `/etc/nginx/modsecurity/custom.conf` 中，每个规则都会设置对应的环境变量 `MODSEC_ATTACK_TYPE`。

### 1. 错误页面文件

错误页面文件已在上一步从 `/root/error` 目录复制到 `/etc/nginx/error-html/`，包含以下文件：

- `403_通用安全威胁.html` - 通用安全威胁
- `403_XSS攻击.html` - XSS 攻击
- `403_SQL注入攻击.html` - SQL 注入攻击
- `403_命令注入攻击.html` - 命令注入攻击
- `403_文件包含攻击.html` - 文件包含攻击
- `403_路径遍历攻击.html` - 路径遍历攻击
- `403_敏感文件访问.html` - 敏感文件访问
- `403_敏感路径访问.html` - 敏感路径访问
- `403_恶意编码攻击.html` - 恶意编码攻击

### 2. 配置说明

**注意**：错误页面处理逻辑已经配置在 `/etc/nginx/nginx.conf` 主配置文件中，包括：

- `map` 指令：根据 `$modsec_attack_type` 环境变量映射到对应的错误页面
- `@custom_403` location：处理 403 错误，根据攻击类型显示不同页面
- `/error-html/` location：错误页面路由，所有站点共用

这些配置已经自动应用到所有站点，无需在每个站点配置文件中重复添加。

### 3. 站点配置

站点配置文件（如 `/etc/nginx/enabled/default.conf`）只需配置站点特定参数：

- 端口监听（listen）
- 服务器名称（server_name）
- 根目录（root）
- 索引文件（index）
- 站点日志路径
- PHP FastCGI 配置（不同站点可使用不同 PHP 版本）
- HTTPS 证书配置

错误页面处理由 nginx.conf 统一管理，简化了站点配置。
# 验证配置语法
nginx -t

# 平滑重启 Nginx 生效
sudo systemctl reload nginx
```

### 5. 测试不同攻击类型的错误页面

```bash
# 测试 XSS 攻击
curl -i "http://服务器IP/<script>alert(1)</script>"

# 测试 SQL 注入攻击
curl -i "http://服务器IP/?id=1%27%20union%20select"

# 测试命令注入攻击
curl -i "http://服务器IP/?cmd=;ls"

# 测试路径遍历攻击
curl -i "http://服务器IP/?file=../../../etc/passwd"

# 测试敏感文件访问
curl -i "http://服务器IP/config.env"
```

**预期结果**：所有攻击请求都会返回 403 状态码，并显示自定义的错误页面。

---

## 配置 Systemd 服务

实现 systemctl 命令管理 Nginx，支持开机自启、平滑重启、故障自动重启：

```bash
sudo tee /etc/systemd/system/nginx.service << EOF
[Unit]
Description=Nginx HTTP Server with ModSecurity
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s stop
Restart=on-failure
RestartSec=5
User=www-data
Group=www-data
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
EOF
```

### 启动服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable nginx

# 启动 Nginx 服务
sudo systemctl start nginx

# 查看服务状态
sudo systemctl status nginx
```

**预期输出**: `active (running)`

### 故障排查

如果服务启动失败，按以下步骤排查：

**1. 检查服务状态和错误日志**

```bash
# 查看详细错误信息
sudo systemctl status nginx

# 查看 systemd 日志
sudo journalctl -xeu nginx.service

# 查看 Nginx 错误日志
tail -f /var/log/nginx/global.error.log
```

**2. 常见问题：权限不足**

如果提示权限错误，可能是因为编译安装时目录权限配置问题。临时解决方案：

```bash
# 临时修改服务文件，使用 root 用户启动（用于排查）
sudo sed -i 's/User=www-data/User=root/' /etc/systemd/system/nginx.service
sudo sed -i 's/Group=www-data/Group=root/' /etc/systemd/system/nginx.service
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

**生产环境建议**：解决权限问题后，应改回 www-data 用户运行：

```bash
# 确保所有目录权限正确
sudo chown -R www-data:www-data /etc/nginx /var/log/nginx /var/www/html
sudo chown -R www-data:www-data /etc/nginx/modsecurity/tmp
sudo chmod -R 755 /var/www/html
sudo chmod -R 700 /etc/nginx/ssl /etc/nginx/modsecurity/tmp

# 恢复 www-data 用户运行
sudo sed -i 's/User=root/User=www-data/' /etc/systemd/system/nginx.service
sudo sed -i 's/Group=root/Group=www-data/' /etc/systemd/system/nginx.service
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

**3. 验证 Nginx 可执行文件路径**

```bash
# 确认 Nginx 安装路径
which nginx
# 预期输出: /usr/sbin/nginx

# 如果路径不同，修改服务文件中的 ExecStart
sudo vim /etc/systemd/system/nginx.service
# 修改 ExecStart=/usr/sbin/nginx 为实际路径
```

**4. 验证配置文件语法**

```bash
# 在启动服务前，先验证配置
nginx -t
```

---

## 创建共用配置片段

将常用的配置提取为共用片段，简化多站点管理：

```bash
sudo mkdir -p /etc/nginx/snippets
sudo tee /etc/nginx/snippets/common.conf << 'EOF'
# ============================================
# Nginx 共用配置片段
# 包含：ModSecurity 错误处理、静态资源优化、安全设置
# ============================================

# ---------- ModSecurity 攻击类型错误页面处理 ----------
# 捕获 ModSecurity 设置的环境变量
set $modsec_attack_type $MODSEC_ATTACK_TYPE;

# 自定义 403 错误页面（保持 403 状态码）
error_page 403 @custom_403;

location @custom_403 {
    root /etc/nginx;
    try_files $error_page_path /error-html/403_通用安全威胁.html =404;
    internal;
}

# 自定义 404 错误页面
error_page 404 /error-html/404_资源未找到.html;

# 错误页面路由
location /error-html/ {
    root /etc/nginx;
    internal;
}

# ---------- 默认 location 配置 ----------
location / {
    try_files $uri $uri/ =404;
}

# ---------- PHP 处理配置 ----------
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
    fastcgi_connect_timeout 300s;
    fastcgi_send_timeout 300s;
    fastcgi_read_timeout 300s;
}

# ---------- 安全设置 ----------
# 禁止访问 .htaccess 文件
location ~ /\.ht {
    deny all;
}

# 禁止访问隐藏文件
location ~ /\. {
    deny all;
}

# ---------- 静态文件缓存优化 ----------
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
EOF
```

## 创建默认站点

适配多站点管理模式，创建默认站点（使用共用配置）：

```bash
sudo tee /etc/nginx/enabled/default.conf << EOF
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    root /var/www/html;
    index index.html index.htm;

    access_log /var/log/nginx/site/default.access.log main;
    error_log /var/log/nginx/site/default.error.log warn;

    # 引入共用配置（包含错误页面、PHP处理、安全设置、静态缓存）
    include snippets/common.conf;

    # HTTPS 预留配置
    # listen 443 ssl http2;
    # ssl_certificate /etc/nginx/ssl/default.crt;
    # ssl_certificate_key /etc/nginx/ssl/default.key;
}
EOF
```

### 创建默认首页

```bash
sudo tee /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Default Site | Nginx + ModSecurity</title>
    <meta charset="UTF-8">
</head>
<body>
    <h1>Default Site Enabled</h1>
    <p>Nginx 1.25.4 + ModSecurity 3.0.10 + OWASP CRS 3.3.4</p>
    <p>GeoIP/MaxMind Disabled</p>
</body>
</html>
EOF
```

### 平滑重启 Nginx

```bash
sudo systemctl reload nginx
```

---

## 功能验证

### 1. 访问验证

通过浏览器访问服务器 80 端口（`http://服务器IP`），显示 "Default Site Enabled" 且包含 "GeoIP/MaxMind Disabled" 即为成功。

### 2. ModSecurity 拦截验证

验证 ModSecurity 核心拦截能力，确保 XSS、SQL 注入等攻击能被有效拦截：

```bash
# 模拟 XSS 攻击请求，替换为你的服务器 IP
curl -i "http://服务器IP/<script>alert(1)</script>"
```

**预期结果**: 返回 `HTTP/1.1 403 Forbidden`，并显示自定义 403 错误页面，表示 ModSecurity 拦截成功。

### 3. 404 错误页面验证

访问不存在的页面，验证自定义 404 错误页面：

```bash
curl -i "http://服务器IP/不存在的页面"
```

**预期结果**: 返回 `HTTP/1.1 404 Not Found`，并显示自定义 404 错误页面。

### 4. 不同攻击类型错误页面验证

验证配置了不同攻击类型后显示对应错误页面的功能（需要先完成"根据攻击类型显示不同错误页面"章节的配置）：

```bash
# 测试 XSS 攻击
curl -i "http://服务器IP/?q=<script>alert(1)</script>"

# 测试 SQL 注入攻击
curl -i "http://服务器IP/?id=1' OR '1'='1"

# 测试路径遍历攻击
curl -i "http://服务器IP/../../../etc/passwd"

# 测试敏感文件访问
curl -i "http://服务器IP/.env"
```

**预期结果**: 根据不同的攻击类型，显示对应的自定义错误页面（如 XSS 攻击显示 403_XSS攻击.html，SQL 注入显示 403_SQL注入攻击.html）。

---

## 多站点管理

基于 enabled/disabled 目录实现多站点一键管理，操作时无需中断 Nginx 服务。

> **提示**: 推荐使用 [Nginx站点管理工具使用指南.md](./Nginx站点管理工具使用指南.md) 交互式管理站点，支持自动创建配置、申请 SSL 证书、多版本 PHP 检测等功能。

### 手动新增站点

1. 在 `/etc/nginx/enabled/` 目录下创建 `.conf` 配置文件，参考默认站点配置（`default.conf`）
2. 配置站点根目录权限：
   ```bash
   sudo chown -R www-data:www-data 站点根目录
   ```
3. 平滑重启 Nginx 生效：
   ```bash
   sudo systemctl reload nginx
   ```

**站点配置示例**（使用共用配置，支持不同 PHP 版本）：

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example.com;
    index index.php index.html;

    access_log /var/log/nginx/site/example.com.access.log main;
    error_log /var/log/nginx/site/example.com.error.log warn;

    # 引入共用配置（错误页面、安全设置、静态缓存）
    include snippets/common.conf;

    # 覆盖 PHP 处理配置（使用 PHP-FPM 8.1）
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # HTTPS 配置
    # listen 443 ssl http2;
    # ssl_certificate /etc/nginx/ssl/example.com.crt;
    # ssl_certificate_key /etc/nginx/ssl/example.com.key;
}

# 另一个站点使用 PHP-FPM 7.4
server {
    listen 80;
    server_name legacy.example.com;
    root /var/www/legacy.example.com;
    index index.php index.html;

    access_log /var/log/nginx/site/legacy.example.com.access.log main;
    error_log /var/log/nginx/site/legacy.example.com.error.log warn;

    # 引入共用配置（错误页面、安全设置、静态缓存）
    include snippets/common.conf;

    # 覆盖 PHP 处理配置（使用 PHP-FPM 7.4）
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

**注意**：
- 通过 `include snippets/common.conf;` 引入共用配置，包含错误页面、安全设置、静态缓存等
- 如需自定义 PHP 版本，在 include 之后重新定义 `location ~ \.php$` 块即可覆盖
- 站点配置只需关注站点特定参数（端口、域名、根目录、PHP 版本、SSL 证书等）

### 暂停站点

```bash
sudo mv /etc/nginx/enabled/站点名.conf /etc/nginx/disabled/
sudo systemctl reload nginx
```

### 恢复站点

```bash
sudo mv /etc/nginx/disabled/站点名.conf /etc/nginx/enabled/
sudo systemctl reload nginx
```

### 查看站点列表

```bash
# 查看运行中的站点
ls /etc/nginx/enabled/

# 查看暂停的站点
ls /etc/nginx/disabled/
```

---

## HTTPS 部署

基于配置文件中预留的 SSL 配置，3 步即可完成 HTTPS 部署：

### 1. 上传 SSL 证书

将 SSL 证书的公钥（`.crt`/`.pem`）、私钥（`.key`）文件上传至 `/etc/nginx/ssl/` 目录：

```bash
# 本地向服务器上传示例
scp 本地证书.crt root@服务器IP:/etc/nginx/ssl/
scp 本地证书.key root@服务器IP:/etc/nginx/ssl/
```

### 2. 修改站点配置

编辑对应站点的配置文件，取消 443 端口注释并替换证书路径：

```bash
sudo vim /etc/nginx/enabled/站点名.conf
```

修改以下配置：

```nginx
listen 443 ssl http2;
ssl_certificate /etc/nginx/ssl/你的证书.crt;
ssl_certificate_key /etc/nginx/ssl/你的证书.key;
```

### 3. 验证并重启

```bash
# 验证配置语法
nginx -t

# 平滑重启 Nginx 生效
sudo systemctl reload nginx
```

---

## 常见问题排查

### 问题 1：权限报错

**原因**: www-data 用户无目录/文件读写权限

**解决方案**:

```bash
sudo chown -R www-data:www-data /etc/nginx /var/log/nginx /var/www/html /etc/nginx/modsecurity/tmp
sudo chmod -R 755 /var/www/html
```

### 问题 2：误拦截正常业务请求

**原因**: OWASP CRS 规则匹配到正常业务请求

**解决方案**: 在 `modsec.conf` 中添加放行规则（加载 CRS 规则前）：

```bash
sudo vim /etc/nginx/modsecurity/modsec.conf
```

添加示例规则：

```nginx
# 放行特定 URL
SecRule REQUEST_URI "@beginsWith /api/health" "id:100,phase:1,allow,nolog"
# 放行特定请求参数
SecRule ARGS:param_name "@equals safe_value" "id:101,phase:2,allow,nolog"
```

生效命令：

```bash
sudo systemctl reload nginx
```

### 问题 3：Git 克隆失败

**原因**: 服务器网络不通或 443 端口未开放

**解决方案**:

```bash
# 检查网络连通性
ping github.com

# 确保 443 端口开放（若开启了 ufw 防火墙）
sudo ufw allow 443/tcp
```

若仍无法克隆，使用各步骤中的手动下载压缩包方案。

### 问题 4：Nginx 启动失败

**解决方案**:

1. 重新创建 systemd 服务文件，参考第九章完整命令
2. 重新加载并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl start nginx

# 查看错误详情，定位根因
journalctl -xeu nginx.service
```

### 问题 5：自定义错误页面不显示

**原因**: 错误页面权限配置有误或权限不正确

**解决方案**:

```bash
# 检查错误页面目录权限
ls -la /etc/nginx/error-html/

# 重新设置权限
sudo chown -R www-data:www-data /etc/nginx/error-html
sudo chmod -R 644 /etc/nginx/error-html/*.html

# 验证配置
nginx -t
sudo systemctl reload nginx
```

---

## 版本兼容性说明

> **重要**: 请严格遵守以下版本兼容性要求

- **禁止**将 Nginx 升级至 1.26+，暂未适配 ModSecurity 3.0.10
- ModSecurity 3.0.10 仅兼容 Nginx 1.25.x 及以下稳定版
- OWASP CRS 3.3.4 需与 ModSecurity 3.x 配套使用，不兼容 2.x 版本

---

## 日志配置与管理

### 日志目录结构

完整的日志目录结构如下：

```
/var/log/nginx/
├── global.access.log       # Nginx 全局访问日志
├── global.error.log        # Nginx 全局错误日志
├── modsecurity/            # ModSecurity 专用日志目录
│   ├── audit.log           # ModSecurity 审计日志（JSON 格式）
│   └── debug.log           # ModSecurity 调试日志（生产环境建议关闭）
└── site/                   # 各站点独立日志目录
    ├── default.access.log  # 默认站点访问日志
    └── default.error.log   # 默认站点错误日志
```

### 一、全局日志配置

全局日志在 Nginx 主配置文件 `nginx.conf` 中配置，所有站点共享。

#### 1. Nginx 全局访问日志

在 `nginx.conf` 的 `http` 块中配置：

```nginx
# 自定义访问日志格式
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

# 全局访问日志
access_log  /var/log/nginx/global.access.log  main;
```

**日志格式字段说明**：
- `$remote_addr`：客户端 IP 地址
- `$remote_user`：客户端用户名（基本认证时使用）
- `$time_local`：访问时间（本地时区）
- `$request`：请求方法和路径
- `$status`：HTTP 响应状态码
- `$body_bytes_sent`：响应体大小（字节）
- `$http_referer`：来源页面 URL
- `$http_user_agent`：客户端浏览器/设备信息
- `$http_x_forwarded_for`：代理服务器转发的真实客户端 IP

#### 2. Nginx 全局错误日志

在 `nginx.conf` 的顶层配置：

```nginx
# 全局错误日志路径和级别：debug/info/notice/warn/error/crit
error_log  /var/log/nginx/global.error.log warn;
```

**错误日志级别**（从低到高）：
- `debug`：调试信息（最详细，生产环境不建议）
- `info`：一般信息
- `notice`：通知
- `warn`：警告（推荐生产环境使用）
- `error`：错误
- `crit`：严重错误

#### 3. ModSecurity 全局日志

在 `modsec.conf` 中配置：

```nginx
# 调试日志（生产环境建议关闭，设置 SecDebugLogLevel 0）
SecDebugLog /var/log/nginx/modsecurity/debug.log
SecDebugLogLevel 0

# 审计日志（记录所有拦截的请求，JSON 格式便于分析）
SecAuditLog /var/log/nginx/modsecurity/audit.log
SecAuditLogFormat JSON
SecAuditLogType Serial
```

### 二、独立站点日志配置

每个站点可以配置独立的访问日志和错误日志，便于单独分析和监控。

#### 1. 站点访问日志

在站点配置文件（如 `/etc/nginx/enabled/default.conf`）的 `server` 块中添加：

```nginx
server {
    listen 80;
    server_name example.com;

    # 站点独立访问日志
    access_log /var/log/nginx/site/example.com.access.log main;

    # 其他配置...
}
```

#### 2. 站点错误日志

```nginx
server {
    listen 80;
    server_name example.com;

    # 站点独立错误日志
    error_log /var/log/nginx/site/example.com.error.log warn;

    # 其他配置...
}
```

#### 3. 完整站点配置示例

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    root /var/www/html;
    index index.html index.htm;

    # ========== 站点独立日志配置 ==========
    access_log /var/log/nginx/site/default.access.log main;
    error_log /var/log/nginx/site/default.error.log warn;

    # 静态文件处理
    location / {
        try_files $uri $uri/ =404;
    }

    # PHP 处理（根据站点需求配置不同 PHP 版本）
    # 示例：使用 PHP-FPM 8.1
    # location ~ \.php$ {
    #     fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    #     fastcgi_index index.php;
    #     fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    #     include fastcgi_params;
    # }
}
```

**注意**：错误页面处理（包括基于攻击类型的 403 页面）已由 `nginx.conf` 统一配置，无需在每个站点中重复添加。

### 三、ModSecurity 审计日志格式详解

ModSecurity 审计日志使用 JSON 格式，便于工具解析和分析。

**关键字段说明**：

| 字段 | 说明 |
|------|------|
| `transaction.request_id` | 请求唯一 ID |
| `transaction.client_ip` | 客户端 IP |
| `transaction.time_stamp` | 请求时间戳 |
| `transaction.request.uri` | 请求 URI |
| `transaction.request.method` | 请求方法（GET/POST 等）|
| `transaction.response.http_code` | 响应状态码 |
| `transaction.messages` | 触发的规则信息 |
| `transaction.messages[].msg` | 规则消息 |
| `transaction.messages[].id` | 规则 ID |

**审计日志查看示例**：

```bash
# 查看最新 10 条拦截记录
tail -n 10 /var/log/nginx/modsecurity/audit.log

# 仅提取规则 ID 和消息
tail -f /var/log/nginx/modsecurity/audit.log | jq -r '.transaction.messages[] | "\(.id): \(.msg)"'

# 统计攻击类型分布
cat /var/log/nginx/modsecurity/audit.log | jq -r '.transaction.messages[].msg' | sort | uniq -c | sort -rn
```

### 四、日志轮转配置（logrotate）

配置日志自动轮转，避免日志文件过大占用磁盘空间。

创建 Nginx 日志轮转配置文件：

```bash
sudo tee /etc/logrotate.d/nginx << 'EOF'
/var/log/nginx/global*.log
/var/log/nginx/site/*.log
/var/log/nginx/modsecurity/*.log {
    daily
    rotate 30
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
EOF
```

**配置说明**：
- `daily`：每天轮转一次
- `rotate 30`：保留 30 天的日志
- `compress`：压缩旧日志（gzip 格式）
- `delaycompress`：延迟压缩，保留最近一天的未压缩日志
- `postrotate`：轮转后通知 Nginx 重新打开日志文件

**手动测试日志轮转**：

```bash
# 测试配置是否正确
sudo logrotate -d /etc/logrotate.d/nginx

# 强制执行一次轮转
sudo logrotate -f /etc/logrotate.d/nginx
```

### 五、日志查看常用命令

#### Nginx 日志

```bash
# 实时查看全局访问日志
tail -f /var/log/nginx/global.access.log

# 实时查看全局错误日志
tail -f /var/log/nginx/global.error.log

# 查看特定状态码的请求（如 403 拦截）
grep ' 403 ' /var/log/nginx/global.access.log

# 统计访问量最高的 IP
awk '{print $1}' /var/log/nginx/global.access.log | sort | uniq -c | sort -rn | head -20
```

#### 站点独立日志

```bash
# 实时查看默认站点访问日志
tail -f /var/log/nginx/site/default.access.log

# 查看默认站点错误日志
tail -n 50 /var/log/nginx/site/default.error.log

# 多站点日志汇总查看
ls -lh /var/log/nginx/site/
```

#### ModSecurity 日志

```bash
# 实时查看审计日志
tail -f /var/log/nginx/modsecurity/audit.log

# 查看最近 50 条拦截记录
tail -n 50 /var/log/nginx/modsecurity/audit.log

# 统计拦截的攻击类型（需 jq）
cat /var/log/nginx/modsecurity/audit.log | jq -r '.transaction.messages[].msg' 2>/dev/null | sort | uniq -c | sort -rn

# 查看被拦截的请求详情（按时间）
grep -A 50 'XSS' /var/log/nginx/modsecurity/audit.log
```

### 六、日志权限配置

确保日志目录权限正确，Nginx 能够写入：

```bash
# 设置日志目录所有者和权限
sudo chown -R www-data:adm /var/log/nginx
sudo chmod -R 750 /var/log/nginx

# 确保子目录权限
sudo chmod -R 750 /var/log/nginx/site
sudo chmod -R 750 /var/log/nginx/modsecurity
```

---

## 常用命令速查

### Nginx 管理

```bash
nginx -t                          # 验证配置语法（必备，修改配置后先执行）
sudo systemctl start nginx        # 启动服务
sudo systemctl stop nginx         # 停止服务
sudo systemctl restart nginx      # 强制重启（不推荐生产环境）
sudo systemctl reload nginx       # 平滑重启（生产环境推荐）
sudo systemctl status nginx       # 查看服务状态
ps -ef | grep nginx               # 查看 Nginx 进程
```

### 全局日志

```bash
tail -f /var/log/nginx/global.access.log           # 实时查看全局访问日志
tail -f /var/log/nginx/global.error.log            # 实时查看全局错误日志
grep ' 403 ' /var/log/nginx/global.access.log     # 查看被拦截的请求
```

### ModSecurity 日志

```bash
tail -f /var/log/nginx/modsecurity/audit.log  # 实时查看拦截审计日志
tail -f /var/log/nginx/modsecurity/debug.log  # 查看调试日志（仅测试环境开启）
tail -n 50 /var/log/nginx/modsecurity/audit.log  # 查看最近50条拦截记录
```

### 站点日志

```bash
tail -f /var/log/nginx/site/default.access.log  # 实时查看默认站点访问日志
tail -f /var/log/nginx/site/default.error.log   # 实时查看默认站点错误日志
ls -lh /var/log/nginx/site/                     # 查看所有站点日志文件
```

### 日志轮转

```bash
sudo logrotate -d /etc/logrotate.d/nginx  # 测试日志轮转配置
sudo logrotate -f /etc/logrotate.d/nginx  # 强制执行一次轮转
```

### 站点管理

```bash
ls /etc/nginx/enabled/   # 查看运行中的站点
ls /etc/nginx/disabled/  # 查看暂停的站点
```

---

## 生产环境运维建议

### 1. 日志管理
- **关闭调试日志**: 保持 `SecDebugLogLevel 0`，避免调试日志占满磁盘空间
- **配置日志轮转**: 确保已配置 logrotate，参考"日志配置与管理"章节
- **定期检查日志大小**:
  ```bash
  # 查看日志目录占用空间
  du -sh /var/log/nginx/
  
  # 查看各日志文件大小
  ls -lh /var/log/nginx/
  ls -lh /var/log/nginx/site/
  ls -lh /var/log/nginx/modsecurity/
  ```
- **定期分析 ModSecurity 日志**: 及时发现新的攻击模式和潜在误报

### 2. 配置管理
- **定期备份配置**:
  ```bash
  tar -zcvf nginx-conf-backup-$(date +%Y%m%d).tar.gz /etc/nginx
  ```
- **修改配置后验证**: 始终先执行 `nginx -t` 验证语法，再平滑重启

### 3. 性能优化
- **优化 Nginx 性能**: 根据服务器核心数，将 `nginx.conf` 中的 `worker_processes` 从 `auto` 改为具体数值（如 4/8）
- **调整 worker_connections**: 根据并发需求适当调整 `worker_connections` 参数

### 4. 安全加固
- **开启防火墙**: 仅开放 80/443 业务端口
  ```bash
  sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw enable
  ```
- **权限加固**: 保持核心目录（`/etc/nginx/ssl`、`/etc/nginx/modsecurity/tmp`）700 权限，避免未授权访问
- **禁止自动升级**: 配置 apt 禁止 Nginx 自动升级，避免版本兼容问题
  ```bash
  sudo apt-mark hold nginx
  ```

### 5. 规则管理
- **更新安全规则**: 本方案使用自定义规则，定期根据业务需求更新 `/etc/nginx/modsecurity/custom.conf`
- **监控误报**: 定期检查 ModSecurity 审计日志，及时发现并调整误拦截规则

### 6. 日志监控与分析
- **部署日志分析系统**: 推荐使用 ELK Stack（Elasticsearch + Logstash + Kibana）或 Grafana Loki + Promtail
- **关键监控指标**:
  - 403 拦截请求数量及趋势
  - 攻击类型分布（XSS、SQL 注入等）
  - 访问量 TOP IP（识别潜在攻击源）
  - 站点错误率
- **告警配置**:
  - 短时间内大量 403 拦截触发告警
  - 磁盘空间使用率超过 80% 触发告警
  - Nginx 服务异常停止触发告警

### 7. 日常检查
- **定期检查错误页面**: 定期检查自定义错误页面是否正常显示
- **监控磁盘空间**: 确保日志分区有足够空间
- **服务状态检查**:
  ```bash
  # 检查 Nginx 服务状态
  sudo systemctl status nginx
  
  # 检查 Nginx 进程
  ps aux | grep nginx
  ```

---

## 总结

### 核心要点

- **核心调整**: 使用自定义安全规则（不依赖 OWASP CRS），配置美观的自定义错误页面，支持不同攻击类型显示对应错误页面
- **日志管理**: 完善的全局日志和独立站点日志分离配置，支持日志自动轮转（logrotate），便于分析和监控
- **路径核心**: 全英文目录 + 精准的 www-data 权限配置，解决 ModSecurity 路径解析缺陷
- **运维核心**: 修改配置后先执行 `nginx -t` 验证语法、重启使用 `systemctl reload nginx` 平滑重载、报错优先查看日志
- **版本核心**: 严格遵循 Nginx 1.25.4 + ModSecurity 3.0.10 版本兼容组合
- **管理核心**: enabled/disabled 目录分离，实现多站点一键启停，操作不中断服务
- **规则管理**: 自定义规则统一存放在 `/etc/nginx/modsecurity/custom.conf`，便于管理和更新
- **安全防护**: 自定义规则覆盖 XSS、SQL 注入、命令注入、文件包含、路径遍历、敏感文件访问等常见攻击类型
- **用户体验**: 配置美观的自定义错误页面，并根据不同攻击类型显示对应的错误页面，提升用户体验和安全提示
- **高级功能**: 使用 ModSecurity 自定义规则设置环境变量，结合 Nginx 的 map 指令实现动态错误页面选择

本方案搭建的 Nginx+ModSecurity 服务完全适配生产环境，兼顾安全性、稳定性和运维便捷性。
