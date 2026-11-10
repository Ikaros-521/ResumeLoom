# Research Agent Implementation Plan

**Goal:** 交付可演示、可评测、能解释边界的岗位研究Agent。

**Architecture:** 索引与结构化事实层、受范围控制的检索/统计工具、有界编排、带出处回答和可查看执行记录。保持FastAPI/SQLite/React，配置后才启用真实向量与模型。

**Tech Stack:** Python、SQLite FTS5、httpx、Pydantic、React/TypeScript、pytest及Playwright。

本会话直接执行，用户已要求升级；不新增审批、不提交未请求的Git提交。

- [x] 在`backend/tests/test_research.py`先覆盖工资别名、版本投影、范围、增量、无命中、薪资统计、追问、向量融合/缓存/失败和执行记录。运行`python -m pytest .../test_research.py -q`观察缺失能力。
- [x] 创建`research/index.py`、`query.py`、`providers.py`、`retrieval.py`、`tools.py`、`agent.py`及类型合同，按模块运行对应失败测试至通过。
- [x] 在`main.py`接入范围会话、研究运行接口、索引状态/重建接口；保留旧引用校验与服务端历史规则。`chat.py`支持已检索证据和结构化工具上下文。
- [x] 前端`ResearchTrace.tsx`、`api.ts`、`ScopedChat.tsx`与岗位问答显示结构化统计及可折叠执行事件；范围不变。
- [x] 创建版本化`evals/retrieval.json`与`evals/run.py`，运行关键词基线和BM25对比，输出机器可读报告、错误样本及延迟；未配置dense时注明未测。
- [x] 独立测试库完成83项既有回归与新测试，生产构建、UI脚本、真实库索引和只读工资问题验证，更新架构/面试讲解/验证文档。
