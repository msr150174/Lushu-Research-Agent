# 机器 D：引用图谱与 arXiv 追踪（模块四 + 五）

> **本机承担创新性 25% 中的 10%（图谱挖掘）+ 8%（技术方案创新）。**
> 负责：引用图谱、图论挖掘、Future Work 去重、arXiv 检索追踪

---

## 一、任务清单

| # | 任务 | 优先级 | 模块 |
|---|---|---|---|
| D1 | arXiv API 检索 | P1 | 四 |
| D2 | 参考文献自动抓取入库 | P2 | 四 |
| D3 | arXiv 定时追踪 | P2 | 四 |
| D4 | GitHub 开源代码关联 | P1 | 四 |
| D5 | 个性化推荐 + 理由 | P1 | 四 |
| D6 | **引用关系解析** | **P0** | 五 |
| D7 | **图谱构建 + 图论挖掘** | **P0** | 五 |
| D8 | 图谱可视化数据接口 | **P0** | 五 |
| D9 | 自动综述生成 | P1 | 五 |
| D10 | **Future Work 去重** | P1 | 五 |
| D11 | 引用含金量筛选 | P1 | 四 |

---

## 二、D6：引用关系解析（**图谱的地基**）

从 A 机器解析出的 `paper_refs` 出发，把"引用字符串"变成"指向库内论文的边"。

### 解析链路（三级匹配）

```python
def resolve_reference(raw: str, title: str, authors: list) -> str | None:
    # ① 精确匹配：已在库中
    if hit := db.match_by_raw(raw):
        return hit.paper_id

    # ② arXiv ID 提取：raw 里常含 arXiv:2210.03629
    if m := re.search(r'arXiv:\s*(\d{4}\.\d{4,5})', raw):
        return db.match_by_arxiv(m.group(1)) or fetch_and_store(m.group(1))

    # ③ 标题模糊匹配：Semantic Scholar / 本地
    if hit := s2.match_title(title):
        return hit.paper_id

    # ④ 未解析 → 保留为外部引用，图上画"虚节点"
    return None
```

> 💡 **虚节点也是加分点**：图谱中显示"库外引用"（灰色），
> 说明你的 Agent 知道"有哪些重要文献我还没入库"——这是诚实的学术态度。

---

## 三、D7：图论挖掘（**10% 的核心**）

### 基石论文识别

```python
import networkx as nx

def build_graph(paper_ids, edges):
    G = nx.DiGraph()
    G.add_nodes_from(paper_ids)
    for e in edges:
        G.add_edge(e.source_id, e.target_id, weight=e.weight)

    # PageRank：被引越多越重要
    pr = nx.pagerank(G, alpha=0.85)

    # 中介中心性：跨领域的桥梁
    bc = nx.betweenness_centrality(G)

    # 入度
    indeg = dict(G.in_degree())

    return G, pr, bc, indeg

def classify_role(pid, pr, bc, indeg):
    pr_rank = sorted(pr.values(), reverse=True).index(pr[pid]) / len(pr)
    indeg_median = statistics.median(indeg.values())

    # 基石：PageRank 前 20% 且入度显著高于中位数
    if pr_rank < 0.20 and indeg[pid] >= indeg_median * 2:
        return "keystone"
    # 边缘衍生：入度低
    if indeg[pid] <= indeg_median:
        return "peripheral"
    return "normal"
```

### 社区发现（聚类）

```python
from networkx.algorithms.community import greedy_modularity_communities
G_undirected = G.to_undirected()
clusters = greedy_modularity_communities(G_undirected)
# 每个节点分配 cluster 标签，前端按簇着色
```

### 引用含金量判定（D11）

**区分实质性引用与应付式引用**——赛题明写要求"自动剔除应付式引用"。

```python
PERFUNCTORY_PATTERNS = [
    r'(?i)\b(see|e\.g\.|etc\.)\b[^.]{0,40}\[\d+\]',   # "see [5]" 一句话带过
    r'(?i)(several|many|various)\s+(works|studies|papers)\s*\[[\d,\s]+\]',  # 罗列式
]

def citation_quality(context: str) -> str:
    """context 是该引用在正文中的上下文（前后 200 字）"""
    if not context or len(context.strip()) < 80:
        return "perfunctory"          # 没有上下文 = 应付式

    for p in PERFUNCTORY_PATTERNS:
        if re.search(p, context):
            return "perfunctory"

    # 实质引用特征：有方法对比、数据引用、明确的论述
    substantive_markers = ['compare', 'outperform', 'unlike', 'in contrast',
                           'following', 'based on', 'improve', 'achieve']
    if any(m in context.lower() for m in substantive_markers):
        return "substantive"

    return "perfunctory" if len(context) < 150 else "substantive"
```

> ⚠️ **图谱默认过滤 `perfunctory` 边**，但提供开关展示。这直接对应"引用含金量"评分点。

---

## 四、D10：Future Work 去重（**创新性加分项**）

赛题要求：**"自动排除已被后续论文解决的旧 idea"**。

### 实现思路

```
1. 提取每篇论文的 Future Work / Limitations 段落（按关键词定位）
2. 用 LLM 抽取"待解决问题"结构化条目
3. 对每条 idea 检索库内其他论文：
   - 若后续论文的 Abstract/Conclusion 中提到已解决 → 标 solved
4. 输出三个分组：solved / partially_solved / open
```

```python
FUTURE_WORK_PATTERNS = [
    r'(?i)(future work|future research|open problem|' +
    r'remains? (?:an )?open|we leave .{0,20} for future|' +
    r'limitation[s]? of (?:our|this) work)',
]

def extract_future_work(paper) -> list[dict]:
    snippets = []
    for anchor in paper.anchors:
        if re.search('|'.join(FUTURE_WORK_PATTERNS), anchor.text):
            snippets.append(anchor)

    ideas = []
    for s in snippets:
        ideas.extend(llm_extract_ideas(s.text, anchor_id=s.anchor_id))
    return ideas

def check_solved(idea, later_papers) -> dict:
    """later_papers: published_at > 原论文的库内论文"""
    for p in later_papers:
        verdict = llm.judge_solved(idea["idea"], p.abstract + p.conclusion)
        if verdict == "solved":
            return {"status": "solved", "solved_by": [p.paper_id],
                    "reason": verdict.evidence}
        if verdict == "partial":
            return {"status": "partially_solved", "solved_by": [p.paper_id]}
    return {"status": "open", "solved_by": []}
```

> 💡 **答辩时可演示**："这篇 2017 年论文提出的'降低注意力复杂度'，系统自动识别出已被 Linformer(2020) 解决——所以不再作为前沿方向推荐。"
> 这一句演示，足以说明系统做到了赛题最硬的要求。

---

## 五、D1/D4/D5：arXiv 检索与推荐

### arXiv API

```python
import urllib.parse, feedparser

ARXIV_API = "https://export.arxiv.org/api/query"

def arxiv_search(q: str, max_results: int = 20):
    params = {"search_query": f"all:{q}",
              "start": 0, "max_results": max_results,
              "sortBy": "relevance"}
    url = f"{ARXIV_API}?{urllib.parse.urlencode(params)}"
    feed = feedparser.parse(url)
    return [{
        "arxiv_id": e.id.split('/abs/')[-1],
        "title": e.title.strip().replace('\n', ' '),
        "abstract": e.summary.strip(),
        "authors": [a.name for a in e.authors],
        "published": e.published,
        "pdf_url": next(l.href for l in e.links if l.type == 'application/pdf')
    } for e in feed.entries]
```

> 📌 **坑位记录（继承路书的"坑"习惯）**：
> - arXiv API 有**速率限制**：官方建议 3 秒 1 次请求，连续请求会被 403
> - `arxiv.org/abs/xxx` 是 HTML 页，**PDF 直链是 `arxiv.org/pdf/xxxvN`**
> - 返回的 `id` 带版本号，如 `1706.03762v7`，**去重时要去掉 vN**

### GitHub 代码关联（**推荐加分**）

```python
def check_github(title: str) -> dict | None:
    # 用标题在 GitHub 搜索
    r = requests.get("https://api.github.com/search/repositories",
                     params={"q": title, "sort": "stars"},
                     headers={"Authorization": f"token {GITHUB_TOKEN}"})
    for repo in r.json().get("items", [])[:3]:
        if title.lower()[:30] in repo["name"].lower() or \
           title.lower()[:30] in (repo.get("description") or "").lower():
            return {"url": repo["html_url"], "stars": repo["stargazers_count"]}
    return None
```

**推荐排序规则**（赛题明确要求）：

```python
score = 0.5 * similarity \
      + 0.3 * (1 if has_github else 0) \
      + 0.1 * recency_score \
      + 0.1 * citation_quality_score
```

> "优先推荐附带开源 GitHub 代码（易复现）的论文"——**赛题原话，必须实现**。

---

## 六、图谱可视化数据格式

给前端（ECharts Graph）的直接可用格式：

```json
{
  "nodes": [
    { "id": "p_0001", "name": "Attention Is All You Need",
      "symbolSize": 48, "role": "keystone", "year": 2017,
      "pagerank": 0.182, "category": 0, "cluster": "transformer" }
  ],
  "edges": [
    { "source": "p_0002", "target": "p_0001", "value": 1,
      "lineStyle": { "opacity": 1 } }
  ],
  "categories": [
    { "name": "transformer" }, { "name": "bert" }
  ]
}
```

`symbolSize` 建议 = `20 + pagerank * 200`，让基石论文视觉上最大。

---

## 七、验收标准

- [ ] 20 篇论文构建图谱 → 前端渲染 < 2s，节点/边数正确
- [ ] 基石论文（如 Attention）被正确标为 `keystone` 且视觉最大
- [ ] 应付式引用（"see [5]"）默认被过滤
- [ ] Future Work 中"降低注意力复杂度"被标为 `solved` 并给出解决论文
- [ ] arXiv 检索返回 20 条，带 PDF 直链
- [ ] 推荐列表中带 GitHub 的论文排在前面
- [ ] 频繁请求 arXiv 不被封（加 3 秒间隔）

---

## 八、依赖

| 关系 | 说明 |
|---|---|
| **等 A** | 需要 `paper_refs`（参考文献解析结果） |
| **可先做** | **D1/D4 不依赖任何人，可立刻开工**；D7 图论算法可用构造数据先写 |

> **建议顺序**：D1（arXiv）→ D7 图论算法（用假数据先测）→ 等 A 的表 → D6 接线 → D10。
