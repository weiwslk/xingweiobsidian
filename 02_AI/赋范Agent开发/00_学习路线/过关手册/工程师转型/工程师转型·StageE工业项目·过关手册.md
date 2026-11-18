# 工程师转型 · Stage E 工业项目 · 过关手册

> 本册对应「工程师转型 Agent 开发」路径的收官阶段：E1 复刻并改造 FF-GBrain 编译式 RAG，E2 从 advanced-rag 或 agentic-graphrag 二选一做进作品集。
> 练习向。题目从两个真实项目的源码、README/AGENTS/PRD 与课程课件出，数字和字段名逐字核对。回看锚统一写成 `engineer-transform#节点id`。
> 用法：先自己作答，再翻参考答案。面试自测每题带一层追问，务必连追问一起答。

---

## 一、随堂练习

### E1 · FF-GBrain 编译式 RAG（回看锚 engineer-transform#E1）

**练习 1｜画出 BFF 桥怎么调后端。** 用一句话说清 FF-GBrain 的 5 个在线接口最终是怎么触达 GBrain 知识引擎的，写出关键的进程调用形式与三条硬约束。

【参考答案】所有在线接口都经 `src/lib/gbrain-bridge.ts` 的 `callGBrain(op, params)` 触达后端：它用 `execFile("bun", ["src/cli.ts", "call", op, JSON.stringify(params)])` 起一个子进程，`cwd` 指向 `backend`，不走 shell。三条硬约束：超时 `timeout: 30_000`（30 秒）、`maxBuffer: 32 * 1024 * 1024`（32MB）、调用前先校验 `OPENAI_API_KEY` 以 `sk-or-v1-` 开头，否则抛 `backend_not_configured`。
【自查要点】能说出「子进程 + 无 shell + JSON.parse(stdout)」；能背出 30 秒 / 32MB 两个数；知道桥是所有在线能力的唯一枢纽。
<!--src:A; ref:code:gbrain-wiki/src/lib/gbrain-bridge.ts-->

**练习 2｜GBRAIN_HOME 的正确配法。** 灌完库后 `get_stats` 读不到数据，最常见的一类配置错误是什么？说清 `GBRAIN_HOME` 到底该填哪一层目录。

【参考答案】`GBRAIN_HOME` 要填 GBrain 数据主目录的**父目录**绝对路径（须绝对、不含 `..`）：`backend/src/core/config.ts` 会自动追加 `.gbrain`。填成 `backend` 时数据实际落在 `backend/.gbrain`。最常见错误是自己又补了 `.gbrain` 后缀，路径变成 `.gbrain/.gbrain`，于是灌进去的库和读取路径对不上，`get_stats` 读空。
【自查要点】记住「父目录 + 代码自动补 .gbrain」；能复现 `.gbrain/.gbrain` 这个双后缀坑；知道缺 `sk-or-v1-` 前缀会让所有在线接口返回 502。
<!--src:A; ref:code:gbrain-wiki/AGENTS.md（常见故障表）-->

**练习 3｜/api/graph 为什么不逐页调 CLI。** `/api/graph` 要出「节点 + 边」的知识网络，它取节点和取边分别用了什么手段？为什么不全走后端？

【参考答案】节点：`callGBrain("list_pages", { limit: 500 })` 一次拿回页列表。边：本地 walk `sample-vault/` 目录，用正则 `\[\[([^\]|#]+)(?:\|[^\]]*)?\]\]` 抽每篇的 wikilink，只保留两端 slug 都在节点集合里的边。设计上用「本地 fs 解析」替代「50 次串行子进程 CLI 调用」，是为性能；vault 目录缺失时 catch 掉，退化成 nodes-only 图。
【自查要点】能说出节点走 CLI、边走本地 fs 抽 wikilink 的分工；知道「两端都在 slug 集合」这条过滤；理解 50 次串行子进程是它要规避的开销。
<!--src:A; ref:code:gbrain-wiki/src/app/api/graph/route.ts-->

### E2 · 知识库项目组二选一（回看锚 engineer-transform#E2）

**练习 4｜advanced-rag 文本范式的四种模式。** advanced-rag 的「文本检索优化」范式并列了哪四种模式？其中 hybrid 融合的是哪两路、用什么方法融？

【参考答案】四种模式并列对比：`baseline` / `small-to-big` / `hybrid`（Vector + BM25，用 RRF 融合）/ `router`。hybrid 融合稠密向量检索与 BM25 关键词检索两路，用 RRF（Reciprocal Rank Fusion）把两个排名列表融成一个。文本范式只需第一项 OpenAI 兼容 Key 即可跑通。
【自查要点】四种模式名能报全；知道 hybrid = 向量 + BM25 的 RRF 融合；记得文本范式最低门槛只要一把 OpenAI 兼容 Key。
<!--src:A; ref:code:advanced-rag/README.md（核心功能表）-->

**练习 5｜advanced-rag 五范式的外部服务落点。** 说出五范式各自依赖的外部服务与向量库落点，并指出哪些范式必须先起 Milvus。

【参考答案】文本 → OpenAI 兼容 API（gpt-4o + text-embedding-3-small）；结构化 → PandasQueryEngine 把自然语言转 Pandas 代码执行；图文 PDF → MinerU 解析 + DashScope 多模态向量入 Milvus；图像 → DashScope 的 qwen2.5-vl-embedding 做文搜图/图搜图；视频 → TwelveLabs（Marengo 检索 + Pegasus 生成）；Agent RAG → LangChain Agent 编排 Milvus 双流。Milvus 监听 `localhost:19530`，仅「图文 PDF 的 Agent 模式」与「Agent RAG」需要先起。
【自查要点】五范式对应服务不串台；Milvus 端口 19530 记得住；知道只有 PDF Agent 与 Agent RAG 两个场景强依赖 Milvus。
<!--src:A; ref:code:advanced-rag/AGENTS.md（外部服务与凭证表）-->

**练习 6｜agentic-graphrag 的四段链路。** 垂域高精度 GraphRAG 系统把一份文档走完哪四段？默认向量库是什么、抽取和 Embedding 各用哪个模型？

【参考答案】四段：抽取 → 建库 → 问答 → 溯源。先用 LangExtract + DeepSeek（`deepseek-chat`）做高精度实体/关系抽取并**保留原文字符位置**，再把抽取结果与原文写入向量库，之后基于检索增强做 SSE 流式问答，每条答案按字符位置回跳原文证据高亮。默认向量库是本地持久化的 ChromaDB（可切 Qdrant），Embedding 用 DashScope 的 `text-embedding-v4`。
【自查要点】四段顺序背得出；抽取=DeepSeek、Embedding=DashScope text-embedding-v4 不混；知道「保留字符位置」是溯源高亮的前提；默认 ChromaDB 而非 Qdrant。
<!--src:A; ref:code:agentic-graphrag/README.md（项目简介 · 技术栈表）-->

---

## 二、误区自查

对下面 6 条，逐条判断自己是不是踩着，踩着的当场纠正。

1. **「GBRAIN_HOME 要精确到 .gbrain 目录」**，错。它是父目录，`config.ts` 自动追加 `.gbrain`；你手写后缀会变成 `.gbrain/.gbrain`，库读空。
2. **「/graph 的全量 17k 视图也是接真后端的」**，错。`src/lib/api.ts` 里 `view === "real"` 直接 `return fallback` 走 `graph-17k.json` mock，真后端不可能拉 1.7 万节点；只有教学 500 视图在 demo 下可接接口。
3. **「首页那个 BrainBench P@5 49.1% 是本仓库实测出来的」**，错。它是 `today` 接口里写死的展示值（论文口径，对照 vector RAG 22.1%），不是本工程台实测。
4. **「跑 advanced-rag 必须先装 Milvus」**，错。仅体验文本检索范式只需一把 OpenAI 兼容 Key；Milvus 只有 PDF Agent 模式与 Agent RAG 才要。
5. **「agentic-graphrag 直接 docker compose up 就能起前后端」**，错。`frontend/Dockerfile` 在仓库中不存在，compose 起不了前端；教学环境优先用 `start.sh` 本地方式。
6. **「改造 GBrain 就是去改 backend/ 里的知识引擎源码」**，错。`backend/` 是上游 GBrain v0.28.6 源码，红线是勿改；你的改造做在前端/BFF 或检索逻辑层，且必须是自己手写的 diff。

---

## 三、迁移挑战

**迁移 1｜把 GBrain 的子进程桥经验迁到你的 Agent API 服务。** 你在 D 阶段手写过 Agent API 服务（回看锚 engineer-transform#D2）。GBrain 桥的三处设计，无 shell 的 `execFile`、错误信封前的 `maskKey`、缺 key 直接拒而不是塞占位符，分别对应你 API 服务里的哪个安全动作？各写一句你会怎么落到自己的服务上。

【参考答案要点】① `execFile` 不走 shell = 避免命令注入，对应你服务里对外部命令/参数一律用参数数组传、不拼字符串；② `maskKey` 把 `sk-or-v1-[a-z0-9]+` 抹成 `sk-or-v1-***` 再进错误信封 = 你的服务在日志/错误响应里也要脱敏密钥，别把上游报错原样透出；③ 缺 `OPENAI_API_KEY` 直接抛 `backend_not_configured`、在线接口 502 = 你的服务对缺凭证要快速失败并返回规范错误码，不能用占位符继续跑出假成功。
<!--src:A; ref:code:gbrain-wiki/src/lib/gbrain-bridge.ts-->

**迁移 2｜把 hybrid RRF 融合迁到你 B2 框架 Agent 的检索工具。** 你在 B2 用 LangChain 或 LangGraph 搭过带工具调用的多步 Agent（回看锚 engineer-transform#B2）。advanced-rag 文本范式的 hybrid 是「向量 + BM25 两路召回，RRF 融合」。若给你的框架 Agent 加一个检索工具，你会怎么把这套融合思路搬过去？说清召回、融合、精排、截断的先后顺序。

【参考答案要点】顺序不能乱：先各路召回放大（向量 + BM25 各取较多候选）→ 用 RRF 把两个排名列表融成一个统一排名（RRF 用排名而非绝对分，天然抹平两路分数不可比）→ 再对融合后的全量候选做 Reranker 精排 → 最后截断取 top-k → 需要时才做阈值过滤。顺序反了（先截断再融合，或先阈值过滤再精排）会把正确候选提前杀光。工具产出要能追溯到命中来源，衔接你后面要做的可溯源问答。
<!--src:A; ref:pack:rag-advanced-pack（rag-hybrid-fusion-tuning 能力）-->

---

## 四、排错挑战

下面是一位同学部署 FF-GBrain 在线模式时的 `backend/.env.local`（凭证已打码）与现象。请按【症状】【根因】【修法】三段定位。

```dotenv
# backend/.env.local
OPENAI_API_KEY=or-v1-abcd1234efgh5678
GBRAIN_HOME=/srv/case-10-gbrain-wiki/backend/.gbrain
```

灌库脚本跑完没报致命错，`bun src/cli.ts call get_stats '{}'` 却显示 `page_count` 为 0；前端切「在线」后首页、精准检索、知识网络全部拿不到真实数据，devtools 里在线接口返回 502。

【症状】灌库命令表面跑通，但 `get_stats` 读到 0 页；所有在线接口 502、前端在线徽章不亮，自动退回沙箱 mock。

【根因】两处叠加：① `OPENAI_API_KEY` 写成 `or-v1-...`，少了前缀里的 `sk`，桥在 `callGBrain` 里校验 `startsWith("sk-or-v1-")`，不匹配就抛 `backend_not_configured`，于是每个在线接口经 `bridgeError` 返回 502。② `GBRAIN_HOME` 末尾自带了 `.gbrain`，代码会再自动追加一次，真实数据目录变成 `.gbrain/.gbrain`，灌库写到的库和读取路径对不上，`get_stats` 自然读空。

【修法】① key 改成以 `sk-or-v1-` 开头的真实 OpenRouter key；② `GBRAIN_HOME` 去掉 `.gbrain` 后缀，填父目录 `/srv/case-10-gbrain-wiki/backend`，让代码自己补 `.gbrain`；改完重跑灌库四步（`init --pglite` → `import --no-embed` → `embed --all` → `extract`），再 `get_stats` 期望约 50 页。
<!--src:A; ref:code:gbrain-wiki/src/lib/gbrain-bridge.ts + AGENTS.md（常见故障表）-->

---

## 五、面试自测

10 题，每题连追问一起答。这是你带 E 阶段作品去技术面时最可能被追的方向（并入 career-sprint 前的预演）。

**面试 1｜编译式 RAG 和传统向量 RAG 的本质区别。**
【考察点】是否理解「编译时算 vs 检索时算」，而不是把 GBrain 当又一个向量库。
【答题要点】传统向量 RAG 是「检索时算」：查询来了才做相似度召回，关系型问题（谁投了谁、某想法怎么演变）召回很差。编译式 RAG 是「编译时算」：预先把资料结构化成 typed-link 知识网络（self-wiring 抽 wikilink 建边，零 LLM），查询时既能语义检索又能沿关系跳。GBrain 底座是 Markdown 原生 + hybrid RAG，用 embedded PGLite + pgvector 存。
【追问】那首页写的 BrainBench P@5「49.1% vs vector RAG 22.1%」说明什么、可信度如何？说明在该基准上编译式检索的前 5 命中准确率高出约 27 个点；但要点破：这个数是 `today` 接口里写死的论文口径展示值，不是本工程台实测，面试时主动讲清出处更稳。
<!--src:A; ref:code:gbrain-wiki/src/app/api/today/route.ts-->

**面试 2｜FF-GBrain 的 BFF 子进程桥怎么设计的。**
【考察点】能否讲清「Next.js 前端怎么安全地驱动一个 Bun CLI 后端」。
【答题要点】BFF 路由跑在 `runtime = "nodejs"`，经 `gbrain-bridge.ts` 用 `execFile("bun", ["src/cli.ts","call",op,JSON])` 起子进程：无 shell（防注入）、`cwd=backend`、30 秒超时、32MB maxBuffer、从 `backend/.env.local` 懒加载并注入环境变量、`JSON.parse(stdout)` 取结果。失败经 `maskKey` 抹掉密钥再包成 502 信封。
【追问】为什么这套设计不适配 Vercel serverless？因为它依赖 `spawn` bun 子进程，Vercel 的 serverless 函数不能起子进程；README/PRD 明确把 Vercel 标为不推荐，容器化要在镜像里装 bun + node 并 `bun build --compile` 出单文件。
<!--src:A; ref:code:gbrain-wiki/src/lib/gbrain-bridge.ts-->

**面试 3｜/api/graph 的性能取舍。**
【考察点】能否识别「批量 subprocess 调用」这类性能坑并给出替代。
【答题要点】节点走一次 `list_pages(limit:500)` 拿全；边如果也逐页调 CLI，就是约 50 次串行子进程，慢。所以改成本地 walk `sample-vault/`、正则抽 wikilink、只留两端都在节点集的边，纯 fs 读，快很多。
【追问】vault 目录不在项目根会怎样？抽边的 try 块 catch 掉，返回 nodes-only 图；README 常见问题也提到 `/api/graph` 30s 超时多半是 `sample-vault/` 放错位置（应在项目根而非 backend），部署验证要求 `node_count >= 50` 且 `link_count >= 60`。
<!--src:A; ref:code:gbrain-wiki/src/app/api/graph/route.ts-->

**面试 4｜typed-link 边是怎么抽出来的，为什么零 LLM。**
【考察点】理解 self-wiring 建图是确定性解析而非模型生成。
【答题要点】GBrain 的 `extract`（self-wiring）纯用正则解析 markdown 里的 `[[...]]` wikilink 建边，不调 LLM，所以入库这一步的抽边成本约等于零、可增量。工程台 `/api/graph` 复刻了同一思路：本地正则抽 wikilink。
【追问】写了 `[[张三]]` 却抽不到边，根因是什么？裸 `[[名字]]` 没有目录前缀，不在白名单会被静默忽略；要写成 `[[目录/slug]]` 并建对应实体存根页，`extract` 才连得上边（`extract` 返回 0 边且不报错是典型静默失败）。
<!--src:A; ref:pack:compiled-rag-pack（references/gbrain-cli-and-pitfalls 静默失败表）-->

**面试 5｜hybrid 检索的三路融合与顺序。**
【考察点】RRF 融合原理与检索管线的正确装配顺序。
【答题要点】三路召回（BM25 关键词 + 稠密向量，可再加 Reranker）用 RRF 融合：RRF 按每路里的排名倒数加权，不看绝对分数，天然解决两路分数不可比。装配顺序：召回放大 → RRF 融合 → Reranker 精排全量 → 截断 top-k → 阈值过滤。
【追问】RRF 那个常数 k 一般取多少、取错会怎样？工程惯例取 60（Cormack & Clarke 2009 提出）；k 太小会让高排名项权重过陡、太大则各路趋同失去区分，属架构级融合参数，先按 60 起步再按数据调。
<!--src:B; ref:external（RRF 常数 k=60，Cormack, Clarke & Büttcher, SIGIR 2009 原论文；pack rag-advanced 教 reciprocal_rerank 融合但未记该常数）-->

**面试 6｜结构化数据检索用的什么方案。**
【考察点】Text-to-SQL / Text-to-Pandas 这类「自然语言转可执行代码」的检索范式。
【答题要点】advanced-rag 结构化范式用 LlamaIndex 的 PandasQueryEngine，把自然语言问题转成 Pandas 代码在 Excel/CSV 上执行再回答，本质是让 LLM 生成可执行查询而非做向量召回。
【追问】直接执行 LLM 生成的查询代码，生产上要补什么？要加校验环：生成后先校验/修复（表名列名是否存在、SQL 或代码是否合法）再执行，配合 Schema 增强与 Few-shot 提示，不直接跑生成结果，否则列名表名不存在就报错或跑出错值。
<!--src:A; ref:pack:rag-advanced-pack（rag-text-to-sql-hardening 能力）-->

**面试 7｜GraphRAG 的 local / global / drift 检索怎么选。**
【考察点】按问题类型选检索模式，而不是一把梭。
【答题要点】Local（实体级）：答具体实体及其邻域的细节问题；Global（全局归纳）：答需要跨社区汇总的宏观/归纳型问题（如「整篇讲了哪几大主题」）；DRIFT：介于两者的探索式；另有 Basic 作向量对照。自建流水线则在 NetworkX 内存图上用 search_entities/get_neighbors 之类工具多步检索。
【追问】什么时候必须走 global 而不是 local？当问题要对全库做归纳汇总、答案分散在多个实体社区、单点实体邻域覆盖不到时；local 只在种子实体附近展开，回答不了「全局主题有哪些」这类多跳归纳问题。
<!--src:A; ref:pack:rag-graphrag-pack（graphrag-query-mode-selection 能力）-->

**面试 8｜agentic-graphrag 怎么做到可溯源问答。**
【考察点】证据溯源的实现前提与流式问答的事件设计。
【答题要点】抽取阶段 LangExtract + DeepSeek 就保留了每个实体/关系在原文里的字符位置；问答走检索增强、SSE 流式返回，答案带 sources 与实体，前端按字符位置在原文高亮证据。默认向量库 ChromaDB，Embedding 用 DashScope text-embedding-v4。
【追问】只配了 DeepSeek、没配 DashScope，RAG 端点会怎样？RAG（检索/问答）端点会 500 报「未配置 DASHSCOPE_API_KEY」，因为 Embedding 依赖 DashScope；只配 DeepSeek 时仅「抽取」功能可用，检索/问答不可用。
<!--src:A; ref:code:agentic-graphrag/AGENTS.md（验证标准 · 常见故障表）-->

**面试 9｜这两个项目的凭证与安全边界。**
【考察点】密钥治理与「哪些不能碰」的红线意识。
【答题要点】凭证一律只进 `.env`（GBrain 是 `backend/.env.local`），从 `.env.example` 复制后填，绝不写进代码或提交 git；桥/服务对错误信息做密钥脱敏（GBrain 的 `maskKey`）。红线：不改 GBrain 的 `backend/` 上游源码与 `sample-vault/`；不改 advanced-rag 的上传件与源码逻辑；不改 agentic-graphrag 的场景 Prompt 与演示数据。
【追问】review 时发现代码里硬编码了 `sk-or-v1-*` 密钥，怎么处理？应报告而非扩散（GBrain 桥已内置 `maskKey` 兜底），把密钥挪进 `.env`、从 git 历史清除、轮换该 key；绝不把硬编码 key 复制到别处或提交。
<!--src:A; ref:code:gbrain-wiki/AGENTS.md（安全与边界）-->

**面试 10｜服务化里的无状态与单写者约束。**
【考察点】把「本机脚本」变「可扩展服务」时的状态处理，衔接 D 阶段。
【答题要点】Agent 服务化核心是无状态化 + 幂等：把会话状态外置，接口才能水平扩展；GBrain 工程台的 BFF 本身无状态，靠子进程调后端。但底层 PGLite 是单写者：写操作和 `serve`（持写锁）不能并发，大批量 sync 前要先停 serve。
【追问】并发写 GBrain 报 `Timed out waiting for PGLite lock` 怎么办？那是 `serve` 持着写锁、PGLite 只允许单写者；按「停 serve → 写 → 重启 serve」处理，别让多个写入者同时抢锁。
<!--src:A; ref:pack:compiled-rag-pack（references/gbrain-cli-and-pitfalls 静默失败表）-->

---

## 六、模拟面试官指令

把下面这段发给你的 Agent，让它扮演面试官，围绕 E 阶段两个作品对你连问带追。

```text
你是一家做企业知识库 / RAG 平台公司的资深工程师，正在面我。我的作品集里有两个工业级 RAG 项目：
一个是私有化部署并改造过的 FF-GBrain 编译式 RAG，一个是 advanced-rag（五范式检索）或 agentic-graphrag
（垂域可溯源 GraphRAG）二选一。请按下面节奏面我，每轮只问一个问题，等我答完再追问或纠偏：

1. 先让我用 90 秒讲清我选的那个知识库项目的检索/编排流水线（抽取→建库→问答→溯源，或五范式并列）。
2. 挑我讲得含糊的一环深挖：比如「编译时算 vs 检索时算」到底省了什么、hybrid 的 RRF 为什么用排名不用绝对分、
   local 和 global 检索的边界在哪。
3. 追我一处工程细节：BFF 子进程桥的三条安全约束、GBRAIN_HOME 的父目录坑、缺 DashScope 时 RAG 端点的行为。
4. 逼我讲「我改造了哪一处、为什么这么改、改完怎么验证测试仍通过」——这是区分「跑通 demo」和「真读懂」的题。
5. 最后让我说这个项目还有什么没做好、若上生产我会先补哪一件（无状态化 / 评估 / 凭证治理 / 观测）。

每轮结束用一句话点评我答得实不实、有没有背课件而非真懂，别直接给我标准答案。
```

---

## 七、能力自评表

按「能跟做 / 能独立做 / 能教别人」三档，对每个技能点如实打勾。三档都空的技能点，就是你毕业前要补的洞。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|--------|:---:|:---:|:---:|
| 按 AGENTS.md 把 FF-GBrain 部署跑通并逐项自检（curl `/api/today` live、`/api/graph` node≥50/link≥60） | ☐ | ☐ | ☐ |
| 讲清编译式 RAG 的「编译时算 vs 检索时算」与 typed-link 零 LLM 建图 | ☐ | ☐ | ☐ |
| 读懂并复述 BFF 子进程桥（execFile 无 shell / 30s / 32MB / maskKey / 缺 key 502） | ☐ | ☐ | ☐ |
| 手写改造 GBrain 一处检索/编译逻辑，给出改造理由且测试仍通过 | ☐ | ☐ | ☐ |
| 部署 advanced-rag 并讲清五范式各自的服务落点与 Milvus 依赖边界 | ☐ | ☐ | ☐ |
| 讲清 hybrid 检索的三路召回 + RRF 融合 + 精排截断顺序 | ☐ | ☐ | ☐ |
| 部署 agentic-graphrag 并讲清抽取→建库→问答→溯源四段与字符位置溯源 | ☐ | ☐ | ☐ |
| 说清 local/global/drift 检索选型与 Text-to-Pandas 校验环 | ☐ | ☐ | ☐ |
| 把子进程桥 / 无状态 / 密钥脱敏经验迁到自己的 Agent API 服务 | ☐ | ☐ | ☐ |
| 面试里用 90 秒把一个知识库项目讲成能过技术面的故事 | ☐ | ☐ | ☐ |

---

## 八、落地对照

拿这几条对照你的作品集与目标岗位 JD，逐条落实到「我做过、能讲、有产物」。

1. **私有化部署一个工业级 RAG 项目**：FF-GBrain 按 AGENTS.md「部署完成的验证标准」逐项打勾（`curl /` 200、`/api/today` 返回 `live:true` 且 metrics 长 6、`/api/graph` node≥50/link≥60、`/api/ask` 传 `typed-link` 有 sources、`click-through-live.mjs` 14 项断言全过）。对应你要在简历写「能私有化部署并自检 RAG 服务」（回看锚 engineer-transform#E1）。
2. **手动改造一处核心逻辑并说清理由**：GBrain 的一处检索/编译逻辑，手写 diff、写改造理由、改后自检/测试仍通过，这是面试区分「跑通 demo」和「读懂源码」的关键产物（回看锚 engineer-transform#E1）。
3. **交付一个知识库作品并讲清检索架构**：advanced-rag 或 agentic-graphrag 二选一部署跑通 + 一次可追溯来源的真实问答 + 一段架构讲解文档，做进作品集（回看锚 engineer-transform#E2）。
4. **把服务化能力串起来**：用 D 阶段的接口/容器化/评估意识（回看锚 engineer-transform#D2）回看这两个项目，能对 JD 里「Agent/RAG 服务上线与运维」条目给出具体动作（无状态化、凭证治理、评估集）。
5. **面试讲解预演**：任取 E 阶段一个项目，不看笔记向面试官讲清它的检索/编排流水线并指出一处可优化，这条直接对接求职冲刺（career-sprint）里的项目讲解环节。

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
