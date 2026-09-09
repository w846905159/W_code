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


好，我用大白话重新讲一遍。先说整体，再拆开。

## 整体一句话

埋点数据要写进**你的硬盘文件**里。写硬盘慢，所以设计成"**先快速收下、记在账本上，后台慢慢誊抄到正式文件**"。崩溃防丢靠的是一本"**流水账本**"，防重复靠的是"**每张纸编号**"。

---

## 1. 什么是 WAL

WAL = Write-Ahead Log。你就记成 **"流水台账"**：

想象你在邮电所上班，用户来寄信（=程序产生一条埋点）。你不能用户一走就把信连夜送进仓库（=直接改正式文件），因为收信很快、送仓库慢。

现在改成：
1. **用户递来一封信，你马上把信封抄进一本"流水台账"上**（就是往 `events.wal` 文件里追加一行）。这个动作极快，几微秒。
2. **用户可以走了**，数据已经"记在账上"了。
3. **后台员工（writer 线程）隔一会儿（500ms）把台账里新抄的内容，誊抄到正式的分类账**（`events_日期.jsonl`，每个埋点一个文件，按天分）。
4. **誊抄完一批，就把台账上对应的页撕掉**（清空 WAL），下一批再从新页开始。

为什么这样防崩溃？

- 崩溃 = 邮电所突然停电/关门。
- 用户已交的信件（=埋点），**因为已经抄进台账，而台账是写在硬盘上的一个文件**，所以信没丢。
- 停电再开店时（=进程重启），你先翻台账，**把上次没来得及誊抄到分类账的部分，补抄一遍**，然后再正常营业。这一步就是代码里的"启动 recovery"（TelemetryEmitter.java:44-54）。

一句话：**先把单子落到一个磁盘文件上（快、不丢），再异步誊抄到正式文件（慢、慢慢做）**。这就是 WAL。

---

## 2. 为什么需要 seq（编号）和"幂等"

先解释"幂等"：**同一件事做了两遍，结果等于做一遍** = 幂等。比如盖章，盖两下还是同一个章。

为什么要编号？因为后台誊抄可能在中间崩掉：

- 台账里有信件 1、2、3、4、5 五封。
- 后台誊抄了 1、2、3，抄到一半停电了。
- 重启后重新翻台账，**从第 1 封开始再抄一遍** —— 结果信件 1、2、3 在分类账里出现了**两次**（重复埋点）。

怎么避免？给每封信编号：

- 誊抄时**把编号也写进分类账那行**（就是代码里的 `seq` 字段）。
- 读的时候（`loadAll`），**看到编号 2 出现过，就跳过第二次**。
- 抄两遍没关系，反正读的时候只认第一遍。这就是 **seq 作为"幂等键"去重**（TelemetryStore.java 的 `loadAll`）。

所以：
- **WAL 解决"丢"**：数据先进磁盘台账，崩溃补抄。
- **seq 解决"重复"**：补抄会导致重复，靠编号读到重复就忽略。

---

## 3. 那还剩什么丢数据的窗口？

需要诚实告诉你两件"仍可能丢"的事（我们之前聊过）：

**情况一：断电（真停电/拔电源）**，不是进程崩溃。
- 每次你"抄进台账"，其实是先写到**系统内存的草稿区**（page cache），真正落到硬盘是后台攒一批才强制写一次（fsync）。
- 如果刚抄完还没来得及强制写盘，电断了 → 最后那几条（最多约 500ms 内、最多一次批量）没了。
- 为什么不能每一条都强制写盘？因为强制写一次大约几毫秒，埋点一多就把程序卡死了。500ms 攒一次是"快"和"稳"的折中。

**情况二：极端残行**。写一半断电，最后一行只写了一半 → 重启读它读不懂（损坏行），就跳过它，等于丢这一条。这个概率极低。

---

## 4. 一张图收尾

```
程序产生埋点
   │
   ▼
①emit(): 立刻抄进"台账"events.wal（加编号seq，快，防丢的根基）
   │                       │
   │               后台writer线程每500ms：
   │                       ▼
   │                ②誊抄到正式账本 events_日期.jsonl
   │                       │
   │                ③确认抄好→撕掉台账对应页（清空WAL）
   ▼
返回，程序继续跑

任何一步崩了（kill -9/OOM）：
重启时 Wi— 先翻台账，把没誊完的补抄进正式账本，再营业
                     └→ 补抄产生重复行？读账时按编号 seq 去重忽略
```

- **生死线**：`emit()` 必须"把单子写进台账"成功后才返回 —— 这是不丢的根（TelemetryEmitter.java:168-193）。
- **兜底**：抄写失败（磁盘满等）不改台账，下次 500ms 再试；再崩溃就留到下次启动补抄。
- **唯一的商量余地**：断电的 ≤500ms 窗口，和"每条约几毫秒"二选一，遥测选前者。

现在回看你会出现的词就都懂了：**WAL=流水台账（防丢）、seq=编号（防重复）、recovery=重启补抄、fsync=强制写进硬盘（防断电）、at-least-once=最坏情况多几次（靠编号去重），但绝不丢**。
