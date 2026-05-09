# HammuraAI 初步设计

<div class="hero-panel">
  <p class="hero-kicker">Design Notes · Phase One</p>
  <h2>一套为法律服务场景设计的智能问答与案件协同底座</h2>
  <p class="hero-lead">
    当前文档用于维护 HammuraAI 的早期技术设计。第一阶段聚焦于搭建可演示、可评测、可扩展的基础结构，
    让数据库、检索、重排、生成、会话与 Prompt 规则先形成稳定骨架。
  </p>
  <div class="hero-actions">
    <a class="md-button md-button--primary" href="phase-one-deliverables/">查看第一阶段交付</a>
    <a class="md-button" href="architecture/">查看系统架构</a>
  </div>
</div>

<div class="annotation-showcase">
  <div class="annotation-showcase__visual" markdown>

<div class="code-window">
  <div class="code-window__bar">
    <span></span><span></span><span></span>
    <strong>answer-api.yml</strong>
  </div>

``` yaml
answer_api:
  trace_id: required # (1)
  citations: required # (2)
  refusal_on_no_evidence: true # (3)
retrieval:
  statutes: enabled
  cases: enabled
session:
  streaming: sse
  persistence: enabled # (4)
```
</div>

1. 每次回答都要返回 `trace_id`，方便定位一次问答经过与检索记录。
2. 只要系统给出可回答结果，就必须附带最小引用来源。
3. 没有依据时明确拒答，而不是生成看起来很像真的法律结论。
4. 会话内容在流式生成过程中持续落库，保证多轮咨询可追踪。

  </div>
  <div class="annotation-showcase__copy" markdown>

<p class="hero-kicker">Showcase · Code Annotations</p>

## Code annotations

你想实现的这块效果，核心就是 Material for MkDocs 自带的 **code annotations**。  
左侧代码块里的 `# (1)`、`# (2)` 这类标记，会和下方对应编号的说明自动关联，形成一种优雅的文档演示方式。

对于 HammuraAI，这种表现形式很适合拿来展示：

- Answer API 的输出契约
- 检索链路的关键约束
- 法律回答为什么必须带引用
- 会话与持久化如何协同工作

它不是单独的前端组件，而是 **Markdown 语法 + Material 原生能力 + 两栏布局样式** 的组合。

[查看第一阶段交付](phase-one-deliverables.md){ .md-button .md-button--primary }
[查看系统架构](architecture.md){ .md-button }

  </div>
</div>

## 当前阶段

第一阶段不是直接做完整法律产品，而是先证明主链路成立：

1. `后端服务 + 检索链路 + LLM 生成` 可以跑通。
2. 回答能带最小引用与来源结构。
3. 系统结构足够清晰，能继续接入 FastGPT、权限治理和业务工作台。

## Design showcase

<div class="showcase-grid">
  <a class="showcase-card" href="modules/database/">
    <span class="showcase-card__eyebrow">Foundation</span>
    <h3>数据库搭建</h3>
    <p>定义 PostgreSQL、ORM 模型、迁移入口和多用户业务数据的基础结构。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/retrieval/">
    <span class="showcase-card__eyebrow">Retrieval</span>
    <h3>检索工具</h3>
    <p>组织法规索引、案例索引和 Embedding 接入，让法律问题先有依据再有回答。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/rerank/">
    <span class="showcase-card__eyebrow">Reasoning Input</span>
    <h3>重排功能</h3>
    <p>对候选法规和案例做轻量排序修正，为生成层提供更可信的上下文。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/llm/">
    <span class="showcase-card__eyebrow">Generation</span>
    <h3>LLM 调用模块</h3>
    <p>统一模型入口、流式输出和结构化摘要，让推理链路稳定可维护。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/profile/">
    <span class="showcase-card__eyebrow">User Surface</span>
    <h3>用户档案管理</h3>
    <p>承接登录、订阅、钱包和个人中心，为法律服务场景提供用户侧骨架。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/multi-session/">
    <span class="showcase-card__eyebrow">Runtime</span>
    <h3>多用户会话功能</h3>
    <p>把消息持久化、流式事件和案件记忆组织成可连续追踪的咨询过程。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card" href="modules/prompts/">
    <span class="showcase-card__eyebrow">Guidance</span>
    <h3>Prompt 模板管理</h3>
    <p>维护法律咨询、追问、摘要和上下文注入模板，约束系统输出边界。</p>
    <span class="showcase-card__meta">查看模块设计</span>
  </a>
  <a class="showcase-card showcase-card--accent" href="phase-one-deliverables/">
    <span class="showcase-card__eyebrow">Phase One</span>
    <h3>第一阶段交付</h3>
    <p>查看当前立项阶段的边界、交付口径和验收判断，作为后续推进基线。</p>
    <span class="showcase-card__meta">查看交付页面</span>
  </a>
</div>

## 进度视图

<div class="grid cards" markdown>

- :material-database-outline:{ .lg .middle } **数据库与基础模型**

    ---

    **状态**：进行中  
    已具备 SQLAlchemy 模型、AsyncSession、Alembic 迁移骨架。

- :material-magnify-scan:{ .lg .middle } **检索工具**

    ---

    **状态**：进行中  
    已具备案例索引、法规索引、Embedding 微服务接入能力。

- :material-tune-variant:{ .lg .middle } **重排与引用**

    ---

    **状态**：进行中  
    已具备轻量重排、`citations` 与 `rag_block` 输出结构。

- :material-robot-outline:{ .lg .middle } **LLM 调用**

    ---

    **状态**：进行中  
    已具备统一 `LLMService`、模型选择与流式输出能力。

- :material-account-badge-outline:{ .lg .middle } **用户档案管理**

    ---

    **状态**：已落地骨架  
    已有认证、订阅、钱包、个人中心聚合接口。

- :material-chat-processing-outline:{ .lg .middle } **多用户会话**

    ---

    **状态**：已落地骨架  
    已有 `ChatSession`、`ChatMessage` 与 SSE 流式会话运行时。

- :material-text-box-edit-outline:{ .lg .middle } **Prompt 模板管理**

    ---

    **状态**：进行中  
    已有集中式 prompts 模块，后续需继续拆分和治理版本。

</div>

## 第一阶段交付产物

<div class="showcase-grid showcase-grid--compact">
  <a class="showcase-card" href="phase-one-deliverables/">
    <span class="showcase-card__eyebrow">Demo</span>
    <h3>法律问答后端基础服务</h3>
    <p>可运行、可启动、可承载第一阶段主链路。</p>
  </a>
  <a class="showcase-card" href="modules/retrieval/">
    <span class="showcase-card__eyebrow">Retrieval</span>
    <h3>检索增强回答路径</h3>
    <p>将法规与案例检索结果稳定送入生成层。</p>
  </a>
  <a class="showcase-card" href="modules/retrieval/">
    <span class="showcase-card__eyebrow">Knowledge</span>
    <h3>首版法律资料索引</h3>
    <p>支撑第一轮法规与案例召回验证。</p>
  </a>
  <a class="showcase-card" href="modules/rerank/">
    <span class="showcase-card__eyebrow">Trust</span>
    <h3>最小引用返回结构</h3>
    <p>为结果解释、前端呈现和后续审计做准备。</p>
  </a>
  <a class="showcase-card" href="phase-one-deliverables/">
    <span class="showcase-card__eyebrow">Evaluation</span>
    <h3>测试题与评测标准</h3>
    <p>作为第一阶段统一验收口径。</p>
  </a>
</div>

### 对内必须完成的工程资产

| 交付物 | 内容 |
| --- | --- |
| 数据层骨架 | PostgreSQL 连接、ORM 模型、Alembic 迁移入口 |
| 向量数据骨架 | pgvector 扩展启用方案、`document_chunks` 目标模型、HNSW 索引设计 |
| 检索层骨架 | 法规索引、案例索引、Embedding 服务对接、PG 混合检索验证 |
| 生成层骨架 | LLM 统一调用、流式输出、基础工作流摘要、引用约束消费 |
| 业务层骨架 | 用户、会话、档案、订阅、积分相关模型与接口 |
| Prompt 资产 | 面向法律咨询、案情分析、追问引导的模板库 |
| 文档资产 | 本设计文档与第一阶段交付口径 |

## 设计原则

### 法律场景优先

- 不编造来源
- 不越过事实边界
- 不把演示能力误当成生产能力
- 所有复杂能力优先服务于“可解释”和“可追溯”

### 工程策略

- 先搭底座，再接业务前台
- 先保证结构清晰，再做深度优化
- 尽量让检索、重排、生成、会话彼此解耦
- 文档与实现同步演进

## 文档入口

<div class="grid cards" markdown>

- [第一阶段交付](phase-one-deliverables.md)
- [系统架构](architecture.md)
- [数据库搭建](modules/database.md)
- [检索工具](modules/retrieval.md)
- [重排功能](modules/rerank.md)
- [LLM 调用模块](modules/llm.md)
- [用户档案管理](modules/profile.md)
- [多用户会话功能](modules/multi-session.md)
- [Prompt 模板管理](modules/prompts.md)

</div>
