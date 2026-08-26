Claude Code 长效记忆与自学习系统总结

本文介绍了一套为 Claude Code 设计的持久化记忆与自我学习系统，旨在解决 AI 助手“忘事”的问题。该系统通过行为观测、模式提炼和记忆注入三个核心层，实现从被动记录到主动学习的闭环，显著提升了开发效率并降低了 Token 消耗。

1. 系统架构总览

系统由三个协同工作的子系统构成，形成完整的自学习闭环：

行为观测层 (Observation Engine)：数据采集源。
模式提炼层 (Instinct Engine)：核心大脑，提炼行为规律。
记忆注入层 (Memory Engine)：知识持久化与应用。
2. 行为观测层：确定性数据捕获

为了确保数据的质量与完整性，系统摒弃了依赖模型主动调用的 Skill 机制，转而采用 Claude Code 原生的 Hook 机制实现 100% 确定性触发：

触发机制：
PreToolUse：仅在 Bash 调用前记录意图。
PostToolUse：匹配所有工具 (.*)，确保后置采集率 100%。
Stop：会话结束时触发分析与提炼流程。
数据存储：观测数据以 JSONL 格式存储，包含工具名、时间戳、输入参数等。
生命周期管理：通过 observations_rotate.py自动按月份分片归档（超过 5MB 或 8000 行），主文件仅保留最近 30 天数据，防止膨胀。
3. 模式提炼层：双路径智能分析

会话结束时，auto-analyze-instincts.py 并行执行两条路径，将观测数据提炼为原子化的 Instinct 规则：

路径 A：统计模式检测
方式：基于硬编码规则的高频工具调用序列检测。
置信度演化：首次发现为 0.5，重复验证增加自信，长期未触发则衰减（< 0.55 标记为废弃）。
路径 B：AI 语义分析
方式：调用本地 Claude 模型对观测摘要进行语义理解，捕捉统计方法无法识别的深层逻辑。
优势：互补性强，语义路径能发现复杂模式。
规则聚合与去重
数据模型：每个 Instinct 为独立的 Markdown 文件，包含 Trigger、Action、Evidence 及 Confidence。
语义去重：采用基于 Jaccard 相似度的 Union-Find 算法。通过提取英文技术关键词，跨语言识别相似意图，合并冗余规则。
领域聚合：按 Domain（如 workflow, testing, git）分组，生成 evolved-*.md 文件，最终合并写入 ~/.claude/rules/auto-evolved.md，供下次会话自动加载。
4. 记忆注入层：语义检索与精准注入

与规则化的 Instinct 不同，记忆层侧重存储知识性内容（如 Bug 修复、技术决策、项目上下文）。

核心链路
触发时机：SessionStart Hook 驱动，在第一条消息前自动就绪。
查询构造：结合当前工作目录 ($PWD) 和最近 3 条 Git Commit Message 构建查询向量，确保语境相关。
向量检索：使用本地 Embedding 模型 (nomic-embed-text) 和 Qdrant 向量库，计算余弦相似度，召回 Top-5 最相关记忆。
隐私保护：全本地运行，避免敏感代码上传云端。
上下文注入：将召回的记忆以结构化 Markdown 格式（含 [feedback], [project] 等标签）注入系统提示词。
5. 整体设计理念
数据流设计：从 Hook 采集 -> JSONL 存储 -> 双路径提炼 -> 去重聚合 -> 规则文件生成 -> 下次会话注入。
防膨胀设计：
数据层：观测日志归档、低置信度 Instinct 废弃、记忆 TTL 管理。
索引层：MEMORY.md 行数裁剪、规则文件每次覆盖重写、Jaccard 去重。
其他原则：原子性优先、隐私边界严格、置信度动态演化（具备“遗忘”能力）、Hook 优于 Skill。
6. 实际效果与收益

经过数月积累，系统已提炼数百条 Instinct，实际收益显著：

冷启动提速：上下文准备时间从 10 分钟降至 30 秒，首条响应即具备项目感知。
Token 节省：平均每会话节省约 78% 的 Token，通过精准召回替代全量粘贴。
错误降低：重复性错误率下降 80%，有效规避已知陷阱（如 CLI 检查、提交规范）。
知识复利：随着时间推移，规则积累产生指数级价值，AI 行为日益贴合个人习惯，形成高度定制化的编程伙伴。
7. 总结

该系统成功实现了 AI 助手的跨会话记忆与主动行为优化。通过确定性的数据采集、智能的模式提炼以及语义化的记忆注入，不仅解决了上下文丢失痛点，更通过“观察-提炼-应用”的闭环，让 AI 具备持续进化的能力，大幅提升了人机协作的效率与质量。

请你理解上述内容，并且分析本项目中，哪些上述功能实现了，哪些没实现，实现了的又有哪些不同

tooluse孤儿问题：
防线一：同进程内工具执行被 interrupt（StreamingExecutor）
  工具执行线程被中断时，StreamingExecutor 捕获 InterruptedException 并补一个错误结果（StreamingExecutor.java:122-124）：

  catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      results.add(new ToolExecResult(call.toolId(), "Error: interrupted", true));  // 闭合配对
  }

  防线二：Agent 主循环被外部取消（用户 Ctrl+C / 中断）
工具根本还没执行完，agent 也没来得及写 result。看 MewCodeModel.java:1643-1664 的中断处理：

runningAgent.cancel();                              // 停掉 agent 循环线程
...
Thread.ofVirtual().name("agent-interrupt-repair").start(() -> {
    runningAgent.awaitTermination(5000);
    if (oldQueue != null) oldQueue.clear();
    conversation.repairDanglingToolUses();          // ← 关键：修补孤儿配对
});

还有两处同样的关切
fork 路径：AgentTool.buildForkedConversation 对"带了 pending tool_use 但没结果"的 assistant 消息补 "(tool execution interrupted by fork)" 占位（AgentTool.java:365-371）。
上下文压缩：ContextCompactor.java:404,433 在压缩时也会避免把 tool_use/tool_result 这对拆开丢掉一半，防止压缩后留下孤儿。

tooluse不完整问题（json不完整）：

场景A：流正常读到 EOF，但没发 [DONE]（干净断流）
比如对端发了 Connection: close 关掉 socket，readLine() 返回 null，while 正常退出 → 走安全网 flushPendingToolCalls。

此时参数是半截 JSON，flushPendingToolCalls 里专门有 try/catch 兜底（OpenAiCompatClient.java:311-318）：

try {
    args = MAPPER.readValue(rawArgs, Map.class);   // 半截 JSON 会抛异常
} catch (Exception e) {
    args = Map.of();                                // 解析失败 → 空参数
}
queue.put(new StreamEvent.ToolCallComplete(callId, name, args));
结果：半截工具调用被当成"一个参数为空的完整调用"发出去 → Agent.agentLoop 收集到该 ToolCallInfo → StreamingExecutor 执行它。

执行时怎么办：空参数意味着必填参数缺失，工具通常返回 ToolResult.error("missing required arg ...")（如 AgentTool.java:269 的 description/prompt 校验）。这个错误作为"观察结果"回灌对话 → ReAct 进入下一轮 → 模型看到错误后自行修正/重新发起。所以这种"半截"不会硬崩，而是"执行失败→观察→自我纠正"。

场景B：抛 IOException（硬网络中断）
readLine() 直接抛异常 → doStream 的异常冒泡到 stream() 的 catch（OpenAiCompatClient.java:86），只发一个 StreamEvent.Error(...)：

queue.put(new StreamEvent.Error(classifyError(e).getMessage()));
此时没有 StreamEnd，也没有 ToolCallComplete。Agent.agentLoop 收到 Error 走 streamError 分支（Agent.java:416-445）：

若是可重试错误（context too long / rate limit）→ continue 重试
否则 break 结束回合 → 半截工具调用直接丢弃（因为 conv.addAssistantFull(...) 在错误检查之后才执行，Agent.java:479，根本没加到对话里）
硬错误下，局部变量 toolCalls 里的半截调用从未进对话、也从未执行，被静默丢弃。循环的 finally 补发一个 LoopComplete(0)（Agent.java:540-543），保证有终态。
