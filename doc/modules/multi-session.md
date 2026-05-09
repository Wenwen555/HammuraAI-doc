# 多用户会话功能

## 模块目标

多用户会话能力是 HammuraAI 的运行时核心。法律咨询不是一次性问答，而是连续补充事实、追问证据、逐步成形的过程。

## 当前实现

核心文件：

- `backend_app/app/models/database.py`
- `backend_app/app/api/v1/endpoints/chat.py`
- `backend_app/app/services/chat_runtime.py`
- `backend_app/app/services/case_memory.py`

## 当前设计

### 会话实体

- `ChatSession` 表示一个用户会话容器
- `ChatMessage` 表示会话内的消息记录
- `case_memory` 存储案件结构化记忆

### 运行时流

`ChatRuntime` 负责：

- 新建 assistant 消息占位
- 启动 turn
- 写入事件流
- 推送 SSE
- 在生成过程中增量持久化内容

### 多用户边界

当前已经具备：

- 用户与会话绑定
- 会话与消息绑定
- 通过 session cookie 识别当前用户
- 文档读取时做用户归属判断

## 第一阶段交付重点

1. 一个用户可以稳定拥有多个会话。
2. 每轮消息可以持久化。
3. 流式输出不会破坏消息完整性。

## 当前不足

- 会话级案件权限还不够细
- 断线恢复和多 worker 行为仍需持续验证
- 法律咨询上下文压缩策略还在演进

## 后续演进方向

- 引入更强的会话恢复机制
- 会话与案件、客户、知识库建立更清晰的关联
- 进一步收敛事件日志与审计日志
