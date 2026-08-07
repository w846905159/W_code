# 埋点模块实现说明

## 1. 概述

MewCode（Java 版）内置的埋点（Telemetry）模块，负责记录 Agent 运行过程中的关键事件与指标，落盘为 JSONL 数据文件，供统计分析与 `AskUserDialog`/命令面板展示。

核心能力：
- **事件采集**：LLM 请求、文件变更、工具调用、Agent 错误、中断。
- **归因分析**：区分文件变更是来自 **Agent**（AI 写入）还是 **USER**（外部编辑器/IDE 修改）。
- **异步落盘**：事件经内存队列由虚拟线程批量写入 JSONL，不阻塞主流程。
- **统计展示**：`/telemetry` 命令聚合统计 AI/用户代码比例、token、时延、错误等。

## 2. 架构与数据流

```
┌────────────────────────────────────────────────────────────┐
│                          事件源                              │
│  Agent (LLM请求/错误)  StreamingExecutor (工具调用)         │
│  中断处理 (interrupt)   FileWriteService (AGENT写入)        │
│  FileWatcher (USER外部编辑)                                 │
└──────────────────────────┬─────────────────────────────────┘
                           │ emit(...)
                           ▼
                  ┌──────────────────┐
                  │  TelemetryEmitter │  单例
                  │  LinkedBlockingQueue(1024) 内存队列
                  └────────┬─────────┘
                           │ 虚拟线程 telemetry-writer 轮询
                           ▼
                  ┌──────────────────┐
                  │   TelemetryStore  │  append()
                  │  events_yyyymmdd.jsonl
                  └──────────────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │ /telemetry 命令    │  loadAll() 聚合统计
                  │  (CommandRegistry) │
                  └──────────────────┘
```

### 归因辅助结构

```
FileStateRegistry (内存快照 ConcurrentHashMap<path, content>)
        ▲                                  ▲
        │ set(path,content)                │ 对比快照 vs 磁盘
   FileWriteService                  FileWatcher
   (AGENT 写入时更新快照)              (检测外部改动，差异归因 USER)
```

## 3. 包结构

```
com.mewcode.telemetry
├── TelemetryEmitter.java      # 埋点核心（单例、异步队列、生命周期）
├── TelemetryEvent.java        # 事件类型定义（sealed interface + record）
├── TelemetryStore.java        # 持久化（JSONL 落盘 / 读取）
├── FileWriteService.java      # 统一写入层（diff + 发 AGENT 事件）
├── FileStateRegistry.java     # 文件内容快照注册表（归因共享）
└── FileWatcher.java           # 文件系统监听（检测 USER 外部改动）
```

关联模块：
- `com.mewcode.session.SessionManager.newId()` 生成会话 ID（`yyyyMMdd-HHmmss`）。
- `com.mewcode.command.CommandRegistry` 提供 `/telemetry` 统计命令。

## 4. 组件详解

### 4.1 TelemetryEvent（事件类型）

`sealed interface`，统一基方法：`eventType()`、`sessionId()`、`timestamp()`、`model()`。

| 事件类型 | `eventType` | 关键字段 | 触发方 |
|---|---|---|---|
| LlmRequestEvent | `llm_request` | input/outputTokens、cacheRead/CreationTokens、latencyMs、stopReason、success、errorMessage | Agent |
| FileChangeEvent | `file_change` | filePath、extension、addedLines、removedLines、source(AGENT/USER) | FileWriteService / FileWatcher |
| ToolCallEvent | `tool_call` | toolName、category、success、elapsedSec | StreamingExecutor |
| AgentErrorEvent | `agent_error` | errorType(stream_error/timeout/rate_limit/context_too_long)、message | Agent |
| InterruptEvent | `interrupt` | （无扩展字段） | 中断处理 |

辅助 record：`FileDiff(addedLines, removedLines)`。

### 4.2 TelemetryEmitter（核心单例）

- **初始化**：`init(workDir, modelName, sessionId)`，存储目录 `{workDir}/.mewcode/telemetry/`。重复 init 会先 shutdown 旧实例。
- **异步写入**：`LinkedBlockingQueue(1024)` 容量；虚拟线程 `telemetry-writer` 每秒 poll 队列，把事件交给 `store.append`。队列满则 `offer` 丢弃（不阻塞）。
- **生命周期**：`shutdown()` 置 running=false，线程退出前 `drainRemaining()` 冲刷队列。
- **静态访问**：`emit(event)`、`getSessionId()`、`getModelName()`、`getStore()`。
- **关闭**：`shutdownInstance()` 同步关闭并置空。

### 4.3 TelemetryStore（持久化）

- 按日分文件：`events_yyyyMMdd.jsonl`。
- `append(event)`：把事件转成 `LinkedHashMap`（基字段 `eventType/sessionId/ts/model` + 各事件扩展字段），`ObjectMapper.writeValueAsString` 追加写 JSONL。
- `loadAll()`：扫描 baseDir 下所有 `.jsonl`，逐行反序列化为 `Map`。
- 写入/读取 IO 异常均被捕获忽略，不干扰主业务。

### 4.4 FileWriteService（统一写入层 + AGENT 归因）

所有文件写入统一走此层。调用方完成实际写入，传入 **before 内容**，据此做真实 diff：

- `write(path, before, content, source)`：发 `file_change` 事件，并更新快照。
- `edit(path, before, after, source)`：同上（替换场景）。
- `diff(before, after)`：**基于集合的相似行匹配**——after 中与 before 匹配的行视为保留，其余计为 added/removed，返回 `FileDiff`。

调用方：`WriteFileTool` 调 `write`，`EditFileTool` 调 `edit`（均以 `EditSource.AGENT`）。

### 4.5 FileStateRegistry（快照注册表）

- 单例，`ConcurrentHashMap<String, String>` 保存**规范化绝对路径 → 文件内容快照**。
- 关键作用：让 Agent 写入路径与文件系统监听共享同一份快照，用于区分改动归属。

### 4.6 FileWatcher（USER 外部改动检测）

- `start(workDir)`：用 `WatchService` 递归注册目录树，监听 `ENTRY_CREATE/ENTRY_MODIFY`，平台线程 `file-watcher` 守护运行，带 500ms 去抖。
- **忽略**：`IGNORED_DIRS`（target/build/node_modules/.git/.gradle/.idea/.mewcode 等）；只关注 `CODE_EXTS` 代码/文本扩展名。
- **启动基线**：`collectPreExisting()` 记录启动时已存在的文件为基线，不算 USER 贡献。
- **process(path)**：
  - 无快照且非启动即存在 → 会话中新建文件，全部内容算 USER。
  - 有快照 → 与磁盘当前内容 diff，发 `file_change`（source=USER），更新快照。
  - 内容与快照一致 → 跳过（避免与 Agent 写入重复计数，防双计）。

## 5. 配置接线（MewCodeModel 启动）

```java
TelemetryEmitter.init(workDir, selectedProvider.getModel(), sessionId); // sessionId = SessionManager.newId()
com.mewcode.telemetry.FileStateRegistry.init();
com.mewcode.telemetry.FileWatcher.start(workDir);
```

启动即初始化埋点、快照注册表与文件监听。运行中模块自行注册 shutdown hook 停止 watcher。

## 6. 事件与统计展示

### 事件接入点

| 接入点 | 位置 |
|---|---|
| LLM 请求成功/失败 | `Agent.java`（StreamEnd/Error 后构造 LlmRequestEvent） |
| 流超时 | Agent 流超时分支发 AgentErrorEvent(timeout) |
| 工具调用 | `StreamingExecutor.java` 发 ToolCallEvent |
| 中断 | `MewCodeModel.java` 发 InterruptEvent |
| 文件写入（AGENT） | WriteFileTool / EditFileTool → FileWriteService |
| 文件外部改动（USER） | FileWatcher |

### `/telemetry` 命令

`CommandRegistry` 注册 `/telemetry`（别名 `/tl`）。读取 `loadAll()`，聚合：
- **perFile AI vs 用户代码归因**：`fp -> [aiAdd, userAdd, aiRem, userRem]`
- LLM 请求总数/成功率、总 input/output token、总时延
- 工具调用次数、错误总数与错误类型分布、中断次数

## 7. 依赖

| 依赖 | 用途 |
|---|---|
| `com.fasterxml.jackson.core:jackson-databind` (2.21.3) | 事件序列化/反序列化 |

> 纯 JDK + Jackson，无第三方重量依赖；文件监听用 JDK 自带 `WatchService`。

## 8. 测试

| 测试类 | 覆盖 |
|---|---|
| `SessionManagerTest` | 会话 ID 与持久化（关联） |

> 埋点核心逻辑（Emitter/Store/diff）当前以手工验证为主；`FileWriteService.diff` 的集合匹配算法为纯函数，适合后续补单测。

## 9. 已知限制与后续

- **队列有界**：队列满时 `offer` 直接丢弃事件，突发高吞吐下可能丢点，可考虑加背压或落盘备用队列。
- **diff 为近似**：基于行的集合匹配非严格 LCS 差异，重复行较多时 added/removed 计数可能不准。
- **文件监听目录规模**：递归注册全树，超大工程监听 key 较多，可能存在资源开销，可考虑按需懒注册。
- **归因边界**：Agent 写后内容与他人随后改动同时发生时，Watcher 靠快照式对比近似归因。
