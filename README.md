# ResumeLoom

用事实档案串起求职准备的纯 Skill 套件。以中文简历为主，也可按用户要求输出其他语言。

`dig → tailor → humanize → audit → apply`

第一版只有 Markdown 指令和 YAML/Markdown 模板。不绑定模型提供商，没有 Web 应用、运行时代码、算法评分器或浏览器驱动。执行效果依赖宿主模型及其文件、网页工具。

| Skill | 输入 | 输出 |
|---|---|---|
| [resume-dig](skills/resume-dig/SKILL.md) | 自述、旧简历、访谈 | 带来源和确认状态的事实档案 |
| [resume-tailor](skills/resume-tailor/SKILL.md) | 事实档案、JD | 定向草稿、JD 要求与事实映射 |
| [resume-humanize](skills/resume-humanize/SKILL.md) | 草稿、事实档案、映射 | 表达修订稿、事实追溯侧录 |
| [resume-audit](skills/resume-audit/SKILL.md) | 修订稿、事实档案、JD、侧录 | 事实门禁、ATS 与表达两轨报告 |
| [resume-apply](skills/resume-apply/SKILL.md) | 用户确认的定稿、有效报告、事实档案、岗位链接 | 填写清单或已填待用户检查的页面、追踪记录 |
| [job-pipeline](skills/job-pipeline/SKILL.md) | 上述材料或已有运行目录 | 顺序执行、退回整改、可恢复的运行记录 |

## 使用

把 `skills/` 下所需目录完整复制到宿主支持的 Skill 目录。原子 Skill 的模板均位于自身目录内，可单独安装；`job-pipeline` 必须和五个原子 Skill 安装在同一父目录。安装位置和命令语法以宿主当前文档为准，本仓库不会自动安装或更改全局配置。

在支持 `$skill-name` 的宿主中可直接引用名字；其他宿主可明确要求读取对应 `SKILL.md` 并执行。以下示例使用自然语言，不假定斜杠命令已注册：

> 使用 job-pipeline。原始经历在我提供的材料中，目标 JD 在 jd.md。把运行文件保存到 runs/candidate-a/backend-001/。先完成准备和审查，投递时只填写，不提交。

> 使用 resume-dig，把我的旧简历整理成 facts.yaml。暂时不做定向改写。

> 使用 resume-audit，审查 resume-humanized.md、humanized.meta.yaml、facts.yaml、jd.md 和 requirements.yaml。没有实际 PDF 时，明确记录文件解析未验证。

尚无文件读写工具时，Skill 返回可保存的文本，不会声称已创建文件。尚无浏览器时，apply 输出离线字段清单，不会声称已填写网页。目标岗位页面必须由用户提供或由可用工具实际读取，不能推测表单字段。

## 数据与边界

- `facts.yaml` 是事实依据；每个事实有来源、确认状态和稳定 ID。简历只使用已确认事实，用户自述不等于独立认证。
- 简历正文不嵌入内部事实 ID；单独的 `.meta.yaml` 绑定原句、事实 ID 与版本，供审查使用。
- humanize 保留技术术语与职责范围；audit 的事实门禁、ATS 轨、表达轨分别报告，不能相互抵消。
- ATS 轨是可解释的规则审查。没有真实文件及提取工具就不能声称解析通过；表达诊断不推断作者是否使用 AI。
- apply 不点击提交、不代签声明、不绕过登录或验证码。上传附件可能立即向平台传送资料，应在执行前取得对该平台和该文件的授权。用户自己提交后，可依据用户确认或回执更新记录。
- audit 保存实际输入快照，apply 比较当前文件与快照；即使忘记改版本号，也会使变更后的旧报告失效。两轮整改后仍阻塞就输出问题，等待用户处理。
- 真实产物放在被 Git 忽略的 `runs/` 中；忽略规则不是加密。`examples/` 只存虚构数据。

## 示例与验证

从 [虚构端到端样例](examples/backend-demo/README.md) 开始阅读。它展示正常链路、一个被排除的旧简历数字、一个缺失的 JD 技能，以及无浏览器时的申请准备记录。

[实施计划](docs/implementation-plan.md)、[行为场景](docs/validation-scenarios.md) 和 [实际验证记录](docs/validation-results.md) 说明交付范围与验证边界。

本版本按对话中的职责划分独立编写，没有复制或打包第三方 Skill、代码或规则集，也不声称实现某个开源项目的完整算法。候选开源项目的能力与许可证尚未核验。

## 开源协议

本仓库按 [MIT License](LICENSE) 发布，Copyright (c) 2026 Ikaros-521。

JobScope 岗位情报工作台是另一个仓库，见 [Ikaros-521/JobScope](https://github.com/Ikaros-521/JobScope)，那边采用 AGPLv3 与商业授权的双重许可，条款以其 `LICENSE` 为准。
