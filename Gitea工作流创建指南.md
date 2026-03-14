# Gitea 工作流创建指南

## 什么是 Gitea 工作流？

Gitea 工作流是一种自动化工具，允许你在代码仓库中定义和执行 CI/CD 流程。通过工作流，你可以自动化构建、测试、部署等操作，提高开发效率和代码质量。

## 前提条件

1. 已部署 Gitea 服务器
2. 已安装并配置 Gitea Actions 运行器
3. 拥有仓库的管理权限

## 创建工作流的步骤

### 步骤 1：进入仓库的工作流页面

1. 登录 Gitea 并进入你的仓库
2. 点击顶部导航栏中的「工作流」选项卡
3. 如果你是第一次创建工作流，会看到「目前还没有工作流」的提示

### 步骤 2：创建工作流文件

工作流文件需要存放在仓库的 `.gitea/workflows/` 目录中，文件扩展名为 `.yml` 或 `.yaml`。

#### 方法一：通过 Gitea 界面创建

1. 在工作流页面点击「创建工作流」按钮
2. 选择一个模板或创建空白工作流
3. 编辑工作流文件内容
4. 点击「保存」按钮

#### 方法二：手动创建工作流文件

1. 在本地仓库中创建 `.gitea/workflows/` 目录（如果不存在）
2. 在该目录中创建一个 YAML 文件，例如 `main.yml`
3. 编辑工作流配置
4. 提交并推送更改到 Gitea 仓库

## 工作流文件结构

一个最基础的学习用工作流文件结构如下：

```yaml
name: 学习测试工作流

# 触发条件
on:
  # 当代码推送到 main 或 master 分支时触发
  push:
    branches: [ main, master ]
  # 当创建或更新拉取请求时触发
  pull_request:
    branches: [ main, master ]

# 作业
jobs:
  # 构建作业
  构建测试:
    # 运行环境
    runs-on: linux_amd64
    # 步骤
    steps:
    
    # 检查 Node.js 版本
    - name: 检查 Node.js 版本
      run: |
        node --version
        npm --version
    
    # 检出代码
    - name: 检出代码
      uses: actions/checkout@v3
    
    # 输出Hello World
    - name: 输出测试信息
      run: echo "Hello, Gitea 工作流！"
    
    # 查看系统信息
    - name: 查看系统信息
      run: uname -a
```

## 关键配置说明

### 触发条件（on）

定义工作流何时触发，常见的触发事件包括：

- `push`：当代码推送到指定分支时
- `pull_request`：当创建或更新拉取请求时
- `schedule`：按计划执行
- `workflow_dispatch`：手动触发

### 作业（jobs）

工作流由一个或多个作业组成，每个作业在独立的运行器上执行：

- `runs-on`：指定运行环境，如 `linux_amd64`、`ubuntu-latest`、`windows-latest` 等
- `steps`：作业的具体步骤

### 步骤（steps）

每个步骤可以是：

- `uses`：使用一个动作（action）
- `run`：执行一个命令
- `name`：步骤的名称
- `with`：传递给动作的参数



## 示例：Node.js 项目工作流

```yaml
name: Node.js CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: linux_amd64
    steps:
    - uses: actions/checkout@v3

    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '16'

    - name: Install dependencies
      run: npm install

    - name: Build
      run: npm run build

    - name: Test
      run: npm test
```

## 管理工作流运行

1. 在工作流页面查看所有工作流的运行状态
2. 点击具体工作流查看详细运行日志
3. 可以手动触发、取消或重新运行工作流

## 常见问题排查

### 工作流不运行

- 检查触发条件是否正确
- 确认运行器是否在线
- 检查工作流文件语法是否正确

### 工作流运行失败

- 查看详细日志找出错误原因
- 检查依赖项是否正确安装
- 确认运行环境是否满足要求

## 高级配置

### 使用环境变量

```yaml
jobs:
  build:
    runs-on: linux_amd64
    env:
      API_KEY: ${{ secrets.API_KEY }}
    steps:
    - name: Build
      run: echo "Using API key: $API_KEY"
```

### 矩阵构建

```yaml
jobs:
  build:
    runs-on: linux_amd64
    strategy:
      matrix:
        node-version: [14, 16, 18]
    steps:
    - uses: actions/checkout@v3
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
```

## 总结

通过本文档的指导，你应该能够在 Gitea 中成功创建和管理工作流。工作流可以大大提高开发效率，确保代码质量，是现代软件开发中不可或缺的工具。

如果需要更多帮助，请参考 Gitea 官方文档或查看工作流模板库获取更多示例。