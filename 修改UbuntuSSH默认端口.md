# 修改 Ubuntu SSH 默认端口技术文档

## 文档信息
- **标题**: 修改 Ubuntu SSH 默认端口
- **适用系统**: Ubuntu 18.04 / 20.04 / 22.04 / 24.04
- **操作权限**: root 或 sudo 用户
- **风险等级**: 中（配置错误可能导致无法远程连接）

---

## 一、操作目的

将 SSH 服务的默认端口从 22 修改为自定义端口（如 2222），以提高服务器安全性，减少自动化扫描和暴力破解攻击。

---

## 二、前置检查

### 2.1 确认当前 SSH 端口
```bash
# 查看当前 SSH 服务监听的端口
ss -tlnp | grep ssh
# 或
netstat -tlnp | grep ssh
```

### 2.2 确认 SSH 服务状态
```bash
# 检查 SSH 服务是否运行
systemctl status ssh
# 或
systemctl status sshd
```

### 2.3 备份配置文件
```bash
# 备份原始 SSH 配置文件
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%Y%m%d_%H%M%S)
```

### 2.4 检查防火墙状态
```bash
# 查看防火墙状态
ufw status
# 或查看 iptables 规则
iptables -L -n | grep 22
```

### 2.5 检查 SELinux 状态（如适用）
```bash
# 检查 SELinux 状态
sestatus
```

---

## 三、修改 SSH 端口步骤

### 3.1 选择新端口
- 建议使用 1024-65535 范围内的端口
- 避免使用常用服务端口（如 80、443、3306 等）
- 示例新端口: 2222

### 3.2 修改 SSH 配置文件

#### 3.2.1 编辑配置文件
```bash
# 使用文本编辑器打开 SSH 配置文件
nano /etc/ssh/sshd_config
# 或
vim /etc/ssh/sshd_config
```

#### 3.2.2 修改 Port 配置
```bash
# 找到以下行（可能被注释）
#Port 22

# 修改为（取消注释并修改端口号）
Port 2222

# 如需保留 22 端口作为备用，可添加多行
Port 22
Port 2222
```

#### 3.2.3 可选：修改监听地址
```bash
# 如需限制监听特定 IP（默认监听所有地址）
#ListenAddress 0.0.0.0
ListenAddress 192.168.1.100
```

### 3.3 保存并退出
- nano: `Ctrl+O` 保存，`Ctrl+X` 退出
- vim: `:wq` 保存并退出

---

## 四、配置防火墙

### 4.1 UFW 防火墙配置
```bash
# 添加新端口到防火墙允许列表
ufw allow 2222/tcp

# 如需删除旧端口规则（确认新端口可用后再执行）
ufw delete allow 22/tcp

# 重新加载防火墙
ufw reload

# 查看防火墙状态
ufw status
```

### 4.2 iptables 配置
```bash
# 添加新端口规则
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT

# 保存规则（Ubuntu 使用 iptables-persistent）
netfilter-persistent save

# 或手动保存
iptables-save > /etc/iptables/rules.v4
```

### 4.3 云服务器安全组配置
如使用云服务器（阿里云、腾讯云、AWS、Azure 等），需在控制台安全组中添加新端口的入站规则。

---

## 五、配置 SELinux（如启用）

### 5.1 检查 SELinux 状态
```bash
sestatus
```

### 5.2 添加 SELinux 端口标签
```bash
# 为 SSH 服务添加新端口标签
semanage port -a -t ssh_port_t -p tcp 2222

# 或修改现有端口（如果 2222 已被其他服务使用）
semanage port -m -t ssh_port_t -p tcp 2222
```

### 5.3 验证 SELinux 配置
```bash
# 查看 SSH 相关端口
semanage port -l | grep ssh
```

---

## 六、重启 SSH 服务

### 6.1 测试配置文件语法
```bash
# 检查配置文件语法是否正确
sshd -t
# 或
/usr/sbin/sshd -t
```

### 6.2 重启 SSH 服务
```bash
# 方法1：使用 systemctl（推荐）
systemctl restart sshd

# 方法2：使用 service 命令
service ssh restart
# 或
service sshd restart
```

### 6.3 验证服务状态
```bash
# 检查 SSH 服务状态
systemctl status sshd

# 确认新端口正在监听
ss -tlnp | grep 2222
# 或
netstat -tlnp | grep 2222
```

---

## 七、验证连接

### 7.1 本地测试
```bash
# 在服务器本地测试新端口连接
ssh -p 2222 localhost
# 或
ssh -p 2222 用户名@127.0.0.1
```

### 7.2 远程测试
```bash
# 从另一台机器测试新端口连接
ssh -p 2222 用户名@服务器IP地址
```

### 7.3 测试通过后关闭旧端口
确认新端口可以正常连接后，可删除 22 端口配置：
```bash
# 编辑配置文件，删除或注释 Port 22
nano /etc/ssh/sshd_config

# 重启 SSH 服务
systemctl restart sshd

# 删除防火墙规则
ufw delete allow 22/tcp
```

---

## 八、故障排查

### 8.1 无法连接新端口

#### 检查服务状态
```bash
systemctl status sshd
journalctl -u sshd -n 50
```

#### 检查端口监听
```bash
ss -tlnp | grep ssh
ss -tlnp | grep 2222
```

#### 检查配置文件语法
```bash
sshd -t
```

#### 检查防火墙规则
```bash
ufw status
iptables -L -n | grep 2222
```

#### 检查 SELinux 日志
```bash
cat /var/log/audit/audit.log | grep ssh
cat /var/log/audit/audit.log | grep denied
```

### 8.2 配置文件恢复
如果配置错误导致无法连接，可通过控制台或 VNC 登录恢复：
```bash
# 恢复备份的配置文件
cp /etc/ssh/sshd_config.bak.XXXXXX /etc/ssh/sshd_config
systemctl restart sshd
```

### 8.3 常见错误

| 错误信息 | 可能原因 | 解决方案 |
|---------|---------|---------|
| Connection refused | 服务未启动或端口未监听 | 检查服务状态和端口配置 |
| Connection timed out | 防火墙或安全组阻止 | 检查防火墙和安全组规则 |
| Permission denied | SELinux 阻止 | 配置 SELinux 端口标签 |
| Bad configuration option | 配置文件语法错误 | 检查配置文件语法 |

---

## 九、回滚方案

### 9.1 快速回滚命令
```bash
# 1. 恢复配置文件
cp /etc/ssh/sshd_config.bak.XXXXXX /etc/ssh/sshd_config

# 2. 重启 SSH 服务
systemctl restart sshd

# 3. 恢复防火墙规则
ufw allow 22/tcp
ufw delete allow 2222/tcp
ufw reload
```

### 9.2 紧急恢复（无法 SSH 时）
- 通过云服务商控制台 VNC 登录
- 通过物理服务器本地登录
- 修改配置文件后重启服务

---

## 十、安全建议

### 10.1 额外安全措施
```bash
# 1. 禁用 root 登录
PermitRootLogin no

# 2. 使用密钥认证
PasswordAuthentication no
PubkeyAuthentication yes

# 3. 限制允许登录的用户
AllowUsers username1 username2

# 4. 设置空闲超时
ClientAliveInterval 300
ClientAliveCountMax 2

# 5. 限制登录尝试次数
MaxAuthTries 3
```

### 10.2 监控和日志
```bash
# 查看 SSH 登录日志
tail -f /var/log/auth.log

# 查看失败的登录尝试
grep "Failed password" /var/log/auth.log

# 查看成功的登录
grep "Accepted" /var/log/auth.log
```

---

## 十一、脚本编写要点

### 11.1 脚本应包含的功能
1. 自动检测当前 SSH 端口
2. 备份现有配置文件
3. 验证新端口是否可用（未被占用）
4. 修改配置文件
5. 配置防火墙
6. 检查并配置 SELinux（如启用）
7. 验证配置文件语法
8. 重启 SSH 服务
9. 测试新端口连接
10. 提供回滚机制

### 11.2 脚本注意事项
- 必须使用 `set -e` 确保错误时退出
- 所有修改前必须备份
- 提供交互式确认和自动模式选项
- 记录所有操作日志
- 保留原始配置以便回滚

---

## 十二、参考命令速查表

| 操作 | 命令 |
|-----|------|
| 查看 SSH 状态 | `systemctl status sshd` |
| 重启 SSH 服务 | `systemctl restart sshd` |
| 检查端口监听 | `ss -tlnp \| grep ssh` |
| 测试配置文件 | `sshd -t` |
| 备份配置 | `cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak` |
| 防火墙添加端口 | `ufw allow 2222/tcp` |
| 防火墙删除端口 | `ufw delete allow 22/tcp` |
| SELinux 添加端口 | `semanage port -a -t ssh_port_t -p tcp 2222` |
| 查看登录日志 | `tail -f /var/log/auth.log` |

---

## 十三、相关文件路径

| 文件/目录 | 说明 |
|----------|------|
| `/etc/ssh/sshd_config` | SSH 服务端配置文件 |
| `/etc/ssh/sshd_config.d/` | SSH 配置片段目录 |
| `/etc/ssh/ssh_config` | SSH 客户端配置文件 |
| `/var/log/auth.log` | 认证日志（Ubuntu/Debian） |
| `/var/log/secure` | 认证日志（CentOS/RHEL） |
| `/var/log/audit/audit.log` | SELinux 审计日志 |

---

## 版本历史

| 版本 | 日期 | 修改内容 |
|-----|------|---------|
| 1.0 | 2026-03-11 | 初始版本 |
