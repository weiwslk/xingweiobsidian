# 企业落地 / 一人公司 · Stage B 选型打样 · 过关手册

> 本册对应「企业落地 / 一人公司」路径的核心阶段：B1 用一张决策矩阵在知识库 / 流程 / 运维三个业务场景里选定一个先落地，B2 用课程对应项目当脚手架，把选定的那一个打样成本机可 demo 的 MVP。
> 练习向。题目从三个场景的真实项目源码（README / AGENTS.md）、rag-advanced-pack 能力包与课程课件出，端口、字段名、数字逐字核对。回看锚统一写成 `business-landing#节点id`。
> 用法：先自己作答，再翻参考答案。面试自测每题带一层追问，务必连追问一起答。

---

## 一、随堂练习

### B1 · 场景选型决策（回看锚 business-landing#B1）

**练习 1｜三个候选场景各自的样例项目，哪个开箱零依赖。** 知识库 / 流程 / 运维三个场景在课程里各对应哪些样例项目？其中哪一个场景的样例项目「零外部服务、零 API key」就能完整跑通一次业务链路？

【参考答案】知识库场景对应 advanced-rag 与 ff-companybrain；流程场景对应 ai-document-review；运维场景对应 ff-sre-incidentops。运维场景的 ff-sre-incidentops 零外部依赖：后端默认用 `MockSreService` 回放 `backend/app/data/fixtures/` 里 24 个脱敏运维数据 JSON，不接任何第三方 API、不需要任何 key，即可跑通「收敛→诊断→根因→审批→恢复→复盘」一次完整事故指挥。
【自查要点】三场景对应项目不串台；记住只有 ff-sre 是回放态零依赖；知道接真实系统才走 `RealSreService` 二开。
<!--src:A; ref:code:ff-sre-incidentops/AGENTS.md（外部服务与凭证 · 零依赖）-->

**练习 2｜知识库场景两个样例项目的定位差别。** advanced-rag 和 ff-companybrain 都归在知识库场景，但选型时不能混为一谈。用一句话各说清它俩是什么，并说明什么诉求选哪个。

【参考答案】advanced-rag 是「五范式检索工作台」：文本 / 结构化 / 图文 PDF / 图像 / 视频五范式并列，同一问题下横向对比「基线 vs 进阶」的检索效果，本质是单用户内存态的检索演示系统。ff-companybrain 是「企业知识大脑平台」：把 Nano Brain / Traditional RAG / GraphRAG 三条独立链路聚合到一个统一工作台，有统一身份、14 个场景模板、唯一管理员、PostgreSQL 多 database。只想快速验证「哪种检索最适合我的资料」选 advanced-rag；要一个像产品、给企业员工用的知识库选 ff-companybrain。
【自查要点】advanced-rag = 检索范式对比工作台；ff-companybrain = 聚合三链路的平台；记住 14 场景模板 / 唯一管理员 / 多 database 是后者标志。
<!--src:A; ref:code:advanced-rag/README.md + ff-companybrain/README.md（项目简介）-->

**练习 3｜「数据 / 依赖就绪度」在三个场景里具体指什么。** B1 决策矩阵有一维是「数据就绪度」，为什么它能当选型的先决筛子？把三个场景各自打样前要备的外部依赖列出来对比。

【参考答案】知识库（advanced-rag）：只体验文本范式最低只要一把 OpenAI 兼容 Key（`OPENAI_API_KEY` + `OPENAI_BASE_URL`）；要图文 PDF / 图像 / Agent 范式还得备 DashScope、MinerU 并起 Milvus。流程（ai-document-review）：要 Azure 订阅、`az login`、Terraform 部署基础设施、再部署 PromptFlow endpoint 与 App Service，依赖最重。运维（ff-sre-incidentops）：零外部依赖，回放态直接起。依赖越重、数据越不就绪，打样越容易卡在「快速开始」这一步，所以先落地依赖最轻、数据最全的那个。
【自查要点】三场景依赖从轻到重：ff-sre（零）< advanced-rag 文本范式（一把 Key）< ai-document-review（整套 Azure）；知道依赖重的场景后置。
<!--src:A; ref:code:advanced-rag/AGENTS.md + ai-document-review/docs/getting_started.md + ff-sre-incidentops/README.md（外部服务准备）-->

### B2 · 选定场景打样（回看锚 business-landing#B2）

**练习 4｜advanced-rag 文本范式的四种模式。** 若你选了知识库场景走 advanced-rag 打样，它「文本检索优化」范式并列了哪四种模式？其中 hybrid 融合的是哪两路、用什么方法融？

【参考答案】四种模式并列对比：`baseline` / `small-to-big` / `hybrid`（Vector + BM25，用 RRF 融合）/ `router`。hybrid 融合稠密向量检索与 BM25 关键词检索两路，用 RRF（Reciprocal Rank Fusion）把两个排名列表按排名倒数合并成一个。这一范式的教学主材料在 `backend/core/text_rag.py`。
【自查要点】四种模式名报得全；hybrid = 向量 + BM25 的 RRF 融合；知道核心逻辑在 `backend/core/text_rag.py`。
<!--src:A; ref:code:advanced-rag/README.md（核心功能表）-->

**练习 5｜打样跑通的最低门槛与三条本机验收。** advanced-rag 想先「原样跑通默认 demo」，最少要配哪些凭证？跑起来后拿哪几条命令 / 现象证明后端真的活了？

【参考答案】只跑文本范式，最少配 `OPENAI_API_KEY` 与 `OPENAI_BASE_URL`（写进 `backend/.env`）。三条本机验收：① `curl localhost:8002/` 返回含 `Advanced RAG System Backend is Running!` 的 JSON；② `curl localhost:8002/config` 返回 `"has_openai": true`；③ 浏览器打开 `localhost:3000` 看到五范式模式选择首页。再进一步做文本范式冒烟：`POST /api/text/build`（modes 含 baseline）→ `POST /api/text/chat` 得到非空 `results`。
【自查要点】后端端口 8002、前端 3000 记得住；`/config` 的 `has_openai` 是读没读到凭证的探针；冒烟走 build 再 chat。
<!--src:A; ref:code:advanced-rag/AGENTS.md（部署完成的验证标准）-->

**练习 6｜换成自己业务数据后效果变差，先动哪里。** 按 B2 的节奏，你把项目默认数据换成自己的（脱敏）业务数据后效果明显下滑。第一步该查什么、不该先做什么？

【参考答案】先别改模型。九成是数据清洗 / 切分或检索参数的问题，按项目 `docs/PRD.md` 的已知问题章节与 rag-advanced-pack 的调优点逐项核对。具体到 hybrid 检索，装配顺序不能乱：召回放大（`recall_top_k` 课程基线 30）→ RRF 融合 → Reranker 精排全量（`top_n = recall_top_k`）→ 截断到 `rerank_top_k`（课程用 5）→ 最后做阈值过滤。顺序反了（先截断或先阈值过滤再精排）会把正确候选提前杀光。
【自查要点】先查数据 / 参数不先换模型；背得出 recall_top_k=30、rerank_top_k=5；精排要全量（top_n=recall_top_k）不是只精排 5 个。
<!--src:A; ref:pack:rag-advanced-pack（rag-hybrid-fusion-tuning 融合校验环）-->

---

## 二、误区自查

对下面 6 条，逐条判断自己是不是踩着，踩着的当场纠正。

1. **「选型就选技术最酷的那个场景」**，错。落地失败大多不是败在技术，而是败在选错场景。先看数据 / 依赖就绪度：依赖没到位的场景无论多诱人都后置，先落地数据最全、依赖最轻的那个。
2. **「跑 advanced-rag 必须先装 Milvus」**，错。仅体验文本检索范式只需一把 OpenAI 兼容 Key；Milvus 只有「图文 PDF 的 Agent 模式」与「Agent RAG」两个场景才要，监听 `localhost:19530`。
3. **「advanced-rag 和 ff-companybrain 是一回事」**，错。前者是五范式检索对比工作台（单用户内存态演示、后端 8002），后者是聚合三链路的企业知识大脑平台（14 场景模板、唯一管理员、PostgreSQL 多 database、API 在 3001）。选型时别混。
4. **「运维场景 ff-sre 要接真实告警或 DeepSeek 才能跑」**，错。默认 `MockSreService` 回放 24 个脱敏 JSON fixtures，零外部依赖、零 key 即可跑通一次完整事故指挥；接真实系统是 `RealSreService` 二开路线，默认交付不启用。
5. **「hybrid 检索先截断 top5 再融合」**，错。顺序是召回放大（30）→ RRF 融合 → 精排全量（`top_n = recall_top_k`）→ 截断 `rerank_top_k`（5）→ 最后阈值过滤；反了会把正确候选提前杀光，阈值一收紧结果全空。
6. **「选型矩阵让 Agent 生成一份模板就行」**，错。B1 的阶段徽章要你自己打分、自己权衡、自己拍板，Agent 只当资料员帮你查每个场景要备什么。打分依据得是你自己填的，不是套模板。

---

## 三、迁移挑战

**迁移 1｜把 hybrid 的 RRF 融合迁到你选定场景的检索工具。** advanced-rag 文本范式的 hybrid 是「向量 + BM25 两路召回、RRF 融合」。若给你选定场景的 MVP 加一个检索工具，你会怎么把这套融合思路搬过去？说清召回、融合、精排、截断、阈值的先后顺序，以及为什么 RRF 用排名不用绝对分。

【参考答案要点】顺序不能乱：先各路召回放大（向量 + BM25 各取较多候选，`recall_top_k` 如 30）→ 用 `QueryFusionRetriever(mode="reciprocal_rerank")` 做 RRF 融合（`num_queries=1` 不做 query 扩写省 LLM 调用）→ 对召回的全部节点做 Reranker 精排（`top_n = recall_top_k`，不是只精排最终要的 5 个）→ 截断到 `rerank_top_k`（5）→ 最后按观测分数分位做阈值过滤，别拍脑袋 0.5。RRF 按每一路里的排名倒数加权，不看各自不可比的原始分数，天然抹平两路分数尺度差异。用内积 IP 度量时 embedding 必须先 L2 归一化，多路统一 512 维。
<!--src:A; ref:pack:rag-advanced-pack（rag-hybrid-fusion-tuning 融合校验环）-->

**迁移 2｜把 ff-sre 的 service seam 迁到你的 MVP，让 demo 态和生产态可切换。** ff-sre 用一层 `SreService` 抽象接口，默认挂 `MockSreService` 回放数据、预留 `RealSreService` 接真实系统，路由层只依赖抽象接口。你打样时（回看锚 business-landing#B2）也想「先用假数据跑通产品形态，再决定接不接真实后端」，会怎么把这个模式搬到自己的 MVP？

【参考答案要点】① 先定一个能力抽象接口（把「检索一次 / 审一份 / 处置一次」这类业务动作抽成方法），前端 / 上层只依赖接口，不直接调具体实现；② 默认实现回放脱敏样例数据，零外部依赖就能端到端演示一次完整业务动作（对应 B2 第 9-10h 的 demo 验收）；③ 真实实现留成二开空实现，等选型确认、数据就绪后再接真实模型 / 数据库；④ 切换靠配置而非改路由，保证 demo 态和生产态同一套接口。这样打样阶段先验证产品形态站不站得住，再投入接真实系统的成本。
<!--src:A; ref:code:ff-sre-incidentops/README.md + AGENTS.md（service seam 设计）-->

---

## 四、排错挑战

下面是一位同学打样知识库场景、部署 advanced-rag 时的配置与现象。请按【症状】【根因】【修法】三段定位。

```bash
# 目录布局（.env 放在了项目根，不在 backend/）
advanced-rag/
├── .env                 # OPENAI_API_KEY=sk-xxxx  （只填了这一行）
├── backend/
│   └── main.py
└── frontend/

# 启动后现象
$ curl localhost:8002/config
{"has_openai": false, ...}
# 切到「图文 PDF 的 Agent 模式」范式：请求一直转圈，最后报连接超时
```

【症状】`curl localhost:8002/config` 返回 `has_openai=false`；文本范式对话拿不到结果；切图文 PDF Agent / Agent RAG 范式时请求连接超时。

【根因】两处叠加：① `.env` 放在了项目根而不是 `backend/` 目录，后端读不到凭证，`/config` 的 `has_openai` 就是 false；而且只填了 `OPENAI_API_KEY`，漏了同样必填的 `OPENAI_BASE_URL`。② 图文 PDF 的 Agent 模式与 Agent RAG 范式依赖 Milvus 监听 `localhost:19530`，本机没起 Milvus，所以连接超时。

【修法】① 把 `.env` 移到 `backend/.env`，`OPENAI_API_KEY` 与 `OPENAI_BASE_URL` 两项都填真实值（只跑文本范式配齐这两项就够）；重启后端，`/config` 应返回 `has_openai=true`。② 要跑图文 PDF Agent / Agent RAG，再用 Docker 起 Milvus 于 `localhost:19530` 后重试；只想先验证文本检索，不必装 Milvus。改完先 `curl localhost:8002/` 拿到 `Advanced RAG System Backend is Running!` 再继续。
<!--src:A; ref:code:advanced-rag/AGENTS.md（常见故障表 + 部署完成的验证标准）-->

---

## 五、面试自测

10 题，每题连追问一起答。这是你带 B 阶段的「选型矩阵 + 打样 MVP」去聊落地时最可能被追的方向。

**面试 1｜为什么落地要先选型再写代码。**
【考察点】是否理解「选错场景」比「技术不行」更致命，而不是上来就开跑。
【答题要点】落地失败大多不是败在技术，而是败在选错场景。B 阶段先用一个决策矩阵把候选场景（知识库 / 流程 / 运维）摊开评清楚，按数据就绪度、预期 ROI、落地风险、可复制性四维打分，选定一个投入产出比最高、数据最就绪的先落地；不贪多，一个跑通胜过三个半成品。
【追问】三个场景怎么排先后？用数据 / 依赖就绪度做先决筛子：ff-sre 零依赖可先跑通产品形态，advanced-rag 文本范式一把 Key 就能验证检索，ai-document-review 要整套 Azure 基础设施最重、最后上；依赖越重越靠后，别被「最想做」牵着走。
<!--src:A; ref:code:advanced-rag/AGENTS.md + ai-document-review/docs/getting_started.md（外部服务对比）-->

**面试 2｜知识库场景两个样例项目怎么选。**
【考察点】能否分清「检索范式演示工作台」和「企业级聚合平台」两种产品形态。
【答题要点】advanced-rag 是五范式检索工作台，横向对比 baseline vs 进阶检索效果，单用户内存态演示，后端 8002；ff-companybrain 是把 Nano Brain / Traditional RAG / GraphRAG 三条链路聚合成统一工作台的企业知识大脑，有统一身份、14 场景模板、唯一管理员、PostgreSQL 多 database。要先搞清「哪种检索适合我的资料」选前者，要一个给员工用、像产品的知识库选后者。
【追问】ff-companybrain 为什么要 PostgreSQL 多 database？因为它有模块边界约束：每个 RAG 模块要有独立的上传 / 处理 / 检索链路，各库可在同一实例但 database 必须独立；统一 API 与 Agent Gateway 不允许直连模块业务数据库，避免模块间耦合。
<!--src:A; ref:code:ff-companybrain/AGENTS.md（模块边界约束 · 共享环境变量）-->

**面试 3｜advanced-rag 五范式各自的外部服务落点与 Milvus 依赖边界。**
【考察点】能否说清每个范式吃哪个外部服务，而不是笼统「都要一堆 key」。
【答题要点】文本 → OpenAI 兼容 API（gpt-4o + text-embedding-3-small）；结构化 → PandasQueryEngine 把自然语言转 Pandas 代码执行；图文 PDF → MinerU 解析 + DashScope 多模态向量入 Milvus；图像 → DashScope 的 qwen2.5-vl-embedding 做文搜图 / 图搜图；视频 → TwelveLabs（Marengo 检索 + Pegasus 生成）；Agent RAG → LangChain Agent 编排 Milvus 双流。Milvus 监听 `localhost:19530`，只有「图文 PDF 的 Agent 模式」与「Agent RAG」才强依赖。
【追问】只想先验证文本检索，最少配什么？只配 `OPENAI_API_KEY` 与 `OPENAI_BASE_URL` 两项写进 `backend/.env` 即可，不必装 Milvus、不必备 DashScope / MinerU / TwelveLabs。
<!--src:A; ref:code:advanced-rag/AGENTS.md（外部服务与凭证表）-->

**面试 4｜hybrid 检索的 RRF 融合与装配顺序。**
【考察点】RRF 融合原理与检索管线的正确装配顺序。
【答题要点】两路（BM25 关键词 + 稠密向量）用 `QueryFusionRetriever(mode="reciprocal_rerank")` 做 RRF 融合：RRF 按每路里的排名倒数加权，不看绝对分数，天然解决两路分数不可比。装配顺序：召回放大（`recall_top_k`=30）→ RRF 融合（`num_queries=1` 不扩写）→ Reranker 精排全量（`top_n = recall_top_k`）→ 截断 `rerank_top_k`（5）→ 最后阈值过滤。用 IP 内积度量时 embedding 必须先 L2 归一化，多路统一 512 维。
【追问】Reranker 为什么要精排全量而不是只精排 top5？课程源码有一条明确修复注释：要对全部召回节点精排（`top_n = recall_top_k`），为后面的阈值过滤留足候选；若只精排最终要的 5 个再阈值过滤，阈值一收紧结果就全空。
<!--src:B; ref:external（RRF, Reciprocal Rank Fusion；参数基线取自 pack rag-advanced-pack rag-hybrid-fusion-tuning）-->

**面试 5｜结构化数据检索三方案怎么选。**
【考察点】PandasQueryEngine / Text-to-SQL / 向量+BM25 三条路的适用边界。
【答题要点】PandasQueryEngine 适合小数据（可全量载入内存）的精确聚合 / 排序 / 数值范围；Text-to-SQL 适合中大、已在数据库、要多表 JOIN 的复杂业务，但依赖 Schema 质量、列名幻觉要校验器兜底；向量 + BM25 混合适合语义模糊 + 关键词并存，但不擅长数值 / 聚合。口诀：精确聚合 / 数值 → Pandas（小）/ SQL（大）；语义 + 关键词 → 向量 + BM25；两类都有上智能路由分流。
【追问】直接执行 LLM 生成的查询代码，生产上要补什么？要加校验环：Schema 增强（注入字段业务含义消列名幻觉，性价比最高）+ Few-shot（3–5 个「问题→SQL」示范）+ SQL 校验器（校验→模糊修复错误表名 / 列名，cutoff 0.6→重试）+ 高频查询缓存；不直接跑生成结果。
<!--src:A; ref:pack:rag-advanced-pack（rag-text-to-sql-hardening 四层加固）-->

**面试 6｜图文 PDF 解析四工具怎么选。**
【考察点】按文档特征选解析工具，而不是一把梭。
【答题要点】SimpleDirectoryReader（L1）纯文本简单排版，图片会丢、多栏跨列串读；PyMuPDF（L2）能提图做多模态，但复杂表格被压扁成字符流、LLM 读表幻觉；LlamaParse（L4 企业）擅长原生 PDF 复杂表格 / 多栏，但付费、扫描件 / 中文密集下降；MinerU（L4 本地）擅长扫描件 / 中文密集 / 图文分离 / 要本地化，但部署重、原生复杂表格不如 LlamaParse。
【追问】中文密集的扫描件走哪个、为什么？走 MinerU：四特征（扫描件 / 中文 / 图文分离 / 要本地化）命中任一就选它；这类文档 LlamaParse 会下降，MinerU 本地解析更稳。
<!--src:A; ref:pack:rag-advanced-pack（rag-parser-selection PDF 解析四工具）-->

**面试 7｜流程场景 ai-document-review 的部署形态与打样成本。**
【考察点】能否识别「重基础设施依赖」的项目不适合当轻量打样起点。
【答题要点】ai-document-review 是 Azure 云原生方案：先 `az login`、`task infra-init` 初始化 Terraform、`task infra-deploy` 用 Terraform provision Azure 资源（部署后自动在 app/ui、app/api、flows 生成 `.env`），再 `task app-build` / `app-deploy` 发到 App Service，还要 `task flow-deploy` / `flow-deploy-endpoint` 把 PromptFlow 审核流部署到 Azure AI Foundry。上传 PDF 后由 flow endpoint 逐 chunk 流式返回问题项。
【追问】为什么它不适合当「20 分钟快车道」的打样起点？因为它强依赖 Azure 订阅、`az login`、Terraform 基础设施部署和 PromptFlow endpoint，装起来重、成本高；打样阶段要的是最短路径先跑通产品形态，流程场景更适合数据 / 云资源都就绪后再上，或先用更轻的方式验证审核逻辑。
<!--src:A; ref:code:ai-document-review/docs/getting_started.md（Deploying the solution）-->

**面试 8｜运维场景 ff-sre 的 service seam 设计。**
【考察点】理解「抽象接口 + 回放默认 / 真实二开」这种可切换设计。
【答题要点】ff-sre 用 `SreService` 抽象接口收口全部能力面，默认挂 `MockSreService` 回放 `backend/app/data/fixtures/` 的 24 个脱敏 JSON，预留 `RealSreService` 接真实系统；后端 13 个路由模块共 35 个 `/api` 端点只依赖抽象接口。前端每个动作都真发 HTTP 打后端，所以即便是回放态，链路也是真的。
【追问】这套设计对「选型打样」阶段的价值是什么？零外部依赖就能端到端跑通一次完整业务链路，先验证产品形态站不站得住；确认要落地后再在 `RealSreService` 接真实模型 / 数据库，切换靠配置不用改路由，把「接真实系统」的成本推迟到选型确定之后。
<!--src:A; ref:code:ff-sre-incidentops/AGENTS.md（项目速览 · service seam）-->

**面试 9｜B2 的 MVP 打样怎么算验收通过。**
【考察点】能否说清「打样跑通」的客观标准，而不是「我觉得能用」。
【答题要点】B2 收尾要接入你自己的（脱敏）业务数据、端到端跑通一次核心业务动作（问答一次 / 审一份文档 / 处置一次告警），录屏或截图存档，并对照选定项目 `AGENTS.md`「部署完成的验证标准」里与本机运行相关的条目逐项打勾。打样分三段推进：第 3-4h 原样跑通默认 demo、第 6-8h 换成自己的数据、第 9-10h MVP 闭环成形可 demo。
【追问】以 advanced-rag 为例，自检清单里哪几条能当本机验收？`curl localhost:8002/` 返回 running JSON、`curl localhost:8002/config` 返回 `has_openai=true`、浏览器 `localhost:3000` 显示五范式首页、文本范式冒烟 `POST /api/text/build`（含 baseline）后 `POST /api/text/chat` 得到非空 `results`。
<!--src:A; ref:code:advanced-rag/AGENTS.md（部署完成的验证标准）-->

**面试 10｜换成自己业务数据后效果变差，怎么定位。**
【考察点】排障顺序意识：先查数据 / 参数，别一上来换模型。
【答题要点】先别改模型，九成是数据清洗 / 切分或检索参数问题，按项目 `docs/PRD.md` 已知问题章节与 rag-advanced-pack 的调优点逐项核对。项目跑不通同理：README 常见问题 + PRD 第 13 章已知问题都如实写着，先查这两处再动手。hybrid 检索尤其要核对召回放大→融合→精排全量→截断→阈值这条顺序有没有装反。
【追问】hybrid 里阈值设了结果全空怎么办？多半是阈值高于真实分数、且在精排前就过滤了。正确做法是先精排全量召回节点再截断，阈值按观测分数分位定（如文搜图观测分数约 0.43 就别拍 0.5），不要在精排前用高阈值把候选杀光。
<!--src:A; ref:pack:rag-advanced-pack（rag-multimodal-service-debugging 阈值全空）-->

---

## 六、模拟面试官指令

把下面这段发给你的 Agent，让它扮演面试官，围绕 B 阶段的「选型矩阵 + 打样 MVP」对你连问带追。

```text
你是一家要把 AI 落地到具体业务的公司技术负责人，正在面我。我手里有一份场景选型决策矩阵
（在知识库 / 流程 / 运维三个场景里选定了一个），以及选定场景打样出的一个本机可 demo 的 MVP。
请按下面节奏面我，每轮只问一个问题，等我答完再追问或纠偏：

1. 先让我用 90 秒讲清：我为什么在三个场景里选了这一个，选型矩阵四个维度上它各是怎么打分的，
   数据 / 依赖就绪度这一维起了什么决定作用。
2. 挑我讲得含糊的一环深挖：比如 advanced-rag 和 ff-companybrain 到底差在哪、ff-sre 零依赖是怎么做到的、
   ai-document-review 为什么打样成本高。
3. 追我一处工程细节：advanced-rag 文本范式只配哪两项就能跑、hybrid 的 RRF 为什么用排名不用绝对分、
   Reranker 为什么要精排全量。
4. 逼我讲「打样验收我是怎么自检的」——录屏了哪次业务动作、对照 AGENTS.md 哪几条本机标准逐项打了勾。
5. 最后让我说这个 MVP 还差什么才能上线（接口化 / 容器化 / 评估 / 驾驭约束），我打算下一步先补哪一件。

每轮结束用一句话点评我答得实不实、有没有背项目文档而非真做过，别直接给我标准答案。
```

---

## 七、能力自评表

按「能跟做 / 能独立做 / 能教别人」三档，对每个技能点如实打勾。三档都空的技能点，就是你毕业前要补的洞。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|--------|:---:|:---:|:---:|
| 手写一张选型决策矩阵（四维打分 + 唯一选定结论），自己拍板不外包给 Agent | ☐ | ☐ | ☐ |
| 说清知识库 / 流程 / 运维三场景的样例项目与各自数据 / 依赖就绪度 | ☐ | ☐ | ☐ |
| 讲清 advanced-rag 与 ff-companybrain 的产品形态差别与选型含义 | ☐ | ☐ | ☐ |
| 只配 OpenAI 兼容 Key 把 advanced-rag 文本范式跑通并做本机三条验收 | ☐ | ☐ | ☐ |
| 说清 advanced-rag 五范式各自的外部服务落点与 Milvus 依赖边界 | ☐ | ☐ | ☐ |
| 讲清 hybrid 的两路召回 + RRF 融合 + 精排全量 + 截断 + 阈值顺序 | ☐ | ☐ | ☐ |
| 说清结构化三方案与图文 PDF 四工具的选型口诀 | ☐ | ☐ | ☐ |
| 讲清 ff-sre 的 service seam（Mock 默认 / Real 二开）并能迁到自己的 MVP | ☐ | ☐ | ☐ |
| 把项目默认数据换成自己业务数据并端到端跑通一次核心业务动作 | ☐ | ☐ | ☐ |
| 对照选定项目 AGENTS.md「部署完成的验证标准」逐项自检打样 MVP | ☐ | ☐ | ☐ |

---

## 八、落地对照

拿这几条对照你的作品与目标（自己的业务 / 岗位 JD），逐条落实到「我做过、能讲、有产物」。

1. **一张能拍板的选型决策矩阵**：候选场景 × 数据就绪度 / 预期 ROI / 落地风险 / 可复制性，四维每个都有你自己填的打分与依据，并圈出唯一选定场景和一句理由。这是本路径 portfolio 的第一件，也是第二场景复用的模板（回看锚 business-landing#B1）。
2. **选定场景的生产可用 MVP**：知识库 / 流程 / 运维三选一，接入你自己的（脱敏）业务数据、端到端跑通一次核心业务动作、录屏存档（回看锚 business-landing#B2）。
3. **一份本机自检记录**：对照选定项目 AGENTS.md「部署完成的验证标准」里本机相关条目逐项打勾（如 advanced-rag 的 `/config has_openai=true`、五范式首页、文本范式冒烟）；这条直接衔接接口化上线时的验收（回看锚 business-landing#C1）。
4. **一套可复制的选型 + 打样方法**：把「怎么选、怎么最短路径跑通、怎么换自己的数据」沉淀成能套到第二场景的做法，验证方法可复制（回看锚 business-landing#D4）。
5. **一条能讲清的场景取舍**：面对业务方或面试官，用 90 秒讲清你为什么选这个场景、放弃另两个的依据是什么，指出这个 MVP 上线前还差哪一件（接口化 / 容器化 / 评估 / 驾驭约束）。

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
