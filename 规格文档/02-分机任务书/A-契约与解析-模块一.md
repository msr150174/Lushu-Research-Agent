# 机器 A：契约与解析（模块一）

> **本机是项目起点，产出阻塞其他所有机器。优先完成契约确认。**
> 负责：数据模型落地、PDF 解析、语义去重

---

## 一、任务清单

| # | 任务 | 优先级 | 交付物 |
|---|---|---|---|
| A1 | **确认并冻结 JSON Schema**（首个产出，通知全员） | **P0** | `docs/schema.json` |
| A2 | 建库建表 + 初始数据 | **P0** | `db/init.sql` |
| A3 | PDF 上传接口 | **P0** | `POST /papers/upload` |
| A4 | 矢量 PDF 解析（PyMuPDF） | **P0** | `parse/pdf_parser.py` |
| A5 | **锚点生成器** | **P0** | `parse/anchor_builder.py` |
| A6 | **语义去重**（官方用例 3.1） | **P0** | `POST /papers/duplicate-check` |
| A7 | 章节树提取 | P1 | `sections` 表填充 |
| A8 | 扫描件 OCR（PaddleOCR） | P1 | `parse/ocr_parser.py` |
| A9 | 参考文献解析 | P1 | `paper_refs` 表填充 |
| A10 | 版本差异对比 | P1 | `GET /papers/{a}/diff/{b}` |
| A11 | 长文档分级解析 | P1 | 按需加载 |
| A12 | 批量上传 + 任务队列 | P2 | `POST /papers/batch-upload` |

---

## 二、技术要点

### A1：先冻结 Schema（**今天必须完成**）

向所有人公布解析输出的 JSON 结构：

```json
{
  "paper_id": "p_0001",
  "title": "...",
  "authors": [{"name": "", "affiliation": ""}],
  "abstract": "...",
  "arxiv_id": "1706.03762",
  "version": 7,
  "published_at": "2023-08-02",
  "page_count": 15,
  "content_hash": "a3f9c1...",
  "is_scanned": false,
  "sections": [{"section_id":"sec-3.1","level":2,"title":"...","page_start":7,"page_end":8}],
  "anchors": [{"anchor_id":"an_012","page":7,"bbox":[120,340,480,392],
               "char_span":[1240,1386],"text":"...","text_sha256":"...",
               "block_type":"paragraph","is_ocr":false,"confidence":1.0}],
  "references": [{"ref_index":11,"raw":"...","citation_quality":"substantive"}]
}
```

> **B/C/D 三台机器都等这个结构开工。你有 2 小时。**
> 结构可以先"够用"，但**字段名一旦公布不得再改**。

### A4：矢量 PDF 解析

```python
import fitz  # PyMuPDF

doc = fitz.open(path)
for pno in range(doc.page_count):
    page = doc[pno]
    d = page.get_text("dict")
    for block in d["blocks"]:
        if block["type"] != 0:      # 0=文本, 1=图片
            continue
        for line in block["lines"]:
            for span in line["spans"]:
                # span["text"], span["bbox"], span["size"], span["font"]
                pass
```

**关键处理**：

| 情况 | 处理 |
|---|---|
| 双栏排版 | 按 `bbox` 的 x 中位数把 block 分左右栏，左栏先读 |
| 公式 | `span["font"]` 含 `CMMI`/`CMSY`/`CMEX` → `block_type = "formula"` |
| 表格 | 用 `page.find_tables()` → `block_type = "table"` |
| 参考文献区 | 检测 `References` / `Bibliography` 标题后进入特殊模式 |
| 页眉页脚 | 首页/末页 5% 高度内的重复文本剔除 |

### A6：语义去重（官方用例 3.1，**必须做对**）

```
1. 存入 v7  → content_hash = H(M(v7正文))
2. 上传 v1  → content_hash = H(M(v1正文))
3. 相似度 = SimHash/Jaccard(M(v1), M(v7))
4. 若 ≥ 0.90 且 arxiv_id 相同
     → is_duplicate = true
     → match_basis = "content"          ★ 不是 filename！
     → relation: v1 < v7  → "older_version"
     → suggestion: "库中已有更新版本（v7），建议保留现有版本"
```

**规范化函数**（两边都要用同一个）：

```python
def normalize_for_hash(text: str) -> str:
    t = re.sub(r'\s+', '', text)        # 去所有空白
    t = re.sub(r'\[\d+\]', '', t)       # 去引用编号
    t = re.sub(r'\d+', '#', t)          # 数字归一（页码/行号差异）
    return t.lower()
```

> ⚠️ **绝对不能用 `arxiv_id` 或文件名判重**。
> v1 和 v7 的 `arxiv_id` 都是 `1706.03762`，用 arxiv_id 判重会把它们当"同一份"直接覆盖，
> 就**演示不出"语义去重"了**，这个用例等于没做。

### A8：扫描件 OCR

```python
# 渲染 300 DPI
pix = page.get_pixmap(dpi=300)
img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)

# PaddleOCR
result = ocr.ocr(np.array(img), cls=True)
# result 输出坐标是图片像素 → 换算回 PDF 点
# PDF 点 = 像素 * 72 / 300
```

**置信度必须如实写**：`confidence = 识别置信度`，`is_ocr = True`。

### A11：长文档（541 页）

```python
# 禁止：一次性解析全部页面
# 正确：
def ensure_page_parsed(paper_id, page_range):
    if paper.parsed_pages.issuperset(page_range):
        return
    for p in page_range:
        parse_single_page(paper_id, p)   # 增量存
        paper.parsed_pages.add(p)
    save(paper)
```

首次只解析目录页（TOC）+ 前 20 页。其余按需。

> **内存红线**：单篇解析峰值 < 500MB。超过就往增量+落盘改。

---

## 三、验收标准

- [ ] `db/init.sql` 可重复执行，`IF NOT EXISTS` 无报错
- [ ] 上传 `1706.03762v7.pdf` → `parse_status = done`，`anchors` 数量 > 200
- [ ] 先传 v7 再传 v1 → `is_duplicate = true`，`match_basis = "content"`
- [ ] 任一锚点的 `text` 重新算 sha256 == 库里的 `text_sha256`
- [ ] `1706.03762v1_img.pdf` → `is_scanned = true`，锚点带 `confidence < 1.0`
- [ ] 541 页 UDL 首次打开响应 < 5s，内存不崩
- [ ] `GET /papers/{id}` 返回结构与 `docs/schema.json` 完全一致

---

## 四、依赖与其他机器

| 关系 | 说明 |
|---|---|
| **A → B** | B 依赖 A 的 `anchors` 与 `chunks` 结构 |
| **A → C** | C 依赖 A 的 `GET /papers/{id}/layout` |
| **A → D** | D 的图谱依赖 A 解析出的 `references` |
| **A 无前置依赖** | **可以立刻开工** |

> **开工顺序：A 的 A1（Schema 冻结）→ 全员并行。**
