# 文件契约与失效传播

所有相对产物路径以运行目录为基准。Skill 内的链接以 SKILL.md 目录为基准。`schema_version: 1` 是格式版本；`revision` / `version` 是内容版本，二者不能混用。

## 产物

| 文件 | 写入者 | 下游读取 |
|---|---|---|
| facts.yaml | dig | 全部阶段；下游只读 |
| jd.md | tailor 保存原文快照 | tailor、audit、apply |
| requirements.yaml | tailor；独立 audit 可补建 | humanize、audit |
| resume-draft.md + draft.meta.yaml | tailor | humanize |
| resume-humanized.md + humanized.meta.yaml | humanize | audit、apply |
| output/html/*.html + output/pdf/*.pdf + export-manifest.json（按需） | 宿主导出步骤 | audit、apply、用户修改排版 |
| audit.yaml + audit-inputs/报告版本/ | audit | apply、恢复检查 |
| application.yaml | apply | 用户检查及提交记录 |
| run.yaml | job-pipeline | 恢复与依赖检查 |

原子 Skill 的 assets 给出各自完整模板。未知字段用 null，集合用空列表。各阶段必须填写实际输入绑定，不能直接照抄模板中的 v1。facts.status 为 partial 不阻止使用其 confirmed 子集。

正文面向招聘者，不包含内部 ID。侧录 claims 包含稳定 `C-xxx`、正文原句、位置、`fact_ids`，覆盖姓名、技能、时间和经历等全部事实。事实档案 `F-xxx` 保留来源 `S-xxx`，状态只有 confirmed / unconfirmed / disputed / inferred。删除或更正事实不能回收 ID。

## 版本与快照

内容改动必须更新版本。阶段重跑前将将被覆盖的旧产物保留在 `history/<阶段>/<旧版本>/`，不要删除用户材料。快照目录不能覆盖。不同岗位各用独立运行目录。

audit 将实际审查的每项输入保存到 `audit-inputs/<报告版本>/` 并记录 live/snapshot 对照。恢复时先读当前文件，再与报告快照逐项比较。对 Markdown/YAML 比完整文本，对二进制附件比较完整内容或宿主实际计算的摘要；不要捏造哈希。没有文件工具时不能声称快照已保存或相等。

快照不是防篡改签名；用户若同时修改快照，则需要重新审查。它用于发现日常手改内容却未更新版本的情况，不提供密码学完整性保证。

正式附件导出时，`export-manifest.json` 记录正文版本与实际计算的摘要、HTML/PDF 路径和版本、导出工具、实际页数、文字提取与逐页视觉检查结果；未执行的检查明确标为未验证。可在 `run.artifacts.export` 引用该清单，并在 history 记录导出及附件复查。正文只因内容变化而升版，样式变化更新 HTML/PDF 及导出版本。audit 保存当前实际附件快照；apply 比较的附件须与审查文件一致。历史文件先归档，真实个人资料留在本地运行目录，不写进公开模板或代码。

## 失效规则

| 变化 | 失效范围与恢复入口 |
|---|---|
| facts | requirements、草稿、修订稿、audit、apply；从 tailor 重跑，事实未确认先 dig |
| JD | requirements 及以后；从 tailor 重跑 |
| requirements 或 draft | humanize 及以后；先确认 tailor 侧录一致 |
| humanized 正文或侧录 | audit、apply；内容与侧录不一致先 tailor/humanize 修复再 audit |
| 附件新建或变化 | 附件相关审查与上传记录；重跑 audit |
| audit | apply 重新核对；旧审查的放行级别不可继承 |
| 表单字段/页面变化 | apply 重新读取核对，不能沿用旧已填状态 |

即使逻辑版本不变，内容差异仍触发失效。旧定稿确认不适用于修改后的正文或附件。已经提交的历史记录保留为历史，不因新版本改写成“未提交”；新申请/新版本另起记录。

## 失败恢复

阶段状态：pending / completed / stale / waiting_user / blocked。整体状态：in_progress / waiting_user / blocked / completed。

工具不可用时保留实际已完成阶段，下一步写成获取缺失材料/交还用户操作。审查阻塞时记录 route（dig、tailor、humanize、repair_export），不将整条链重置从头。整改计数达到 2 后保留现场并先报告阻塞；本次发现新 JD、手改文件或“直接投递”的催促都不自动重置计数。只有在向用户报告该阻塞后，用户明确要求开启新整改周期，才归档上一周期计数与原因、将 repair_rounds 置 0 并恢复。补充材料本身只更新待处理输入，不隐含允许继续循环。
