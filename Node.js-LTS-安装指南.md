# Node.js LTS 安装指南

## 1. 使用管理脚本安装

本指南使用 `Node.js-LTS管理脚本.sh` 来安装和管理 Node.js LTS 版本，该脚本提供了简单的命令行界面来完成安装和卸载操作。

### 1.1 脚本位置

脚本位于：`/root/服务脚本库/执行脚本/Node.js-LTS管理脚本.sh`

### 1.2 安装 Node.js LTS

使用以下命令安装最新的 LTS 版本的 Node.js：

```bash
# 使用短选项
./Node.js-LTS管理脚本.sh -i

# 或使用长选项
./Node.js-LTS管理脚本.sh --install
```

安装过程会自动：
- 安装 nvm 到 `/opt/nvm` 目录
- 安装最新 LTS 版本的 Node.js
- 创建系统级符号链接
- 为 git 用户配置 nvm 环境
- 验证安装结果

### 1.3 卸载 Node.js 和 nvm

使用以下命令彻底卸载 Node.js 和 nvm：

```bash
# 使用短选项
./Node.js-LTS管理脚本.sh -u

# 或使用长选项
./Node.js-LTS管理脚本.sh --uninstall
```

卸载过程会自动：
- 移除系统级符号链接
- 删除 nvm 目录
- 清理 git 用户和 root 用户的 nvm 配置
- 移除残留目录
- 验证卸载结果

### 1.4 查看帮助信息

使用以下命令查看脚本的帮助信息：

```bash
# 使用短选项
./Node.js-LTS管理脚本.sh -h

# 或使用长选项
./Node.js-LTS管理脚本.sh --help
```

## 2. 手动安装方法（可选）

如果您需要手动安装，可以按照以下步骤操作：

### 2.1 安装 nvm

使用 curl 命令从 GitHub 仓库下载并安装 nvm：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

### 2.2 配置 nvm 环境

将 nvm 安装到 `/opt/nvm` 目录以提高安全性：

```bash
mkdir -p /opt/nvm
cp -r "$HOME/.nvm/"* "/opt/nvm/"
chown -R git:git /opt/nvm
chmod -R 755 /opt/nvm
```

### 2.3 配置用户环境

为 git 用户配置 nvm：

```bash
echo "" >> /home/git/.bashrc
echo "# Load NVM" >> /home/git/.bashrc
echo "export NVM_DIR=\"/opt/nvm\"" >> /home/git/.bashrc
echo "[ -s \"\$NVM_DIR/nvm.sh\" ] && \. \"\$NVM_DIR/nvm.sh\"  # This loads nvm" >> /home/git/.bashrc
echo "[ -s \"\$NVM_DIR/bash_completion\" ] && \. \"\$NVM_DIR/bash_completion\"  # This loads nvm bash_completion" >> /home/git/.bashrc
chown git:git /home/git/.bashrc
```

### 2.4 安装 Node.js LTS

```bash
source /opt/nvm/nvm.sh
nvm install --lts
nvm use --lts
nvm alias default node
```

### 2.5 创建系统级符号链接

```bash
local node_version=$(nvm current)
local node_bin_dir="/opt/nvm/versions/node/${node_version}/bin"
ln -sf "${node_bin_dir}/node" /usr/local/bin/node
ln -sf "${node_bin_dir}/npm" /usr/local/bin/npm
ln -sf "${node_bin_dir}/npx" /usr/local/bin/npx
chmod +x /usr/local/bin/node /usr/local/bin/npm /usr/local/bin/npx
```

## 3. 验证安装

验证 Node.js 和 npm 版本：

```bash
node -v
npm -v
```

## 4. 常用 nvm 命令

- 查看已安装的 Node.js 版本：
  ```bash
  nvm ls
  ```
- 切换到特定版本：
  ```bash
  nvm use <版本号>
  ```
- 查看可用的 Node.js 版本：
  ```bash
  nvm ls-remote
  ```

## 5. 注意事项

- nvm 安装在 `/opt/nvm` 目录，提高了安全性
- 脚本会自动为 git 用户配置 nvm 环境，确保 Act Runner 可以使用 Node.js
- 系统级符号链接确保所有用户都能访问 Node.js
- 卸载功能会彻底清理所有相关文件和配置

***

**文档创建时间**：2026-03-14
**适用系统**：Linux
**测试环境**：Ubuntu/Debian 系列
**更新内容**：添加了管理脚本使用方法，nvm 安装到 /opt/nvm 目录，支持命令行选项