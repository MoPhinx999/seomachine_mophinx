# SEO Machine 重建方案（窗帘品牌 thehues · 上游选题/写作引擎）

> 本文是「大刀阔斧改动」的正式方案文档。目标读者：项目所有者（eugenia）。
> 过程沟通用中文，最终博客成品用英文、面向美国消费者。

---

## 0. 一句话定位

把 seomachine 从"通用 SEO 内容生成器"重建为一个**感知存量内容的、模仿 AI 搜索扇出机制的「上游选题 + 初稿引擎」**：
每天从种子扇出大量子问题 → 校验真实需求 → 反查 ~200 篇存量博客 → 决策"新写 / 改旧 / 跳过+内链" → 写出懂窗帘、贴美国人、无 AI 味的英文初稿（Markdown）→ 交给已有的 `blog_workflow_manager` 配图上传。

**北极星**：每天低人工地稳定产出"值得写的选题 + 能直接审的高质量初稿"。

---

## 1. 系统边界（最重要的认知校正）

下游 `E:\GitHub\python_script\seo_tools\blog_workflow_manager` 已经是一条成熟的 7 步发布管线，**不要重造**。
而 `markdown_importer.py` 已经为 seomachine 预留了接口契约。两边分工固定如下：

```
┌─ seomachine（本 repo）= 上游的脑 ────────────────────────────────┐
│  种子: 热点/季节趋势 + 核心关键词/产品主题 + (手动大题)            │
│    ↓ ① 扇出   模型像 AI search 那样把种子发散成大量子问题/子主题   │
│    ↓ ② 校验   DataForSEO 验证哪些子问题真有搜索量/PAA/相关需求     │
│    ↓ ③ 反查   对照 ~200 篇存量博客索引，判断是否已被回答/答得好    │
│    ↓ ④ 决策   分流：WRITE_NEW / IMPROVE_EXISTING / SKIP_LINK     │
│    ↓ ⑤ 写作   懂窗帘 + 贴美国人 + 无AI味 + GEO/AEO 结构           │
│    └→ 产出: drafts/*.md（带契约 frontmatter）+ 当日选题决策报告    │
└─────────────────────────────────────────────────────────────────┘
                         ↓ 人工审（选题+初稿） ↓
┌─ blog_workflow_manager = 下游的手（已存在）─────────────────────┐
│  markdown_importer 读 MD → Step3 配图 → 上传 Shopify              │
│  （Step4 内链 / Step5 Meta T/D / Step6 产品匹配 默认关闭，         │
│    因为上游已在 frontmatter 提供）                                 │
└─────────────────────────────────────────────────────────────────┘
```

**重叠的那一点（上游也要做）**：Meta Title / Meta Description / URL Slug / 内链建议 / 外链建议——写进 MD frontmatter，下游直接采用。

---

## 2. 初稿交付契约（Markdown，必须严格遵守）

依据下游 `markdown_importer.py` 实际解析逻辑，初稿格式如下。元数据块**不强制在文件顶部**（解析器会全文找），但建议放顶部，用 `---` 包裹：

```markdown
---
Title: Best Blackout Curtains for Nurseries (2026 Buyer's Guide)
Meta Title: Best Blackout Curtains for Nurseries | thehues
Meta Description: How to pick blackout curtains that actually darken a nursery...
Primary Keyword: blackout curtains for nursery
Secondary Keywords: nursery window treatments, baby room darkening, ...
URL Slug: blackout-curtains-for-nursery
Internal Links:
  - /blogs/news/how-to-measure-curtains
  - /blogs/news/linen-vs-polyester-curtains
External Links:
  - https://www.sleepfoundation.org/...
Word Count: 1800
Generated: 2026-06-18
---

# Best Blackout Curtains for Nurseries (2026 Buyer's Guide)

正文从 H1 开始……（下游会把 H1 提为 title、首段提为 summary、其余转 HTML）
```

契约要点（来自源码）：
- 识别为"文章"的触发字段：`Meta Title / Meta Description / URL Slug / Primary Keyword` 任一存在即可。
- `URL Slug` 取最后一段作为 `handle`（也是你要的"用 handle 命名"）。
- `Internal Links / External Links` 用 `- ` 列表；下游存为 `{"provided": [...]}` 并默认关闭自己的加链接步骤。
- 正文第一个 `# H1` 会被当作标题并从正文移除；第一段 `<p>` 作为 summary。
- **图片不要在初稿里塞**——配图是下游 Step3 的职责。

> 建议：文件名直接用 `URL Slug`（handle）命名，例如 `drafts/blackout-curtains-for-nursery.md`，与存量博客索引的 handle 命名一致。

---

## 3. 新的每日引擎（5 个阶段）

| 阶段 | 输入 | 处理 | 输出 |
|---|---|---|---|
| ① 种子 Seed | 热点采集 + 核心关键词/产品主题表 + 可选手动大题 | 汇总当日种子清单 | `output/seeds/YYYY-MM-DD.json` |
| ② 扇出 Fan-out | 种子 | 模型对每个种子发散 N 个子问题/长尾意图（模仿 AI 搜索 query fan-out） | `output/fanout/YYYY-MM-DD.json` |
| ③ 校验 Validate | 扇出子问题 | DataForSEO 拉搜索量/PAA/相关查询，过滤掉无需求的"车轱辘"子题 | 带数据的子题清单 |
| ④ 反查决策 Decide | 校验后子题 + 存量博客索引 | 对每个子题检索最相关旧文，判定 已答好/答不清/没答，分流 | `output/decisions/YYYY-MM-DD.json` |
| ⑤ 写作 Draft | WRITE_NEW / IMPROVE 的子题 | 窗帘专业 + 美国消费者视角写正文 → 质量门 → 输出 MD 契约 | `drafts/*.md` + `output/topic_report/YYYY-MM-DD.md` |

④ 的决策规则：

```
对每个 (校验通过的) 子问题:
  related = 在存量索引里检索 top-k 最相关旧文（标题+正文语义/关键词匹配）
  if 没有相关旧文:                      → WRITE_NEW（新写）
  elif 有旧文但没真正回答/答得浅:        → IMPROVE_EXISTING（标注旧文 handle + 缺口）
  else 旧文已答好:                       → SKIP_LINK（跳过，记为内链候选）
```

`SKIP_LINK` 的产物不是浪费——它们自动成为 WRITE_NEW 初稿 frontmatter 里的 `Internal Links` 候选。

人工审在 ⑤ 之后：你看"当日选题决策报告 + 初稿"，确认后再丢进下游。

---

## 4. 存量博客的数据层

**来源**：用你现有的 `get_shopify_data.py` → 选 `4. 博客文章 (Articles)`，导出 Excel（含 `handle / title / content_html / seo_title / seo_description / tags / url`）。这是反查引擎最理想的单一数据源。

**建议管线**（新建一个轻量索引器，放本 repo）：
1. 读 Articles Excel → 清洗 `content_html` 为纯文本（复用 `content_scrubber.py`）。
2. 抽取每篇的"它回答了哪些问题"（标题 + H2/H3 + 关键段落）。
3. 建一个本地索引文件 `corpus/index.json`（按 handle 命名），字段：`handle, title, url, questions_answered[], keywords[], summary, word_count, last_updated`。
4. 反查时：先关键词/BM25 粗筛，再让模型对 top-k 做"是否真正回答了这个子问题"的判定（答好/答浅/没答）。

> 200 篇规模不需要向量库；关键词粗筛 + 模型精判足够准且简单。后续要扩可再上 embedding。
> 索引重建按需触发（你导出新 Excel 后跑一次），不必每日。

---

## 5. 质量系统（直击你最不能忍的 4 点）

你的质量痛点 → 对应机制：

| 痛点 | 机制 |
|---|---|
| 太 AI 味/不像人写 | 复用并强化 `humanizer` skill + `topic-reviewer` 的 AI-language 检测，作为写作后的强制门 |
| 车轱辘话/不帮用户 | ③ 校验阶段先砍掉无搜索需求的子题；写作阶段强制"每段必须回答一个具体问题/给一个可执行结论" |
| 不懂窗帘/不专业 | 新建 `context/curtains-expertise.md`（选购/尺寸/面料/遮光/安装/清洗等行业事实库），写作时强制引用 |
| 不贴美国消费者 | 新建/重写 `context/us-consumer-profile.md`（场景、术语、单位、季节、搜索习惯），扇出与写作都以它为锚 |

GEO/AEO：写作结构默认"问题前置 + 直接答案 + 结构化（清单/表格/FAQ）"，便于被 ChatGPT/Google AI 引用。`ethan-smith-perspective` 作为选题优先级的策略视角保留。

---

## 6. 清理清单：删 / 留 / 改

### 🔴 删除（方向错 / 重复下游 / 与窗帘日更无关）
- `commands/publish-draft.md`（WordPress 发布——你是 Shopify 且走下游管线）
- `commands/landing-*.md`（5 个）+ `agents/landing-page-optimizer.md`（落地页不是日更博客使命）
- `data_sources/modules/wordpress_publisher.py` + `wordpress/` Yoast 插件
- 落地页/CRO 相关 Python：`landing_page_scorer / landing_performance / above_fold_analyzer / cta_analyzer / cro_checker / trust_signal_analyzer`
- 通用营销技能库里与日更无关的（约 18 个）：`paid-ads / pricing-strategy / referral-program / email-sequence / popup-cro / paywall-upgrade-cro / signup-flow-cro / onboarding-cro / launch-strategy / form-cro / page-cro / marketing-psychology / marketing-ideas / free-tool-strategy / ab-test-setup / analytics-tracking / competitor-alternatives / product-marketing-context`

### 🟡 改造（核心，重组进新引擎）
- `daily-topic-workflow` → 重建为「种子→扇出→校验→反查→决策→写作」编排器（不再是纯热点）
- `hotspot-collector` → 降级为**种子来源之一**（热点/季节）
- `topic-generator` → 改造成**扇出 + 决策引擎**（吃存量索引 + DataForSEO）
- `topic-reviewer` → 保留为**质量门**，扩充窗帘专业度 + 美国消费者校验
- `commands/write.md` / `article.md` → 改造成输出 §2 契约的窗帘写手
- `agents/`：`seo-optimizer / editor / headline-generator / content-analyzer` → 合并为"写作 + 质量门"，砍掉与下游重复的部分
- `agents/internal-linker / meta-creator / keyword-mapper` → **不独立保留**，逻辑内化进写手（只产出 frontmatter 的链接/Meta 建议）
- `commands/rewrite.md` → 改造为 IMPROVE_EXISTING 的执行入口

### 🟢 保留（有用，基本不动或轻改）
- `humanizer`、`ethan-smith-perspective`（质量与策略视角）
- Python：`dataforseo.py / search_intent_analyzer / keyword_analyzer / content_length_comparator / readability_scorer / opportunity_scorer / content_scrubber`
- `context/` 大部分（品牌、风格、内链图），新增窗帘专业库 + 美国消费者画像
- GA4/GSC 模块：保留但非日更主链（用于事后表现复盘）

---

## 7. 目标 repo 结构（重建后）

```
seomachine/
├── CLAUDE.md                      # 重写：反映新定位与边界
├── REBUILD-PLAN.md                # 本文
├── .claude/
│   ├── commands/
│   │   ├── daily.md               # 主入口：跑整条日更引擎
│   │   ├── write.md               # 单篇：按契约写一篇初稿
│   │   ├── improve.md             # 改进既有旧文（IMPROVE_EXISTING）
│   │   └── index-corpus.md        # 重建存量博客索引
│   └── agents/
│       ├── fanout-strategist.md   # 扇出
│       ├── gap-analyst.md         # 反查+决策
│       ├── curtains-writer.md     # 写手（懂窗帘/贴美国人/出契约MD）
│       └── quality-gate.md        # 质量门（AI味/专业度/价值）
├── corpus/
│   ├── articles_export.xlsx       # get_shopify_data 导出的存量博客
│   └── index.json                 # 反查索引（按 handle）
├── context/
│   ├── curtains-expertise.md      # 新增：窗帘行业事实库
│   ├── us-consumer-profile.md     # 新增：美国消费者画像
│   ├── brand-voice.md / style-guide.md / seo-guidelines.md ...（保留）
├── data_sources/modules/          # 精简后的 Python（见 §6）
├── output/                        # 引擎中间产物（seeds/fanout/decisions/report）
└── drafts/                        # 交付给下游的 *.md（按 handle 命名）
```

---

## 8. 实施分期

- **Phase 0 — 瘦身**：执行 §6 删除清单，repo 先变干净（低风险，先做）。
- **Phase 1 — 数据层**：写 corpus 索引器（读 Articles Excel → `index.json`）；接 DataForSEO 校验。
- **Phase 2 — 引擎骨架**：实现 `daily.md` 编排 + 4 个新 agent，先用少量种子跑通"扇出→校验→反查→决策"，产出决策报告（暂不写正文）。
- **Phase 3 — 写手 + 契约**：实现 `curtains-writer` 输出 §2 契约 MD，端到端跑一篇，丢进下游 `markdown_importer` 验证能正常导入+配图+上传。
- **Phase 4 — 质量门 + 调优**：接 `quality-gate` + `humanizer`，补 `curtains-expertise.md` / `us-consumer-profile.md`，按你审稿反馈迭代标准。
- **Phase 5 — 自动化**：把 `daily` 定时化（可选），稳定后逐步放大每日产量。

---

## 9. 待你拍板的开放问题

1. **每日产量**：每天扇出后最终写几篇初稿？（建议先 1–3 篇跑稳）
2. **种子主题表**：核心关键词/产品主题清单由你给，还是我从你的 collections/产品（`get_shopify_data` 也能导）自动生成初版？
3. **改进旧文(IMPROVE)的产物**：是直接生成"改写后的整篇 MD"覆盖式交付，还是只给"缺口清单 + 补丁段落"让你手动并入？
4. **删除是否需要先备份**：Phase 0 删的东西要不要先 `git` 建分支留存，以防以后想捡回 landing/通用技能。

---

*下一步：你确认本方案（尤其 §6 删除清单和 §9 的 4 个问题）后，我从 Phase 0 开始动手。*
