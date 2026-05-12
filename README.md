# learn-agent

个人学习 LLM Agent 开发的笔记仓库。按"问题—回答—参考资料"的方式持续整理。

## 文档索引

| 编号 | 主题 | 文档 |
| --- | --- | --- |
| 01 | Agent 开发的最佳实践与入门推荐 | [docs/01-agent-development-best-practices.md](docs/01-agent-development-best-practices.md) |
| 02 | 哪种开发语言更适合开发 Agent | [docs/02-language-choice-for-agents.md](docs/02-language-choice-for-agents.md) |

## 速览结论

- **入门路径**：裸 SDK → 工具调用 → 控制循环（ReAct）→ RAG/记忆 → 框架 → 评估与可观测性 → 工程化。
- **语言选择**：原型/智能层用 **Python**；服务层用 **TypeScript / Go / Java**；底层组件可用 **Rust**。多语言分层是常态。
- **稳定性 80% 来自**：结构化输出、严格的工具 schema、控制循环上限、可观测性、评测集——而不是换语言或换框架。
