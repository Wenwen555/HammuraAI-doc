# 数据库搭建

## 模块目标

数据库模块是 HammuraAI 的系统底座，当前目标不是做复杂数据平台，而是先支撑法律问答与多用户会话的主链路。

## 当前实现

核心文件：

- `backend_app/app/core/config.py`
- `backend_app/app/core/database.py`
- `backend_app/app/models/database.py`

### 连接与访问方式

- 使用 PostgreSQL 作为主数据库
- 使用 SQLAlchemy AsyncEngine + AsyncSession
- 运行期只做连接检查，不自动建表
- 表结构迁移统一交给 Alembic

### 一体化 RAG 存储路线

第一阶段建议将 PostgreSQL 扩展为 **业务数据 + 向量数据的一体化存储底座**。具体做法是在 PostgreSQL 中启用 `pgvector`，让数据库同时承担：

- 用户、会话、订阅、钱包等业务数据管理
- 文档元数据与法律资料治理
- 文本块 chunk 的 embedding 向量存储
- 基于用户、租户、时间、文档类型的过滤检索

这样可以避免业务主库和外部向量库之间的双写不一致，也更贴合当前已经存在的异步 ORM 与 Alembic 迁移体系。

### 已有核心模型

| 模型 | 用途 |
| --- | --- |
| `User` | 用户主体与认证关联 |
| `ChatSession` | 多用户、多轮会话容器 |
| `ChatMessage` | 会话消息持久化 |
| `Document` | 用户上传文档 |
| `LegalCase` | 法律案例资料 |
| `WorkflowTask` | 工作流任务记录 |
| `Plan` / `Subscription` | 套餐与订阅 |
| `Usage` | 用量统计 |
| `Wallet` / `CreditLedger` | 积分钱包与流水 |

## 目标知识库模型

当前代码中 `LegalCase.embedding_vector` 仍是 `ARRAY(Float)`，法规与案例索引也有一部分通过外部文件加载。后续应逐步收敛为 pgvector 管理的文档切片模型。

建议模型拆为两层：

| 模型 | 作用 |
| --- | --- |
| `Document` | 存储文档标题、来源、用户/租户、文档类型、状态、时间等元数据。 |
| `DocumentChunk` | 存储文档分块内容、chunk 序号、embedding、引用定位和检索辅助字段。 |

概念映射示例：

```python
from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column
from pgvector.sqlalchemy import Vector

class Document(Base):
    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(255))
    user_id: Mapped[int] = mapped_column(index=True)
    doc_type: Mapped[str] = mapped_column(String(50))


class DocumentChunk(Base):
    __tablename__ = "document_chunks"

    id: Mapped[int] = mapped_column(primary_key=True)
    doc_id: Mapped[int] = mapped_column(
        ForeignKey("documents.id", ondelete="CASCADE")
    )
    content: Mapped[str] = mapped_column(Text)
    embedding: Mapped[list[float]] = mapped_column(Vector(1536))
```

`Vector(1536)` 的维度必须和实际 Embedding 模型保持一致。若后续切换模型，需通过迁移或新字段处理维度变化，不能在运行时隐式改变表结构。

## Alembic 迁移规范

本项目已经明确“不在运行时自动建表”。pgvector 路线也必须继续遵守这一点。

首次引入 pgvector 时，迁移脚本需要包含：

```python
def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")
```

随后通过 Alembic 创建 `document_chunks` 表、普通过滤索引和向量索引。向量索引建议优先使用 HNSW：

```python
op.execute(
    """
    CREATE INDEX IF NOT EXISTS ix_document_chunks_embedding_hnsw
    ON document_chunks
    USING hnsw (embedding vector_cosine_ops)
    """
)
```

同时保留必要的标量索引，例如：

- `document_chunks.doc_id`
- `documents.user_id`
- `documents.doc_type`
- `documents.created_at`
- `document_chunks.source_ref`

这些索引用于在向量检索前先缩小候选范围，避免所有查询都退化为大范围扫描。

## 第一阶段职责

第一阶段数据库模块负责：

1. 让主业务实体可以持久化。
2. 让会话和消息可以追踪。
3. 让知识库文档、文本块和向量字段有统一数据基础。
4. 让后续检索、计费和档案功能都处在可迁移的 schema 管理下。

## 技术判断

### 当前设计优点

- 模型划分已经覆盖主要业务面
- `AsyncSession` 适合 FastAPI 异步调用路径
- 迁移和运行时职责边界清楚
- PostgreSQL + pgvector 可复用现有数据库治理、备份和权限体系

### 当前风险

- 知识库文档与向量索引仍未完全收敛到统一数据模型
- 部分法律资料仍通过外部索引文件加载，不完全在数据库中治理
- 后续案件级权限如果深入，会需要更细粒度关联字段
- 当前依赖中尚未确认 `pgvector` Python 包和数据库扩展的安装闭环

## 下一步建议

- 为知识库与检索元数据补统一表结构：`documents`、`document_chunks`
- 引入 `pgvector` Python 包，并通过 Alembic 开启 PostgreSQL `vector` 扩展
- 将 `ARRAY(Float)` 类型的向量字段演进为 `Vector(dim)` 或迁移至独立 chunk 表
- 明确公共资料和案件资料的分层关系
- 为审计记录预留稳定 schema
