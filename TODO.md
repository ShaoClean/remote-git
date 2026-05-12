# Remote Git Manager - 开发计划

## 项目概述

在本地 Mac 上通过 Web UI 管理远程 Windows 机器上的 Git 仓库，解决挂载盘方式（如 Fork）操作延迟高、体验差的问题。

## 架构方案

### 推荐方案：混合架构（SSH 为核心 + 可选 Remote Agent）

```
┌───────────────── Mac (本地) ─────────────────┐
│  React Frontend  ◄──►  NestJS Backend         │
│                          │                     │
│                     SSH2 / WebSocket          │
│                          │                     │
│              ┌──── 内网通道 ────┐              │
└──────────────┼─────────────────┼──────────────┘
               │                 │
┌──────────────┼─────────────────┼──────────────┐
│  Windows (远程)                │              │
│  ┌────────────────────────────┘              │
│  │  SSH Server (OpenSSH)                     │
│  │  └─► Git CLI 执行命令                     │
│  │                                           │
│  │  [可选] Remote Agent (轻量 NestJS 服务)    │
│  │  └─► 提供 REST/WebSocket API              │
│  │  └─► 文件监听、仓库索引、实时推送         │
└───────────────────────────────────────────────┘
```

### 核心连接方式：SSH（ssh2）

**选择理由：**
- 远程 Windows 已有 SSH Server（OpenSSH），无需额外安装
- ssh2 是纯 Node.js 实现，跨平台，无需本地 SSH 依赖
- 支持公钥/密码认证、SFTP 文件传输、端口转发
- 可以执行任意 shell 命令，天然适配 Git CLI 操作

**可选增强：Remote Agent 模式**
- 在 Windows 上部署一个轻量 NestJS 服务
- 通过 WebSocket 实时推送仓库状态变更（比轮询 SSH 更高效）
- 支持 chokidar 文件监听，即时反映工作区变化
- 首期可不做，后期作为性能优化加入

---

## Phase 0: 项目初始化

- [ ] 初始化 monorepo 结构（nx 或 turborepo）
  ```
  remote-git/
  ├── apps/
  │   ├── web/          # React 前端
  │   └── server/       # NestJS 后端
  ├── packages/
  │   ├── shared/       # 共享类型/工具
  │   └── ssh-client/   # SSH 连接封装
  ├── package.json
  └── turbo.json
  ```
- [ ] 配置 TypeScript、ESLint、Prettier
- [ ] 初始化 NestJS 项目（`nest new`）
- [ ] 初始化 React 项目（Vite + React）
- [ ] 配置 Git 仓库、.gitignore

## Phase 1: SSH 连接层

### 1.1 SSH 连接管理模块

- [ ] 安装 `ssh2` 依赖：`npm i ssh2`
- [ ] 实现 SSH 连接服务（`ssh-client` 包）
  - 连接配置管理（host, port, username, auth）
  - 支持密码认证和公钥认证（`ssh2.Utils.parseKey`）
  - 连接池管理（复用连接、超时断开、自动重连）
  - 连接状态事件：`connect`, `disconnect`, `error`
  - SSH Config 文件解析（`~/.ssh/config`），自动读取已有配置
- [ ] 实现 SFTP 操作服务
  - 读取远程目录结构（ls / stat）
  - 文件内容读写（用于 diff 查看、文件编辑）
  - 文件上传/下载（批量操作时使用）

### 1.2 远程命令执行服务

- [ ] 封装 `execCommand(cmd: string, cwd?: string)` 方法
  - 基于 `ssh2.Client.exec()`
  - 支持 stdout/stderr 流式输出（用于长时间运行的 git 操作）
  - 超时控制与取消（`channel.close()`）
  - 返回 `{ exitCode, stdout, stderr }`
- [ ] Git 命令封装层
  - `gitStatus(repoPath)` → 解析 `git status --porcelain=v2 --branch`
  - `gitLog(repoPath, opts)` → 解析 `git log --format=...`
  - `gitDiff(repoPath, opts)` → 解析 `git diff` 输出
  - `gitBranchList(repoPath)` → 解析 `git branch -a -v`
  - `gitStashList(repoPath)` → 解析 `git stash list`
  - `gitRemoteList(repoPath)` → 解析 `git remote -v`
  - `gitExecute(repoPath, args)` → 通用 git 命令执行
- [ ] 命令输出解析器
  - porcelain v2 格式状态解析
  - log 格式化解析
  - diff 统一格式解析（可使用 `diff2html` 库渲染）

## Phase 2: NestJS 后端核心

### 2.1 连接管理模块

- [ ] `ConnectionModule` / `ConnectionService`
  - CRUD 远程主机配置（加密存储密码/密钥）
  - 测试连接（SSH handshake 验证）
  - 连接状态管理（已连接/断开/重连中）
- [ ] `ConnectionController`
  - `POST /connections` - 添加连接
  - `GET /connections` - 列出连接
  - `DELETE /connections/:id` - 删除连接
  - `POST /connections/:id/test` - 测试连接

### 2.2 仓库管理模块

- [ ] `RepositoryModule` / `RepositoryService`
  - 扫描远程目录查找 Git 仓库（`find . -name .git -type d`）
  - 仓库收藏/置顶（本地数据库存储）
  - 仓库状态概览（branch, status, ahead/behind）
- [ ] `RepositoryController`
  - `GET /repositories/scan?connectionId=&path=` - 扫描仓库
  - `GET /repositories/:id/status` - 仓库状态
  - `GET /repositories/:id/log` - 提交历史
  - `GET /repositories/:id/diff` - 文件差异
  - `GET /repositories/:id/branches` - 分支列表
  - `GET /repositories/:id/stashes` - stash 列表

### 2.3 Git 操作模块

- [ ] `GitModule` / `GitService`
  - Stage/Unstage 文件（`git add` / `git reset`）
  - Commit（`git commit -m` / 打开编辑器暂不支持，前端输入信息）
  - Push/Pull/Fetch（`git push` / `git pull` / `git fetch`）
  - Branch 操作（create/switch/delete/merge/rebase）
  - Stash 操作（save/pop/apply/drop）
  - Checkout 文件（`git checkout -- <file>`）
  - Reset（`git reset --soft/--mixed/--hard`）
  - Cherry-pick / Revert
- [ ] `GitController`
  - `POST /repositories/:id/stage` - stage 文件
  - `POST /repositories/:id/unstage` - unstage 文件
  - `POST /repositories/:id/commit` - 提交
  - `POST /repositories/:id/push` - 推送
  - `POST /repositories/:id/pull` - 拉取
  - `POST /repositories/:id/branch` - 分支操作
  - ...其他 git 操作端点

### 2.4 文件浏览模块

- [ ] `FileModule` / `FileService`
  - 远程目录树浏览（SFTP）
  - 文件内容读取（用于查看未追踪文件 diff）
  - 简易文件编辑（修改后保存，触发 git diff）

### 2.5 WebSocket 网关（实时通信）

- [ ] `EventsGateway`（`@WebSocketGateway`）
  - 实时推送命令执行输出（git push/pull 长时间操作）
  - 仓库状态变更通知
  - 连接状态变更通知
- [ ] 事件定义
  - `connection:status` - 连接状态变更
  - `repo:status` - 仓库状态变更
  - `command:output` - 命令执行输出（流式）
  - `command:exit` - 命令执行完成

## Phase 3: React 前端

### 3.1 基础框架

- [ ] 项目脚手架（Vite + React + TypeScript）
- [ ] UI 组件库选型与引入
  - 推荐 Ant Design（功能全面，表格/树/表单组件丰富）
  - 备选：Arco Design、Shadcn UI
- [ ] 状态管理（Zustand，轻量适合本项目）
- [ ] 路由配置（React Router v6）
- [ ] API 封装（axios + SWR/React Query）
- [ ] WebSocket 连接管理（socket.io-client）

### 3.2 连接管理页面

- [ ] 连接列表页
  - 展示已配置的远程主机
  - 连接状态指示器（在线/离线/重连）
  - 添加/编辑/删除连接
- [ ] 连接配置表单
  - 主机地址、端口、用户名
  - 认证方式切换（密码 / 密钥文件 / SSH Agent）
  - SSH Config 导入
  - 连接测试按钮

### 3.3 仓库仪表盘

- [ ] 仓库列表页
  - 卡片/列表视图切换
  - 显示：仓库名、当前分支、ahead/behind、最后提交
  - 快捷状态图标（clean / dirty / conflicting）
  - 扫描新仓库功能
- [ ] 仓库详情页
  - 左侧：文件树 / 文件列表
  - 右侧：diff 视图

### 3.4 Git 操作界面

- [ ] **Changes 视图**（核心功能）
  - 未暂存文件列表 + 变更类型标识（M/A/D/R/U）
  - 已暂存文件列表
  - 文件 diff 查看（side-by-side / unified 切换）
  - 单文件 stage/unstage
  - 全部 stage/unstage
  - Commit 输入框（消息 + 描述）
  - 最近 commit 消息快速选择
- [ ] **History 视图**
  - 提交历史列表（虚拟滚动，支持大量 commit）
  - commit 详情面板
  - 分支过滤
  - 搜索（作者/消息/SHA）
  - diff 查看（commit 间对比、与 HEAD 对比）
- [ ] **Branch 视图**
  - 分支列表（local / remote / tags）
  - 当前分支高亮
  - 创建/切换/删除分支
  - 合并/变基操作
  - ahead/behind 统计
- [ ] **Stash 视图**
  - Stash 列表
  - apply / pop / drop 操作
  - stash 内容查看
- [ ] **Remote 视图**
  - 远程仓库列表
  - push / pull / fetch 操作
  - force push（需二次确认）
  - 冲突提示与解决引导

### 3.5 Diff 渲染组件

- [ ] 集成 `react-diff-viewer` 或 `diff2html`
  - unified / split view 切换
  - 语法高亮（monaco-editor 或 highlight.js）
  - 行号显示
  - 文件头部信息（变更统计 +X -Y）
  - 文件过滤（只看某类变更）

## Phase 4: 安全与增强

- [ ] 凭据加密存储（使用 `crypto` 模块 AES-256 加密）
- [ ] SSH 密钥安全处理（不落盘，内存中使用）
- [ ] 连接超时与自动重连
- [ ] 操作审计日志
- [ ] 危险操作确认（force push、hard reset 等）
- [ ] 大仓库性能优化（diff 分页、log 懒加载）

## Phase 5: 可选增强（Remote Agent 模式）

- [ ] Windows 端轻量 Agent 服务
  - 基于 NestJS 的独立微服务
  - 自启动注册（Windows Service 或 PM2）
  - 文件系统监听（chokidar → WebSocket 推送）
  - 仓库索引缓存
- [ ] Mac 端自动发现 Agent（UDP 广播或手动配置）
- [ ] 连接模式切换（SSH 直连 / Agent 模式）

---

## 技术栈清单

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | React 18 + TypeScript | Vite 构建 |
| UI 组件 | Ant Design 5 | 表格/树/表单等企业级组件 |
| 状态管理 | Zustand | 轻量，适合中小项目 |
| 数据请求 | React Query + axios | 缓存与自动刷新 |
| 实时通信 | Socket.IO | 命令输出流式推送 |
| Diff 渲染 | react-diff-viewer-continued | 支持 unified/split |
| 语法高亮 | Prism.js / highlight.js | diff 代码高亮 |
| 后端框架 | NestJS | 模块化架构 |
| SSH 客户端 | ssh2 | 纯 JS 实现，无需本地 SSH |
| 数据存储 | SQLite (better-sqlite3) | 轻量，本地配置/凭据存储 |
| 加密 | Node.js crypto | AES-256-GCM 凭据加密 |
| 进程管理 | PM2（可选） | Agent 模式守护进程 |

## 关键依赖

```json
{
  "dependencies": {
    "ssh2": "^1.15.0",
    "@nestjs/core": "^10.x",
    "@nestjs/websockets": "^10.x",
    "@nestjs/platform-socket.io": "^10.x",
    "better-sqlite3": "^11.x",
    "socket.io-client": "^4.x",
    "react": "^18.x",
    "antd": "^5.x",
    "zustand": "^4.x",
    "@tanstack/react-query": "^5.x",
    "react-diff-viewer-continued": "^4.x",
    "diff2html": "^3.x",
    "axios": "^1.x",
    "react-router-dom": "^6.x"
  }
}
```

## 开发顺序建议

```
Phase 0 (项目初始化)
    ↓
Phase 1 (SSH 连接层) ← 核心基础设施
    ↓
Phase 2.1 + 2.2 (连接 + 仓库管理) ← 最小可用后端
    ↓
Phase 3.1 + 3.2 + 3.3 (前端基础 + 连接 + 仓库列表) ← 最小可用前端
    ↓
Phase 2.3 (Git 操作 API) + 3.4 (Changes 视图) ← MVP 核心功能
    ↓
Phase 2.5 + 3.5 (WebSocket + Diff) ← 体验增强
    ↓
Phase 2.4 + 3.4 其余 (文件浏览 + History/Branch/Stash) ← 功能完善
    ↓
Phase 4 (安全增强) ← 生产就绪
    ↓
Phase 5 (Agent 模式) ← 可选，按需开发
```

## MVP 目标（Phase 0-3.4）

首个可用版本应实现：
1. 添加 SSH 连接到远程 Windows
2. 扫描并选择 Git 仓库
3. 查看 status / stage / commit / push / pull
4. 查看基本 diff