# MewCode 自学习模块说明

日期：2026-08-05
范围：行为规则闭环（`com.mewcode.learn`）+ 记忆闭环（`com.mewcode.memory`）

---

## 1. 总览

自学习模块由**两条互补轨道**组成，内部均为「写入沉淀 → 磁盘持久化 → 跨会话读取注入」三环，全部**尽力而为、非阻塞**，任何失败不破坏主对话。

| 轨道 | 学什么 | 包 |
|---|---|---|
| A 行为规则闭环 | 工具「该按什么顺序 / 怎么做」的做事习惯 | `com.mewcode.learn` |
| B 记忆闭环 | 用户是谁 / 偏好 / 反馈 / 项目事实 | `com.mewcode.memory` |

```
工具调用观测 ──▶ 轨道A: 观测→提炼→Instinct 规则 ──▶ 注入(≥0.65)
对话内容   ──▶ 轨道B: extract→.md记忆 → 核心常驻 + 按需召回
                    │ 写入
                    ▼
            磁盘持久化(.mewcode/)
                    │ 下次会话读入
                    ▼
            注入系统提示 / 按需注入
```

---

## 2. 轨道 A：行为规则闭环

### 采集观测
- `ObservationStore implements ToolObserver`：每次工具执行后由 `StreamingExecutor` 回调 `observed(...)`，写一条观测（会话/轮次/序号、工具名、参数摘要、成败、耗时、时间戳）。
- 存储：`.mewcode/observations/observations_YYYYMM.jsonl`（按月分片）；超 5MB 或 8000 行滚入 `archive/`；超 90 天清理（`prune`）。

### 提炼（`InstinctEngine`）
- **统计路径**：`detectFrequentSequences` 检测高频工具连续序列（如 `Bash->Bash`）；置信度首见 0.5、复现 +0.15、上限 0.95、30 天半衰期衰减、<0.55 废弃回收。
- **AI 语义路径**：`AiInstinctExtractor` 复用 `LlmClient` 侧查，把观测摘要提炼成 `Trigger/Action` 规则；带水位线（≥50 新增触发）与熔断（连续失败 3 次暂停 AI 路径）。
- **去重聚合**：`JaccardDedup`（Union-Find + Jaccard）聚类近似规则、合并证据、取最大置信度；按 `instincts.json` / `auto-evolved.md` 落盘。

### 存储与注入
- `InstinctRepository`：机器状态 `instincts.json`（原子写）+ 人类可读 `auto-evolved.md`（按 domain 分组、覆盖重写）；user / project 双作用域目录。
- `InstinctInjector`：合并 user + project，project 覆盖 user，只注入高置信度（≥0.65）规则。

### 触发与接线（`MewCodeModel`）
- 每轮 `LoopComplete` 后 `executeLearn()` 在虚拟线程异步跑「统计增量 + AI 批量」。
- `/learn` 手动触发提炼并展示已学规则。
- 每次工具执行时由 `StreamingExecutor` 采集观测（`agent.setToolObserver(observationStore)`）。

---

## 3. 轨道 B：记忆闭环

### 写入
- `MemoryManager.extract()`：每 5 轮把对话喂给 LLM，按类型分类提炼：
  - `user`（用户级）：偏好 / 角色 / 背景，跨项目
  - `feedback`（用户级）：纠正 / 验证过的方法
  - `project`（项目级）：技术栈 / 约定 / 事实
  - `reference`（项目级）：外部资源
- 产出**单一事实源**：带 frontmatter 的 `.md` 记忆文件（`type` / `description`），写入 user（`~/.mewcode/memory`）与 project（`.mewcode/memory`）目录。`auto_memory.json` 保留为兼容读入口。

### 注入分层（本次改造后）
- **核心层（常驻）**：`getCoreMemoriesFrom` 只取 `type=user` 记忆（剥离 frontmatter），注入系统提示——量小、常驻，用作安全网。
- **按需层（召回）**：每条消息 `prefetchRelevantMemories` 从持久化索引召回相关记忆，两级筛选：
  1. `MemoryEmbeddingStore.searchTopK`（向量粗筛，复用 `LuceneVectorStore`，索引位于 `.mewcode/memory-vec`）；
  2. `MemoryRecall.selectRelevantMemories`（LLM 精挑 top-5）→ `renderReminder` 注入。
- 已移除会话开始时全量 `injectMemories` 的兜底，避免上下文被全量记忆撑爆。

### 降级保障
- 召回为空 / 索引 / embedding 失败 → 返回空不注入，回退「核心层常驻 + 原 embedSvc 路径」，主流程不受影响。

---

## 4. 两轨协作与上下文预算

- **分工**：A 管「怎么做」（行为规则），B 管「记住什么」（用户偏好 / 事实）。
- **都跨会话**：A 规则持久化并在下会话注入；B 核心常驻 + 其余按需召回。
- **注入分层**：系统提示只带核心记忆 + 高置信度规则；每条消息只按需召回相关记忆与规则，上下文保持常量级。

---

## 5. 验证

- `com.mewcode.learn.*` 与 `com.mewcode.memory.*` 单测全部通过。
- `build -x test` BUILD SUCCESSFUL（编译 + shadowJar）。
- 全量 160 个测试中 3 个失败均为**既有无关**问题：`ContextCompactorTest`、`TeamManagerTest`、`SessionSnapshotTest`（已在原始 `main` 上确认同样失败）。

> 构建说明：当前环境公网 Maven/Gradle 被认证代理拦截，需用 `huawei-mirror.init.gradle` 内网镜像 init script 构建，如 `.\gradlew.bat test --init-script huawei-mirror.init.gradle`。
