# 机器 C：PDF 阅读器与联动翻译（模块三）★ 工作量最大

> **本机工作量为六模块之最，且与后端耦合最浅，可与 A/B 完全并行。**
> 负责：PDF 渲染、标题树、双向跳转、中英联动

---

## 一、任务清单（**先跑通再优化**）

| # | 任务 | 优先级 | 说明 |
|---|---|---|---|
| C1 | PDF.js 单页渲染 | **P0** | 先做单栏，别管双栏 |
| C2 | **锚点高亮定位** | **P0** | 点击引用 → 滚动 + 高亮 bbox |
| C3 | 标题树导航 | **P0** | 左侧目录树 |
| C4 | 引用角标 `[11]` 双向跳转 | **P0** | 正文 ↔ References |
| C5 | 公式渲染（KaTeX） | P1 | |
| C6 | 双栏排版还原 | P1 | |
| C7 | 划词翻译 | P1 | |
| C8 | **中英联动滚动** | P1 | 两侧 `scrollTop` 同步 |
| C9 | 整篇翻译 | P2 | |
| C10 | 悬停翻译气泡 | P2 | |

---

## 二、战略提醒（**务必先读**）

> **模块三最容易吃掉全部时间，而且拿分性价比不如模块二。**

**建议节奏**：
- **前 40% 时间**：C1 + C2 + C3 → 做出"能点引用跳到原文"的效果，这已经能演示
- **中 40% 时间**：C4 + C5 + C8 → 双向跳转 + 联动滚动，这是评分考察点
- **后 20% 时间**：C6 + C7 + C9 → 双栏、划词、整篇

**不要一开始就死磕双栏排版还原。** 单栏能演示、能高亮，就先交付。双栏属于"排版还原度高"（3% 里的部分），并非生死线。

---

## 三、技术实现

### 依赖

```json
{
  "pdfjs-dist": "^4.0.379",
  "vue": "^3.4.21",
  "element-plus": "^2.4.0",
  "katex": "^0.16.9",
  "echarts": "^5.4.0"
}
```

### C1：基础渲染

```vue
<script setup>
import * as pdfjsLib from 'pdfjs-dist'
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'

async function renderPage(pageNo) {
  const page = await pdfDoc.getPage(pageNo)
  const viewport = page.getViewport({ scale })
  const canvas = canvasRef.value
  canvas.width = viewport.width
  canvas.height = viewport.height
  await page.render({ canvasContext: canvas.getContext('2d'), viewport }).promise
}
</script>
```

### C2：锚点高亮（**核心**）

后端返回的 `bbox` 是 **PDF 坐标（左上原点，y 向下）**，与 PyMuPDF 一致。

```js
function bboxToRect(bbox, viewport) {
  const [x0, y0, x1, y1] = bbox
  const [vx0, vy0] = viewport.convertToViewportPoint(x0, y0)
  const [vx1, vy1] = viewport.convertToViewportPoint(x1, y1)
  return {
    left:   Math.min(vx0, vx1),
    top:    Math.min(vy0, vy1),
    width:  Math.abs(vx1 - vx0),
    height: Math.abs(vy1 - vy0)
  }
}

async function locateAnchor(anchor) {
  await renderPage(anchor.page)                    // 1. 渲染目标页
  const rect = bboxToRect(anchor.bbox, viewport)
  highlight.value = { ...rect, text: anchor.text } // 2. 画高亮层
  scrollIntoView(highlight.value)                  // 3. 滚到可视区
}
```

**高亮层**用绝对定位的 div 叠在 canvas 上：

```css
.anchor-highlight {
  position: absolute;
  background: rgba(47, 107, 255, .18);
  border-left: 3px solid #2F6BFF;
  pointer-events: none;
  transition: background .2s;
}
```

> ⚠️ **坐标方向务必统一为「左上原点、y 向下」**，与 PyMuPDF 和 PDF.js 一致。
> 若发现高亮位置上下颠倒，检查是否误用了 PDF 规范坐标系（左下原点）。

### C3：标题树

```js
// 后端 GET /papers/{id}/outline 返回
// [{ section_id, level, title, page_start }]
<el-tree :data="outline" @node-click="n => gotoPage(n.page_start)" />
```

### C4：引用双向跳转（考察点）

**正向：正文 `[11]` → References 第 11 条**

```js
async function jumpToReference(refIndex) {
  const ref = references.find(r => r.ref_index === refIndex)
  await locateAnchor(ref.ref_anchor_id)
  highlightListItem(refIndex)
}
```

**反向：References 第 11 条 → 正文所有 `[11]` 出现处**

需要后端在解析时记录**每个 `[n]` 在正文中的出现位置**（也是 anchor）：

```json
{
  "ref_index": 11,
  "occurrences": [
    { "anchor_id": "an_088", "page": 6 },
    { "anchor_id": "an_102", "page": 7 }
  ]
}
```

> 📌 **若 A 机器未提供 `occurrences`，立刻反馈**，这是本功能的前置数据。

### C6：双栏还原（P1，别太早做）

```js
// 后端返回 columns: [{x0,x1}, {x0,x1}]
// 前端按 block.bbox.x0 归到该栏，按 y 排序
function layoutColumns(blocks, columns) {
  return columns.map(col =>
    blocks.filter(b => b.bbox[0] >= col.x0 - 2 && b.bbox[2] <= col.x1 + 2)
          .sort((a, b) => a.bbox[1] - b.bbox[1])
  )
}
```

### C8：中英联动滚动（考察点）

```js
let syncing = false

function onScroll(source, target) {
  if (syncing) return
  syncing = true
  const ratio = source.scrollTop /
                (source.scrollHeight - source.clientHeight)
  target.scrollTop = ratio * (target.scrollHeight - target.clientHeight)
  requestAnimationFrame(() => { syncing = false })
}
```

**优雅处理**：拖动时有 100ms 防抖，松手后做一次"段落对齐"微调（按 anchor 段落序号对齐，比纯比例更准）。

```js
function alignByParagraph(source, target) {
  const visible = getTopmostVisibleAnchor(source)
  if (!visible) return
  const t = target.querySelector(`[data-anchor="${visible.anchor_id}"]`)
  if (t) t.scrollIntoView({ block: 'start', behavior: 'smooth' })
}
```

### C5：公式渲染

后端 `block_type = "formula"` 时返回 `latex` 字段：

```js
import katex from 'katex'
katex.render(block.latex, el, { throwOnError: false, displayMode: true })
```

### C7：划词翻译

```js
function onSelectionTranslate() {
  const sel = window.getSelection()
  const text = sel.toString().trim()
  if (!text) return
  // 找选区所属 anchor（用 range 与 bbox 交集判断）
  const anchor = findAnchorByRange(sel.getRangeAt(0))
  api.translate({ paper_id, scope: 'selection',
                  anchor_ids: [anchor.anchor_id], preserve_math: true })
}
```

> ⚠️ **`preserve_math: true` 时公式、变量名、代码必须保持英文**，这是赛题明写要求。

---

## 四、性能要求

| 场景 | 要求 |
|---|---|
| 单页渲染 | < 300ms |
| 锚点定位 | < 500ms |
| 541 页文档翻页 | 不卡顿（**按需渲染，勿全量加载**） |
| 内存 | 打开长文档 < 800MB |

**长文档关键**：**只渲染当前页 + 前后各 1 页缓存**，销毁远处的 canvas。

```js
// 虚拟化：只保留最近 3 页的 canvas
function evictFarPages(currentPage) {
  canvases.forEach((c, p) => {
    if (Math.abs(p - currentPage) > 1) c.remove()
  })
}
```

---

## 五、验收标准

- [ ] 打开 `1706.03762v1.pdf` → 正常显示，标题树可点
- [ ] 调 `locateAnchor` → 高亮框**准确覆盖目标文字**（不偏移、不上下颠倒）
- [ ] 点击 `[11]` → 跳到 References 第 11 条
- [ ] 点击 References 第 11 条 → 跳回正文
- [ ] 公式正常渲染（LaTeX 不是乱码）
- [ ] 中英双栏拖动 → 两侧同步滚动
- [ ] 541 页文档翻页不卡死，内存不超 800MB
- [ ] 划词翻译 → 公式保持英文

---

## 六、依赖

| 关系 | 说明 |
|---|---|
| **等 A** | 需要 `GET /papers/{id}/layout`、`outline`、`references` |
| **等 B** | 引用卡片数据格式（citation 事件） |
| **可先做** | **PDF.js 基础渲染不依赖任何人，可立刻开工** |

> **建议顺序**：先用任意公开 PDF 做 C1 渲染（不依赖后端）→ 后端就绪后接 C2/C3 → C4/C8。
