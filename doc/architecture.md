# 系统架构

## 目标结构

HammuraAI 当前的设计方向是一个分层式法律 AI 后端，不把所有能力塞进单个模块，而是拆为六层：

1. 接入层
2. 业务层
3. 检索层
4. 生成层
5. 数据层
6. 文档与规则层

## 核心技术路线

第一阶段的 RAG 底座采用 **PostgreSQL + pgvector 一体化架构**。也就是说，业务数据、文档元数据、文本块和向量索引优先放在同一个 PostgreSQL 体系内治理，而不是一开始就拆成多个独立存储系统。

这条路线的价值在于：

| 设计原则 | 说明 |
| --- | --- |
| 高一致性 | 用户权限、文档元数据和向量片段处在同一事务边界内，减少分布式双写带来的脏数据风险。 |
| 混合检索 | 通过 SQL 同时完成租户过滤、时间过滤、全文检索和向量相似度计算。 |
| 工程规范 | 延续当前 `AsyncSession`、SQLAlchemy ORM、Alembic 迁移体系，不在运行时自动建表。 |
| 可退出 | 当数据量或性能超过 PostgreSQL 单库能力时，可将 `document_chunks` 迁出到 Milvus 等专业向量库，业务主表仍保持稳定。 |

当前代码已经具备 PostgreSQL、`asyncpg`、SQLAlchemy 异步引擎和 Alembic 骨架。下一步是把已有的 `ARRAY(Float)` 向量字段和外部索引文件逐步收敛为 pgvector 管理的知识库切片模型。

## 分层说明

### 接入层

接入层由 FastAPI 提供 HTTP 接口，对外暴露：

- 认证与登录
- 聊天与会话
- 个人中心
- 文档与知识输入入口

主入口位于：

- `backend_app/app/main.py`
- `backend_app/app/api/v1/endpoints/*.py`

### 业务层

业务层负责把法律场景的结构化信息、会话状态、计费和权限聚合起来。

当前重点模块：

- `chat_runtime.py`
- `workflow.py`
- `entitlements.py`
- `billing.py`

### 检索层

检索层由三部分组成：

- Embedding 计算
- 法规索引
- 案例索引

它的职责不是直接回答用户，而是产出一组可引用、可排序的候选上下文。

目标形态下，检索层会围绕 `documents` 与 `document_chunks` 展开：

1. 离线入库时把文档解析为 chunks，并写入原文、元数据和 embedding。
2. 在线查询时把用户问题向量化，再用 pgvector 做相似度检索。
3. 在同一条 SQL 中挂载用户、租户、文档类型、时间范围等过滤条件。
4. 将 Top-K 结果交给重排与生成层，形成带引用的上下文。

### 生成层

生成层由 `llm.py` 与 `prompts.py` 驱动，负责：

- 选择模型
- 调用模型
- 组织消息
- 生成摘要、回答、追问建议

### 数据层

数据层以 PostgreSQL 为主，Redis 为运行时辅助：

- PostgreSQL: 用户、会话、消息、订阅、钱包、用量
- pgvector: 法律资料 chunks、向量字段、HNSW 向量索引
- Redis: Session、事件流、运行时锁

### 文档与规则层

文档与规则层承担两个职责：

- 维护技术设计与阶段边界
- 维护 Prompt 与法律输出规范

## 第一阶段重点链路

```text
用户请求
  -> Chat API
  -> Workflow 结构化分析
  -> 查询向量化
  -> PG + pgvector 混合检索
  -> 轻量融合 / 重排
  -> Context Engine 拼装上下文
  -> LLM 生成回答
  -> SSE / 消息持久化
  -> 返回 citations 与 rag_block
```

## 第一阶段架构判断

当前架构已经具备一个不错的早期形态：

- 数据模型足够支撑多用户和多会话
- 检索链路已有骨架
- 引用结构已有定义
- Prompt 与上下文引擎已经开始分层

后续最值得继续加强的方向：

- 检索召回质量
- 重排模块独立化
- Prompt 资产治理
- 审计与权限边界
