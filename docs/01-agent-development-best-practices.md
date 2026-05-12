# Agent 开发的最佳实践与入门推荐

> 本文聚焦"如何系统地开发一个 LLM Agent"，覆盖从概念到上线的全流程关键实践。
> 对象是已具备一定后端/脚本开发能力，但首次系统开发 Agent 的工程师。

---

## 1. 先理清概念：什么是 Agent

一个生产级 Agent 通常由以下要素组成：

| 要素 | 说明 |
| --- | --- |
| LLM（大模型） | 推理/决策核心，提供"思考"能力 |
| Prompt / System Instruction | 角色设定、行为约束、输出格式 |
| 工具（Tools / Function Calling） | 让 Agent 能调用外部能力（搜索、数据库、API、Shell 等） |
| 记忆（Memory） | 短期上下文 + 长期向量记忆/知识库（RAG） |
| 规划（Planning / Control Loop） | ReAct、Plan-and-Execute、Reflexion 等控制循环 |
| 执行环境（Runtime） | 沙箱、并发、超时、可观测性 |
| 评估（Eval） | 自动化指标、回归集、A/B |

入门时务必区分清楚：**单轮问答（Chat）≠ 工具调用（Function Call）≠ 多步 Agent（Agent Loop）≠ 多智能体（Multi-Agent）**。先做单 Agent + 工具调用，再考虑多 Agent。

---

## 2. 入门学习路径（建议按顺序）

1. **基础**：熟练使用任意一个主流 LLM API（OpenAI / Anthropic / 通义 / 智谱 / DeepSeek 等），跑通 Chat、Function Calling、流式输出三件套。
2. **Prompt 工程**：理解 system / user / assistant / tool 四类角色；学会写"角色 + 约束 + 输出格式 + 少量示例"的稳定 prompt。
3. **工具调用**：手写一个带 2~3 个工具的最小 Agent（如：搜索 + 计算器 + 文件读写），不要先上框架。
4. **控制循环**：实现 ReAct 风格循环（Thought → Action → Observation → Thought …），理解最大步数、终止条件、错误重试。
5. **RAG / 记忆**：接入向量库（如 pgvector / Milvus / Qdrant / Chroma），做一个能"基于资料回答"的助手。
6. **框架选型**：体验 1~2 个框架（LangGraph、LlamaIndex、AutoGen、CrewAI、Semantic Kernel、Spring AI 等），但**不要被框架绑死**。
7. **可观测性与评估**：接入 Trace（如 LangSmith / Langfuse / Phoenix / OpenTelemetry），建立测试集和回归评测。
8. **工程化**：超时/并发/限流/成本控制/安全审计/Prompt 版本化。

入门时常见的两个坑：
- **过早上框架**：不知道底层在做什么，出 bug 无从下手。**先用裸 SDK 写一个 100 行的 Agent，再看框架**。
- **过早做多 Agent**：单 Agent 还没稳定就堆 Multi-Agent，问题会被放大几倍。

---

## 3. 最佳实践清单（按主题）

### 3.1 Prompt 与上下文

- System Prompt 要"短而硬"：角色、能力边界、禁止项、输出 schema。
- 强制结构化输出：优先 JSON Schema / Tool Schema，不要让模型"自由发挥格式"。
- 上下文窗口要"按需注入"：不要一次性把所有历史都塞进去，做摘要 + 选择性召回。
- Prompt 当代码管理：放入版本控制（Git），打 tag，做 A/B；不要散落在业务代码字符串里。

### 3.2 工具设计（Function / Tool）

- **工具粒度要"刚刚好"**：太粗（一个 `do_everything`）模型选不准；太细（10 个相似的 `get_xxx`）模型会混淆。
- **工具名 + 描述就是 prompt**：用自然语言写清楚"什么时候用、参数含义、返回结构、失败语义"。
- **参数 schema 严格化**：用 JSON Schema 限定类型、枚举、必填；非法参数直接拒绝并返回结构化错误。
- **幂等与可重试**：工具调用要假设会被重复执行，写操作加幂等键。
- **危险工具加确认**：删除、转账、发邮件等副作用工具，强制走"二次确认"或"Human-in-the-loop"。
- **错误回传要可被模型理解**：返回 `{"ok": false, "error_code": "...", "message": "..."}`，而不是抛栈。

### 3.3 控制循环（Agent Loop）

- **设最大步数 / 最大 token / 最大耗时**：防死循环和成本失控。
- **检测重复动作**：连续两次相同 (tool, args) 就强制中断或换策略。
- **失败重试要有上限**：同一个工具调用失败 N 次后让 Agent 改变计划，而不是无限重试。
- **分离"规划"和"执行"**：复杂任务先让模型出一个 plan，再逐步执行，便于观测和回滚。
- **必要时引入 Reflection**：让模型在结束前自检"是否真的完成了用户目标"。

### 3.4 记忆与 RAG

- 短期记忆：会话内做滚动摘要，避免上下文爆炸。
- 长期记忆：分类存储（事实/偏好/历史任务），并打时间戳，**召回时考虑时效性**。
- RAG：先做好"切片 + embedding + 重排（rerank）"，比换模型更有效。
- 切片策略按"语义边界"切，不要按固定字符数硬切。
- 评估 RAG 时同时看 **召回率（Recall）** 和 **答案忠实度（Faithfulness）**。

### 3.5 安全与稳健性

- **Prompt Injection 防御**：不要无条件信任工具返回内容、网页内容、用户输入；做内容隔离（"以下是不可信内容"）。
- **越权防御**：工具层做权限校验，**不要依赖模型"自觉"**。
- **PII / 敏感信息**：日志脱敏，trace 中也要脱敏。
- **沙箱执行**：代码执行、Shell 操作一律走容器/沙箱，限制网络与文件系统。
- **成本上限**：每个会话/用户/任务设 token 与金额上限。

### 3.6 可观测性（Observability）

- 必须有 Trace：每一次 LLM 调用、每一次工具调用都要可追溯（输入、输出、耗时、token、错误）。
- 业务指标：任务完成率、平均步数、平均成本、回退率、用户满意度。
- 线上日志要能"重放"：保存完整 prompt + tool 调用序列，便于事后复盘。
- 推荐：LangSmith、Langfuse、Arize Phoenix、OpenTelemetry GenAI semantic conventions。

### 3.7 评估（Eval）

- 建一个"金标准任务集"（哪怕只有 30~50 条），每次 prompt/模型/工具变更都跑一遍。
- 评估方式分层：
  - **规则评估**：能用正则/JSON 校验就别用 LLM。
  - **LLM-as-Judge**：用更强模型评分，注意打分一致性。
  - **人工评估**：关键场景必须有。
- 区分 **离线评测**（开发时）与 **在线评测**（生产灰度 + A/B）。

### 3.8 工程化与团队协作

- Prompt、工具 schema、模型版本、Agent 配置 → 全部进 Git。
- 用配置（YAML/JSON）描述 Agent，而不是硬编码：方便切换模型、调参、灰度。
- CI 中跑评测集，prompt 改动也要触发 CI。
- 成本与延迟做 SLO：例如 P95 < 10s、单次任务成本 < $0.02。

---

## 4. 推荐的最小可运行 Agent 结构（伪代码视角）

```text
loop:
    messages = build_messages(system, history, user_input)
    response = llm.chat(messages, tools=tool_schemas)

    if response.tool_calls:
        for call in response.tool_calls:
            result = registry[call.name].run(call.args)   # 带超时/校验/审计
            history.append(tool_result(call.id, result))
        continue

    if is_final_answer(response):
        return response.content

    if step >= MAX_STEPS or cost >= MAX_COST:
        return fallback_answer()
```

记住三件事：**外层循环、工具注册表、终止条件**。所有花哨的框架本质都是在这三件事上做扩展。

---

## 5. 推荐资料（按"投入产出比"排序）

- **官方文档**：OpenAI / Anthropic / Google 的 Function Calling 与 Agents 文档（最权威，更新最快）。
- **Anthropic 的 ["Building effective agents"]**：少有的把"什么时候不该用 Agent"讲清楚的文章。
- **LangGraph 文档**：即使不用它，也值得读它的 State Machine 思路。
- **Lilian Weng 的 "LLM Powered Autonomous Agents" 博客**：概念全景。
- **OpenAI Cookbook / Anthropic Cookbook**：可运行的最小示例。
- **论文**：ReAct、Toolformer、Reflexion、Voyager、AutoGPT 论文。读思路即可，不必复现。

---

## 6. 一句话总结

> **先用 100 行裸 SDK 跑通一个带工具调用的单 Agent，再加记忆，再加评估，再考虑框架和多 Agent。**
> 工程上 80% 的稳定性来自：**结构化输出 + 工具 schema 严格 + 控制循环有上限 + 可观测性 + 评测集**。
