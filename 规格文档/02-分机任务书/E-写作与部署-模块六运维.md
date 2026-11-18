# 机器 E：写作辅助与部署（模块六 + 运维）

> **本机负责最后闭环：模块六 + Docker 部署 + 在线演示保活。**
> 在线演示是赛道四**必交项**，评审期必须可用——这是本机的头等责任。

---

## 一、任务清单

| # | 任务 | 优先级 | 模块 |
|---|---|---|---|
| E1 | **Docker Compose 全栈编排** | **P0** | 运维 |
| E2 | **在线演示部署 + 保活** | **P0** | 运维 |
| E3 | `.env.example` + 依赖清单 | **P0** | 交付物 |
| E4 | 论文框架生成（无幻觉 References） | P1 | 六 |
| E5 | 拓扑图脚本生成（TikZ/Mermaid/Graphviz） | P1 | 六 |
| E6 | CSV → 图表 + Figure Caption | P1 | 六 |
| E7 | 学术翻译 | P2 | 六 |
| E8 | 健康检查与监控 | P1 | 运维 |
| E9 | README + 复现指南 | **P0** | 交付物 |

---

## 二、E1：Docker Compose（**可复现性 3% 的分数在这**）

### 目录

```
deploy/
├── docker-compose.yml
├── .env.example
├── backend.Dockerfile
├── frontend.Dockerfile
└── nginx.conf
```

### docker-compose.yml

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: research_agent
      POSTGRES_USER: ${DB_USER:-app}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-app_pass}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ../db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-app}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

  backend:
    build:
      context: ..
      dockerfile: deploy/backend.Dockerfile
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/research_agent
      REDIS_URL: redis://redis:6379/0
      LLM_API_KEY: ${LLM_API_KEY}
      LLM_BASE_URL: ${LLM_BASE_URL}
      EMBED_MODEL: BAAI/bge-m3
      CHROMA_DIR: /data/chroma
    volumes:
      - chromadata:/data/chroma
      - ./pdfs:/data/pdfs:ro        # 官方测试集挂载
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8000:8000"

  frontend:
    build:
      context: ..
      dockerfile: deploy/frontend.Dockerfile
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  pgdata:
  redisdata:
  chromadata:
```

### backend.Dockerfile

```dockerfile
FROM python:3.11-slim

# OCR 依赖（PaddleOCR 需要）
RUN apt-get update && apt-get install -y \
    libgl1 libglib2.0-0 poppler-utils \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY backend/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 预下载 embedding 模型（重要：避免首次请求超时）
RUN python -c "from sentence_transformers import SentenceTransformer; \
    SentenceTransformer('BAAI/bge-m3')"

COPY backend/ .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### .env.example（**提交时必须存在**）

```bash
# ============ 大模型 ============
LLM_PROVIDER=deepseek
LLM_API_KEY=                    # ★ 必填，切勿提交真实密钥
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_MODEL=deepseek-chat
LLM_TIMEOUT=8

BACKUP_LLM_PROVIDER=qwen
BACKUP_LLM_API_KEY=
BACKUP_LLM_BASE_URL=

# ============ 数据库 ============
DB_USER=app
DB_PASSWORD=change_me_in_production
DATABASE_URL=postgresql://app:change_me@localhost:5432/research_agent
REDIS_URL=redis://localhost:6379/0

# ============ 向量库 ============
VECTOR_STORE=chroma             # chroma | milvus
CHROMA_DIR=./data/chroma
EMBED_MODEL=BAAI/bge-m3
RERANK_MODEL=BAAI/bge-reranker-large

# ============ 外部 API ============
ARXIV_RATE_LIMIT=3              # 秒
GITHUB_TOKEN=                   # 可选，提高速率
SEMANTIC_SCHOLAR_KEY=           # 可选

# ============ 检索阈值（Guardrails）============
SCORE_REFUSE=0.45               # 低于此值拒答
SCORE_HEDGE=0.62                # 低于此值降级

# ============ 演示账号（评审用）============
DEMO_USER=demo
DEMO_PASSWORD=chuanzhi2026
```

---

## 三、E2：在线演示部署（**必交，评审期需保活**）

### 部署要求

| 项 | 要求 |
|---|---|
| 公网可访问 | 域名或 IP 均可 |
| **评审期可用** | 至少持续到 **2027-02 底**（评审意见 1 月底发布） |
| 测试账号 | `/login` 页显示演示账号，或直接在 README 写明 |
| HTTPS | 建议配置（部分浏览器限制） |

### 低成本方案

| 方案 | 成本 | 说明 |
|---|---|---|
| 国内轻量云（腾讯/阿里） | ~50 元/月 | 2C4G 起步，需备案（麻烦） |
| 海外 VPS（Vultr/DigitalOcean） | ~$6/月 | 免备案，访问稳定 |
| Railway / Render | 免费档 | 会休眠，**不适合评审期保活** |

> ⚠️ **务必现在就买、现在就部署**，别等到 11-18。备案通常要 7-20 天。

### 保活监控

```bash
# cron 每 5 分钟探活，失败告警
*/5 * * * * curl -sf https://your-domain/api/v1/health || \
  echo "DOWN" | mail -s "Research Agent Down" you@example.com
```

**加一个简单的心跳页**（评委也会看到）：

```
GET /api/v1/health
{
  "status": "UP", "db": true, "vector": true,
  "llm": "ok", "ocr": true,
  "uptime_hours": 128, "papers_indexed": 42
}
```

### 降级预案（重要）

> 评委访问时**最怕 AI 挂了页面白屏**。

**必须实现**：
- LLM 挂了 → 走纯检索模式（只列原文片段），页面仍可用
- OCR 挂了 → 提示"扫描件解析暂不可用"，其余功能正常
- 向量库挂了 → 走关键词检索兜底

**并且首页放一个"演示模式"开关**：预置 3 个成功案例（去重、溯源问答、图谱），**保证即使全部外部依赖挂掉，评委仍能看到核心效果**。

```javascript
// 预置演示数据，不依赖任何外部 API
const DEMO_CASES = [
  { id: 'case1', title: '语义去重：v1 vs v7', ... },
  { id: 'case2', title: '抗幻觉问答 + 原文锚点', ... },
  { id: 'case3', title: '20 篇引用图谱与基石挖掘', ... },
]
```

> 💡 **这一条能救命。** 赛题"完整性与材料质量"以及在线演示可用性都看这个。

---

## 四、E4：无幻觉论文框架生成

```python
def generate_outline(idea: str, paper_ids: list[str]):
    # 1. 检索相关文献作为素材
    context = retrieve_for_outline(idea, paper_ids)

    # 2. 生成
    sections = llm.generate_outline(idea, context)

    # 3. ★ 关键：校验每条 References
    verified, unverifiable = [], []
    for ref in sections["references"]:
        if ref.get("anchor_id") and verify_anchor(ref["anchor_id"]):
            verified.append({**ref, "verified": True})
        else:
            # 无法溯源的引用 → 显式标注，绝不伪装
            unverifiable.append({**ref, "verified": False})

    return {
        "abstract": sections["abstract"],
        "sections": sections["sections"],
        "references": verified,
        "unverifiable_references": unverifiable   # ★ 必须暴露
    }
```

**Prompt 要点**：

```
生成论文框架时，参考文献只能从【已入库文献】中选择。
严禁虚构任何文献标题、作者或年份。
如果某个引用找不到原文依据，请将其放入 unverifiable 列表并说明原因。
```

> ⚠️ **`unverifiable_references` 非空时，前端必须显式提示**：
> "以下 2 条引用无法在库中验证，请自行核实。"

---

## 五、E5：拓扑图生成

```python
def generate_diagram(desc: str, dtype: str) -> str:
    prompts = {
        "mermaid":  "生成 Mermaid graph TD 语法...",
        "tikz":     "生成 TikZ picture 环境...",
        "graphviz": "生成 DOT 语法...",
        "matplotlib":"生成 Python matplotlib 代码...",
    }
    return llm.generate(prompts[dtype] + f"\n\n描述：{desc}")
```

**校验**：生成后用对应工具解析一遍，语法错误就让 LLM 修一次：

```python
def validate_and_fix(code: str, dtype: str) -> str:
    try:
        validate(code, dtype)
        return code
    except SyntaxError as e:
        return llm.fix(code, str(e), dtype)
```

---

## 六、E6：CSV → 图表 + Caption

```python
@app.post("/viz/from-csv")
async def viz_from_csv(file: UploadFile, chart_type: str):
    df = pd.read_csv(file.file)

    # 1. 生成 ECharts option
    spec = build_echarts_option(df, chart_type)

    # 2. ★ Caption 必须提炼结论，不是描述图表
    caption = llm.generate(
        f"""根据以下数据生成学术规范的中文图注。
要求：一句话说明图表展示什么，一句话给出核心趋势结论（带关键数值）。
不要写"这是一张折线图"这类无信息量的话。

数据摘要：{df.describe().to_string()}
列名：{list(df.columns)}""")

    return {"chart_spec": spec, "caption": caption,
            "table": {"columns": list(df.columns),
                      "rows": df.head(100).values.tolist()}}
```

**Caption 好/坏示例**：

- ❌ 差："这是一张展示不同模型 BLEU 分数的柱状图。"
- ✅ 好："图 1. 各模型在 WMT14 EN-DE 上的 BLEU 分数对比。Transformer 以 28.4 显著领先，较 LSTM 基线提升 2.8 个点。"

---

## 七、E9：交付物清单（**最后核对**）

### 代码仓库（Public）

```
├── README.md                    ← E9，含 10 分钟复现指南
├── .env.example                 ← E3，必须
├── requirements.txt / pnpm-lock.yaml
├── docker-compose.yml
├── db/init.sql
├── backend/
├── frontend/
├── deploy/
└── docs/
    ├── 架构图.png
    └── prompts/                 ← AI 协同记录（建议）
```

### README 必备章节

```markdown
# 科研助手 Agent

## 项目简介
## 核心特性（对应六大模块）
## 技术架构（架构图）
## 快速开始（10 分钟复现）
  1. 克隆仓库
  2. 复制 .env.example 为 .env，填入 LLM_API_KEY
  3. docker compose up -d
  4. 访问 http://localhost
## 演示账号
## 官方测试用例验证结果（表格）
## 团队信息
## 许可证
```

### 提交材料清单（**逐项打勾**）

- [ ] 代码仓库 URL（**Public**）
- [ ] 演示视频 MP4 ≥1080p，5–8 分钟
- [ ] 技术文档 PDF ≤30 页
- [ ] **在线演示地址**（公网可访问）
- [ ] **评审测试账号**（写在 README + 录像里演示）
- [ ] 确认仓库**无 API 密钥**（`git log -p | grep -E "sk-|pat_"`）

---

## 八、验收标准

- [ ] `docker compose up -d` 后 3 分钟内全部服务 UP
- [ ] 全新机器 clone + 复制 .env + compose up → 能跑（**这是"可复现性"的实测**）
- [ ] `/api/v1/health` 返回全绿
- [ ] 公网地址可访问，演示账号能登录
- [ ] **断网测试**：拔掉 LLM API → 页面不白屏，走纯检索
- [ ] CSV 上传 → 生成图表 + Caption 有实质结论
- [ ] Mermaid/TikZ 脚本通过语法校验

---

## 九、依赖

| 关系 | 说明 |
|---|---|
| **可先做** | **E1/E2/E3 不依赖任何人，今天就该开工** |
| 等 A/B/C/D | E4–E7 需要后端接口就绪 |

> **强烈建议：E1 今天先搭起来**，哪怕后端是空的 `GET /health`。
> 这样其他机器可以随时部署自己的分支联调，最后不会在部署上卡住。
