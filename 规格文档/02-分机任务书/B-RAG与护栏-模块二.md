# 机器 B：RAG 与 Guardrails（模块二）★ 最高优先级

> **本机承担全项目最大的分值池：抗幻觉 RAG 8% + Guardrails 5% = 13%，且直接决定其他维度印象。**
> 负责：检索管道、溯源问答、护栏体系

---

## 一、任务清单

| # | 任务 | 优先级 | 交付物 |
|---|---|---|---|
| B1 | **移植路书治理层** | **P0** | `agent/guardrails/policy.py` |
| B2 | 向量化流水线 | **P0** | `rag/indexer.py` |
| B3 | 混合检索（Dense + Sparse） | **P0** | `rag/retriever.py` |
| B4 | **Rerank** | P1 | `rag/reranker.py` |
| B5 | **SSE 流式问答** | **P0** | `POST /chat/completions` |
| B6 | **引用锚点输出** | **P0** | 见 `03-溯源锚点规范.md` |
| B7 | **引用校验接口** | **P0** | `POST /chat/verify-citation` |
| B8 | 无覆盖协议 | **P0** | 触发 3001 话术 |
| B9 | 三层降级链 | P1 | `agent/fallback_chain.py` |
| B10 | 多文献对比推理 | P1 | `scope.mode = multi` |
| B11 | 冲突检测 | P1 | `guardrail: conflict` |
| B12 | 会话管理 | P2 | sessions / messages |

---

## 二、B1：移植路书治理层（**本机最重要的一件事**）

路书知识库头部写着三条铁律，**逐条代码化**：

### 铁律 1：「事实只来自知识库，没有就答"现有资料未覆盖"」

```python
# backend/app/agent/guardrails/policy.py

class NotCoveredPolicy:
    """迁移自 路书 notCoveredPolicy"""

    THRESHOLD_REFUSE = 0.45     # 低于此值直接拒答
    THRESHOLD_HEDGE  = 0.62     # 低于此值降级回答

    @staticmethod
    def evaluate(retrieved: list[dict]) -> dict:
        if not retrieved:
            return {"triggered": True, "type": "no_source",
                    "action": "refuse",
                    "message": "现有资料未覆盖该问题。"}

        top_score = max(r["score"] for r in retrieved)

        if top_score < NotCoveredPolicy.THRESHOLD_REFUSE:
            return {"triggered": True, "type": "no_source",
                    "action": "refuse",
                    "message": "现有资料未覆盖该问题。"}

        if top_score < NotCoveredPolicy.THRESHOLD_HEDGE:
            return {"triggered": True, "type": "low_confidence",
                    "action": "hedge",
                    "message": "以下内容基于相似度较低的片段，仅供参考："}

        return {"triggered": False}
```

### 铁律 2：「把二手内容标成原文会被判数据造假」

来源分级由**数据自动判定**，不允许人工指定。见 `03-溯源锚点规范.md` §六。

### 铁律 3：「每个结论必须能点回 PDF 的页 + 坐标」

见 B6。

### 额外迁移自路书的设计

| 路书资产 | 新实现 |
|---|---|
| `data_gaps`（缺口登记） | 每次拒答写一条 `unanswered_queries` |
| SUPPORT 层最高优先级 | **SAFETY 层**：先查 `retraction_flags`，命中则 block |
| 「不承诺录取与薪资」 | 「不虚构引文、不编造数据」 |

> **写进技术文档的表述**：
> "团队将垂直知识库场景验证过的『无覆盖协议』迁移至科研场景，并扩展了撤稿拦截、冲突检测、引用哈希校验三层机制。"

---

## 三、B2：向量化（注意分级）

```python
# 分级索引，长文档不至于爆内存
def index_paper(paper):
    # 1. 文档级（上传即建）
    index_chunk(level="doc", text=f"{paper.title}\n{paper.abstract}")

    # 2. 章节级（上传即建）
    for sec in paper.sections:
        index_chunk(level="section", text=sec.full_text)

    # 3. 段落级（懒加载！）
    #    ⚠️ 541 页长书不能全建，首次访问该章时再建
    for sec in paper.sections:
        if sec.accessed:
            for anchor in sec.anchors:
                index_chunk(level="chunk", anchor_id=anchor.anchor_id,
                            text=anchor.text)
```

**Embedding**：`BGE-M3`（多语言，中英通吃）

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("BAAI/bge-m3")
```

**向量库**：Chroma，collection `paper_chunks`

```python
metadata = {
    "paper_id": ..., "anchor_id": ...,   # ★ 必须带 anchor_id
    "level": "chunk", "page": 7, "section_id": "sec-3.1"
}
```

> ⚠️ **每个 chunk 必须挂 `anchor_id`**，否则无法溯源，等于白做。

---

## 四、B3：混合检索

```python
def hybrid_retrieve(query, paper_ids, top_k=8):
    # Dense：向量相似度
    dense = chroma.query(query_emb, n_results=top_k*2)

    # Sparse：BM25（关键词精确匹配，抗幻觉靠它兜底）
    # 注意：学术术语如 "β_1"、"BLEU"、"P100" 关键词匹配必须准
    sparse = bm25.search(query, top_k=top_k*2)

    # 融合：RRF (Reciprocal Rank Fusion)
    fused = rrf_fusion(dense, sparse)

    # 过滤：已撤稿论文
    fused = [f for f in fused if not is_retracted(f.paper_id)]

    return fused[:top_k]

def rrf_fusion(*result_lists, k=60):
    scores = {}
    for results in result_lists:
        for rank, item in enumerate(results):
            scores[item.id] = scores.get(item.id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

> **为什么 Sparse 在本项目特别重要**：
> "β_1 / β_2 设置"、"8 NVIDIA P100"、"BLEU 28.4" 这类**细粒度数值**，
> 纯向量检索经常检索不到，关键词匹配才靠得住。这是**官方明确考察点**。

---

## 五、B5 + B6：流式问答 + 锚点输出

```python
async def stream_chat(question, scope):
    # 1. 检索
    yields("retrieval", {"chunks": [...]})

    # 2. Guardrails 前置检查
    verdict = NotCoveredPolicy.evaluate(retrieved)
    if verdict["triggered"] and verdict["action"] == "refuse":
        yields("guardrail", verdict)
        yield_text(verdict["message"])
        yields("done", {"has_citation": False, "reason": "no_source"})
        log_unanswered(question)      # 写 unanswered_queries
        return

    # 3. 撤稿拦截（SAFETY 层最高优先级）
    for r in retrieved:
        if is_retracted(r.paper_id):
            yields("guardrail", {"type": "retracted", "action": "block"})
            return

    # 4. 流式生成（带 citation 指令）
    prompt = build_prompt(question, retrieved)
    async for delta in llm.stream(prompt):
        yield_text(delta)

    # 5. 提取引用并校验
    citations = extract_citations(full_answer, retrieved)
    for c in citations:
        valid = verify_hash(c["anchor_id"], c["quote"])
        if not valid:
            yields("guardrail", {"type": "citation_unverified",
                                 "action": "invalidate"})
            log_to_call_logs(...)
            return          # ★ 整条答案作废

    yields("done", {"has_citation": True})
```

### Prompt 模板（**这是 5% Guardrails 的一部分**）

```
你是学术文献助手。严格遵守以下规则：

1. 只能依据下方【参考资料】回答，不得使用你自己的知识。
2. 每个事实性陈述后必须紧跟引用标记 [n]，n 是参考资料编号。
3. 引用内容必须逐字摘自参考资料，不得改写。
4. 如果参考资料不足以回答，直接输出：「现有资料未覆盖该问题。」
5. 不确定的数值（如超参数、GPU 型号）必须精确引用原文，禁止猜测。
6. 输出格式：Markdown，数字与单位保留原文写法。

【参考资料】
[1] (p.7, §3.1) "We trained on 8 NVIDIA P100 GPUs for 12 hours."
[2] (p.7, §3.2) "..."
```

---

## 六、B7：引用校验（**技术壁垒**）

见 `01-核心规格/03-溯源锚点规范.md` §三。

```python
@app.post("/chat/verify-citation")
def verify_citation(anchor_id: str, quoted_text: str):
    anchor = db.get_anchor(anchor_id)
    recomputed = anchor_hash(quoted_text)
    valid = (recomputed == anchor.text_sha256)
    return {
        "code": 0, "method": "ok",
        "data": {
            "valid": valid,
            "page": anchor.page,
            "bbox": anchor.bbox,
            "hash_match": valid
        }
    }
```

---

## 七、B9：三层降级链

迁移自 `智能无感/AiService.java`：

```python
async def generate(prompt):
    # ① 主模型
    try:
        return await primary_llm.stream(prompt, timeout=8)
    except Exception:
        pass

    # ② 备用模型
    try:
        return await backup_llm.stream(prompt, timeout=8)
    except Exception:
        pass

    # ③ 纯检索模式：不生成新文字，只列原文片段
    return retrieval_only_response()
```

> **第 ③ 层绝不调用任何模型**，只把检索到的片段原样列出。
> 符合"宁可不答，不可瞎答"。

---

## 八、验收标准（六条全过才算达标）

- [ ] 问"Attention 用了多少块 GPU" → 答案带锚点，点击能定位到第 7 页高亮
- [ ] 问论文中不存在的"作者是否提到量子计算" → 答"现有资料未覆盖"
- [ ] 篡改 `quoted_text` 调校验 → `valid: false`
- [ ] 上传撤稿论文后引用它 → 被 `retracted` 拦截
- [ ] 扫描件引用带 `is_ocr: true` 徽章
- [ ] 断开大模型 API → 降级到纯检索，**不报错**

---

## 九、依赖

| 关系 | 说明 |
|---|---|
| **等 A** | 需要 `anchors` 表和 `chunks` 结构 |
| **给 C** | C 的引用卡片渲染依赖 B 输出的 citation 事件格式 |
| 可先做 | **B1 治理层不依赖任何人，可立刻开工** |

> **建议顺序：B1（治理层）→ 等 A 的 schema → B2/B3 → B5/B6/B7。**
