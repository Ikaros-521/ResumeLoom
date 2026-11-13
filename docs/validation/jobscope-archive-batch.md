# 岗位档案：筛选与批量编辑/删除（2026-11-10）

岗位档案此前只能按关键词+地点筛选（且复用采集表单的值），没有任何编辑和删除能力。本次加了信息源/薪资区间筛选、卡片多选、批量编辑（地点/薪资/标签）与批量删除（含按筛选一键全选）。

## 实现口径

- **薪资数值键**：`jobs.payload` 新增 `salary_monthly_min/max`（单位：元，由 `parse_salary` 从原文算出）和 `salary_parsed` 标记。`save_job` 每次写入都重算；`Repository` 启动时对缺键的老档案回填一次（有标记或薪资为空的不重复扫）。面议/年薪/时薪/外币解析不出每月数值，键为 None。
- **薪资筛选是重叠语义**：岗位月薪区间与给定区间有交集即命中（如 `3-5万` 会命中 20–35K 的筛选）；解析不出数值的岗位任何薪资筛选都不命中。前端输入单位 K/月，提交时 ×1000。
- **信息源筛选**：`payload.source` 精确匹配；`q`、`location` 口径不变，可叠加。
- **`GET /api/jobs/ids`**：同一套筛选返回全部命中 id（上限 20000），供"选筛选全部"。
- **`PATCH /api/jobs`**：body `{ids: 1..1000, location?, salary?, tags?}`，缺省字段不改；显式空串=清空，`tags: []`=清空标签。批量编辑字段只有地点/薪资/标签——批量改 JD、链接、公司名没有真实场景，且模型 `extra=forbid` 挡住。
- **编辑不重排 id、不写新证据**：`update_jobs` 原位改 payload（`save_job` 的 id 由 title/company/location 派生且每次落一条证据快照，批量编辑都不该触发）；改薪资时重算数值键。
- **删除级联**（`delete_jobs`，单事务）：删 `job_evidence` 链接、`reports`、仅被待删岗位引用的 evidence（多岗位共享的公司证据保留）、scope 指向这些岗位的 `conversations`，最后删 jobs 行。没有回收站，确认弹窗兜底。

## 验证（本机 8765 实例，1729 条真实档案）

```powershell
cd E:\Github_Pro\ResumeLoom\jobscope
.venv\Scripts\python.exe -m pytest backend/tests -q   # 464 passed（含新增 backend/tests/test_job_batch.py 9 项）
cd frontend
npm run build                                          # tsc -b + vite build 通过
```

- 老数据回填：重启实例后首条 `1-1.7万` → 10000/17000、`salary_parsed=true`；真实库筛选总数合理（zhaopin 1430，20–35K 1114，叠加 882）。
- API 全链路（自测导入岗，跑完即删）：`面议` 导入后数值键为 None → PATCH 改地点/薪资/标签 → id 不变、证据数不变、数值键变 15000/25000 → `q`+`salary_min` 能筛到 → batch-delete 返回 `{deleted:1}`，再查 404，档案总数恢复 1729。
- 浏览器 UI 实测：筛选栏（关键词/信息源/地点/薪资 K）筛出"岗位档案 882"，与 API 一致；勾选 1 条出现 批量编辑/删除/取消选择；批量编辑弹窗提示"将应用到已选的 1 份档案"且留空不改；删除确认弹窗写明连带删除证据、报告与对话；两弹窗取消均无副作用；清除筛选恢复 1729。

## 2026-11-10 补充：岗位研究页一屏工作台

用户反馈留白多、来回翻页滚动麻烦。桌面端（≥1200px 宽且 ≥700px 高）把岗位研究页改成固定高度工作台：顶栏 48px、页头一行、检索区单行、双栏面板占满剩余视口，岗位列表与详情面板各自内部滚动，页脚与次要说明行隐藏；页面本身不再整体滚动。小屏保持原布局。

- 卡片从 6 行压成 3 行等高（标题单行省略 / 薪资·地点·首个标签 / 公司·来源·时间，82px）；每页 20→30 条；选择栏并入面板标题行。
- 全局 `input { min-height: 44px }` 规则曾让筛选栏每行涨到 49px，筛选栏内控件已解除该限制并收紧内边距。
- 顺手修了一个老 bug：ChatWorkspace 挂载/消息更新时 `scrollIntoView({ block: 'end' })` 会连带滚动详情面板等祖先容器，选中岗位后面板被滚过标题区（稳定停在 scrollTop≈198）。现改为只滚 `.chat-thread` 自身，`selectJob` 时面板回顶。
- 验证（1440×900）：整页高度=视口，无页面滚动；列表区 464px 可见 5 张整卡；筛选、勾选、批量编辑/删除弹窗复测正常；选中岗位后面板停在顶部、标题可见；数据与采集、我的简历等普通视图布局正常。

## 2026-11-10 补充：单页 100 条 + 修地点下拉挤偏列表

- 单页条数 30 → 100（后端 `page_size` 上限本就是 100），1699 条档案 17 页。
- 修地点筛选框挤压：城市下拉宽 208px 从地点框右缘伸出面板（595 > 面板右缘 576），浏览器为让焦点元素可见把 `overflow: hidden` 的列表面板横向滚 ~40px，整块内容被推左裁掉。两处修复：筛选栏内下拉改右对齐（`right: 0`）完全收在面板内；列表面板曾改 `overflow: clip` 想杜绝焦点滚动横移，但 clip 让面板在栅格里失去高度约束、被 100 条内容撑到 8498px 且外层不可滚——上下都滚不动了，已回退为 `hidden`（右对齐已消除溢出，clip 属多余保险）。复测：面板回到 725px、列表内部滚动正常（scrollTop=300 生效）、输入“上海”触发下拉后卡片 x 坐标不变（312）、面板 `scrollLeft=0`、下拉右缘 563 < 576。

## 2026-11-10 补充：浏览器采集数量上限 1000

用户要求采集数量输入框支持到 1000。`BrowserCollectInput.limit` le=500 → 1000；智联（state 模式）`pages` 25 → 50（20×50=1000，仍按请求数量封顶、`isEndPage` 到尾页自动停、页间 0.5 秒）；前端输入框 max 与钳制 50 → 1000。测试 `test_zhaopin_page_cap_covers_five_hundred` 同步改为断言 1000。验证：pytest connectors+api 60 passed；模型层 limit=1000 合法、1001 被拒（端点实测 422）；UI 输入 1000 生效、5000 钳回 1000。

注意：boss（listen 模式单页 30）与 51job（fetch 模式单页 50）的既有单页封顶不变——扩这两家需要新写各自的翻页逻辑，本次未做；设置超过其上限的数量时仍按原行为只采到单页封顶值。

## 2026-11-10 补充：BOSS 滚动翻页与前程无忧接口翻页

用户要求 boss 与 51job 也具备大量采集能力。实现沿智联的扩展器模式，`collect()` 按 mode 分发到三个 `_extend_*`：

- **boss（listen 模式）**：`pages=34`（30×34≈1000）。首包收到后循环向下滚动（`window.scrollTo` 到文档底部），`tab.listen.wait` 收下一页 `joblist.json`，按 `encryptJobId` 去重合并进 `zpData.jobList`；滚动后 10 秒无响应、响应无岗位或无新岗位即停，保留已读内容不算失败。
- **51job（fetch 模式）**：`pages=20`（50×20=1000）。同一检索接口 `pageNum` 递增续读（`pageSize` 固定 50，不把 limit 塞进单次请求），按 `jobId` 去重合并进 `resultbody.job.items`；每页失败补试一次。
- **加固**：`_page` fetch 分支收紧验收——200 且可 parse 不再直接算成功（风控校验页也可能是 JSON），必须 `status='1'` 且带岗位列表，否则 2 秒后重试（最多 3 次）。首次实测 51job 因此返回过"0 条+翻页即停"，加固后同一参数成功。

验证：单测 67 passed（新增两源的分页合并/超时停/去重用例 + 封顶断言）；真实采集 boss Python/杭州 limit=60 → completed 60 条、翻 4 页；51job Python/杭州 limit=100 → completed 100 条、翻 2 页。档案里留有这批测试数据（boss 60、51job 100），不需要可批量筛选删除。

## 2026-11-10 补充：猎聘大量采集

猎聘是 HTTP 接口源，本身已按 `currentPage` 翻页，但有三处旧钳制：`connectors.collect` 入口 `min(limit, 50)`、页数上限 3 页、`CollectInput.limit` le=50。另外实测直连接口时 `pagination.hasNext` **恒为 false**（804 个结果、totalPage=10 时也是假）——旧停止逻辑翻完第 1 页就被它骗停。修复：三处钳制放到 1000（页数上限 25×40），停止逻辑改为"去重后无新岗位为主、totalPage 划界、缺 totalPage 才退回 hasNext"；前端"检索岗位"表单增加数量输入（原写死 20，上限 1000）。

验证：真实采集 产品经理/北京 —— limit=100 → 100 条；limit=1000 → 402 条（猎聘该查询 totalPage=10、totalCounts=804，平台把深分页封在约 400 条，这就是单次查询的真实上限），逐页零重叠、去重正常，耗时 404 秒。单测 481 passed（分页语义新用例 + 模型边界）。

注意：`/api/collect` 是同步请求，满额 400 条约 7 分钟，超过浏览器 fetch 的约 300 秒超时——UI 里填到接近猎聘上限时可能超时失败，命令行/API 调用不受影响；若经常需要满额采集，应把该入口也改成后台任务（浏览器采集已是后台模式）。

## 2026-11-10 补充：详情面板紧凑化与内容卡片化

用户反馈右侧详情样式错乱、内容过多。修复与重排：

- **错乱根因**：`detail-heading-bottom`（薪资 + 编辑档案 + 评估适配）用了 space-between，三元素被拉满整行、中间按钮悬空——改为薪资居左、按钮成组靠右。另发现 `.job-chat .chat-page` 固定 `100vh-240px`，在固定高度工作台面板里撑出 693px——工作台内压到定高 320px，对话线程内部滚动。
- **紧凑化**（工作台媒体查询内）：标题区 210→141px（padding 12/20、标题 18px）、jd-state 76→64、选项卡 49→42、正文区 padding 收紧。
- **内容卡片化**：岗位描述变为描边卡片，超过 600 字自动折叠为 220px 内滚 + "展开全文/收起描述"切换（实测 3680 字 JD：220→560→220）；公司情报取数 4→12 条，每页 4 条卡片分页（上一页/下一页 + 页码），切岗位时页码与展开态重置。

验证（1440×900）：详情面板总高从 1248 压到 ~810（视口 725 内小幅内滚）；薪资/按钮一行排布；`npm run build` 通过。

## 2026-11-10 补充：工作台布局优化与响应式导航

用户要求对页面设计布局整体优化（UI/UX Pro Max Skill）。保留绿色主题与全部业务功能，重排导航、检索与阅读动线：

- **侧边栏**：216px 深绿（#173e35）固定栏，导航分「研究工作区 / 工作区管理」两组，44px 触控目标，岗位研究带数量徽标；≤800px 改为顶部横条 + 汉堡按钮（aria-expanded/aria-controls），点导航后自动收起并把焦点交给主区。
- **检索与筛选分离**：采集区「发现新岗位」表单单行栅格（关键词/地点/数据源/数量），全部可见 label；本地档案筛选独立成行（关键词 + 筛选按钮），信息源/地点/月薪 K 收进原生 `<details>`「筛选条件」并带已设计数徽标；有筛选时标题行出现「清除筛选」，清除后焦点还原到搜索框（修焦点落 BODY 的评审问题）。
- **岗位列表加宽**：研究区栅格 `minmax(340px,.48fr) minmax(0,1fr)`，卡片标题两行 clamp、选中整卡浅绿底；选择栏移到筛选下方，空态区分「没有符合条件的岗位」并给清除入口；欢迎清单改 auto-fit 卡片。
- **移动端**：≤1040px 单栏 + 详情面板「返回岗位列表」按钮；选中岗位后的详情滚动从 selectJob 同步执行改为渲染完成后的 effect（scrollIntoView + focus，修加载态短页滚错位置的评审问题）；≤680px 输入 16px 防 iOS 缩放、44px 目标。市场卡片与对话标题补 min-width:0 修内部裁切。
- **紧凑桌面**：≥1200×700 工作台基础上，≤820px 高压缩顶栏 40px、隐藏检索次要说明行。另有 skip-link、main tabIndex=-1、prefers-reduced-motion 规则。

验证（本机 8765 实例，443 条真实档案）：`npm run build` 通过；agent-browser 六种视口（1440×960、1280×720、1024×768、768×1024、812×375、375×812）整页无横向溢出，200% 文本缩放无溢出；薪资 15–30K 筛出 354、清除恢复 443 且焦点在搜索框；移动端选岗详情停在 top=16、返回列表焦点还原；批量编辑弹窗 Tab 循环圈禁、Escape 关闭后焦点回到触发按钮；方向键切换详情选项卡 aria 同步；城市拼音 hz→惠州 键盘可选；全程浏览器零报错。

未验证/未改：prefers-reduced-motion 的运行时模拟在工具里未生效，仅确认 CSS 规则存在；`frontend/scripts/test-mcp-config.mjs` 仍断言 SSE/自定义头被拒（实现早已支持），属过时断言，与本次布局无关，未改动。

## 2026-11-10 补充：采集/扩展/模型设置三页布局统一

继续同一设计语言，重排「数据与采集 / Agent 扩展 / 模型设置」三个管理页，保留绿色主题与全部业务逻辑（采集表单字段、安装/导入动作、密钥与 payload 均未动）：

- **数据与采集**：顶部三卡总览（本地岗位数 / 专用浏览器状态 / 采集记录数）；三个页签（采集工作台 / 数据源 / 任务记录）共用一条 tablist，键盘左右 + Home/End 可环绕；采集页签左侧步骤引导（选平台 → 扫码 → 带回岗位）+ 右侧双列表单；数据源卡 3 列、信号源配置收进 `<details>`「配置来源」；任务表格每格补 data-label，≤680px 转卡片；采集表单 ≤680px 收为单列（实测 375px 下输入从 135px 放大到全宽）。
- **Agent 扩展**：概览两卡（MCP 启用/总计、Skills 启用/总计，未加载时显示 —）；市场/MCP/Skills 改真页签（单 panel、aria-controls 固定，修 skills 页签指向不存在 id 的问题）；市场搜索框提级到工具条第一位，类目 chips 次之，emoji 图标换内联 SVG，卡片 hover 描边；目录缓存时间并入「目录来源与缓存说明」summary，删冗余状态段；MCP/Skills 列表紧凑主从布局，选中项 aria-pressed，空态精简，Skills 导入视图点列表项即退出（修粘滞）。
- **模型设置**：左侧纵向目标页签（对话/语义），卡片显示已存配置摘要（模式 + 模型名），aria-orientation=vertical，共用 navigateTabs 助手扩为方向感知；表单分「01 服务连接 / 02 选择模型 / 03 测试与保存」编号小节，渠道九宫格 + 协议/地址双列（顶对齐修错位），禁用态说明卡、测试/保存结果就地状态行；读取失败可重试（loadError + loadAttempt）；连接帮助 details + 密钥警示文案常显。
- **响应式**：@1280 数据源 2 列 / 维护 2 列；@1100 模型侧栏转静态横排、采集面板单列；@800 扩展主从单列；@680 输入 16px、44px 目标、任务表转卡、概览紧凑；扩展概览计数 strong/label 加 white-space:nowrap 修 768px 下「1 / 1」被挤断行。

验证（本机 8765 实例）：`npm run build` 通过（CSS 100.67 kB）；agent-browser 六种视口（1440×960、1280×720、1024×768、768×1024、812×375 横屏、375×812）三页均无横向溢出；页签键盘导航横/纵两向实测（含环绕与 Home/End）；模型目标页签切换后草稿保留（改模型名 → 切走再切回仍在，恢复原值后保存按钮回到禁用）；市场搜索实测本地过滤（1 字）与在线检索（2 字防抖 400ms，返回 npm/GitHub 结果），清空恢复全目录（12 MCP + 8 Skills）；tabNavigation 单测（node + transpileModule）两方向全过；全程浏览器控制台零报错。

未验证/未改：未点击「测试并发现工具」「开始采集」「测试连接/保存设置」等会启动进程或写配置的按钮；已保存密钥只读不动（模型名草稿已复原）；npm run build 的 >500 kB chunk 警告为既有问题，与本次无关。

## 2026-11-10 补充：简历/资料库/全局对话三页布局统一与 2K 宽屏适配

同一设计语言收尾剩余三页，另修 2K 显示器右侧留空。业务逻辑零改动（简历内容、资料查询、会话数据均只读不动）：

- **我的简历**：整理建议侧栏卡片化（paper-panel + 「整理建议」小节标签），01/02/03 改绿色编号圆标（与采集步骤同款），隐私提示改浅绿底色卡；文件拖放区加上传图标；编辑区「未保存的修改」未保存时琥珀色加粗。
- **公司资料库**：页签从私有 workspace-tabs 样式并入 management-tabs（与数据页同款），键盘左右 + Home/End 实测环绕、aria 同步；公司名从纯文字链接改绿点徽标（点按进入公司对话），列表行加 hover 浅绿底；锁定主体在「公司对话」页签上显示为徽标（限宽省略）；移动端分页按钮与公司徽标提到 44px 触控。
- **全局对话**：补页头（全站此前唯一没有 h1 的页面）：标题 + 说明 + 「会话保存在本机」；对话区高度按视口实算 `calc(100vh - 210px)`，1440×960 下底部 917 < 960 整页零滚动（改前溢出约 107px）；建议问题按钮改主题绿 hover、40px 高；移动端输入框/模型下拉 16px 防 iOS 缩放（修了 680 块规则被 composer 基础规则按源序覆盖的问题，挪进 800 媒体块），发送区首屏可见（高度自适应 + 线程 60dvh 封顶）。
- **2K 宽屏适配**：根因是 `.main` 整体 `max-width: 1920px`，2560 屏主区被掐在 1920 右侧空 ~420px；连同数据页 1440、扩展 1440（含一条被覆盖的旧 1200 规则）、模型设置 1280 全部移除，改纯流式。模型设置 ≥1800px 时「01 服务连接 / 02 选择模型」并排双栏、「03 测试与保存」通栏（`:not(.model-settings-footer)` 保证禁用模式下保存行分隔线不丢），避免单个输入框拉到两千像素宽。

验证（本机 8765 实例，简历页有真实内容仅做几何检查）：`npm run build` 通过；2560×1440 下七视图内容右缘均到 2532（仅剩 28px 主区内边距），模型设置双栏分节几何正确（01: 541–1504 / 02: 1544–2507 / 03 通栏带线）；1920×1080、1440×960、375×812 回归七视图零溢出；资料库页签键盘导航与全局对话页头/建议按钮、composer 字号逐项实测；全程控制台零报错。

未验证/未改：未切换模型「连接方式」下拉（避免动配置草稿，禁用态布局靠 CSS 选择器保证）；未点击任何会写数据或启动进程的按钮；简历正文与格式下拉未触碰。

## 2026-11-10 补充：三列点击聚焦（替代"只有对话可点 + 收起按钮"）

用户要求三列都能点、点谁谁扩张，去掉收起按钮。交互改为 `paneFocus ∈ {archive, info, chat}`（默认 info 均衡态）：点某列即聚焦该列，点其他列自动切换，无需收起动作；再点已聚焦列无副作用（不会误触收缩）。

列宽三态（grid 实测）：`info` 280/512/336（默认均衡）、`chat` 240/380/508（对话主栏）、`archive` 448/380/300（档案主栏，扫列表用）；`no-chat` 未选岗位两栏下同样支持聚焦档案。ChatWorkspace 的 `onCollapse` prop 与收起按钮移除（回归纯展示组件）。列宽 0.25s 过渡保留（真实前台标签页正常动画；后台面板中过渡冻结不影响最终布局）。

## 2026-11-10 更正：对话卡改常驻三栏（替代上下排布方案）

用户反馈"第二列上下排布"太抽象，改为从一开始就是三列：档案 / 岗位信息 / 岗位对话各占一列，对话列从详情面板里拆出成为 `research-layout` 的第三个网格子项。

- 默认三栏 `280px / minmax(0,1fr) / 336px`，对话列整列全高（`chat-page` flex 撑满，不再用 320px 定高）。
- 未选岗位时加 `no-chat` 退化为两栏（对话没有岗位范围无意义，不渲染空列）。
- 点击对话列仍是伸缩态：`240px / minmax(300px,380px) / minmax(0,1fr)`——对话列扩张为最宽主栏、信息列收窄内滚、档案列再收一档；头部"收起"退回。
- 基础样式（中窄屏）补 `.research-layout > .job-chat { grid-column: 1/-1 }`：对话列拆出后在 720–1200px 的两栏网格里不再被塞进 320px 窄列，而是横跨整行接在信息卡下方。

验证（1440×900）：未选岗位 `280/868` 两栏；三栏默认 `280/512/336` 全高对话；扩张态 `240/380/508`；收起还原三栏默认；三态整页均一屏。`npm run build` 通过。

## 2026-11-10 补充：岗位对话卡片动态伸缩

用户期望"搜岗位找信息 → 点对话卡片展开变长、压缩信息卡与档案列"的动态伸缩卡片。实现（工作台桌面端）：

- 详情面板内有岗位时，标题/JD状态/选项卡/正文包进 `.detail-info-col`，岗位对话成为面板内第二块卡片。默认：信息列占满、对话卡 320px 紧凑卡在底部。
- 点击对话卡片（整卡可点，`cursor: pointer` 提示）→ `chat-focus` 态：对话卡展开为右侧主栏（全高、`chat-page` 100%），信息列收窄为 `minmax(320px,400px)` 竖排内滚并加分隔线，档案列 304→258px 同步收窄；列宽带 0.25s 过渡。对话头部出现"收起"按钮（ChatWorkspace 新增可选 `onCollapse` prop，其他对话场景不受影响），点击退回默认态。
- 切岗位不重置伸缩态（在对话流中连续查看多个岗位时保持展开）。

验证（1440×900）：展开态 grid 实测 `258px / minmax(320px,400px 列内 400px) / 对话全高 750px`、收起按钮在位、信息列内部滚动正常；收起态回到信息列满宽 + 对话 345px 紧凑卡；两态整页均一屏。`npm run build` 通过。

## 2026-11-10 补充：公司资料库一屏工作台

用户反馈公司资料库页面太长。复用 main-workbench 框架（evidence 视图同样启用 100dvh 固定高度）：页头/控件行/选项卡紧凑化，来源列表与公司对话面板 `flex: 1` 各自内部滚动，底部分页移除（顶部保留）、页脚隐藏；来源条目压缩（条目 padding 20→10、摘要行 8px、公司链 12px），每页 10→50 条。公司对话 tab 同样全高布局（会话列表 + 对话区）。

验证（1440×900）：整页高度=视口（1777→900）；列表 50 条内滚（scrollTop 拉到底部验证）；来源档案/公司对话两个 tab 均一屏；`npm run build` 通过。

## 2026-11-10 补充：工作区管理三页紧凑化

数据与采集 / Agent 扩展 / 模型设置启用新的 `main-compact` 框架（与 main-workbench 并列，区别是内容超一屏时整页可滚而非强制内滚）：topbar 44px、页头单行 20px、页脚隐藏、paper-panel padding 24→16、区块间距收紧。定点收紧：管理概览卡（图标 32px、padding 10/14）、选项卡行 38px、采集记录表格行（th/td 7px 10px）、浏览器采集面板 gap 20、扩展工具条/列表/详情、模型设置卡（16px padding、field-label 12/6、fieldset 14）。

验证（1440×900）：三页均 `main-compact` 生效、main 固定 900px 页面内滚、document 高度=视口；岗位研究页回归正常（workbench 三栏未受影响）；`npm run build` 通过。

## 2026-11-11 补充：三列聚焦的可发现性标识

用户反馈三列点击聚焦功能太隐蔽，第一眼会误判为布局缺陷。加三层标识：

1. **一次性提示条**：三栏上方居中显示"提示：点击任意一栏可聚焦放大，点其他栏切换焦点"（绿色胶囊 + 关闭按钮），点 × 后写入 localStorage（`jobscope.pane-focus-hint=off`）不再出现；隐私模式写入失败仅在当前会话内不再显示。
2. **悬停高亮**：非聚焦列悬停时边框变浅绿 + 外圈阴影，`cursor: pointer` 全列示意可点。
3. **聚焦绿边**：聚焦列边框常绿（`var(--green)`），一眼看出当前焦点在谁。

验证：提示条显示/文案正确；点 × 后 localStorage 写入 `off`、刷新不再显示；聚焦列边框实测 `rgb(23,62,53)`（深绿）、未聚焦列保持 `rgb(220,228,223)` 浅灰。`npm run build` 通过。

## 2026-11-11 补充：修窄对话列内容撑破卡片

用户反馈对话列样式超出卡片范围。根因：三栏布局下 `.chat-page` 作为 flex/grid 子项默认 `min-width: auto`，composer 工具栏的长键盘提示等不可收缩内容把它撑到容器外（archive 聚焦 300px 窄列时整树右移 66px）。修复：`.job-chat .chat-page` 加 `min-width: 0` + `container-type: inline-size`，容器查询 `≤460px` 时隐藏键盘提示、压缩头部与欢迎区；宽列（>460px，如对话扩张态 508px）提示照常显示。

验证：三态（archive 300 / info 336 / chat 508）对整棵对话子树做越界扫描全部 0 溢出；hint 按容器宽度自适应显隐；`npm run build` 通过。

## 2026-11-11 补充：公司情报多平台异步检索

用户指出公司情报检索只查巨潮、与求职参考的数据打通名不副实。改造：

- **后端** `POST /api/company-research`（body `{company}`）：以精确公司名跑五个渠道——cninfo 公告（接口）、boss/zhaopin/51job（浏览器）、liepin（接口），后台线程逐渠道串行；渠道走 BROWSER_SOURCES 判定（liepin 是接口源）。每渠道独立 try：失败只标记该渠道任务（"该渠道检索失败，已跳过；不影响其他渠道"），不中断编排；端点立即返回任务列表，前端轮询。原 `run_collection` 拆出 `execute_collection` 复用。
- **前端**：按钮"检索公告"→"**多平台检索**"；说明文案改为"多平台检索该公司岗位与公告，结果直接作为求职参考的依据材料；渠道失败不影响整体"；点击后粘性 toast"已启动 5 个渠道…"，各渠道终态时逐渠道 toast 结果（完成 notice / 失败 error），全部结束后刷新详情与元数据。检索回来的证据入 evidence 表，供"从材料里抽取信号""生成参考"直接使用——数据打通即在此。

验证：真实运行"海宁辰曦科技有限公司"五渠道——cninfo/zhaopin/51job completed、boss needs_verification、liepin failed，渠道间零牵连，任务消息如实；单测 test_company_research.py 验证渠道规划与失败隔离（stub 渠道：cninfo 抛错仅自身 failed，liepin completed，浏览器渠道 needs_login）；全套 487 passed；UI 实测按钮文案、粘性 toast、防重复触发。

## 2026-11-11 补充：多平台检索接入企查查/天眼查 + BOSS"卡住"诊断

- **接入快查渠道**：`/api/company-research` 从 5 渠道扩到 7 渠道——cninfo → qcc（企查查）→ tyc（天眼查）→ boss → zhaopin → 51job → liepin。qcc/tyc 复用并发会话开发的 `company_lookup.py`（company_web 传输，读项目 Chrome 公开页，卡片按精确主体名归档为 `web_company_profile` 证据），抽出 `execute_lookup` 供单渠道端点 `/api/company/lookup` 与编排共用。
- **BOSS"卡住"诊断**：用户反馈编排里 boss 拉起浏览器卡住（单独采集成功过）。实测编排中 boss 渠道 38 秒即返回 `needs_verification`——不是卡死，是 BOSS 停在滑块/登录验证页：此前会话验证态过期触发风控。设计上如实把页面留在项目浏览器前台交给用户手动完成，不重试不绕过；在项目 Chrome 窗口里完成一次 BOSS 验证即可恢复。企查查同理 needs_verification（公开页限流，首次使用需在项目浏览器登录）。
- 实测七渠道（海宁辰曦科技有限公司）52 秒全部到终态：cninfo/tyc/zhaopin/51job completed（tyc 解析到 3 家公司卡片）、qcc/boss needs_verification、liepin failed（接口结构问题，如实标记），零卡住、渠道间零牵连。测试扩展到七渠道规划与 qcc 卡片归档、tyc 失败隔离（18 passed）。

## 2026-11-11 补充：公司情报检索结果改预览卡片

用户反馈多平台检索完只剩超链接列表。公司情报区的来源列表从纯链接行替换为 `EvidenceCard` 可折叠预览卡（与公司资料库同组件）：摘要行含标题 + 来源·时间·类型（如"tyc · 企业信息快照"），点击展开全文与出处，保留每页 4 条分页。空态文案同步更新（提示可多平台检索）。列表样式适配卡片（gap 10、摘要 10/12px）。

验证（1440×900，海宁辰曦科技有限公司岗位）：tyc 企业信息快照卡展开显示真实工商字段（法定代表人/注册资本/统一社会信用代码/地址等，天眼查公开页解析），boss 企业自述卡折叠预览正常；`npm run build` 通过。

## 2026-11-11 补充：需人工验证的页面自动前置

用户反馈 BOSS 报验证状态时"没看见打开浏览器标签页"——根因：各渠道各自开新标签，BOSS 停在验证页时标签被后续渠道的新标签埋没，"留在前台"名不副实。修复：`browser.collect` 在 blocked（未收到响应）与 mapper 判定 needs_verification/needs_login 两条路径上调用新增的 `_front(tab)`（`tab.set.activate()`，带保护）；`company_lookup` 的验证路径同样激活。

回归：相关测试 49 passed；BOSS 渠道实跑（验证解除后）直接 completed 读到 5 条——验证态已由用户在项目 Chrome 人工解除，激活逻辑待下次遇到验证页时自然生效。

## 2026-11-11 补充：对话停止改为即时应答+协作中断+迟到丢弃

用户反馈岗位研究对话页发送后按"停止"极慢。根因三层：research 模式整条流水线（索引同步→规划 LLM→embedding/检索→综合 LLM，最坏约 2 分钟）**没有任何取消检查点**，取消只能跑完全程后丢结果；general 模式只在轮间/工具间检查、流式 LLM 调用中不可中断；前端停止后完全等 SSE `cancelled` 事件才复位 UI。三个对话页（全局/岗位/公司）共用 ChatWorkspace+useChatRun，一处修复全覆盖。

重构为三层取消契约（JobScope `e261021`）：

- **L0 即时应答**：`cancel()` 持 `handle.condition`（RLock）原子执行 set 标志→push `cancelled`→`closed=True`→`finish()`，毫秒级返回；会话锁即刻释放可发新消息。幂等：已 done 的 run 取消为 no-op；与 `_settle` 的落库竞争由同一把锁裁决，不存在"停止后冒出答案"的中间态。
- **L1 协作中断**：`ResearchAgent.answer` 新增 `should_stop` 参数，5 个检查点（入口/同步后/规划后/检索后/恢复检索前）命中抛 `RunCancelled`；`stream_turn` chunk 循环内轮询停止标志，流式模型调用可中途断流停止烧 token。
- **L2 有界丢弃**：打断不了的调用（非流式 complete≤60s、MCP≤35s、embedding≤20s、SQLite busy≤15s）本就有超时上界；返回后下一个检查点退出。`RunHandle.closed` 封口：僵尸线程一切 push 丢弃、`_settle` 不落库——停止后迟到结果永不出现。

前端 `stop()` 改乐观收尾：cancel POST 成功立即关流、复位 UI、重载会话；SSE `cancelled` 处理器保留兜底跨标签页场景。行为变化：general 模式停止不再落库部分答案（原 interrupted 标记），与研究模式统一为纯丢弃，符合"中途任务打断丢弃"的要求。

验证：test_chat_runs 重写取消组并新增 4 例（阻塞步骤中取消即时生效+日志封口、检查点抛 RunCancelled、自然完成后取消 no-op、取消后会话立即可续）共 13 passed；全量 506 passed；`npm run build` 通过；8765 实例实测 cancel 立即返回 `done:true, cancelled:true`（旧代码 `done:false`）。测试新增 `join_worker` 辅助：即时取消会留下活到检查点的僵尸线程，必须 join 否则吞掉后续测试 monkeypatch 的假模型流（曾致相邻测试假失败）。

## 2026-11-11 补充：岗位/公司范围下口语泛问召回本公司证据

用户在岗位对话页问"这是个什么公司"答"没有找到证据"，而左栏公司情报里明明有 boss 公司自述。根因：`plan.terms()` 对纯口语泛问切出的 2-gram 几乎全在 STOP 表（"什么""公司"都是停用词），FTS MATCH 零实词可查；未配 embedding 时 dense 通道跳过，recovery 用原问题复查同样零命中——**窄 scope 下口语泛问 BM25 完全失效**。修复（JobScope `8b1bf8d`）：`retrieve()` 在 scope（job/company）模式下把公司名 token 注入检索词——SQL 层 `d.company=?` 已把候选限定在本公司，公司名是绝对可靠的召回信号；相关性 40% 门槛仍按原始查询词计算，公司名命中直接放行（注入只扩召回不抬门槛）；global 范围行为不变（无意义查询仍零命中走澄清引导）。

验证：真实库复现"这是个什么公司"从 0 证据 → 2 条（boss 自述排首位 + 岗位 JD）；8765 重启后端到端走真实岗位 scope，回答 mode=model，引用 boss 自述给出规模/成立年份/业务板块且如实标注"公司自述未经独立核验"；新增 test_colloquial_prompt_recalls_company_scope_evidence（短/长垃圾泛问两条路径 + global 零命中回归），全量 507 passed；探针会话已清理。

## 2026-11-11 补充：向量缓存随换模型静默失效，重建后 dense 通道打通

用户纠正：embedding 一直配着（remote bge-m3 @127.0.0.1:3425）。重查证实服务端 `embedding_configured=true`——上一条"未配 embedding"是我从检索耗时 141ms 的错误推断，实际是 dense 通道**静默空转**：向量缓存 580 条全是旧本地模型 `local:BAAI/bge-small-zh-v1.5` 时代构建的，换成远端 bge-m3 后 identity 变化，检索按新指纹 JOIN `research_vectors` 恒为 0 行 → dense 降级，"向量索引尚未建立"提示只存在于 trace 的 degraded 里，界面不可见——换模型导致向量缓存过期没有任何主动暴露。另一个教训：探针脚本测 `EmbeddingProvider.configured` 必须套 `use_config(ModelSettings(repo).effective())` 上下文（worker 线程的真实环境），否则读到默认 disabled，得出伪象结论。

处置：`POST /api/research/index {"operation":"embed"}` 增量重建 2303 条 bge-m3 向量（330s）。验证："这是个什么公司"从 strategy=bm25 升级为 **hybrid**，boss 自述双通道 `['bm25','dense']` 命中；正常关键词查询同样 hybrid，degraded 清空。BM25 公司名注入（上一条）+ dense 语义召回形成双保险。改进留待后续：换模型时主动提示重建向量缓存（2303 条 × batch24 同步跑 5.5 分钟太重，适合后台任务或首次检索时惰性触发）。

## 2026-11-11 补充：研究模式回答流式输出 + 工具步骤执行详情抽屉

用户要求对话支持流式输出、工具执行内容可折叠查看。研究模式（岗位/公司对话默认）综合回答原本走非流式 `complete()`，要干等 30-60 秒一次性出答案。改造（JobScope `276944b`）：

- **流式**：`model_transport` 新增 `stream_complete`（openai/anthropic 双协议，逐 delta 回调，finish_reason/截断语义对齐 complete）；`answer_question` 加 `on_delta`——模型输出是 JSON 包裹，实现 `_AnswerExtractor` 字符级状态机增量提取 `"answer"` 值（`\"`/`\\`/`\n`/`\uXXXX` 转义、``` 围栏前缀、key 跨 chunk 切断的 32 字符尾窗），反转义增量实时推送；**流式内容仅是预览**，最终答案仍以完整 `json.loads` + 引用一致性校验为准，校验失败照旧走证据摘录降级（流式预览被落库答案替换）。链路 `chat_runs → ResearchAgent.answer → synthesizer(on_delta)` 推 `delta` SSE 事件，前端既有 delta→run.answer→Markdown 渲染路径零改动直接生效。`on_delta=None` 时走原 `complete()` 路径，既有测试零破坏。
- **抽屉**：ToolTimeline 步骤展开体增强——研究步骤 payload 无 `arguments`/`result_preview`（数据平铺为 plan/strategy/terms/candidates/degraded），改为无参数时用现成 `readable()` 把除结构字段外的全部字段渲染为"执行详情"；历史消息 trace 与实时 run 共用渲染。api.ts `ResearchRun.events` 对齐 `ToolStep[]` 修掉 TS 缺口。

验证：新增 `_AnswerExtractor` 单测（转义/\uXXX/围栏/分块、evidence_ids 不外泄、校验失败预览不落库）与 research run 流式集成测试（SSE 含 delta、done 答案完整）；全量 510 passed；`npm run build` 过；8765 重启后真实岗位 scope 端到端：start → 4 step → **326 个 delta 逐字流出** → done，引用正确，探针会话已清理。测试坑：patch `httpx.Client.post` 会连累 TestClient 自身（需 isinstance 放行 original，且须在 patch 前存 original 防递归）。

## 边界与未测

- 大批量（几百条）的真实删除只在单测里跑过（HTTP 层 1000 条上限校验），真实档案上的多量大删由用户自行操作，路径与自测相同。
- `parse_salary` 口径不变：`1-1.7万`、`20-30K`、`3-5万` 可解析；`18元/时`、`薪水面议`、美元年薪不参与薪资筛选。
- 筛选"选筛选全部"依赖 `/api/jobs/ids` 与列表同一 where 构建，单测断言两者 total 一致；未测超过 20000 条的截断行为（当前库 1729 条）。

## 2026-11-11 补充：对话排版、阈值滚动、多渠道模型与上下文压缩

用户反馈：回答行距过大且空行多；流式输出时强制吸底，想上翻回看会被拽回；模型只能配一个全局渠道；缺少上下文用量与一键压缩。

### 排版

- `.md-body` 行高 1.85→1.65、段距 10→6px、标题/列表/引用/分割线边距同步收紧；`.md-body` 显式 `white-space: normal`，阻断 `.message-content` 的 `pre-wrap` 被 Markdown 子树继承（此前源码换行会额外撑出空行）。
- 宽松列表产生的 `li > p` 段距压到 2px；空 `p/li` 不渲染。代码块内容不做任何文本改写（曾试过正则折叠连续空行，会误伤代码块，已撤销）。

### 阈值滚动

- 距底部 ≤80px 视为吸底；超过即暂停自动滚动并显示「↓ 回到底部」。
- 向上滚轮/触控上滑/PageUp/Home/↑ 立即解除吸底，不依赖 scroll 事件与定时器（内嵌浏览器后台面板会节流 scroll/rAF，实测程序化改 `scrollTop` 甚至不派发 scroll）；向下操作后补算恢复。
- 回到底部改为即时滚动：后台面板会冻结 smooth 动画。会话恢复时补一次 150ms 延迟吸底，避免消息与布局分两拍到位停在顶部。

### 多渠道模型

- 新表 `llm_channels(id,payload)`：每条渠道保存名称、预设、协议、地址、密钥、默认模型、模型列表与上下文窗口；同名再保存视为编辑。密钥留空保留旧值，勾选清除才删除。
- 凭证绑定：改地址或协议时必须重填密钥或显式清除（渠道保存与全局设置同一规则）；测试连接/拉模型列表可带 `channel_id`，后端用该渠道已存密钥，而不是全局配置。
- 「启用为默认」把渠道 payload 整体写入 `model_settings.llm`（带 `saved_channel_id` 标记），下游请求链路零改动。
- 会话级 `conversations.llm_channel`：请求里省略=沿用会话已存渠道/模型，空串=重置为渠道默认/全局默认，显式值覆盖。通用 Agent 的可用性校验按“本次实际生效模型”判断，不再只看全局配置。
- 对话输入框模型下拉按渠道分组，保留当前选中项；渠道被删除后显示「渠道已删除 · 模型」占位，可切回全局。上下文窗口按所选渠道登记值切换。

### 上下文用量与一键压缩

- 用量口径：未折叠消息 + 流式增量，按字符/3.2 估算，加 600 tokens 固定基数（system 与工具声明）；60%/85% 变色，`role=progressbar` 可读。
- 压缩：保留最近 4 条，至少 2 条可折叠才可点；旧消息打 `compacted` 标记（不删除、UI 淡化并标「已折叠」），边界插入 `kind=compaction` 摘要检查点。摘要请求使用会话自己的渠道与模型。
- 并发与一致性：压缩期间该会话禁止发送/编辑/删除/再次压缩；写回前比对消息 id 快照（CAS），期间历史有变则 409 且不落任何标记。
- 不静默截断：按消息边界切块（每块 ≤16000 字符）分段提炼再合并；整段超过 240000 字符明确 422 拒绝。上一份摘要会作为可折叠内容参与下一轮压缩。
- 摘要检查点固定注入通用 Agent 上下文（最近 6 条窗口之外单独保留），压缩后继续追问仍能看到早期结论。

### 验证

```powershell
cd E:\Github_Pro\ResumeLoom\jobscope
.venv\Scripts\python.exe -m pytest backend/tests -q   # 524 passed（新增 test_llm_channels 5 项、test_conversation_compress 8 项、通用 Agent 摘要钉入 1 项）
cd frontend
npm run build                                          # tsc -b + vite build 通过
```

- 发布仓库 E:\Github_Pro\JobScope 同步后复跑全量：524 passed；提交 `b3de1b4` 已推送。
- 浏览器实测（1440×900，真实 3425 模型）：行高 23.1px、li 2px、`white-space: normal`；流式生成中上翻 500px 后最终仍停在距底 490px，未被拽回，点「回到底部」归零。
- 渠道：新建本地渠道（2 个模型）→ 下拉出现分组与默认项；有密钥渠道改地址时保存/测试禁用并提示重填；激活后全局表单同步地址/模型/窗口；删除渠道后会话显示失效占位。
- 压缩：3 轮问答后点压缩 → 折叠 2 条、摘要卡含公司/岗位/薪资/证据 ID、旧消息淡化、按钮因历史不足转禁用；验收会话与临时渠道已删除，全局模型配置保持原 3425 网关。

## 2026-11-11 补充：统一范围化通用 Agent 与范围扩展复制跳转

用户要求去掉"资料研究 / 通用 Agent"模式切换：所有入口统一为一个通用 Agent，入口（岗位/公司/全局）只决定检索范围；需要更大范围时由 Agent 出工具卡片，点击后复制会话并跳转，原会话保留。

### 统一入口与范围执行

- 模式切换按钮移除；三个入口都是同一个 Agent（工具调用、停止、思维链、模型/渠道选择、上下文用量与压缩全部保留）。输入框旁显示范围标签：`本岗位 · 岗位名` / `本公司 · 公司名` / `全项目`；岗位页标签旁有「扩大范围 ↗」手动入口（岗位可选"所属公司/全项目"，公司只有"全项目"）。
- `GeneralAgent.start(scope=…)` 接收服务端权威范围（`handle.scope`），旧的模块级全局 SCOPE 不再参与执行；范围随运行状态、trace 与工具确认续跑状态持久化，续跑前重新校验会话范围。
- 工具层隔离（`agent_scope.py`）：`project_search/project_salary` 走既有 `scope_clause` 过滤；`project_read` 直接传证据 ID 也校验主体与岗位映射（`research_job_documents`，兼容多岗位共享证据）；`company_profile/company_detail` 拒绝范围外公司，且计数/分页/引用登记先过滤后返回；`pay_vs_city_median` 等跨公司派生信号在窄范围一律拒绝，本地派生信号需要可信规则来源；查询规划需要澄清时直接返回澄清，不再把未知主体当全局查询。系统提示加入范围说明与扩展指引。
- 运行时统一：`ChatRunInput` 默认 `general`；配置了模型时旧 `research` 请求也走同一范围化 Agent，未配置模型保留离线检索兜底；旧 `/api/chat` 经 RunManager 同步适配器共享同一套锁、持久化与工具确认。
- 回归：全量 580/581 passed（新增 `test_scoped_agent.py` 32 项隔离测试、`test_scope_expansion.py` 25 项含并发幂等与过期待确认不永久阻塞）。

### 范围扩展与复制跳转

- 新工具 `request_scope_expansion(target: company|global, reason)`：只生成提案，不切换权限、不执行扩大后的查询；生成卡片后本轮优雅结束。允许路径仅 岗位→本公司/全局、公司→全局（`scope_expansion.py._target` 服务端推导，模型不能指定别的公司）。
- 卡片在 live 气泡与历史消息中都直接可见（`actions` 随消息持久化，SSE step 携带 `action`），显示目标范围、原因与「复制对话并进入…」；点击后"正在复制…"，成功变「打开已复制对话」，失败可重试。范围标签的手动入口生成同样的卡片。全局入口无此按钮。
- `POST /api/conversations/{id}/scope-expansions` 提案、`POST …/{action_id}/accept` 接受；接受在事务内快照完整消息（含压缩检查点/折叠标记/引用/工具记录/模型渠道/标题），复制简历私密标记，mode=general，记录 lineage；不复制活动运行与待确认队列；生成中/压缩中/有存活待确认时 409，过期的持久化待确认不永久阻塞。同一提案并发/重复接受返回同一目标会话（幂等）。
- 跳转：`App` 统一处理 `{requestId, scope, conversationId}`；公司目标进入公司资料库并按 `chooseCompany` 锁定主体，全局目标进入全局对话；`useChatRun` 显式打开优先于 sessionStorage 恢复，带 selection 代际防过期响应覆盖，成功后消费请求并 toast「范围已扩大，可继续提问」。**点击只复制跳转，不自动发起新的模型请求。** 克隆会话顶部显示来源提示条；范围外的历史卡片不渲染。
- 模型配置一致性沿用上一交付：会话模型/渠道随克隆保留，用量窗口按所选渠道切换。

### 验证（真实模型 3425）

- 岗位页提问"比较这家公司所有岗位"→ Agent 先按岗位范围调用检索与公司工具，发现不足后生成「扩大到公司范围」卡片（原因写明当前仅含本岗位 JD 与同公司非岗位来源）→ 点击复制跳转公司页，锁定证优客，历史 2 条完整保留，未自动发起新请求。
- 公司页手动「扩大范围 ↗」→「全项目」→ 卡片 → 复制跳转全局页，同样保留历史；回到原岗位，卡片已变「打开已复制对话」，再点仍能打开同一克隆（幂等）。三条会话（岗位/公司/全局）消息数一致，均无活动运行。
- 窄栏（508px 岗位对话列）卡片宽度 332px 无横向溢出（thread scrollWidth=clientWidth）。
- 验收会话已删除，浏览器恢复初始状态；8765 已用最终代码重启。

## 2026-11-11 补充：模型设置改为「渠道中心 + 功能绑定」

用户反馈模型设置混乱：渠道、全局大表单、"启用为默认"（复制配置）堆在一页。参考 zcode / AI IDE 形态重构为页内两大标签。

### 渠道中心
- 「模型渠道」标签：左侧渠道列表（名称/host/模型数/「对话默认」标记），右侧详情摘要或编辑表单（预设、协议、地址、密钥+换址守卫、默认模型、对话模型列表、**向量模型列表（新增，供语义检索选用）**、上下文窗口、测试连接、获取模型）。
- 渠道是密钥与连接信息的**唯一事实源**：「设为对话默认」不再复制配置到全局行，而是写一条指向渠道的绑定（`model_bindings` 表）；轮换渠道密钥即刻对所有功能生效。
- 被功能引用的渠道删除时 422，提示先改绑（列出引用功能名）。
- 旧"启用为默认"数据一次性迁移：`saved_channel_id` 指向的渠道若仍存在则提升为 agent 绑定（指向已删渠道则跳过，回落旧行）。

### 功能与模型
- 四张功能卡：对话 Agent（默认模型，必须显式选渠道+模型）、语义检索（渠道›向量模型 / 本地缓存模型 / 关闭·回退 BM25）、可靠性信号抽取 / 简历适配分析（跟随对话默认 / 关闭 / 渠道›模型）。改完即存，立即生效；每卡显示当前实际生效摘要（渠道名·模型），渠道被删显示「渠道已删除：正在回退」。
- 解析链：`settings.effective()` 组装时按绑定覆盖 llm/embedding 并追加 extraction/analysis 两个 key（middleware/use_config 快照全量下发，Agent/传输层零改动）；`purpose_configured/get_purpose_config` 供请求侧调用；extract 与 /api/analyze 门禁、semantic 分别改读自己的 purpose。
- 旧 `PUT /settings/models` 全局行保留为回落（未绑定时照旧生效），前端不再使用。

### 验证
```powershell
.venv\Scripts\python.exe -m pytest backend/tests -q   # 590 passed（新增 test_model_bindings 25 项：绑定/回落/disabled/本地/删除守卫/anthropic 拒绝/迁移/extract+analyze 隔离）
npm run build                                          # 通过
```
- 发布仓库同步后复跑：590 passed；提交 `330856e` 已推送。
- 浏览器实测：两大标签正常；用户的 3425 网关已落成渠道「本机网关 · 3425」（含密钥与向量模型 auto/bge-m3），agent+embedding 绑定指向它；功能页四卡状态与实际生效一致；全局对话发消息确认走绑定渠道回答正常。抽取/分析保持"跟随对话默认"。
- 已知取舍：旧 `PUT /settings/models` 若在存在 agent 绑定后调用会被绑定遮蔽（前端已无入口，仅兼容保留）。

## 2026-11-11 补充：思维链窗自动滚动

用户反馈思维链（reasoning-body）输出不会自动滚动。复用主聊天线程的阈值吸底策略：

- 距底 ≤80px 跟随流式增量；上翻滚轮/触控/PageUp 立即解除并出现「↓ 回到底部」悬浮按钮；向下操作或点按钮恢复跟随（即时滚动，不依赖 smooth）。
- 仅流式（live）且展开时提供按钮与跟随；历史消息的思维链保持自由阅读。details 展开后补一次延迟落底（高度此时才真实存在）。
- 容器加 `overscroll-behavior: contain` 防滚穿；`.reasoning-body-wrap` 相对定位承载按钮。
- 验证（真实模型流式）：跟随阶段 distance=0；上翻后 distance 保持 200 不被拽回、按钮出现；结束后自动折叠，按钮消失。发布仓库 `8aa1c75` 已推送。

## 2026-11-11 补充：消息操作按钮移至气泡下方

用户反馈复制/删除等按钮应放在对话气泡下方。原先按钮藏在气泡头部且 hover 才显示，难以发现。

- `msg-actions` 从气泡头移到气泡内容之后（article 最后一个子元素），常显不再依赖 hover；用户消息右对齐、助手消息左对齐。
- 样式弱化为文字级小按钮（无边框灰字，hover 浅底），避免喧宾夺主；编辑/重发/删除仍仅可变会话中显示，删除二次确认逻辑不变。折叠消息（compacted）继续隐藏操作。
- 验证：三类气泡按钮均渲染在 article 底部（actionsBottom=true、head 无残留）、用户消息按钮与正文右缘对齐、常显可见。发布仓库 `10b51da` 已推送。

## 2026-11-12 补充：扩展市场拆分 MCP 与 Skills 双页

用户要求市场也按类型各管各的、互不干扰。原先单一「市场」tab 里 MCP 服务与 Skills 混排：共用一个搜索框与一套分类筛选（分类取两边并集），在线检索结果（npm + GitHub）也同屏交错。

- `AgentExtensions` 顶层 tab 从 3 个拆为 4 个：MCP 市场 / MCP 工具 / Skills 市场 / Skills，发现页与管理页按类型配对；各 tab 描述文案集中为 `TAB_DESCRIPTIONS`。
- `MarketView` 增加 `kind: 'mcp' | 'skill'`：搜索框、placeholder、分类 chips、目录列表、在线检索结果与空态提示全部按页独立；MCP 页只出 npm 在线结果，Skills 页只出 GitHub 在线结果（后端 `search_online` 不变，前端按 kind 取半边）。
- 后端仅改一处文案：`skills_error` 去掉「MCP 结果不受影响」的跨页表述（该提示现在只出现在 Skills 市场页）。
- 验证：运行副本与发布仓 `tsc -b && vite build` 均通过；`test_extensions_market.py` 10 passed。发布仓库 `73ca32d` 已推送。
- 并发说明：交付时运行副本还叠着一份未提交的「联网搜索指引」文案改动（对应 `feat/agent-web-search-guidance` 分支，无提交）；按用户选择本次提交已剔除该文案，仅交付拆页本身，两仓在该文案上暂不同步。

## 2026-11-12 补充：Agent 联网搜索引导（MCP 接入 + 主动性）

用户希望 Agent 有联网搜索的主动性与探索性，问"接入 MCP 是否就可以"。结论：可以——工具市场已有 Tavily / Brave Search / Fetch 三个搜索类预设；真正缺口是系统提示没有"何时该主动联网"的引导。路线（用户确认）：MCP 接入 + 主动性引导，主动自动搜（auto_tools 放行免逐次确认；读过简历的会话仍逐次确认防泄露）。

### 实现

- `_catalog()` 组装工具目录时按关键词（search/fetch/tavily/brave/web/网页/搜索/检索/抓取/联网）检测启用的 MCP 是否含搜索/抓取类，系统提示条件化注入两分支：
  - **有搜索工具**：项目查不到或涉及工商/新闻/行业/政策等外部时效事实时，先一句话说明要查什么再主动调用搜索 MCP；结果标注来源 URL 与访问时间，严格区分项目证据 [E-…] 与联网信息，禁止伪装成项目证据；无结果如实说明，计入现有 8 轮/12 次/240s 限额。
  - **无搜索工具**：如实说明时效信息无法核实；回答这类问题禁止输出任何 [E-…] 引用（真实测试中模型曾幻觉引用，被引用校验拦截后补强了这条）；可提示用户到「Agent 扩展 → 工具市场」接入搜索 MCP。不编造检索过程。
- 市场三张搜索卡片 desc 追加"设为自动授权后 Agent 会主动联网"引导；扩展页 TAB_DESCRIPTIONS 的 mcp-market/mcp 页说明接入联网引导（与市场拆页重构同时落地的并发会话采纳并整合了文案）。

### 验证

```powershell
.venv\Scripts\python.exe -m pytest backend/tests -q   # 594 passed（新增 test_web_search_guidance 4 项：无搜索引导装安装路径/搜索 MCP 触发主动指引/非搜索 MCP 不误触发/禁用不触发）
npm run build                                          # 通过
```
- 真实模型实测（无搜索 MCP）：问"证优客最近一个月有什么新闻"→ Agent 先在项目内检索两次、如实说明项目范围不含新闻信源、未编造引用；第一轮曾幻觉 [E-…] 引用被引用校验拦截（防线有效），据此补强了无工具分支的"禁止输出证据引用"话术后复测通过。
- 已知边界：@playwright/mcp（浏览器自动化）不会被误判为搜索工具；真实联网验收需用户注册 Tavily/Brave key 后按「市场安装 → 填 key → 测试 → 启用 → 授权设自动」步骤接入，然后问时效问题观察主动调用。
- 发布仓库 `f2fcfbe` 已推送（直接落 main——feat 分支名与并发会话残留分支冲突，结果与内容不受影响）。

## 2026-11-12 补充：对话三档工具权限模式（变更前确认/自主/完全访问）

用户要求参照 ZCode agent 的三档权限模式，给对话加会话级模式切换，并把 MCP 工具执行按档自动化。此前门槛只有每工具静态白名单（`auto_tools`），要免确认得逐工具去扩展管理页勾选。

### 语义（模式是主策略）

| 档位 | MCP 调用行为 |
|---|---|
| `confirm` 变更前确认 | 全部弹审批卡（auto_tools 暂不生效） |
| `autonomous` 自主（默认，存量迁移） | readOnlyHint（服务方声明）或 auto_tools 白名单内直接执行，其余逐次确认 |
| `full` 完全访问 | allowed_tools 内全部自动执行 |

不变项：内置 9 工具三档都直接执行（全只读/仅提案）；**简历红线**——本轮读过简历后外发调用任何档位都强制确认（保住既有隐私底线与测试语义）；allowed_tools 白名单与 scope 边界照常；审批状态机（pending/claim/approve/SSE approval_required）零改动复用。

### 实现

- `conversations` 新列 `permission_mode`（默认 `autonomous`，位置插入两处 INSERT 同步补列）；`ConversationPatch`/`ChatRunInput` 加 Literal 字段，`PATCH /api/conversations/{id}` 与 `POST /api/chat/runs` 读写；`RunManager.start` 仿 model 的 None=沿用/显式写回语义，随每轮 run 透传给 `GeneralAgent.start` 并落 state 快照（resume 沿快照，运行中前端禁切）。
- 判定点 `general_agent._execute`：resume 红线 > full 全放 > confirm 全拦 > autonomous 按 `binding['read_only']`（`_catalog` 时从已存但从未使用的 `annotations.readOnlyHint` 提取进 registry，registry 项不参与 fingerprint、老 pending 状态缺键安全回退）∪ auto_tools 放行；审批 reason 分档文案。
- 系统提示词按档注入确认指引（仿 `_web_search_guidance` 的 `_permission_guidance`），替换原先写死的"MCP调用参数会展示给用户确认"。
- 前端：输入区三段分段控件（确认/自主/完全访问，当前档白底绿字，运行/待确认禁切），底部提示文案随档变化；切往完全访问弹 420px 小确认框（点背景/Escape/取消均可回退），降档直达；档位照 `changeModel` 三段式保存（乐观更新 + PATCH + 新会话随 startChatRun 落库）。顺手清了无引用的死样式 `.agent-mode-select`。
- README「工具权限档位」段落补文档；`@playwright/mcp` 的 browser_tabs（无 readOnlyHint、不在 auto_tools）是天然验收夹具。

### 验证

```powershell
.venv\Scripts\python.exe -m pytest backend/tests -q   # 604 passed（新增 test_permission_modes.py 9 项：confirm 压过 auto_tools/自主三路/全放/简历红线×full/提示词三档/PATCH 回环+422/落库与沿用；test_scope_expansion 的 start 替身补 permission_mode 形参）
npm run build                                          # 通过
```
- 运行副本与发布仓各跑一遍全量（均 604 passed）；发布仓 `ea6883a` 经 `feat/chat-permission-mode` 合入 `ddc440b` 已推送。
- 浏览器实测（本机 8765 + 真实模型 + @playwright/mcp）：存量 11 会话迁移 autonomous；full 档问"调用 browser_tabs"→ 直接执行（时间线 `@playwright/mcp / browser_tabs 5957ms completed`）无审批卡，会话落库 `permission_mode=full`；重开会话控件显示完全访问；切回自主再问 → 审批卡弹出且 reason 为新文案"该工具未声明只读、也未勾选自动执行"，拒绝路径收尾正常（卡片消失、时间线出现用户拒绝）；升档弹窗取消/确认、降档直达、确认档切换均符合预期。
- 流程修正：ResumeLoom 父仓库已把 `jobscope/` 整目录入 .gitignore（源码只在 JobScope 独立仓提交），本次 feat 分支建在 JobScope 仓；ResumeLoom 仅记本交付文档。

## 2026-11-12 补充：工具确认卡「本会话始终允许」授权

用户反馈自主档下 playwright 工具仍逐次确认（navigate/click "基本没风险"不该每次问）。排查：@playwright/mcp 给全部动作类工具标了 `destructiveHint: true`（点官方标注谨慎放宽），readOnlyHint 只有 snapshot/screenshot 等 8 个——所以按标注放行解决不了，真正对症的是用户点名的**会话内固定命令后续都允许**。

### 实现

- `agent_tool_grants` 表（conversation_id+server_id+tool_name 主键）：审批请求 `grant=true` 时在 `GeneralAgent.resume` 登记（批准即记，不依赖执行成败）；删会话路由顺带清理。
- 判定链优先级变为：**简历红线 > 会话授权 > 权限档位**。授权是显式 opt-in，confirm 档也认；跨会话不蔓延（主键含 conversation_id）。
- registry 增加 `destructive` 标记，审批 reason 分文案（destructive="可能修改外部数据或页面状态"）。
- 审批卡第三按钮「本会话始终允许」；想跨会话永久放行仍走 MCP 工具页「自动执行」勾选（文案已引导）。

### 验证

```powershell
.venv\Scripts\python.exe -m pytest backend/tests -q   # 609 passed（test_permission_modes.py 扩至 14 项：授权直跑/会话隔离/confirm 认授权/红线压授权/拒绝不记授权/destructive 文案）
npm run build                                          # 通过
```
- 真实 E2E（8765 + @playwright/mcp browser_tabs）：自主档发起调用 → 审批卡三按钮、reason 为 destructive 文案 → 点「本会话始终允许」→ 授权落库 → 同会话再调同工具时间线直接 completed、无 waiting 步骤。
- 发布仓库 `acf400c` 经 `feat/session-tool-grants` 合入 `3dd648b` 已推送。

## 2026-11-12 补充：MCP 常驻会话池（修有状态服务被逐次冷启动拆散）

用户实测 @playwright/mcp 多步搜索：第一次 navigate 打开的有头浏览器随即被关，后续 snapshot 读到空白页、反复重导航无效。诊断正确：不是授权逻辑关的——`MCPClient._request` 每次调用新建 `AsyncExitStack` 后销毁，stdio 子进程（连带它管理的浏览器）随调用结束退出，每次调用都是全新会话，多步操作天然无法串接。这问题一直存在，只是授权流让用户开始真实串接使用后暴露。

### 实现

- `MCPSessionPool` + `_MCPSession`（extensions.py）：按 `fingerprint(config)` 缓存常驻会话——每个会话一条 daemon 线程 + 常驻 asyncio 事件循环 + 长命 `AsyncExitStack`，stdio/HTTP/SSE 通用；调用经 `run_coroutine_threadsafe` 投递，会话内串行锁保证浏览器操作顺序。
- 回收：空闲 600s 惰性回收（`execute` 入口顺带 reap）；配置指纹变化自动重建；传输异常丢弃会话下次重建；FastAPI shutdown 钩子 `shutdown_all` 优雅关闭（aclose 10s 超时放弃，daemon 线程随进程退出）；实测 playwright Chrome 在 shutdown 后数秒内被收割。
- `MCPClient.call` 改**同步**（走池）；`discover`（测试并发现工具）保持一次性连接——它是显式管理操作，冷启动无副作用。`general_agent` 的 `anyio.run(MCPClient.call, ...)` 改直接调用。
- 业务错误（MCP 返回 isError 结果）不抛异常、不丢弃会话；只有传输类异常才重建。

### 测试适配与新增

- 4 个测试文件的 `MCPClient.call` 替身从 `async def` 改同步——原替身在同步调用下返回未 await 的 coroutine，导致 10 个审批/执行用例失败（首批全量回归暴露，`--lf` 逐个修复）。
- fixture_mcp 新增 `proc_pid` 工具（返回进程 pid）；`test_permission_modes.py` 扩至 16 项：pid 跨调用相同（进程复用）、配置指纹变化 pid 变化（重建）、空闲超时 pid 变化（回收）；真 stdio 用例补 `SESSION_POOL.shutdown_all()` 防孤儿进程。
- 全量回归：**611 passed**（运行副本与 JobScope 发布仓各跑一遍）。

### 真实验收

- 池层脚本（.venv 直调）：`browser_navigate(example.com)` → `browser_snapshot` 读到 "Example Domain" 页面正文——修复前 snapshot 拿到的是全新空白会话。
- UI 全链路（8765 + 真实模型 + @playwright/mcp）：自主档发"navigate 打开 example.com 再 snapshot 读取"→ navigate 首次弹审批卡（服务方标 destructive）→ 「本会话始终允许」授权 → completed；snapshot（readOnlyHint）自主档直跑 → completed 且读到第一次导航的页面；最终回答给出正确标题与正文第一句。多步操作完整串接。
- 发布仓库 `372942b` 经 `feat/mcp-session-pool` 合入 `966101b` 已推送。

## 2026-11-12 补充：放宽 agent 执行限额（工具 50 次）

用户反馈多步浏览器任务"没一会就中断"。原限额 8 轮/12 次工具/48000 字符/240 秒，playwright 任务一轮几个调用、每次 5-7 秒，很快触顶。四项配套放宽（只改一个会换个上限照样中断）：`MAX_CALLS` 12→50、`MAX_STEPS` 8→60、`MAX_CONTEXT` 48000→160000（每条结果仍截 12000）、240 秒时限常量化为 `MAX_SECONDS=900`。系统提示词限额文案按常量动态生成（f-string 段，注意那段原本是普通字符串）。`test_unknown_tools_rejected_and_loop_bounded` monkeypatch 限额回 8 验证循环有界性（否则 8 条脚本耗尽直接 IndexError）。全量 611 passed 两仓；发布仓库 `c6cce67` 经 `feat/raise-agent-limits` 合入 `5c94d01` 已推送。

## 2026-11-12 补充：采集能力接入 Agent 工具层（公司快查 + 岗位采集）

用户反馈：对话里问"查一下浙江的证优客"，Agent 因工具层没有采集类工具，拿 Playwright MCP 手搓浏览器查天眼查（Bing 空结果→爱企查登录墙→天眼查成功）——而项目快查（企查查/天眼查，走用户已登录的项目 Chrome 会话）与岗位采集管线早已存在，只是没暴露给 Agent。

- **管线下沉**：main.py 的采集编排闭包（persist_collection / execute_collection / run_collection / execute_lookup）抽到新模块 `backend/jobscope/collect_service.py`（repo/data_dir 显式传参、importlib 调用时按名加载），页面端点与 Agent 工具共用一条管线；main.py 四个闭包改 thin delegate，端点行为与响应形状不变。
- **两个内置工具**（BUILTINS，直接执行不走 MCP 审批流）：
  - `company_web_lookup(channel, company)`：复用快查管线，卡片照常落 `web_company_profile` 证据；范围纪律与 company_profile 相同（非 global 须 `require_company`）；只有过 `allows_evidence` 的证据经 `_remember` 登记进 `citable_evidence`，其余按外部信息转述防虚构引用；needs_login/needs_verification/backoff 状态原样透传不重试。
  - `collect_jobs(source, query, location, limit≤100)`：job 范围一律拒绝（读不了其他岗位）、company 范围 query 必须含公司名、global 放行；不满足直接 `out_of_scope` 不建任务防污染资料库；返回紧凑摘要（status/companies 前 10/sample 前 5 条四字段），全量岗位走 project_search/project_salary 检索。
- **提示词**：基础段补"工商主体优先 company_web_lookup、岗位数据先 collect_jobs 落库"；新增无条件 `_collection_guidance` 段（快查 vs 搜索 MCP 分工、状态透传纪律、citable_evidence 引用规则）；`_web_search_guidance` 把工商信息从搜索 MCP 首选场景改道到快查（保留全部既有断言短语，test_web_search_guidance 无需改）。
- **测试**：新增 `test_agent_collect_tools.py` 11 项（happy path 落库+登记、三种状态透传、company 范围越界 ScopeBoundaryError 且不建任务、job/company 范围拒绝、collect 摘要不含全量岗位、系统提示断言、/api/chat 端到端）；两仓全量 **624 passed**。
- **并发交付**：与"停止即时断流"会话同窗作业——对方 register_abort/MCP 可中断链在 general_agent.py 同文件混改动，用"HEAD+仅本次改动"脚本经 `hash-object + update-index` 隔离暂存（工作区文件原样留给对方），feat 提交后在独立 worktree 验证提交组合（621+1 环境性 dist 缺失）、再在临时 worktree 合并 main（`93e7d7d` → merge `4ba95de`），merge 后两特性共存 624 全绿；checkout 被并发未提交改动挡住时不强抢，用 symbolic-ref+reset 收尾。发布仓库 `8243e80` 经 `feat/agent-collect-tools` 合入 `4ba95de` 已推送。

## 2026-11-12 补充：对话顺序显示模式 + 系统设置页

用户反馈：对话输出是合并式（思维链聚一段、工具聚「执行过程 N 次调用」、正文压轴），顺序信息全丢；想要 AI IDE 式追加流（思维链被打断就换新卡片，工具/正文/思维链按发生顺序穿插），并要一个设置页改系统参数。

- **合并机制的根源与破解**：`useChatRun` 的 RunState 把 SSE 三类事件分别累进 `steps[]/answer/reasoning` 互不相关的字段，顺序在前端即丢失。SSE 事件本身天然按序到达（重挂整段重放），流式侧只需加一个保序 `segments` 数组；**落库侧**才是真缺口——`general_agent._loop` 每轮重置 `partial/reasoning`、只存最后一轮，历史消息要按序显示必须后端补存分段 timeline。
- **后端 timeline**：`state['timeline']` 全程记录——`_event()` 里每个 step 类事件追加 `{'kind':'step','event':…}`；`_loop` 包装 `on_delta/on_reasoning` 追加 text/reasoning 段（相邻同 kind 合并、每段 8000 字符封顶）；非流式路径（hooks=None）每轮结束后整轮 content 记一段；`_result` 带出、`chat_runs._persist` 落 `message.timeline`。审批暂停/续传经 `agent_pending` JSON 序列化天然保序（`resume` 对旧 pending 状态 setdefault 兼容）。旧消息无 timeline 前端自动回落合并布局。
- **系统设置**：`settings.py` 新增 `SystemSettingsStore`（`system_settings` KV 表）+ `GET/PUT /api/settings/system`；参数：`chat_display_mode`（merged/sequential，默认 merged 不动存量行为）、`show_reasoning`（原 localStorage `jobscope-reasoning` 开关迁移为服务端设置，composer 快捷开关改为写同一 store，跨浏览器一致）、`agent_limits`（轮数/工具次数/上下文/时长，原硬编码 MAX_* 常量改为从设置读取、保存时钳制 1-200/1-300/10k-1M/60-7200）。
- **前端**：新 `systemSettingsStore.ts` 模块级共享 store（fetch 一次 + subscribe + 乐观更新）；`RunState.segments` 保序维护；新 `ChatSegments.tsx`（左侧轨道线、流式时仅最后一段 live 展开、更早段自动折叠=「打断换卡」）；`ReasoningBlock` 抽独立文件防循环依赖（注意 Windows 大小写不敏感：store 文件名 `systemSettingsStore.ts` 与页面 `SystemSettings.tsx` 不能仅差大小写）；流式气泡与历史消息（`ChatWorkspace`/`ChatMessageItem`）按 displayMode 双分支。新「系统设置」页挂侧边栏工作区管理组。
- **测试**：新增 5 项——settings 默认值/钳制/重启持久化/非法枚举 422、非流式 timeline 分段、限额走设置（轮数 1 提前收尾）、流式 timeline 保序（reasoning→agent_model→project_search→reasoning→text→agent_model）、审批暂停续传 timeline 不丢序；原 monkeypatch MAX_STEPS 的循环有界测试改走设置端点。
- **并发窗口**：另一会话同时在运行副本做公司横评（compare）半成品（main.py 路由、api.ts/App.tsx/styles.css/CompanyCompare.tsx、reliability 测试 3 红属其进行中状态）。同步策略：10 个纯我的文件整 cp；main.py/api.ts/App.tsx/styles.css 四个混改文件在 worktree 里对 HEAD 版外科式打我的补丁；独立 worktree `feat/chat-sequential-mode` 上后端 **632 passed** + npm build 通过；快进合 main（`08bc63f→88eb61f`）、删分支、移除 worktree、已推送。
- **验收提示**：8765 旧实例须重启才能出 `/api/settings/system` 与 timeline 落库；切换入口在侧边栏「工作区管理 → 系统设置」。

## 2026-11-12 补充：页面加载提示 + 接口提速（loading 态全覆盖与后端批量化）

用户反馈：切换页面时部分接口响应慢，无任何「加载中」提示，看起来像没数据；顺带提升加载速度。

- **前端 loading 态**（复用 `.loading-state`，不引骨架屏）：公司资料库列表（首载「正在读取来源档案…」）、模型设置（channels 初值改 `null`——「正在读取渠道配置…」/「正在读取功能绑定…」，原先在途请求被渲染成"还没有渠道"或空白面板）、会话列表（`sessionsLoading` 透传 SessionList）、求职参考（`reliabilityLoading`——慢请求不再被误读成「还没生成过」）、横评五图表（layers 在途「正在读取岗位构成…」、reports 在途「正在读取参考报告…」）、sources 页目录/任务表（复用首屏 `loading`）。筛选/翻页时旧数据保留并整体变暗禁点（`.refreshing` 类 + CSS opacity/pointer-events，busy==='filter'/'page-jobs' 驱动）。
- **EvidenceCard 按需全文**：列表端点裁剪 `text` 至 600 字符（`text_truncated` 标记）+ 后端基于全文算 `text_hit` 命中窗口（-60/+200，对齐前端 excerpt——命中常在长报告中段，裁剪后本地找不到）；`<details>` 展开瞬间按需 GET `/api/evidence/{id}` 取全文。资料库/岗位证据 tab/公司情报三处统一。
- **后端提速**：① `model_configuration` 中间件仅对 `/api/` 前缀组装配置（静态资源免每请求 5-8 连接税），`effective()` 单连接一次读齐 model_settings/model_bindings/llm_channels 三表（建表 DDL 挪到 `ModelSettings.__init__`，resolve_purpose 闭包化删除）；② `jobs()`/`page('jobs')` 批量回填 evidence_ids（单连接 `IN` 查询），消灭每行一新连接的 N+1——compare/details-status/reliability local 全链受益；③ 列表裁剪：`/api/jobs` description 截 600（前端列表卡不渲染正文）、evidence 系列端点传 `trim_text`；④ `compare_page` jobs_count 改一条 GROUP BY；⑤ `details_status` 全量岗位只取一次（`pending_jobs` 加可选 jobs 参数）；⑥ research index GET 加 `needs_sync()` 只读探测，稳态不进 `BEGIN IMMEDIATE` 写事务（DDL 加实例级就绪标记跳过）；⑦ `list_conversations` 用 `json_array_length`/`json_each` 在库侧算 preview/count，不再整段 json.loads messages；⑧ 五条二级索引（evidence.company、jobs.updated_at、company_reliability(company_id,created_at)、company_signals.company_id、conversations(scope,updated_at)，放 `_migrate_conversations` 之后防列未建）。
- **踩坑**：effective() 先做过 TTL+invalidate 缓存，`test_missing_channel_falls_back_to_legacy_row` 直接删库行绕过 API 打脸——缓存失效覆盖不了外部直写，改单连接方案后无此问题且同样去掉了固定税。
- **并发手术**：运行副本有「渠道模型元数据」未交付并发线（channels.py 的 MODEL_METADATA、ModelSettings 渠道编辑器 step 化改版、api.ts 的 Channel.note/ModelMetadata 类型、styles.css 大段渠道编辑样式）——整 cp 会把引用不存在字段的半成品带进 main（JobScope 侧 test_channels 立红）。处置：main.py 两处外来 hunk（MODEL_METADATA import、from exc）回退成 main 版；ModelSettings.tsx/styles.css/api.ts 三文件 `git checkout main --` 恢复后重放本次小改动。channels.py 本身不带（并发会话交付责任）。
- **验证**：运行副本与 JobScope 两边后端全量 **638 passed**；前端双仓 build 通过；8766 新实例实测：evidence 裁剪（50 条仅 1 条截断）、text_hit 命中窗口、`/api/evidence/{id}` 全文、compare companies 模式 0.05s、details/status 0.03s、research index 稳态 `updated=0` 直读、conversations SQL 化 preview 正常；Playwright 冒烟 8 项（资料库列表/无空态误报/截断证据展开取回全文 605 字符/渠道区不误报空/横评页/任务表/岗位列表）通过。
- 发布仓库 `c25f78e` 经 `feat/loading-and-perf` 合入 `13e188d` 已推送；feat 分支已删。8765 当前实例（Miniconda，23:29 起）恰已加载新代码 + 新 dist，无需再重启。

## 2026-11-13 补充：求职参考操作按钮改浅白文案 + 悬浮说明

用户反馈：「用已有材料算覆盖度」「从材料里抽取信号」两个按钮是底层术语，用户只认功能。

- **改名**：覆盖度 → 「盘点已有资料」（用已存岗位/材料本地算一遍，报告哪些维度有依据、哪些缺）；抽取 → 「提取资料数字」（从补充的公告资料里读出明确数字补进打分依据）。缺口行的快捷按钮（原「本地重算」「从材料抽取」）同步改成同一套名。
- **悬浮提示**：新增 `ACTION_INFO`（name+tip）统一表，底部两按钮与缺口行按钮共用同一份 `title`；「补资料」也补了说明。沿用项目原生 title 惯例（同 CompanyCompare.tsx）。
- **并发注意**：运行副本 App.tsx 含并发会话未交付的「简历工坊」（ResumeWorkshop）半成品——该文件只能外科式 sed 打标签一行，勿整 cp。
- **验证**：运行副本 tsc + build 通过、dist bundle 含新文案且旧文案清零；发布仓同 build 通过。
- 发布仓库 `a250fd1` 经 `feat/reference-button-labels` 合入 `f80b24c` 已推送；feat 分支已删。8765 伺服 frontend/dist 静态文件，**无需重启，浏览器硬刷新**即见新文案。

## 2026-11-13 补充：简历工坊（JD 定向优化 + 模板预览 + 多格式导出）

用户需求：针对不同公司要求定向优化简历；优化结果可 MD 预览与二次编辑；内置常用模板；导出 PDF 等常用格式。

- **后端 `resume_workshop.py`**（新模块）：3 个内置模板（classic 经典衬线 / modern 现代无衬线 / compact 紧凑单栏，仅 CSS 差异）；渲染管线统一为 md→HTML（Markdown 库 tables 扩展）+ 模板 CSS，iframe 预览 / HTML 导出 / PDF 导出同源所见即所得。PDF 用 **pymupdf Story**（MuPDF 自带 Droid Sans Fallback 中文渲染，spike 实测无 tofu；DocumentWriter 直写 BytesIO 后 `subset_fonts()` + deflate，3.6MB→~40KB）。DOCX 用 python-docx + 标准库 html.parser 轻遍历（表格先缓冲网格再一次性建表——`_Row` 无 add_cell 是坑）。requirements 补 `Markdown`、`python-docx`（.venv 经 Miniconda system-site-packages 本就有）。
- **优化端点** `POST /api/resume-versions/optimize`：analysis purpose（`complete()`，temp .3 / 8000 tokens / 180s）；system prompt 红线——只重组已有事实，严禁虚构经历/技能/数字，JD 有而简历没有的不得写入；输出协议 `<<<RESUME>>>`/`<<<NOTES>>>` 分隔符（缺失容错：全文当简历、notes 空，过短 502）。成功即落库为版本（含 JD 快照 jd_digest、基础简历快照 base_text、notes）。
- **数据层**：`resume_versions` 表（幂等 CREATE）+ repository CRUD；端点组：模板列表/preview、版本 CRUD、`/export?format=pdf|docx|html|md`（Content-Disposition filename* 中文）。未配置 analysis 模型 → 400 引导去模型设置。
- **前端 `ResumeWorkshop.tsx`**：三栏一屏（版本列表 / 优化配置+MD 编辑 / 模板 chips + iframe srcDoc 实时预览（防抖 800ms、sandbox=""）+ 导出按钮组）。优化表单：基础简历=我的简历或粘贴；目标岗位=本地档案 typeahead（listing 摘要标注提醒）或粘贴 JD；补充指示。导出前自动保存未存修改。岗位详情「评估适配」旁新增「优化简历」入口（workshopPrefill 预填跳转）。App.tsx 8 处外科编辑、styles.css 纯追加。
- **测试** `test_resume_workshop.py` 6 例：模板/preview、CRUD、优化落库+红线文案断言进 system、岗位联动与守卫、无模型/坏回复容错、四格式导出（PDF 回提中文、docx 打开验段落表格）。
- **并发手术**：本次撞上 JobScope main 连续两次前进（13e188d loading-perf、f80b24c reference-labels，后者还会动 App.tsx 一行）。运行副本与 main 双向分叉（运行副本含未交付 company_merge_proposals 半成品；main 删了 MODEL_METADATA 等被运行副本持有）——全部走 worktree + **以 main 为底重放**：3 新文件整 cp，models/api/styles/requirements 纯追加，main.py/repository.py/App.tsx 锚点重放（worktree CRLF 检出须先归一化 LF 再匹配锚点，requirements.txt 首轮就因此静默漏改）。rebase 到 f80b24c 无冲突二次 build 复核通过。
- **验证**：运行副本全量 **644 passed**；worktree 上 **643 passed + 1 环境性失败**（test_index_shell 无 frontend/dist 属产物缺失非代码）；双处 npm build 通过；8765 重启（Miniconda 同参数）后实测四格式导出 200（PDF 44.9KB / DOCX 37.1KB / HTML / MD），联调数据已清。
- 发布仓库 `4511f9a` 经 `feat/resume-workshop` 合入 main 已推送；feat 分支与 worktree 已删。上一节提到的「简历工坊半成品」即本次交付，已转正。

## 2026-11-13 补充：岗位收藏功能（收藏夹 + 反查 + 收藏内横评，Phase 1）

用户需求：收藏岗位 → 反查岗位最新情况 → 收藏内做横评 → 直达源帖；后期再做平台侧收藏（本项目浏览器登录态内 JS 注入/模拟点击/调 API），本轮仅文档预留不实现。

- **存储**：`starred`/`starred_at` 进 jobs payload。关键坑：`save_job` 重采集 upsert 整包覆盖 payload——收藏标记加入幸存继承块（repository.py，无条件从 previous 继承），否则一次日常刷新就静默清空全部收藏。`update_jobs` 白名单加 starred（置 True 盖 starred_at、置 False 清空）；`_job_filters`/`page`/`job_ids` 加 `starred` 过滤（`json_extract('$.starred')=1`，只认显式 true）。
- **API**：GET /api/jobs 与 /api/jobs/ids 加 `starred` 查询参数；PATCH /api/jobs 模型加 `starred: bool | None`（exclude_none 保留显式 false）；DetailBatchInput 加 `ids`（≤200，min_length=1）——批量反查与完整JD补采共用任务管道，`run_batch` ids 模式池子按指定 id 且**不限 complete 状态**（反查就是要重读已完整岗位），任务标题区分「收藏岗位反查/完整JD批量补采」。
- **前端**：`Icon.tsx`（从 App 抽出防循环导入，加 star 图标）、`JobCard.tsx`（档案页/收藏夹共用卡片：勾选框+卡体+第三列星标+actions/footer 插槽）、`FavoritesView.tsx`（轻列表：q 筛选、本页全选、批量反查=批量任务+2s 轮询、横评选中公司=公司去重逐个 addCompareCompany 满 8 如实提示、取消收藏、单条反查、卡底反查时间）。档案页：「只看收藏」开关（立即生效）、选择条收藏/取消收藏、详情标题栏收藏按钮、侧栏收藏入口+角标（`GET /jobs?starred=true&page_size=1` 轻请求）。点收藏卡跳岗位研究复用完整详情工作台。styles.css 只追加（卡片第三列网格、星标金色、收藏面板）。
- **踩坑**：① `.job-archive-filters input` 通配 40px/100% 规则把收藏开关复选框撑大——追加同特异性规则收回 16px；② 冒烟脚本首版依赖「首条未收藏」前置状态，跑第二遍即翻转——改为开头 API 复位+状态断言（toast 只加分不断言）；③ 同秒 updated_at 并列时 run_batch 池子顺序不保证——测试断言改无序比较。
- **并发撞车实录**：发布仓在本次交付窗口被并发会话推进三个提交（workshop `4511f9a`、横评门槛 `9c1973a`、实体合并 `74b437b`，后者 10:05 提交恰在冒烟前 2 分钟）。models.py 的 starred/ids 被 74b437b 的 cp 同步**顺带扫进 main**（diff 为 0 直接跳过）；api.ts/main.py 仍是混改（渠道元数据 MODEL_METADATA 并发线在运行副本未交付）——main.py/api.ts 外科打我的 hunks，styles.css 只追加收藏块；App.tsx 因两侧都已带工坊且都无 CompanyMerge 集成，核实 diff 全归属本改动后**整 cp**。8765 伺服静态 dist 且并发会话恰好重启过服务，新后端已生效，无需再重启。
- **测试**：新 `test_job_favorites.py` 6 项（重采集幸存/PATCH 开关与时间戳/过滤/ids 池重读已完整岗位/ids 上限与缺失/端点校验）；运行副本与 JobScope 双仓后端全量 **658 passed**；双仓 tsc+build 通过；Playwright 冒烟 7 步 ×2 轮全过（侧栏入口/星标/只看收藏/详情收藏态/收藏夹卡片与操作行/横评选中公司/取消收藏空态+服务端复核），零控制台错误。
- 发布仓库 `36b9341` 经 `feat/job-favorites` 合入 `c5010c2` 已推送；feat 分支已删。**Phase 2 预留**（平台侧收藏，未实现）：复用 browser.py 会话纪律（port/profile/session_lock），走 `FETCH_JS` in-page fetch(credentials:'include') 调平台收藏接口或 `run_js` 模拟点击，任务走 create_task+轮询；数据字段（如 `platform_favorited`）到时再加。
