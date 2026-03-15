# Gitea 优化技术文档

## 1. 概述

Gitea 完全支持多维度的 clone 加速方案，按使用权限分为客户端零配置立即可用优化、自有 Gitea 服务端全局调优、团队/大规模场景进阶方案三大类，覆盖所有使用场景，可直接按需选用。

## 2. 客户端优化（零配置）

### 2.1 浅克隆（最显著提速）

仅拉取指定分支的最新 1 个提交，完全跳过历史提交、其他分支和标签，数据量减少 90% 以上，提速效果最显著。

```bash
# 最简命令（仅拉取main分支最新版本）
git clone --depth 1 --single-branch --branch main <Gitea仓库地址>

# 后续如需完整历史，执行此命令补全
git fetch --unshallow
```

### 2.2 部分克隆

Gitea 默认开启 Git 部分克隆功能（需服务端 Git≥2.22，未关闭 DISABLE_PARTIAL_CLONE），可过滤掉大文件/历史对象，只拉取必要数据。

```bash
# 无blob克隆：只拉取代码结构和提交元数据，二进制大文件按需下载，适合带大量资源的仓库
git clone --filter=blob:none <Gitea仓库地址>

# 无树克隆：仅拉取最新的目录树，历史树对象完全跳过，适合历史悠久的超大仓库
git clone --filter=tree:0 <Gitea仓库地址>
```

### 2.3 稀疏克隆

只拉取指定目录的文件，其余目录完全不下载，适合 monorepo 大仓库。

```bash
# 1. 初始化稀疏克隆，不拉取完整文件
git clone --filter=blob:none --sparse <Gitea仓库地址>
# 2. 进入仓库目录，指定需要拉取的目录
cd 仓库名
git sparse-checkout set 你需要的目录1 目录2
```

## 3. 协议选择

| 协议 | 优势 | 适用场景 |
|------|------|----------|
| SSH | 长连接复用、免密认证、无 HTTPS 证书开销，长期使用更稳定 | 日常开发、有服务端 SSH 权限、内网环境 |
| HTTPS | 配置简单、支持代理、穿透性强 | 跨网访问、无 SSH 权限、需走代理加速 |
| Git（git://） | 无加密开销，传输速度最快 | 内网公开仓库、只读拉取场景 |

## 4. 代理配置

国内访问境外 Gitea 实例，或跨网访问时，可配置 HTTP/SOCKS5 代理加速：

### 4.1 HTTP/HTTPS 代理

```bash
# 全局代理（所有Git仓库生效）
git config --global http.proxy socks5://127.0.0.1:7890
git config --global https.proxy socks5://127.0.0.1:7890

# 仅对当前Gitea域名生效（推荐，不影响其他仓库）
git config --global http.https://你的gitea域名.proxy socks5://127.0.0.1:7890

# 取消代理
git config --global --unset http.proxy
git config --global --unset http.https://你的gitea域名.proxy
```

### 4.2 SSH 协议代理配置

编辑 ~/.ssh/config 文件：

```
Host 你的gitea域名
  HostName 你的gitea域名
  User git
  ProxyCommand nc -x 127.0.0.1:7890 %h %p
  IdentityFile ~/.ssh/你的私钥文件
```

## 5. Git 参数调优

通过调整 Git 参数，提升传输和处理效率：

```bash
# 降低压缩级别（1=最快速度，默认6，牺牲少量压缩比换取更快传输）
git config --global core.compression 1

# 增大HTTP缓冲区，避免大文件传输断连
git config --global http.postBuffer 524288000
git config --global http.maxRequestBuffer 104857600

# 启用并行拉取，多线程处理对象
git config --global fetch.parallel 0

# 启用文件系统缓存，加速索引加载
git config --global core.fscache true
git config --global core.preloadIndex true
```

## 6. 服务端优化

如果你是 Gitea 服务端管理员，可通过修改 custom/conf/app.ini 配置文件，从底层优化 clone 性能，全局提升所有用户的克隆速度。

### 6.1 [git] - Git 核心行为调优

```ini
[git]
# 启用部分克隆（允许客户端过滤大文件/历史）
DISABLE_PARTIAL_CLONE = true
# 自动GC间隔，优化仓库存储，减少拉取开销
AUTO_GC_INTERVAL = 24h
GC_INTERVAL = 72h

# Git全局配置，优化打包和传输性能
[git.config]
core.compression = 1          # 压缩级别 1（速度优先，默认 6）
pack.threads = 0               # 自动使用所有 CPU 核心打包
pack.windowMemory = 1g         # 打包时窗口内存限制（根据服务器内存调整）
pack.packSizeLimit = 2g        # 单个 pack 文件大小上限
pack.deltaCacheSize = 512m     # 增量缓存大小
http.postBuffer = 524288000    # HTTP 传输缓冲区（500MB）
receive.fsckObjects = false     # 非高安全场景关闭，减少传输校验开销
```

### 6.2 [server] - 服务传输层优化

```ini
[server]
RUN_MODE = prod # 生产模式，关闭调试日志，提升性能
# 启用内置SSH服务，比系统SSH性能更优、可控性更强
START_SSH_SERVER = true
SSH_DOMAIN = 你的Gitea域名
SSH_PORT = 22
SSH_LISTEN_PORT = 22

# 启用GZIP压缩，减少传输体积，平衡压缩比和CPU开销
ENABLE_GZIP = true
GZIP_LEVEL = 4 # 默认6，调低至4，速度优先

# LFS大文件服务启用
LFS_START_SERVER = true
LFS_ALLOW_PURE_SSH = false # 支持SSH协议拉取LFS文件
LFS_CONTENT_PATH = 你的LFS存储路径
```

### 6.3 [repository] - 仓库存储优化

```ini
[repository]
ENABLE_BUNDLE_DOWNLOAD = true # 启用bundle整包下载，超大仓库克隆速度显著提升
GC_ARGS = --auto --aggressive # 仓库GC参数，深度优化仓库结构，优化打包效率
# 限制打包文件大小，避免单文件过大导致传输卡顿
PACKED_GIT_LIMIT = 512m
PACKED_GIT_WINDOW_SIZE = 64m
```

### 6.4 [cache] - 缓存配置

```ini
# 启用Redis缓存，减少磁盘IO和数据库查询，大幅提升热仓库响应速度
[cache]
ENABLED = true
ADAPTER = redis
HOST = redis://127.0.0.1:6379/0 # 本地 Redis 地址
ITEM_TTL = 16h # 缓存项过期时间，根据实际访问模式调整
```

### 6.5 [database] - 数据库配置 

```ini
# 数据库连接池优化，避免连接瓶颈
[database]
MAX_IDLE_CONNS = 20
MAX_OPEN_CONNS = 100
CONN_MAX_LIFETIME = 300s
```

## 7. 大文件处理

仓库包含大量二进制大文件（模型、安装包、素材等）时，必须启用 Git LFS，配合对象存储 + CDN 实现极致加速。

### 7.1 LFS 配置

确保在 server 部分启用 LFS：

```ini
[server]
LFS_START_SERVER = true
LFS_ALLOW_PURE_SSH = true
LFS_CONTENT_PATH = 你的LFS存储路径
```

### 7.2 对象存储配置

建议将 LFS 文件存储在对象存储服务中，如 S3、OSS 等，配合 CDN 加速访问。

## 8. 团队/大规模场景优化

### 8.1 仓库结构优化

- 对于大型项目，考虑拆分为多个小型仓库
- 使用 monorepo 时，合理规划目录结构，利用稀疏克隆

### 8.2 访问控制优化

- 合理设置仓库权限，避免不必要的访问控制开销
- 使用团队管理功能，统一管理权限

### 8.3 监控与维护

- 定期检查仓库大小和健康状态
- 设置合理的 GC 策略，保持仓库清洁
- 监控服务器资源使用情况，及时扩容

## 9. 总结

通过以上优化策略，可以显著提升 Gitea 的克隆速度和整体性能。根据实际使用场景，选择合适的优化方案：

- 客户端用户：优先使用浅克隆、部分克隆等零配置方案
- 服务端管理员：通过配置文件优化服务端性能
- 团队/大规模场景：结合多种方案，实现极致性能

定期维护和监控是保持 Gitea 高性能的关键，建议建立完善的运维流程，确保系统持续稳定运行。