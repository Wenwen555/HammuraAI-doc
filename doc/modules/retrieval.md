# 检索工具

## 模块目标

检索工具负责把“用户的法律问题”变成“可供模型引用的法规、案例和材料候选集”。

在 HammuraAI 第一阶段，检索工具的意义比生成更基础，因为法律场景首先要保证来源与依据。

## 当前实现

核心文件：

- `backend_app/app/services/embedding.py`
- `backend_app/app/services/provisions_db.py`
- `backend_app/app/services/case_db.py`
- `backend_app/app/services/workflow.py`

## 当前能力分解

### 1. Embedding 服务接入

- 通过 HTTP 微服务完成向量计算
- 支持单条与批量编码
- 支持相似度计算

### 2. 法规检索

- 从法规索引文件加载数据
- 按 query 计算语义相似度
- 对法条做基础打分与去重
- 输出法规 payload

### 3. 案例检索

- 支持 JSONL 与二进制索引加载
- 支持 claims / facts / other 向量分区
- 支持候选路径与状态检查
- 作为案例召回的主要入口

## PG + pgvector 检索目标

下一步检索工具应从“外部索引文件 + 内存召回”逐步演进为 **PostgreSQL + pgvector 的统一检索服务**。

目标不是立刻替换所有现有索引，而是先把第一阶段知识库问答的主路径放进数据库：

1. 文档元数据进入 `documents`。
2. 文本切片、原文片段、引用定位和 embedding 进入 `document_chunks`。
3. 查询时通过 SQL 同时完成用户过滤、资料类型过滤、时间过滤和向量相似度排序。
4. 返回统一候选结构，供重排、Prompt 和引用输出使用。

## 双轨数据流

### 离线入库链路

离线入库负责把法律资料变成可检索的 chunks。

```text
文件 / 法规 / 案例
  -> 文档解析
  -> 清洗与分块
  -> 批量 embedding
  -> documents / document_chunks 事务写入
  -> Alembic 管理索引与约束
```

关键要求：

- 分块策略需要保留法条、标题、页码、来源 URL 或案件编号等引用定位。
- Embedding 调用应支持批量与并发，但数据库写入应保持事务边界。
- 同一文档重新入库时，应能按 `doc_id` 或版本字段清理旧 chunks，避免重复召回。

### 在线检索与生成链路

在线检索负责把用户问题变成可引用上下文。

```text
用户问题
  -> 查询向量化
  -> metadata filter
  -> pgvector Top-K
  -> 关键词 / 全文召回补充
  -> 轻量融合排序
  -> citations + rag_block
```

概念查询示例：

```python
stmt = (
    select(DocumentChunk)
    .join(Document)
    .where(Document.user_id == current_user_id)
    .where(Document.doc_type.in_(["statute", "case", "template"]))
    .order_by(DocumentChunk.embedding.cosine_distance(query_vector))
    .limit(5)
)

chunks = await session.scalars(stmt)
```

实际实现时，需要根据所选 Embedding 模型确认向量维度，并在数据库侧建立 HNSW 索引，避免数据量变大后全表扫描。

## 混合检索策略

法律问答不能只依赖语义相似度。第一阶段建议保留三路信号：

| 信号 | 作用 |
| --- | --- |
| 向量检索 | 召回语义接近的法规、案例和文档片段。 |
| 关键词 / 全文检索 | 保住法条编号、专有名词、案号、机构名等精确匹配。 |
| 元数据过滤 | 处理用户、租户、资料类型、时间范围、地域、有效状态等边界。 |

融合方式可以先采用轻量规则：

1. 先用 metadata filter 限定范围。
2. 分别取向量 Top-K 和关键词 Top-K。
3. 对重复 chunk 合并分数。
4. 交给重排模块做最终排序与引用整理。

## 第一阶段交付重点

第一阶段并不要求检索系统做到“最强”，但必须做到：

1. 能稳定召回法规与案例。
2. 能返回基础分数与来源字段。
3. 能与 workflow 和 prompt 链路对接。
4. 能验证 PG + pgvector 路线下的过滤、召回和引用闭环。

## 当前不足

- 检索策略仍偏工程骨架，尚未形成统一召回协调器
- 关键词检索、全文检索和向量检索尚未完全统一
- 知识库分层与权限过滤尚未落到检索入口

## 第二步演进方向

- 建立统一的 `retrieve` 服务接口
- 接入 PostgreSQL 全文检索与 pgvector，形成混合检索基线
- 加入更清晰的 metadata filter 设计
- 为高频检索字段建立标量索引，为 embedding 建立 HNSW 索引
- 当单表向量数据超过 PostgreSQL 舒适区时，再评估 Milvus 等独立向量库迁移
