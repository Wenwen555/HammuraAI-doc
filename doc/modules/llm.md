# LLM 调用模块

## 模块目标

LLM 模块负责把检索得到的上下文、会话历史和业务规则组织成最终生成调用。

它不是简单的 API 封装，而是 HammuraAI 的生成控制层。

## 当前实现

核心文件：

- `backend_app/app/services/llm.py`
- `backend_app/app/services/context_engine.py`
- `backend_app/app/services/workflow.py`

## 当前能力

### 统一客户端入口

`LLMService` 负责：

- 创建 OpenAI 兼容异步客户端
- 根据 tier 和 action 选择模型
- 执行同步或流式调用

### 流式输出

系统支持：

- SSE 流式生成
- token 用量回传
- 最终完成事件

### 结构化辅助生成

当前还承担一些法律场景辅助任务：

- 前端摘要生成
- 结构化案件信息更新
- 对话历史摘要
- 后续问题建议

## 与 RAG 检索链路的关系

在 PostgreSQL + pgvector 路线下，LLM 模块不直接访问数据库，也不直接决定检索排序。它接收检索与重排模块整理好的上下文包，并负责把它转成可控的模型输入。

标准输入应包含：

| 字段 | 作用 |
| --- | --- |
| `question` | 用户原始问题或经过结构化后的法律请求。 |
| `chat_history` | 必要的多轮对话上下文。 |
| `retrieved_chunks` | 来自 pgvector、全文检索和重排后的证据片段。 |
| `citations` | 每个证据片段的来源、标题、条文或案件定位。 |
| `constraints` | 是否允许拒答、是否需要追问、输出格式要求。 |

生成层的职责是：

1. 把 Top-K chunks 组装为 Context。
2. 将引用位置和来源约束写入 Prompt。
3. 调用 OpenAI 兼容异步客户端或后续替换的模型服务。
4. 通过 SSE 流式返回内容。
5. 在证据不足时明确拒答或引导用户补充事实。

概念流程：

```text
retrieved_chunks + citations
  -> Context Engine
  -> Prompt Template
  -> LLMService async / streaming call
  -> answer + citations + trace_id
```

法律场景里，LLM 不应绕过检索结果自行扩展事实。对于没有证据支持的问题，Prompt 模板应优先要求模型说明“当前资料不足”，而不是生成看似完整的法律结论。

## 第一阶段交付重点

1. 模型调用统一化。
2. 流式输出可运行。
3. 可接入检索与 Prompt 模板。
4. 能消费 `citations` 和 `rag_block`，并在回答中保持来源约束。

## 当前不足

- 模型路由策略仍较轻量
- 不同法律子任务的模型配置还没有完全制度化
- 输出质量高度依赖 Prompt 治理和检索质量
- 无依据拒答、引用覆盖率和流式过程审计还需要进一步制度化

## 后续演进方向

- 分离生成模型与工具模型
- 增强 structured output 的稳定性
- 为不同法律任务配置更明确的模型策略
- 将 `trace_id`、检索命中、最终回答和引用快照统一写入审计记录
- 为不同法律任务维护独立 Prompt 模板和评测样例
