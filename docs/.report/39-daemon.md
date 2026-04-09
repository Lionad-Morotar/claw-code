# Unit 39: Daemon — 后台守护进程技术报告

## 一、功能概述

**Feature Flag**: `FEATURE_DAEMON=1`（通常与 `FEATURE_BRIDGE_MODE=1` 组合使用）

Daemon 是 claw-code 的后台守护进程系统，将 CLI 转变为可持续运行的后台服务。主进程（supervisor）管理多个 worker 进程的生命周期，通过 Unix 域套接字进行 IPC 通信。主要应用场景是配合 BRIDGE_MODE 提供远程控制服务。

**核心能力**：
- 持久化后台运行，接受远程会话连接
- 多 worker 进程并发处理不同任务类型
- 进程崩溃自动重启（指数退避策略）
- 优雅关闭与信号处理

---

## 二、实现架构

### 2.1 模块状态总览

| 模块文件 | 状态 | 工作量 | 说明 |
|----------|------|--------|------|
| `src/daemon/main.ts` | ✅ 完整实现 | 大 | Supervisor 主进程：启动 worker、生命周期管理、IPC |
| `src/daemon/workerRegistry.ts` | ✅ 完整实现 | 中 | Worker 注册与类型分发 |
| `src/entrypoints/cli.tsx` | ✅ 路由完成 | - | `--daemon-worker` 和 `daemon` 子命令路由 |
| `src/commands/remoteControlServer/` | ✅ 完整实现 | 大 | TUI 命令封装，与 daemon 集成 |
| `src/bridge/bridgeMain.ts` | ✅ 依赖模块 | 大 | headless bridge 运行核心 |

### 2.2 CLI 入口与路由

**快速路径分两个层级**：

1. **Worker 入口** (`cli.tsx:124-128`):
```typescript
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  const { runDaemonWorker } = await import('../daemon/workerRegistry.js')
  await runDaemonWorker(args[1])
  return
}
```

2. **Supervisor 入口** (`cli.tsx:186-195`):
```typescript
if (feature('DAEMON') && args[0] === 'daemon') {
  profileCheckpoint('cli_daemon_path')
  const { enableConfigs } = await import('../utils/config.js')
  enableConfigs()
  const { initSinks } = await import('../utils/sinks.js')
  initSinks()
  const { daemonMain } = await import('../daemon/main.js')
  await daemonMain(args.slice(1))
  return
}
```

### 2.3 Supervisor 主进程架构

**文件**: `src/daemon/main.ts`

**核心数据结构** (`main.ts:19-26`):
```typescript
interface WorkerState {
  kind: string
  process: ChildProcess | null
  backoffMs: number
  failureCount: number
  parked: boolean
  lastStartTime: number
}
```

**Supervisor 生命周期** (`main.ts:126-198`):
1. 解析 CLI 参数（`--dir`, `--spawn-mode`, `--capacity` 等）
2. 创建 Worker 列表（当前仅 `remoteControl` 类型）
3. 注册 SIGTERM/SIGINT 信号处理器
4. 启动 worker 并持续监控
5. 等待 abort 信号后优雅关闭

**Worker 重启策略** (`main.ts:256-304`):
- 退出码 78 (`EXIT_CODE_PERMANENT`) → 永久错误，park 不重启
- 运行 < 10s 崩溃 → failureCount 累加，≥5 次则 park
- 正常运行后崩溃 → 指数退避重启（初始 2s, 上限 120s, multiplier=2）

**Worker 环境配置** (`main.ts:213-223`):
```typescript
const env: Record<string, string | undefined> = {
  ...process.env,
  DAEMON_WORKER_DIR: dir,
  DAEMON_WORKER_NAME: config.name,
  DAEMON_WORKER_SPAWN_MODE: config.spawnMode || 'same-dir',
  DAEMON_WORKER_CAPACITY: config.capacity || '4',
  DAEMON_WORKER_PERMISSION: config.permissionMode,
  DAEMON_WORKER_SANDBOX: config.sandbox || '0',
  DAEMON_WORKER_CREATE_SESSION: '1',
  CLAUDE_CODE_SESSION_KIND: 'daemon-worker',
}
```

### 2.4 Worker Registry

**文件**: `src/daemon/workerRegistry.ts`

**Worker 类型分发** (`workerRegistry.ts:33-40`):
```typescript
switch (kind) {
  case 'remoteControl':
    await runRemoteControlWorker()
    break
  default:
    console.error(`Error: unknown daemon worker kind '${kind}'`)
    process.exitCode = EXIT_CODE_PERMANENT
}
```

**RemoteControl Worker** (`workerRegistry.ts:57-111`):
- 从环境变量读取配置（dir, spawnMode, capacity, permissionMode, sandbox 等）
- 调用 `runBridgeHeadless(opts, controller.signal)` 进入 headless bridge 循环
- 捕获 `BridgeHeadlessPermanentError` 区分永久/临时错误

### 2.5 RemoteControlServer TUI 命令

**文件**: `src/commands/remoteControlServer/remoteControlServer.tsx`

**启动流程** (`remoteControlServer.tsx:202-247`):
```typescript
function startDaemon(): void {
  const dir = resolve('.')
  const execArgs = [...process.execArgv, process.argv[1]!, 'daemon', 'start', `--dir=${dir}`]
  const child = spawn(process.execPath, execArgs, {
    cwd: dir,
    stdio: ['ignore', 'pipe', 'pipe'],
    detached: false,
  })
  // 捕获 stdout/stderr 日志，最多保留 50 行
}
```

**管理对话框** (`remoteControlServer.tsx:102-181`):
- 停止服务器（发送 SIGTERM → 10s 后 SIGKILL）
- 重启服务器（先 stop 后 start）
- 继续（关闭对话框）

---

## 三、与 BRIDGE_MODE 的关系

**双重门控** (`commands.ts:76-79`):
```typescript
const remoteControlServerCommand =
  feature('DAEMON') && feature('BRIDGE_MODE')
    ? require('./commands/remoteControlServer/index.js').default
    : null
```

**典型工作模式** (`bridgeMain.ts` 与 daemon 协作):
```
┌─────────────────────────────────────────────────────────┐
│                  claude daemon start                    │
│                        (supervisor)                     │
└─────────────────────────────────────────────────────────┘
                           │
                           ├─────────────────────────────┐
                           │                             │
                           ▼                             ▼
┌──────────────────────────────────────┐  ┌─────────────────────────────┐
│     Worker: remoteControl            │  │    (future worker types)    │
│   runBridgeHeadless() 运行中         │  │                             │
│  - 监听远程会话连接                   │  │                             │
│  - 多会话并发（capacity 控制）        │  │                             │
└──────────────────────────────────────┘  └─────────────────────────────┘
```

**环境变量组合**:
```bash
FEATURE_DAEMON=1 FEATURE_BRIDGE_MODE=1 bun run dev
claude daemon                        # 启动守护进程
claude --daemon-worker=assistant     # 以特定 worker 启动
```

---

## 四、使用方式

### 4.1 命令行用法

```bash
# 启动守护进程（默认子命令 start）
claude daemon

# 查看帮助
claude daemon --help

# 启动时指定参数
claude daemon start --dir=/path/to/project --capacity=8 --name=my-daemon

# 查看状态（stub）
claude daemon status

# 停止（stub）
claude daemon stop

# 从 TUI 启动
/remote-control-server
```

### 4.2 子命令参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--dir <path>` | 工作目录 | 当前目录 |
| `--spawn-mode <mode>` | worker 生成模式：`same-dir` \| `worktree` | `same-dir` |
| `--capacity <N>` | 每个 worker 最大并发会话数 | 4 |
| `--permission-mode <mode>` | 权限模式 | 未设置 |
| `--sandbox` | 启用沙箱模式 | 关闭 |
| `--name <name>` | 会话名称 | 未设置 |

---

## 五、关键设计决策

### 5.1 多进程架构
- **优势**: 进程隔离，单个 worker 崩溃不影响其他 worker
- **代价**: IPC 复杂度，进程间通信需要协议设计

### 5.2 指数退避重启
- 初始 2s，每轮翻倍，上限 120s
- 防止快速崩溃循环占用系统资源
- 连续 5 次快速失败后 park 该 worker

### 5.3 永久错误识别
- 退出码 78 (`EX_CONFIG`) 标识不可恢复错误
- 典型场景：trust not accepted, git repo 缺失
- 避免 supervisor 无意义重试

### 5.4 与 BRIDGE_MODE 强绑定
- 守护进程最常见的用途是提供远程控制服务
- 双重门控确保依赖条件满足

---

## 六、文件索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `src/daemon/main.ts` | - | Supervisor 主进程（完整实现） |
| `src/daemon/workerRegistry.ts` | - | Worker 注册与分发（完整实现） |
| `src/entrypoints/cli.tsx` | #L124-L128 | Worker 快速路径路由 |
| `src/entrypoints/cli.tsx` | #L186-L195 | Supervisor 快速路径路由 |
| `src/commands.ts` | #L76-L79 | remoteControlServer 命令双重门控 |
| `src/commands/remoteControlServer/index.ts` | - | TUI 命令入口 |
| `src/commands/remoteControlServer/remoteControlServer.tsx` | - | React 管理界面 |
| `src/bridge/bridgeMain.ts` | - | Headless bridge 运行核心 |

---

## 七、待补全内容

根据原始文档，以下内容需要后续完善：

1. **IPC 协议层**: Supervisor-Worker 通信协议当前未实现（仅 pipe stdout/stderr）
2. **更多 Worker 类型**: 当前仅 `remoteControl`，预期有 `assistant`, `bridge-sync`, `proactive`
3. **状态查询 CLI**: `daemon status` 和 `daemon stop` 为 stub
4. **PID 文件**: 用于外部进程管理

---

## 八、源码引用汇总

| 引用点 | 文件 | 行号 |
|--------|------|------|
| EXIT_CODE_PERMANENT 定义 | `main.ts:9`, `workerRegistry.ts:15` | - |
| 退避参数 | `main.ts:14-17` | - |
| WorkerState 接口 | `main.ts:19-26` | - |
| daemonMain 入口 | `main.ts:41-63` | - |
| runSupervisor | `main.ts:126-198` | - |
| spawnWorker | `main.ts:203-305` | - |
| runDaemonWorker | `workerRegistry.ts:26-41` | - |
| runRemoteControlWorker | `workerRegistry.ts:57-112` | - |
| CLI daemon-worker 路由 | `cli.tsx:124-128` | - |
| CLI daemon 路由 | `cli.tsx:186-195` | - |
| remoteControlServer 门控 | `commands.ts:76-79` | - |
| startDaemon | `remoteControlServer.tsx:202-247` | - |
| ServerManagementDialog | `remoteControlServer.tsx:102-181` | - |

---

**生成时间**: 2026-04-09
**原始文档**: https://ccb.agent-aura.top/docs/features/daemon
