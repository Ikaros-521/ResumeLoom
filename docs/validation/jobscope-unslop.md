# JobScope 前端改版验收

日期：2026-11-07。依据：用户确认的「求职调查档案台」设计图及设计提案。

## 实际变更

- 重写 `jobscope/frontend/src/styles.css`：240px 中性侧栏、64px 顶栏、32px 主内容边距、灰白表面、墨绿操作强调、本地无衬线字体。正文14px，元信息12–13px；去除外部字体请求。
- 调整 `App.tsx`：页面标题改为任务名称；空状态展示岗位要求、公司证据和个人适配三个操作方向；移除轨道插画及重复口号。
- 适配分析采用「岗位要求 / 简历依据 / 判断」表格，显示证据覆盖、未知项、硬性条件、薪酬口径和公司出处；无有效评分时直接显示「暂不生成总分」。
- 简历、公司库、资料问答、数据源、浏览器采集表单和任务记录使用同一套字号、间距及控件样式。
- 弹窗支持初始焦点、Tab/Shift+Tab 焦点循环、Escape关闭、关闭后恢复触发按钮焦点和背景滚动锁定。表格显式声明列标题作用域。
- 后端接口和采集实现未在本次改版中调整。

## 验证结果

| 检查 | 结果 |
| --- | --- |
| `npm --prefix jobscope/frontend run build` | 通过，TypeScript检查与Vite生产构建成功 |
| `python jobscope/scripts/smoke_ui.py` | 通过，在独立8766测试服务执行 |
| 导入岗位→保存简历→适配分析→岗位问答 | 通过 |
| 补充公司证据→公司库精确筛选→公司问答 | 通过 |
| 规则分析无总分与三列表格 | 通过 |
| 弹窗Tab/Shift+Tab、Escape、触发点焦点恢复 | 通过 |
| 四页390px窄屏横向溢出检查 | 均无溢出 |
| 四页390px可见文本字号 | 无低于12px的可见文本 |
| 390px分析表格 | 无页面横向溢出 |
| 四页1440px桌面横向溢出检查 | 均无溢出 |
| 采集503错误提示与按钮恢复 | 通过，使用浏览器请求拦截模拟 |
| 浏览器脚本异常 | 0 |
| 默认服务 `/api/health`、首页 | `ok`、HTTP 200 |
| 默认服务岗位数 | 0，验收材料保存在独立测试库 |

使用本机Chrome和Python Playwright验证。桌面、手机分析页、简历、公司库及采集页已查看截图。前端改版未重跑此前后端测试，不把此前的测试结果当作本轮新结果。

验收数据目录：`jobscope/output/unslop-test-data`。所有「虚构验收」材料仅用于该测试目录。日常服务8765使用默认 `.data`，当前没有配置模型；问答验收覆盖资料检索模式，未验证外部模型综合回答。

本轮未使用真实账号登录BOSS，也未对外部采集接口做成功承诺。采集页仍显示后端返回的真实状态，BOSS接口当前为「需要验证」。

## 截图

- `jobscope/output/unslop/before-desktop.png`：改版前首页。
- `jobscope/output/unslop/after-desktop.png`：改版后默认空库首页。
- `jobscope/output/workbench-analysis.png`：有虚构验收材料的分析及问答页。
- `jobscope/output/unslop/mobile-analysis.png`：390px适配分析。
- `jobscope/output/unslop/after-我的简历.png`：简历页。
- `jobscope/output/unslop/after-公司资料库.png`：公司资料库。
- `jobscope/output/sources.png`：数据与采集页。

设计流程使用 [unslop](https://github.com/heydamir/unslop)，作者 Damir Sokolovsky（[@heydamir](https://x.com/heydamir)），CC BY 4.0。归属信息保留在样式文件与设计文档中。
