# MariaDB APT 安装部署指南

**文档版本**: V1.0  
**适配环境**: Ubuntu 22.04.5 LTS  
**核心版本**: MariaDB 10.11+  
**用途**: 通用数据库服务，支持 InnoDB、Aria 存储引擎，适用于 WordPress 等应用

---

## 目录

1. [系统准备](#系统准备)
2. [目录结构（APT 默认）](#目录结构-apt-默认)
3. [使用 APT 安装 MariaDB 稳定版](#使用-apt-安装-mariadb-稳定版)
4. [配置 MariaDB](#配置-mariadb)
5. [服务管理](#服务管理)
6. [初始化数据库](#初始化数据库)
7. [安全加固](#安全加固)
8. [创建应用数据库](#创建应用数据库)
9. [功能验证](#功能验证)
10. [常用命令速查](#常用命令速查)
11. [常见问题排查](#常见问题排查)
12. [备份与恢复](#备份与恢复)
13. [性能优化建议](#性能优化建议)

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

### 检查系统资源

```bash
# 检查内存（建议至少 2GB）
free -h

# 检查磁盘空间（建议至少 20GB 可用空间）
df -h

# 检查 CPU 核心数
nproc
```

---

## 目录结构（APT 默认）

APT 安装会自动创建以下默认目录结构：

| 目录路径 | 功能说明 |
|---------|---------|
| `/usr` | MariaDB 安装根目录 |
| `/etc/mysql/mariadb.conf.d` | 配置文件目录 |
| `/var/lib/mysql` | 数据目录 |
| `/var/log/mysql` | 日志目录 |
| `/var/run/mysqld` | Socket 和 PID 文件目录 |

### 自定义数据目录（可选）

如果需要使用自定义数据目录（例如独立分区），可以按以下步骤操作：

```bash
# 创建自定义数据目录
sudo mkdir -p /data/mysql

# 复制现有数据（如果已安装）
sudo cp -a /var/lib/mysql/* /data/mysql/

# 设置权限
sudo chown -R mysql:mysql /data/mysql
sudo chmod 700 /data/mysql

# 修改配置文件
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# 将 datadir 改为 /data/mysql

# 重启服务
sudo systemctl restart mariadb
```

---

## 使用 APT 安装 MariaDB 稳定版

### 一键安装命令（推荐）

使用以下命令快速安装 MariaDB（使用默认仓库的稳定版）：

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 MariaDB 服务器和客户端
sudo apt install -y mariadb-server mariadb-client

# 运行安全初始化
sudo mysql_secure_installation

# 启动并启用服务
sudo systemctl start mariadb
sudo systemctl enable mariadb

# 验证安装
mysql --version
```

### 安装完成后

安装完成后，系统会自动：
1. 创建必要的目录结构
2. 配置 Systemd 服务
3. 启动 MariaDB 服务
4. 允许 root 用户本地登录

**预期输出**: `mysql  Ver 10.11.x-MariaDB ...`（版本号可能因系统而异）

---

## 配置 MariaDB（可选）

APT 安装会自动创建默认配置文件。如果需要自定义配置，可以修改以下文件：

### 主配置文件

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

### 客户端配置文件

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-client.cnf
```

### 常用配置建议

如果需要优化配置，可以在主配置文件中添加以下内容：

```ini
[mysqld]
# 字符集配置
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# 连接配置
max_connections = 500

# InnoDB 优化
innodb_buffer_pool_size = 1G
innodb_buffer_pool_instances = 4

# 慢查询日志
slow_query_log = 1
slow_query_log_file = /var/log/mariadb/slow.log
long_query_time = 2
```

---

## 服务管理

APT 安装会自动配置 Systemd 服务。以下是常用的服务管理命令：

```bash
# 启动服务
sudo systemctl start mariadb

# 停止服务
sudo systemctl stop mariadb

# 重启服务
sudo systemctl restart mariadb

# 启用开机自启
sudo systemctl enable mariadb

# 禁用开机自启
sudo systemctl disable mariadb

# 查看服务状态
sudo systemctl status mariadb
```

---

## 初始化数据库

### 安全初始化

MariaDB 安装后，使用官方的安全初始化工具进行配置：

```bash
# 运行 mysql_secure_installation 进行安全加固
sudo mysql_secure_installation
```

**交互式配置步骤**：

1. **设置 root 密码**: 输入并确认新密码
2. **删除匿名用户**: 输入 `Y` 删除匿名用户
3. **禁止 root 远程登录**: 输入 `Y` 禁止 root 远程登录
4. **删除测试数据库**: 输入 `Y` 删除测试数据库
5. **重新加载权限表**: 输入 `Y` 重新加载权限表

### 验证初始化结果

```bash
# 检查服务状态
sudo systemctl status mariadb

# 测试连接（使用设置的 root 密码）
mysql -u root -p -e "SELECT VERSION(); SHOW DATABASES;"
```

### 手动安全加固（可选）

如果需要手动执行安全加固，可以运行以下命令：

```bash
# 登录 MariaDB
mysql -u root -p

# 执行以下 SQL 命令
DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'$(hostname)';
DROP DATABASE IF EXISTS test;
FLUSH PRIVILEGES;

# 退出
EXIT;
```


---

## 安全加固（可选）

### 配置防火墙

```bash
# 如果需要远程访问，开放 3306 端口（仅允许特定 IP）
sudo ufw allow from 192.168.1.0/24 to any port 3306

# 或者完全禁止远程访问（推荐）
sudo ufw deny 3306
```

### 配置 SSL/TLS（生产环境推荐）

**使用自签名证书**：

```bash
# 1. 创建证书目录
sudo mkdir -p /etc/mysql/ssl
cd /etc/mysql/ssl

# 2. 生成 CA 证书
sudo openssl genrsa 2048 > mariadb-ca-private.key
sudo openssl req -new -x509 -nodes -days 3650 \
    -key mariadb-ca-private.key \
    -out mariadb-ca.crt \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=MariaDB/CN=MariaDB CA"

# 3. 生成服务器证书
sudo openssl req -newkey rsa:2048 -days 3650 -nodes \
    -keyout mariadb-server-private.key \
    -out mariadb-server.csr \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=MariaDB/CN=localhost"

sudo openssl x509 -req -in mariadb-server.csr -days 3650 \
    -CA mariadb-ca.crt -CAkey mariadb-ca-private.key -set_serial 01 \
    -out mariadb-server.crt

# 4. 清理和设置权限
sudo rm -f *.csr
sudo chmod 600 *-private.key
sudo chmod 644 *.crt
sudo chown -R mysql:mysql .

# 5. 配置 MariaDB 使用 SSL
sudo tee -a /etc/mysql/mariadb.conf.d/50-server.cnf << 'EOF'

# SSL 配置
ssl
ssl-ca = /etc/mysql/ssl/mariadb-ca.crt
ssl-cert = /etc/mysql/ssl/mariadb-server.crt
ssl-key = /etc/mysql/ssl/mariadb-server-private.key
tls_version = TLSv1.2,TLSv1.3
# require_secure_transport = ON
EOF

# 6. 重启 MariaDB
sudo systemctl restart mariadb
```

---

## 创建应用数据库

### 方式一：自动化创建（推荐）

使用以下脚本实现完全自动化的数据库创建：

```bash
#!/bin/bash
# 配置变量（根据应用需求修改）
DB_NAME="myapp"
DB_USER="app_user"
DB_PASSWORD=$(openssl rand -base64 24)

# 创建数据库和用户（使用 root 免密码登录）
mysql -u root << EOF
-- 创建数据库
CREATE DATABASE IF NOT EXISTS ${DB_NAME} CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 创建用户
CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASSWORD}';

-- 授权
GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;

-- 验证
SELECT "数据库列表:" AS "";
SHOW DATABASES LIKE '${DB_NAME}';
SELECT "用户列表:" AS "";
SELECT User, Host FROM mysql.user WHERE User = '${DB_USER}';
EOF

# 测试连接
if mysql -u "${DB_USER}" -p"${DB_PASSWORD}" "${DB_NAME}" -e "SELECT '连接成功' AS Status;" 2>/dev/null; then
    echo "数据库创建成功！"
    echo "数据库名: ${DB_NAME}"
    echo "用户名: ${DB_USER}"
    echo "密码: ${DB_PASSWORD}"
else
    echo "连接测试失败！"
    exit 1
fi
```

### 方式二：手动创建

```bash
# 登录 MariaDB
mysql -u root -p

# 创建数据库（根据应用需求修改数据库名）
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 创建用户（替换密码）
CREATE USER 'app_user'@'localhost' IDENTIFIED BY '<your_strong_password>';

# 授权
GRANT ALL PRIVILEGES ON myapp.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;

# 验证
SHOW DATABASES;
SELECT User, Host FROM mysql.user WHERE User = 'app_user';

# 退出
EXIT;
```

### 测试连接

```bash
# 使用应用用户测试连接
mysql -u app_user -p myapp

# 查看当前数据库
SELECT DATABASE();

# 查看字符集
SHOW VARIABLES LIKE 'character_set%';

# 退出
EXIT;
```

---

## 功能验证

### 验证 MariaDB 版本

```bash
# 查看 MariaDB 版本
mysql --version

# 登录查看详细信息
mysql -u root -p -e "SELECT VERSION();"
```

### 验证服务状态

```bash
# 查看服务状态
sudo systemctl status mariadb

# 查看 MariaDB 进程
ps aux | grep mysqld

# 查看监听端口
sudo netstat -tlnp | grep 3306
```

### 验证存储引擎

```bash
# 登录 MariaDB
mysql -u root -p

# 查看支持的存储引擎
SHOW ENGINES;

# 查看默认存储引擎
SHOW VARIABLES LIKE 'default_storage_engine';

# 查看 InnoDB 状态
SHOW ENGINE INNODB STATUS\G

# 退出
EXIT;
```

### 验证字符集

```bash
# 登录 MariaDB
mysql -u root -p

# 查看字符集配置
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

# 退出
EXIT;
```

### 性能基准测试（可选）

```bash
# 安装 sysbench
sudo apt install -y sysbench

# 准备测试数据
sysbench oltp_read_write --mysql-host=localhost --mysql-port=3306 --mysql-user=root --mysql-password=<your_password> --mysql-db=test --tables=10 --table-size=100000 prepare

# 运行测试
sysbench oltp_read_write --mysql-host=localhost --mysql-port=3306 --mysql-user=root --mysql-password=<your_password> --mysql-db=test --tables=10 --table-size=100000 --threads=8 --time=60 run

# 清理测试数据
sysbench oltp_read_write --mysql-host=localhost --mysql-port=3306 --mysql-user=root --mysql-password=your_password --mysql-db=test --tables=10 --table-size=100000 cleanup
```

---

## 常用命令速查

### MariaDB 服务管理

| 命令 | 说明 |
|------|------|
| `sudo systemctl start mariadb` | 启动 MariaDB |
| `sudo systemctl stop mariadb` | 停止 MariaDB |
| `sudo systemctl restart mariadb` | 重启 MariaDB |
| `sudo systemctl reload mariadb` | 平滑重载配置 |
| `sudo systemctl status mariadb` | 查看服务状态 |
| `sudo systemctl enable mariadb` | 开机自启 |
| `sudo systemctl disable mariadb` | 禁用开机自启 |

### 数据库操作

| 命令 | 说明 |
|------|------|
| `mysql -u root -p` | 登录 MariaDB |
| `mysql -u user -p dbname` | 登录指定数据库 |
| `SHOW DATABASES;` | 列出所有数据库 |
| `USE dbname;` | 切换数据库 |
| `SHOW TABLES;` | 列出当前数据库的表 |
| `DESC tablename;` | 查看表结构 |
| `CREATE DATABASE dbname;` | 创建数据库 |
| `DROP DATABASE dbname;` | 删除数据库 |

### 用户管理

| 命令 | 说明 |
|------|------|
| `CREATE USER 'user'@'host' IDENTIFIED BY 'password';` | 创建用户 |
| `DROP USER 'user'@'host';` | 删除用户 |
| `GRANT ALL ON dbname.* TO 'user'@'host';` | 授权 |
| `REVOKE ALL ON dbname.* FROM 'user'@'host';` | 撤销权限 |
| `FLUSH PRIVILEGES;` | 刷新权限 |
| `SHOW GRANTS FOR 'user'@'host';` | 查看用户权限 |

### 备份恢复

| 命令 | 说明 |
|------|------|
| `mysqldump -u root -p dbname > backup.sql` | 备份数据库 |
| `mysqldump -u root -p --all-databases > all.sql` | 备份所有数据库 |
| `mysql -u root -p dbname < backup.sql` | 恢复数据库 |
| `mysql -u root -p < all.sql` | 恢复所有数据库 |

---

## 常见问题排查

### 数据库初始化失败

#### 问题 1: Can't lock aria control file

**错误信息**:
```
[ERROR] mariadbd: Can't lock aria control file '/data/mysql/aria_log_control' for exclusive use, error: 11
```

**原因分析**:
数据目录被另一个 mysqld 进程锁定。

**解决方案**:
```bash
# 检查并停止所有 mysqld 进程
sudo pkill -9 mysqld
sleep 2

# 清空数据目录
sudo rm -rf /data/mysql/*

# 重新初始化（APT 安装方式）
sudo mysql_install_db --user=mysql --datadir=/data/mysql
```

#### 问题 2: ERROR 1146 (42S02): Table 'mysql.db' doesn't exist

**错误信息**:
```
ERROR 1146 (42S02) at line 1: Table 'mysql.db' doesn't exist
```

**原因分析**:
数据库未正确初始化，系统表不存在。

**解决方案**:
```bash
# 停止所有 mysqld 进程
sudo pkill -9 mysqld
sleep 2

# 清空数据目录
sudo rm -rf /data/mysql/*

# 重新初始化（APT 安装方式）
sudo mysql_install_db --user=mysql --datadir=/data/mysql

# 启动服务
sudo systemctl start mariadb

# 运行安全初始化
sudo mysql_secure_installation
```

#### 问题 3: 无法启动 MariaDB 服务

**错误信息**:
```
Job for mariadb.service failed because the control process exited with error code.
```

**原因分析**:
- 配置文件错误
- 权限问题
- 端口被占用

**解决方案**:
```bash
# 查看错误日志
sudo tail -f /var/log/mariadb/error.log

# 检查配置文件语法
sudo mysqld --validate-config

# 检查端口占用
sudo netstat -tulpn | grep 3306

# 检查目录权限
sudo ls -la /data/mysql
sudo ls -la /var/run/mariadb
```

#### 问题 4: APT 安装失败

**错误信息**:
```
E: Unable to locate package mariadb-server
```

**原因分析**:
- 仓库未正确添加
- 包名错误
- 网络问题

**解决方案**:
```bash
# 重新添加 MariaDB 仓库
curl -fsSL https://mariadb.org/mariadb_release_signing_key.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/mariadb.gpg
sudo add-apt-repository 'deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/mariadb/mariadb-10.11/ubuntu jammy main'
sudo apt update

# 重新安装
sudo apt install -y mariadb-server mariadb-client

# 启动服务
sudo systemctl start mariadb
```

### 服务无法启动

**问题**: `sudo systemctl start mariadb` 失败

**排查步骤**:

```bash
# 查看服务状态
sudo systemctl status mariadb

# 查看错误日志
sudo tail -f /var/log/mariadb/error.log

# 检查配置文件语法
sudo /usr/local/mariadb10.11/sbin/mysqld --defaults-file=/usr/local/mariadb10.11/etc/my.cnf --help --verbose

# 检查目录权限
ls -la /data/mysql
ls -la /var/run/mariadb
```

**常见原因**:
1. 数据目录权限不正确
2. Socket 目录不存在或权限不足
3. 配置文件语法错误
4. 端口 3306 被占用

### 连接被拒绝

**问题**: `ERROR 2002 (HY000): Can't connect to local MySQL server through socket`

**排查步骤**:

```bash
# 检查 Socket 文件是否存在
ls -la /var/run/mariadb/mariadb.sock

# 检查 Socket 路径配置
grep socket /usr/local/mariadb10.11/etc/my.cnf

# 检查服务是否运行
sudo systemctl status mariadb
```

**解决方法**:
1. 确保 Socket 目录存在并有正确权限
2. 检查配置文件中的 Socket 路径是否正确
3. 重启 MariaDB 服务

### 权限问题

**问题**: `ERROR 1045 (28000): Access denied for user`

**排查步骤**:

```bash
# 使用 root 用户登录（免密码）
mysql -u root

# 查看用户列表
SELECT User, Host FROM mysql.user;

# 退出
EXIT;
```

### 连接问题（DBeaver 等客户端）

**问题**: `Host '127.0.0.1' is not allowed to connect to this MariaDB server` 或 `Access denied for user 'root'@'127.0.0.1' (using password: YES)`

**根本原因**:
1. **连接方式差异**：
   - `localhost` 使用 Unix socket 连接
   - `127.0.0.1` 使用 TCP/IP 连接
   - 两者在 MariaDB 中是不同的访问路径

2. **配置问题**：
   - 配置文件中启用了 `skip-name-resolve` 选项，导致 MariaDB 忽略用户表中的权限条目
   - 权限表中的用户条目不完整

**解决方案**:

#### 步骤 1：检查并修改配置文件

```bash
# 编辑 MariaDB 配置文件
sudo sed -i 's/skip-name-resolve/# skip-name-resolve/' /usr/local/mariadb10.11/etc/my.cnf

# 重启 MariaDB 服务
sudo systemctl restart mariadb
```

#### 步骤 2：重置用户权限

1. **停止 MariaDB 服务**：
   ```bash
   sudo systemctl stop mariadb
   ```

2. **以 skip-grant-tables 模式启动**：
   ```bash
   sudo /usr/local/mariadb10.11/sbin/mysqld --defaults-file=/usr/local/mariadb10.11/etc/my.cnf --skip-grant-tables --skip-networking &
   sleep 5
   ```

3. **重置权限**：
   ```bash
   mysql --skip-password -e "TRUNCATE TABLE mysql.global_priv; INSERT INTO mysql.global_priv VALUES ('localhost', 'root', '{\"access\":18446744073709551615,\"plugin\":\"mysql_native_password\",\"authentication_string\":\"*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9\",\"account_locked\":false,\"password_last_changed\":0}'); INSERT INTO mysql.global_priv VALUES ('127.0.0.1', 'root', '{\"access\":18446744073709551615,\"plugin\":\"mysql_native_password\",\"authentication_string\":\"*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9\",\"account_locked\":false,\"password_last_changed\":0}'); INSERT INTO mysql.global_priv VALUES ('::1', 'root', '{\"access\":18446744073709551615,\"plugin\":\"mysql_native_password\",\"authentication_string\":\"*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9\",\"account_locked\":false,\"password_last_changed\":0}'); FLUSH PRIVILEGES;"
   ```

4. **停止 skip-grant-tables 进程**：
   ```bash
   sudo pkill -9 mysqld
   sleep 3
   ```

5. **正常启动 MariaDB**：
   ```bash
   sudo systemctl start mariadb
   sleep 5
   ```

#### 步骤 3：验证连接

1. **测试 127.0.0.1 连接**：
   ```bash
   mysql -h 127.0.0.1 -u root -e "SELECT VERSION();"
   ```

2. **测试 localhost 连接**：
   ```bash
   mysql -u root -e "SELECT VERSION();"
   ```

#### 步骤 4：DBeaver 连接配置

在 DBeaver 中创建新连接：

1. **基本连接**：
   - 主机：`localhost` 或 `127.0.0.1`
   - 端口：`3306`
   - 用户名：`root`
   - 密码：`123456`

2. **SSH 隧道连接**（推荐）：
   - 按照 SSH 连接指南配置

#### 注意事项

- 默认密码为 `123456`，建议在生产环境中修改
- 确保 `bind-address = 127.0.0.1` 只允许本地连接
- 生产环境建议创建专用数据库用户，而非使用 root

### 性能问题

**问题**: 数据库响应慢

**排查步骤**:

```bash
# 查看慢查询日志
sudo tail -f /var/log/mariadb/slow.log

# 查看当前连接
mysql -u root -p -e "SHOW PROCESSLIST;"

# 查看 InnoDB 状态
mysql -u root -p -e "SHOW ENGINE INNODB STATUS\G"

# 查看变量配置
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb%';"
```

**优化建议**:
1. 增加 `innodb_buffer_pool_size`
2. 优化慢查询 SQL
3. 添加适当的索引
4. 调整 `max_connections`

---

## 备份与恢复

### 全量备份

```bash
# 备份所有数据库
mysqldump -u root -p --all-databases --single-transaction --routines --triggers --events > /backup/mariadb_full_$(date +%Y%m%d_%H%M%S).sql

# 备份指定数据库（替换为实际数据库名）
mysqldump -u root -p --single-transaction --routines --triggers myapp > /backup/myapp_$(date +%Y%m%d_%H%M%S).sql

# 压缩备份
mysqldump -u root -p --all-databases --single-transaction | gzip > /backup/mariadb_full_$(date +%Y%m%d_%H%M%S).sql.gz
```

### 增量备份（使用二进制日志）

```bash
# 启用二进制日志（在 my.cnf 中配置）
# log_bin = /var/log/mariadb/mysql-bin
# binlog_format = ROW

# 刷新日志并记录位置
mysql -u root -p -e "FLUSH LOGS; SHOW MASTER STATUS;"

# 备份二进制日志
cp /var/log/mariadb/mysql-bin.* /backup/binlog/
```

### 恢复数据库

```bash
# 恢复全量备份
mysql -u root -p < /backup/mariadb_full_20250101_120000.sql

# 恢复压缩备份
gunzip < /backup/mariadb_full_20250101_120000.sql.gz | mysql -u root -p

# 恢复指定数据库（替换为实际数据库名）
mysql -u root -p myapp < /backup/myapp_20250101_120000.sql
```

### 自动备份脚本

```bash
sudo tee /usr/local/bin/mariadb_backup.sh << 'EOF'
#!/bin/bash

# 配置
BACKUP_DIR="/backup/mariadb"
RETENTION_DAYS=7
MYSQL_USER="root"
MYSQL_PASSWORD="<your_password>"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份文件名
BACKUP_FILE="$BACKUP_DIR/mariadb_full_$(date +%Y%m%d_%H%M%S).sql.gz"

# 执行备份
mysqldump -u $MYSQL_USER -p$MYSQL_PASSWORD --all-databases --single-transaction --routines --triggers --events | gzip > $BACKUP_FILE

# 删除旧备份
find $BACKUP_DIR -name "mariadb_full_*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $BACKUP_FILE"
EOF

# 添加执行权限
sudo chmod +x /usr/local/bin/mariadb_backup.sh

# 添加到 crontab（每天凌晨 2 点执行）
sudo crontab -e
# 添加以下行
# 0 2 * * * /usr/local/bin/mariadb_backup.sh >> /var/log/mariadb_backup.log 2>&1
```

---

## 性能优化建议

### 内存优化

根据服务器内存调整配置：

| 内存大小 | innodb_buffer_pool_size | innodb_log_file_size |
|---------|------------------------|---------------------|
| 2GB | 512M | 64M |
| 4GB | 1G | 128M |
| 8GB | 4G | 256M |
| 16GB | 8G | 512M |
| 32GB+ | 16G | 1G |

### InnoDB 优化

```ini
[mysqld]
# InnoDB 缓冲池大小（物理内存的 50-70%）
innodb_buffer_pool_size = 4G

# InnoDB 缓冲池实例数（每 1GB 一个实例）
innodb_buffer_pool_instances = 4

# InnoDB 日志文件大小（缓冲池的 25%）
innodb_log_file_size = 1G

# InnoDB 日志缓冲区大小
innodb_log_buffer_size = 64M

# InnoDB 刷盘策略（2=每秒刷盘，性能更好）
innodb_flush_log_at_trx_commit = 2

# InnoDB I/O 线程数
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# InnoDB I/O 容量
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
```

### 连接优化

```ini
[mysqld]
# 最大连接数
max_connections = 500

# 每个用户最大连接数
max_user_connections = 400

# 连接超时时间
wait_timeout = 28800
interactive_timeout = 28800
```

### 查询优化

```ini
[mysqld]
# 查询缓存大小
query_cache_size = 128M
query_cache_type = 1
query_cache_limit = 4M

# 临时表大小
tmp_table_size = 256M
max_heap_table_size = 256M

# 排序缓冲区
sort_buffer_size = 8M

# 读取缓冲区
read_buffer_size = 4M
read_rnd_buffer_size = 16M
```

### 监控与调优

```bash
# 安装性能监控工具
sudo apt install -y mariadb-client

# 查看性能指标
mysql -u root -p -e "SHOW GLOBAL STATUS;"

# 查看变量配置
mysql -u root -p -e "SHOW GLOBAL VARIABLES;"

# 使用 pt-query-digest 分析慢查询
sudo apt install -y percona-toolkit
pt-query-digest /var/log/mariadb/slow.log
```

---

## 附录

### 版本兼容性

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| 操作系统 | Ubuntu 22.04.5 LTS | 推荐版本 |
| PHP | 8.0+ | WordPress 6.4+ 需要 PHP 7.4+ |
| WordPress | 6.4+ | 推荐最新版本 |
| 内存 | 2GB+ | 最小 2GB，推荐 4GB+ |
| 磁盘 | 20GB+ | 数据库数据占用 |

### 配置文件路径汇总

| 配置项 | 路径 |
|--------|------|
| 安装目录 | `/usr` |
| 配置文件 | `/etc/mysql/mariadb.conf.d/50-server.cnf` |
| 数据目录 | `/data/mysql` (自定义) 或 `/var/lib/mysql` (默认) |
| Socket 文件 | `/var/run/mariadb/mariadb.sock` |
| PID 文件 | `/run/mysqld/mysqld.pid` |
| 错误日志 | `/var/log/mariadb/error.log` |
| 慢查询日志 | `/var/log/mariadb/slow.log` |
| 服务文件 | `/etc/systemd/system/mariadb.service` |
| 安装日志 | `/var/log/mariadb_install.log` |
| 密码文件 | `/root/.mariadb_credentials` |

### 自动化脚本说明

#### 安装脚本位置

自动化安装脚本位于：`/root/install_mariadb.sh`

#### 脚本功能

脚本自动完成以下任务：
1. 系统环境检查（内存、磁盘、端口）
2. 安装编译依赖
3. 创建目录结构和 mysql 用户
4. 下载 MariaDB 源码
5. 编译安装（使用多核加速）
6. 配置系统环境变量
7. 创建配置文件
8. 创建 Systemd 服务
9. 初始化数据库
10. 启动服务
11. 安全加固（删除匿名用户、删除测试数据库等）
12. 生成 SSL 证书
13. 验证安装
14. 保存凭证

#### 非交互式实现原理

| 原交互式操作 | 自动化替代方案 |
|-------------|---------------|
| `mysql_secure_installation` | 使用 SQL 脚本直接执行安全加固命令 |
| `openssl req` 交互输入 | 使用 `-subj` 参数指定证书信息 |
| SQL 交互执行 | 使用 Here Document 批量执行 |

#### 常见问题

**Q: 脚本执行失败怎么办？**  
A: 查看日志文件 `/var/log/mariadb_install.log` 获取详细错误信息。

**Q: 如何重新运行脚本？**  
A: 脚本支持幂等性执行，会自动备份已有数据目录后重新安装。

**Q: 密码保存在哪里？**  
A: 所有密码保存在 `/root/.mariadb_credentials`，安装完成后请妥善保管或删除。

**Q: 如何修改默认配置？**  
A: 通过设置环境变量覆盖默认值，详见 [快速自动化安装](#快速自动化安装推荐) 章节。

### 参考资源

- MariaDB 官方文档: https://mariadb.com/kb/en/documentation/
- MariaDB 下载: https://downloads.mariadb.org/
- WordPress 数据库要求: https://wordpress.org/about/requirements/
- 性能优化指南: https://mariadb.com/kb/en/optimization-and-tuning/

---

**文档结束**
