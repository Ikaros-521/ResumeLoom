# 简历 HTML 排版与 PDF 导出

用户要求制作正式简历、Web 转 PDF，或修复分页与留白时读取。输入是已审查的当前正文；输出为自包含 HTML、可选中文字的 PDF，以及实际导出和核验记录。用户指定 Word 等格式时遵从其要求。本指南依赖宿主提供浏览器与 PDF 工具，不内置或自动安装驱动。

## 排版选择

- 从当前正文生成语义化标题、段落和列表，转义用户文本。HTML 不包含事实 ID、审查说明或内部侧录。姓名、求职方向、联系方式、工作及项目经历有清楚的视觉层级。
- 用 A4 打印样式、明确的毫米页边距和可读的中文字体。正文可从 10–11pt、1.3–1.5 倍行高开始调节；实际效果以最终 PDF 为准。优先使用本机字体或本地资源，避免导出依赖远程字体、统计脚本或图片服务。
- 页数由内容量与可读性决定。一页能清楚容纳就用一页；两页合理时按完整项目和章节分段。不要预设必须两页或按总高度平均分成两半，导致每页底部大面积留白。也不要通过整页缩放或极小字号强塞一页。
- 留白过多先检查强制分页、重复间距、页边距与联系方式占行；再调整版面。若需删减文案，返回正文编辑流程并更新侧录，不能只在 HTML 中悄悄删内容。
- 用 `@page { size: A4; margin: 0; }` 配合页容器内边距，或使用 `@page` 页边距，两种方式避免重复叠加。采用自然分页时，标题避免孤立页尾，项目尽量不拆；过长的块允许跨页，不能全局禁止分页而造成溢出。
- 打印时隐藏按钮、背景和预览阴影。可保留“打印 / 保存 PDF”按钮供用户使用。没有必要为单页显示页码；多页页码应准确。

## 浏览器导出

使用宿主已提供的浏览器工具或本地 Playwright/Chromium；按实际环境发现可执行文件及依赖，不在可复用指令中写死本机路径。HTML 如使用分页脚本，应暴露明确的就绪条件；仅等待页面加载不足以保证排版结束。

以下是宿主已能解析 `playwright` 包时的最小 CommonJS 示例，可保存为运行目录内的临时 `.cjs`，以 HTML 和 PDF 的路径作为参数。若 Playwright 默认浏览器不存在，可将已发现的本地 Chromium/Chrome 路径传入 `RESUME_CHROMIUM_PATH`；不要直接照搬报错中的浏览器安装命令。不把候选人信息写入脚本；只有实际执行成功才能报告已导出。

```javascript
const { chromium } = require('playwright');
const { pathToFileURL } = require('node:url');
const path = require('node:path');

(async () => {
  const [htmlPath, pdfPath] = process.argv.slice(2);
  if (!htmlPath || !pdfPath) throw new Error('Usage: export.cjs input.html output.pdf');
  const browser = await chromium.launch({
    headless: true,
    ...(process.env.RESUME_CHROMIUM_PATH
      ? { executablePath: process.env.RESUME_CHROMIUM_PATH } : {})
  });
  try {
    const page = await browser.newPage();
    const errors = [];
    page.on('pageerror', error => errors.push(error.message));
    await page.goto(pathToFileURL(path.resolve(htmlPath)).href, { waitUntil: 'load' });
    await page.emulateMedia({ media: 'print' });
    await page.evaluate(async () => {
      await document.fonts.ready;
      await Promise.all([...document.images].map(img => img.decode()));
    });
    // 使用自定义分页脚本时，在这里等待其真实就绪标志。
    if (errors.length) throw new Error(errors.join('\n'));
    await page.pdf({
      path: path.resolve(pdfPath), format: 'A4', preferCSSPageSize: true,
      printBackground: true, displayHeaderFooter: false, tagged: true
    });
  } finally {
    await browser.close();
  }
})().catch(error => { console.error(error); process.exitCode = 1; });
```

手动打印时选择 A4、100% 比例，关闭浏览器自带网址和日期页眉页脚，确认预览再保存。无法实际导出时交付 HTML 和操作步骤，明确 PDF 尚未生成；不得伪造 PDF 路径或核验结果。

## 实际文件核验

1. 打开生成的 PDF，检查实际页数、A4 尺寸和是否可选中文字。记录工具及当前附件版本。
2. 使用 `pypdf`、`pdfplumber` 或宿主等价工具实际提取最终 PDF 文本，按阅读顺序逐段对照正文。核对中文姓名、邮箱、数字、日期和链接；合理忽略换行、列表符号及排版空格，不能忽略丢字、错字或顺序错乱。发现异常可用第二种提取工具交叉检查；修复后重新核验。
3. 用 Poppler `pdftoppm` 或宿主等价工具把最终 PDF **所有页**渲染成图片并查看。检查字体缺字、重叠、截断、孤立标题、空白页及不合理的大段留白。浏览器截图只用于辅助，不能证明 PDF 打印版式正确。
4. 若改过 HTML/CSS，重新导出并检查最新 PDF；不能拿前一版图片或提取结果背书。只修版面保持正文不变；发生内容差异则同步正文、侧录并重新审查。
5. 归档旧附件，记录正文版本及实际摘要、HTML/PDF 版本与路径、工具、页数和上述核验结果。全文、侧录和附件一同交回 audit 保存新输入快照；附件审查通过后，才允许 `attachment_preparation`。

交付最终 PDF 与 HTML 的可用链接。说明实际页数及已做的核验；本地文字提取和视觉通过不代表任何招聘平台 ATS 通过，也不授予上传或提交权限。
