# 虚构后端岗位样例

从 [input.md](input.md) 与 [jd.md](jd.md) 开始。人物、公司、岗位和授权记录都是虚构的。`example.com` 不是投递目的地。

| 阶段 | 产物 | 展示的决策 |
|---|---|---|
| dig | [facts.yaml](facts.yaml) | F-006 保留旧数字和用户更正，状态为 disputed |
| tailor | [requirements.yaml](requirements.yaml)、[草稿](resume-draft.md)、[侧录](draft.meta.yaml) | Kubernetes 标为 gap；慢 SQL 排查只部分支持优化要求 |
| humanize | [修订稿](resume-humanized.md)、[侧录](humanized.meta.yaml) | 删除冗余名词，保留“参与”和“负责”的边界 |
| audit | [audit.yaml](audit.yaml) | 事实通过、表达通过；ATS 提示缺口，附件解析未验证 |
| apply | [application.yaml](application.yaml) | 离线准备；薪资、声明和附件待处理，未填网页 |
| 编排 | [run.yaml](run.yaml) | 绑定当前版本并停止在用户可检查的准备结果 |

样例文件是作者构造的参考输出，不是从真实招聘网站执行得到的记录。行为验证另外见 [验证记录](../../docs/validation-results.md)。

修改任意事实、JD 或简历正文后，应更新版本并重跑依赖阶段。若只获得实际 PDF，也要追加该附件的审查，不能把本例的文本审查结论复制过去。
