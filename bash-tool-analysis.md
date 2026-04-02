# Bash/Shell 工具实现分析文档

## 目录

1. [架构总览](#架构总览)
2. [核心文件说明](#核心文件说明)
3. [命令执行机制](#命令执行机制)
4. [stdout/stderr 捕获方式](#stdoutstderr-捕获方式)
5. [PTY vs 管道](#pty-vs-管道)
6. [执行结果与终端输出同步](#执行结果与终端输出同步)
7. [stdio 配置与终端集成](#stdio-配置与终端集成)
8. [Shell 命令完整流程](#shell-命令完整流程)
9. [关键设计决策](#关键设计决策)

---

## 架构总览

Bash 工具由五个核心组件协作实现：

| 文件 | 职责 |
|------|------|
| `src/tools/BashTool/BashTool.tsx` | 工具控制器：权限检查、进度循环、结果输出 |
| `src/utils/Shell.ts` | 底层 spawn 封装：文件模式/管道模式选择 |
| `src/utils/ShellCommand.ts` | 子进程生命周期管理 |
| `src/utils/shell/bashProvider.ts` | 命令字符串构建、环境变量配置 |
| `src/utils/task/TaskOutput.ts` | 输出缓冲与轮询 |

---

## 核心文件说明

### `BashTool.tsx`（行 624–1143）

- `call()` 方法（行 624–820）：接收 `BashToolInput`，执行权限检查（`parseForSecurity()`），委托给异步生成器 `runShellCommand()`
- `runShellCommand()` 异步生成器（行 826–1143）：构建执行选项，发起 `Shell.exec()`，维护进度循环

### `Shell.ts`（行 181–420）

核心函数签名：

```typescript
export async function exec(
  command: string,
  abortSignal: AbortSignal,
  shellType: ShellType,
  options?: ExecOptions,
): Promise<ShellCommand>
```

- 调用 `bashProvider` 构建完整命令字符串
- 选择文件模式或管道模式
- 调用 `child_process.spawn()` 启动子进程

### `ShellCommand.ts`（行 114–382）

`ShellCommandImpl` 类负责：

- 监听 `exit`、`error` 事件（行 272–273）
- 处理 abort 信号（行 264–267）
- 超时逻辑（行 275–279）
- 使用 `tree-kill` 终止进程树（行 337–343）

### `bashProvider.ts`（行 77–198）

`buildExecCommand()` 构建完整 shell 命令：

1. Source 环境变量快照
2. Source 会话级环境变量
3. 禁用扩展 glob 模式（安全加固：`shopt -u extglob`）
4. `eval <quoted-command>`
5. `pwd -P` 捕获执行后的工作目录

最终调用形如：`bash -c "<完整复合命令字符串>"`

---

## 命令执行机制

命令从工具控制层经多级封装最终通过 Node.js `child_process.spawn()` 执行：

```
BashTool.call()
  → runShellCommand()
    → Shell.exec()
      → bashProvider.buildExecCommand()   // 构建命令字符串
      → bashProvider.getSpawnArgs()       // [-c, ..., cmd]
      → bashProvider.getEnvironmentOverrides()  // TMUX, TMPDIR 等
      → (可选) SandboxManager.wrapWithSandbox()  // bwrap 沙箱包装
      → child_process.spawn(binShell, shellArgs, { ... })
```

---

## stdout/stderr 捕获方式

实现支持**两种模式**，在 `spawn()` 时选定：

### 文件模式（File Mode，默认）

适用于长时间运行或大输出（>30KB）的命令。

- `Shell.ts` 行 302–313：以 `O_APPEND` 标志打开临时文件
- 子进程的 stdout 与 stderr **均重定向到同一个文件描述符**（合并输出）
- 数据由子进程直接写入，Node.js 不介入数据路径
- POSIX `O_APPEND` 保证每次 `write()` 系统调用的原子性

> 注释原文（Shell.ts 行 289–298）：  
> _"On POSIX, O_APPEND makes each write atomic (seek-to-end + write)"_

### 管道模式（Pipe Mode）

当提供 `onStdout` 回调时激活，用于实时流式监控。

- `stdio: ['pipe', 'pipe', 'pipe']`，三条流均为 Node.js `Readable`
- `ShellCommand.ts` 行 66–104，`StreamWrapper` 类：
  - 包装子进程的 stdout/stderr 流
  - 转换为 UTF-8 字符串
  - 将数据送入 `TaskOutput` 缓冲
- 用于 hooks 与实时监控场景

### 对比总结

| 维度 | 文件模式 | 管道模式 |
|------|---------|---------|
| `stdio` 配置 | `['pipe', fd, fd]` | `['pipe', 'pipe', 'pipe']` |
| stdout/stderr | 合并写入同一文件 | 独立 Node.js 流 |
| 原子性保障 | `O_APPEND`（POSIX） | Node.js 流背压 |
| 适用场景 | 大输出、长时间任务 | 实时回调、hooks |
| Node.js 介入 | 否（子进程直写） | 是（经 StreamWrapper） |

---

## PTY vs 管道

> **结论：本实现不使用 PTY（伪终端）。**

系统完全依赖：
- 标准 Node.js `child_process.spawn()`
- 文件描述符（`O_APPEND` 临时文件）**或**管道（`stdio: 'pipe'`）

这是针对非交互式命令自动化的有意设计，带来以下优势：

- 无 TTY 控制序列干扰输出解析
- 管道模式下 stdout/stderr 可独立处理
- 文件模式下原子写入无需加锁
- 更低的系统开销

---

## 执行结果与终端输出同步

### 进度轮询循环（BashTool.tsx 行 1034–1139）

```typescript
while (true) {
  const progressSignal = createProgressSignal()
  const result = await Promise.race([resultPromise, progressSignal])

  if (result !== null) {
    // 命令已完成，返回最终结果
    return result
  }
  // 否则，向 agent 推送进度更新
  yield { type: 'progress', output, fullOutput, elapsedTimeSeconds, ... }
}
```

### TaskOutput 轮询机制

- `BashTool.tsx` 行 1029：命令启动时调用 `TaskOutput.startPolling()`
- 行 1141：命令结束时调用 `TaskOutput.stopPolling()`
- 共享轮询器，约每 **1 秒**触发一次 tick
- `onProgress` 回调参数：`(lastLines, allLines, totalLines, totalBytes, isIncomplete)`

### 进度事件推送（BashTool.tsx 行 663–678）

每次 tick 向 agent 发出 `bash_progress` 类型的工具使用事件：

```typescript
onProgress({
  toolUseID: `bash-progress-${progressCounter++}`,
  data: {
    type: 'bash_progress',
    output,
    fullOutput,
    elapsedTimeSeconds,
    totalLines,
    totalBytes,
    taskId,
    timeoutMs,
  }
})
```

### CWD 同步（Shell.ts 行 371–400）

工作目录的读取在 `.then()` 微任务中**同步执行**（紧接命令退出之后），防止命令完成与 CWD 更新之间的竞态条件。

### 自动后台化（Auto-backgrounding）

- **BashTool.tsx 行 880–900**：构建时设置 `shouldAutoBackground` 标志
- **行 976–982**：在 assistant 模式下，若命令运行超过 `ASSISTANT_BLOCKING_BUDGET_MS`（15 秒），自动迁移到后台任务：

```typescript
setTimeout(() => {
  if (shellCommand.status === 'running' && backgroundShellId === undefined) {
    assistantAutoBackgrounded = true
    startBackgrounding('tengu_bash_command_assistant_auto_backgrounded')
  }
}, 15_000).unref()
```

---

## stdio 配置与终端集成

### spawn 调用（Shell.ts 行 316–337）

```typescript
const childProcess = spawn(spawnBinary, shellArgs, {
  env: { ...subprocessEnv(), SHELL: binShell, ... },
  cwd,
  stdio: usePipeMode
    ? ['pipe', 'pipe', 'pipe']         // 管道模式
    : ['pipe', outputHandle?.fd, outputHandle?.fd],  // 文件模式
  detached: provider.detached,   // 成为进程组长，便于清理
  windowsHide: true,
})
```

### 环境变量隔离（bashProvider.ts 行 208–253）

| 变量 | 用途 |
|------|------|
| `TMUX` | 隔离 socket，避免干扰用户终端复用器 |
| `TMPDIR` | 沙箱专属临时目录 |
| `TMPPREFIX` | Zsh heredoc 临时文件位置 |
| 会话变量 | 每次调用时从会话上下文加载 |

### 沙箱集成（Shell.ts 行 259–273）

启用时，命令在 spawn 前被 `SandboxManager.wrapWithSandbox()` 包裹：

```typescript
if (shouldUseSandbox) {
  commandString = await SandboxManager.wrapWithSandbox(
    commandString,
    sandboxBinShell,
    undefined,
    abortSignal,
  )
  // 创建权限为 0o700 的沙箱临时目录
}
```

使用 Linux Bubblewrap（`bwrap`）容器化执行环境。

### 文件描述符管理（Shell.ts 行 347–358）

spawn 之后，父进程立即关闭自己持有的输出文件句柄副本；子进程保留其独立的 fd，持续写入，直到退出。

---

## Shell 命令完整流程

```
BashTool.call()
│  ├─ parseForSecurity()          权限与安全检查
│  ├─ 超时 / 后台化参数设置
│  └─ 调用 runShellCommand()
│
runShellCommand() [async generator]
│  ├─ 构建 ExecOptions（timeout, onProgress, shouldAutoBackground）
│  └─ 调用 Shell.exec()
│
Shell.exec()
│  ├─ bashProvider.buildExecCommand()   → 完整 shell 字符串
│  ├─ bashProvider.getSpawnArgs()       → ['-c', ..., cmd]
│  ├─ bashProvider.getEnvironmentOverrides()
│  ├─ (可选) SandboxManager.wrapWithSandbox()
│  ├─ 打开输出文件（文件模式）或准备管道（管道模式）
│  └─ child_process.spawn(binShell, shellArgs, { stdio, detached })
│
子进程执行
│  ├─ 文件模式：子进程 stdout+stderr → O_APPEND 临时文件
│  │    父进程通过 TaskOutput 轮询文件内容
│  └─ 管道模式：子进程 stdout/stderr → StreamWrapper → TaskOutput
│       Node.js 流实时回调
│
ShellCommandImpl (ShellCommand.ts)
│  ├─ 监听 exit / error 事件
│  ├─ 处理 abort 信号
│  ├─ 超时处理（退出码 143 = SIGTERM）
│  └─ 终止：tree-kill(pid, 'SIGKILL')（退出码 137）
│
BashTool 进度循环
│  ├─ Promise.race(resultPromise, progressSignal)
│  ├─ 每 tick 向 agent yield bash_progress 事件
│  └─ 超时自动后台化（>15s，assistant 模式）
│
最终结果返回
   ├─ 大输出（>64MB）截断并存入 tool-results 目录
   └─ 为模型生成输出预览（buildLargeToolResultMessage）
```

---

## 关键设计决策

| 设计决策 | 原因 |
|---------|------|
| **不使用 PTY** | 非交互式自动化无需终端控制序列；管道/文件 fd 更简单可靠 |
| **文件模式为默认** | 避免 Node.js 内存压力（大输出）；`O_APPEND` 保证无锁原子写入 |
| **Generator 流式输出** | `runShellCommand()` 是 async generator，进度增量推送不阻塞主流程 |
| **进程组隔离** | `detached: true` + `tree-kill` 确保 abort/timeout 时子孙进程全部清理 |
| **自动后台化** | 超过 15 秒的命令自动迁移后台，避免阻塞 assistant 响应循环 |
| **安全加固** | 禁用 extglob、沙箱包装（bwrap）、命令解析验证、环境变量隔离 |
