# JobScope 已确认设计稿实施计划

用户已确认设计图并要求开始开发。以 `docs/proposals/2026-11-07-jobscope-unslop-design.md` 和对话中的设计图为依据。

## 交付范围

重做 `jobscope/frontend/src/styles.css`，调整 `App.tsx` 的文案与展示组件，保留 API、业务行为和默认真实空库。更新现有浏览器验收脚本中的页面标题，增加改版的阅读和键盘检查。后端不变更。

## 顺序

1. [x] 间距：240px导航、64px顶栏、32px页面边距、4px节奏；统一控件。
2. [x] 颜色：中性灰白表面、三档文字、墨绿操作强调；语义状态有文字。
3. [x] 图标：保留当前单套线性图标，统一尺寸，移除装饰标记。
4. [x] 文案：页面名替代口号，缩短标签，保留必要的数据限制说明。
5. [x] 字体：可用本地无衬线字体，14–15px正文、12–13px元信息、数字等宽。
6. [x] 布局：岗位主从结构、分析映射表、证据行、资料源与浏览器表单一致。
7. [x] 细节：空分数直接说明、硬条件警示、可展开出处、模态框键盘焦点。
8. [x] 视觉：数据与出处承担重点，空库给出操作入口，不加入虚构数据。
9. [x] 动效：只保留控件反馈、加载与短暂淡入，尊重减少动态效果设置。
10. [x] 验收：前端构建、原有端到端UI流程、桌面与390px窄屏、字号与焦点检查；最多三轮页面截图。

## 运行与验收

- `npm --prefix jobscope/frontend run build`。
- 用独立 `JOBSCOPE_DATA_DIR=jobscope/output/unslop-test-data` 启动8766端口，运行 `python jobscope/scripts/smoke_ui.py`；虚构验收材料不写入默认数据库。
- 用Chrome对空库与有数据的分析页面、公司库、简历和采集页检查布局、错误状态与键盘焦点。
- 将结果记录到 `docs/validation/jobscope-unslop.md`，保留改版截图在 `jobscope/output/unslop/`。

本设计流程使用 unslop，作者 Damir Sokolovsky（@heydamir），CC BY 4.0，来源 https://github.com/heydamir/unslop 。

实施完成。构建、UI回归、390px分析表格、四页字号/溢出、弹窗键盘与采集错误恢复均已验证。详见 `docs/validation/jobscope-unslop.md`。
