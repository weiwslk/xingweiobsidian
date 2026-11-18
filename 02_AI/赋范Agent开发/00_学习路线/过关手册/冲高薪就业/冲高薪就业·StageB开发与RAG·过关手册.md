# 冲高薪就业 · Stage B（开发与 RAG）过关手册

> 这一册对着 career-sprint 的 Stage B 出。七个节点：B1 LangChain 1.0 运行时、B2 LangGraph、B3 RAG 入门、B4 RAG 进阶（LlamaIndex）、B5 RAG 基础 & 向量库（深潜 gate）、B6 企业级 RAG 评估、B7 GraphRAG。
> 这一阶段是你简历里"会做企业知识库、会搭会调工具的 Agent"的底气。求职向手册，重头在后面的面试自测与落地对照，前面的练习先把手练顺。
> 每道题后面有一行看不见的机读标注（HTML 注释），是给核查用的来源出处，你读正文不受影响。

---

## 一、随堂练习

### B1 · LangChain 1.0 运行时框架

**练习 1.1** 用 `create_agent` 搭一个能调工具、带记忆的最小 Agent，并用 `init_chat_model` 一行切换厂商。写出你会怎么装配。

【参考答案】主链路三件：`model = init_chat_model("...")` 统一接入模型（换厂商只改这一行的模型串，不动别的）；`agent = create_agent(model, tools, system_prompt=...)` 返回可直接调用体；给 `agent.invoke({"messages":[{"role":"user","content":...}]})` 传入消息。要记忆就 `compile` 时挂 Checkpointer，并在 `config={"configurable":{"thread_id": ...}}` 里带上会话标识。
【自查要点】你没有再包 `AgentExecutor`；入参用的是 `messages` 而不是 0.x 的 `{"input": ...}`；换厂商只动 `init_chat_model` 一处。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》create_agent 大一统 / pack:langchain-agent-pack-->

**练习 1.2** 你要给 Agent 装三个中间件：PII 脱敏、上下文压缩摘要、模型故障回退。它们分别挂在哪个生命周期钩子上？

【参考答案】PII 脱敏 `PIIMiddleware` 与摘要压缩 `SummarizationMiddleware` 都挂在 `before_model`（模型调用前）；模型故障回退 `ModelFallbackMiddleware` 挂在 `wrap_model_call`（包裹模型调用）。PII 必须最先经过。
【自查要点】你分得清 6 个钩子里 `before_model` 是每次模型调用前、`wrap_model_call` 是洋葱包裹；没把回退中间件错挂到 `after_model`。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》内置中间件按钩子分类 / pack:langchain-agent-pack-->

**练习 1.3** 结构化输出要 Agent 稳定回 JSON，课件的硬红线是什么？迁到 1.0 后老写法 `create_json_agent` 换成什么？

【参考答案】红线是结构化输出走 `"json"` 校验环（response_format 传 Pydantic 模型），让框架校验并在不合规时重试，而不是自己解析字符串。0.x 的 `create_json_agent(...)` 迁到 1.0 统一为 `create_agent(..., response_format=<PydanticModel>)`。
【自查要点】你知道 1.0 的 `create_agent` 不再需要 `agent_scratchpad` 占位符，迁移时要删掉那一行否则报参数错。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》0.x→1.0 迁移映射 / pack:langchain-agent-pack-->

### B2 · LangGraph

**练习 2.1** 你的 State 里 `messages` 字段要支持"人工改掉某一条历史消息"，Reducer 该挂 `operator.add` 还是 `add_messages`？为什么？

【参考答案】挂 `add_messages`。`operator.add` 只会把新值追加到列表末尾，改不了已存在的消息；`add_messages(left, right)` 会按 ID 合并——right 与 left 同 ID 就替换、不同 ID 才追加，这正是要能"人工更新某条已存在消息"时它存在的理由。不挂 Reducer 则是新值直接覆盖旧值。
【自查要点】你能说清"不挂 Reducer = 覆盖、operator.add = 只追加、add_messages = 按 ID 合并"三种行为。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》State 与 Reducer / pack:langgraph-pack-->

**练习 2.2** 搭好带检查点的 ReAct 循环，多轮对话却发现记忆丢失、两个会话串了。最可能漏了什么？

【参考答案】漏传 `thread_id`。加了 checkpointer 就必须在 `config={"configurable":{"thread_id": ...}}` 里带会话标识，thread_id 是隔离会话的唯一标识，换 thread_id 就是换一个独立会话。串了就是多个会话共用了同一个 thread_id 或没配。
【自查要点】你知道 SqliteSaver 的 `from_conn_string` 是上下文管理器，要在 `with` 里用；`:memory:` 换成 `.db` 路径即持久化。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》Checkpointer + thread_id / pack:langgraph-pack-->

**练习 2.3** 你要编排一个多智能体系统，希望好扩展、由唯一节点决定下一步调谁。选哪种架构？关键件是什么？

【参考答案】选 Supervisor（主管）。唯一 supervisor 节点做中央路由，扩展性好，是大多数工作流编排的首选。关键件：`class AgentState(MessagesState): next: str` + `Router` TypedDict（`next: Literal[*options]`）+ `FINISH` 标识（路由到 END）。区别于 Network：Network 每个子代理都能自行 END，Supervisor 是唯一决策者。
【自查要点】你没把 Supervisor 和 Network 搞反；知道要看子图内部输出得开 `subgraphs=True`。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》多代理五架构选型 / pack:langgraph-pack-->

### B3 · 大模型 RAG 入门

**练习 3.1** 画出 RAG 的两阶段数据流，各自包含哪些步骤？

【参考答案】离线索引：加载 → 切分 → 向量化 → 建 FAISS 索引 → 持久化。在线查询：查询向量化 → Top-K 检索 → 阈值过滤 → Token 预算拼接 → 组 RAG Prompt → 生成。这条链路对应 RAG 六大核心组件。
【自查要点】你能把"离线只做一次的索引"和"每次提问都走的查询"两阶段分清，不会把切分放到在线阶段。
<!--src:A; ref:讲次《大模型 RAG 入门》两阶段数据流与六大组件 / pack:rag-basics-pack-->

**练习 3.2** RAG 的 System Prompt 必须带哪三条约束？少了会怎样？

【参考答案】三条：① 只使用【参考资料】中的信息回答、不要编造；② 引导模型引用来源而非自由创作；③ 参考资料不足时明确说"根据现有资料无法回答"而非硬答。缺一条模型就会用先验知识编造，RAG 退化成普通对话。
【自查要点】你没把"温度调低"当成 Prompt 约束（那是生成参数，不是这三条）。
<!--src:A; ref:讲次《大模型 RAG 入门》RAG Prompt 三约束 / pack:rag-basics-pack-->

**练习 3.3** 召回片段拼进 Prompt 前为什么要做 Token 预算控制？怎么拼？

【参考答案】不控预算会超上下文窗口报错或成本失控。做法：按相似度降序贪心累加片段，逼近 `max_tokens` 预算即停止，宁可少拼几个低分片段，不无限拼接。
【自查要点】你拼接是"降序贪心 + 到预算就停"，不是"把召回的全塞进去"。
<!--src:A; ref:讲次《大模型 RAG 入门》Token 预算控制 / pack:rag-basics-pack-->

### B4 · RAG 进阶（LlamaIndex）

**练习 4.1** 把一个 LlamaIndex 的 Text-to-SQL 查询引擎加固到能上生产，四层分别做什么？

【参考答案】第 1 层 Schema 增强：注入字段业务含义与业务规则，消列名幻觉（性价比最高）；第 2 层 Few-shot：给 3–5 个"问题→SQL"示范；第 3 层 SQL 校验器：校验→修复→重试环，模糊修复错误表名列名（如 product→products）；第 4 层查询缓存：高频重复查询走缓存，数据会变的表设 TTL。
【自查要点】你的链路是"LLM 生成 SQL 后先过校验器再执行"，不是直接跑生成的 SQL。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》Text-to-SQL 四层加固 / pack:rag-advanced-pack-->

**练习 4.2** 一批图文混排 PDF：有扫描件、中文密集、要本地化。选哪个解析工具？为什么不用 SimpleDirectoryReader？

【参考答案】选 MinerU（Magic-PDF，本地 L4）。四特征命中任一就走 MinerU：扫描件 / 中文 / 图文分离 / 要本地化。SimpleDirectoryReader 是 L1，只提文本、图片丢失、多栏跨列会串读。若是原生 PDF 的复杂表格、可付费、无本地化要求，才优先 LlamaParse。
【自查要点】你按"四特征命中任一走 MinerU"判，而不是一把梭用同一个解析器。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》图文混排 PDF 解析四工具 / pack:rag-advanced-pack-->

**练习 4.3** 用 CLIP 做文搜图，返回分数只有 0.24 左右，同事说"是不是坏了"。你怎么判断，中文查询要注意什么？

【参考答案】没坏。CLIP 文搜图正常分数就天然偏低（约 0.24），看排序不看绝对值。中文查询分数更低（约 0.26–0.28）、区分度差，中文必上两阶段 rerank；查询别超 77 token，超了会被截断。
【自查要点】你判好坏看的是排序和 rerank 后结果，不是拿 0.5 当阈值卡绝对分。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》多模态模型与分数速查 / pack:rag-advanced-pack-->

### B5 · RAG 基础 & 向量数据库（深潜 gate）

**练习 5.1**（关掉能力包自己写）手写一段 Embedding→余弦检索最小脚本，本 gate 的两条硬红线是什么？

【参考答案】① 索引与查询必须用同一个 Embedding 模型（同模型、同版本、同维度），跨空间检索分数看着正常但召回全错；② 用 `IndexFlatIP` 前，库向量和查询向量都要 `faiss.normalize_L2`——IndexFlatIP 算内积，只有 L2 归一化后内积才等价余弦相似度，漏掉任一侧，长文本因模长大被系统性高估、排序错乱。
【自查要点】你交的是可独立运行的脚本 + 一个已知答案集算出的 Recall@k 数值，不是让 Agent 打分。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》索引/查询一致性 · L2 归一化 / pack:rag-vectordb-pack-->

**练习 5.2** 意图识别，你的第一反应是"算文本相似度匹配一下"。课程实测这条路准确率多少？更优的怎么做？

【参考答案】文本匹配检索实测仅 79%（0.7945），是所有方法里最低。有标注训练数据时，用 Embedding 向量当特征训练 XGBoost/LightGBM 可到 98%~100%（XGB 达 100%、投票融合 100%）；无标注（零样本）用 Function calling 优先 GPT-4（98%）；Function calling 误判就上 Few-shot 错误样本修正（GPT-3.5 从 90% 提到 99%）。
【自查要点】你能背出"文本匹配 79% 是基线、别默认甩它"，并按"有没有标注数据"分流选型。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》借助 Embedding 提升意图识别准确率 / code:03_RAG基础 & 向量数据库应用/4. 借助Embedding提升用户意图识别准确率.ipynb-->

**练习 5.3** 给海量文本批量做 Embedding，怎么控 token 成本？三条输入红线是什么？

【参考答案】批量成本控制：数组一次调用批量嵌入、`tiktoken` 预估 token 与费用、超大数据先抽样、`encoding_format=float`。输入红线：input 非空且 ≤8192 tokens、`\n` 换行必须清洗成空格、索引/查询同模型同维度；`cl100k_base` 编码不可改。
【自查要点】你入库前先跑成本预估，不是直接把全量甩给 API。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》Embedding API 工程调用与成本控制 / pack:rag-vectordb-pack-->

### B6 · 企业级 RAG 评估

**练习 6.1** RAG 评估分哪两层？各看什么指标？

【参考答案】检索层：命中率 Hit Rate、召回率 Recall@k、精确率 Precision@k、MRR、NDCG（MRR/NDCG 是有序指标，看排序质量）。生成层：忠实度 Faithfulness、答案相关性 Answer Relevancy、事实依据性 Groundedness。忠实度是 RAG 的生命线，低忠实度等于幻觉严重，比答不出更危险。
【自查要点】你两层都评，不会只评检索或只评生成就下结论。
<!--src:A; ref:讲次《企业级 RAG 技术优化 & 评估》检索/生成双层评估 / pack:rag-basics-pack-->

**练习 6.2** 评估结果 Precision 高但 Recall 低，怎么开处方？如果是 Faithfulness 低呢？

【参考答案】Precision 高 Recall 低 = 召回太保守：调大 top_k、降 threshold、换更强 Embedding、或加关键词召回补漏。Faithfulness 低（幻觉）= 收紧 Prompt 三约束、降生成温度、明确"资料不足就说不知道"。注意别把 Recall 低误当"噪声多"去加 rerank，方向反了越调越差。
【自查要点】你能区分"Recall 低靠调 top_k/换 Embedding、Precision 低才靠 rerank/提 threshold"。
<!--src:A; ref:讲次《企业级 RAG 技术优化 & 评估》按症状开优化处方 / pack:rag-basics-pack-->

**练习 6.3** 没有人工标注，怎么半自动搭评估的标准答案集（golden set）？

【参考答案】用 `generate_question_context_pairs` 让 LLM 基于每个 chunk 反向生成代表性问题，得到"问题 → 该问题的标准来源 chunk"的问答对（qrels），这就是判分标准。也可用 LlamaIndex `RetrieverEvaluator.from_metric_names(["mrr","hit_rate"])` 批量评。
【自查要点】你没有因为"没人标注"就放弃评估。
<!--src:A; ref:讲次《企业级 RAG 技术优化 & 评估》golden set 半自动构建 / pack:rag-basics-pack-->

### B7 · GraphRAG 应用实战

**练习 7.1** GraphRAG 的 `settings.yaml` 两个模型块是什么？为什么不能拿 deepseek-r1 当文本模型？

【参考答案】两块都要有：`default_chat_model`（文本）与 `default_embedding_model`（向量），每块必填 `api_base` / `api_key`（引用 `.env` 的 `${GRAPHRAG_API_KEY}`）/ `model`。deepseek-r1 系列是推理模型，不可作文本模型接入 GraphRAG，会导致 index 失败；文本改用 Qwen2.5 / llama3.3 等。
【自查要点】你配的是双模型，不是只配一个 chat 就跑 index。
<!--src:A; ref:讲次《GraphRAG 应用实战》settings.yaml 双模型 / pack:rag-graphrag-pack-->

**练习 7.2** `graphrag index` 跑完，怎么判断成功？检索强依赖哪五类核心产物？

【参考答案】日志出 `All workflows completed successfully` 即成功，同时生成 `cache/`、`logs/`、`output/` 三个文件夹。`output/` 下五类核心 parquet：`entities`（实体，local 起点）、`relationships`（关系，图的边）、`communities`（社区结构，global 分层）、`community_reports`（社区报告，global 摘要来源）、`text_units`（文本单元，溯源片段）。
【自查要点】你验收看日志成功标志 + 五类 parquet 齐全，不是看有没有报错就完事。
<!--src:A; ref:讲次《GraphRAG 应用实战》索引产物与成功标志 / pack:rag-graphrag-pack-->

**练习 7.3** 问"这个公司整体在做什么方向"和问"张三和哪个项目有关"，分别用哪种 `--method`？口诀是什么？

【参考答案】整体/主题归纳类 → `global`（遍历社区报告 map-reduce 汇总）；具体实体/关系类 → `local`（从实体出发沿图谱找邻居）。另有 `drift`（探索式，local 基础上动态扩展）与 `basic`（传统向量检索对照基线）。口诀：问实体→local，问整体→global，探索→drift，对照→basic。
【自查要点】你选 method 按"问的是具体实体还是全局主题"分，不是默认全用 local。
<!--src:A; ref:讲次《GraphRAG 应用实战》检索命令与选型 / pack:rag-graphrag-pack-->

---

## 二、误区自查

对照下面 8 条，命中任何一条说明这一阶段还有洞，回看对应节点。

1. **以为索引和查询能用不同 Embedding 模型**。错。必须完全一致（同模型同版本同维度），换模型要重建整个索引，只换查询端会召回全错且分数看着正常极难查。回看 career-sprint#B3、career-sprint#B5。
   <!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》索引一致性红线 / pack:rag-basics-pack-->
2. **建了 `IndexFlatIP` 却忘了 `normalize_L2`**。IndexFlatIP 算内积不是余弦，不做 L2 归一化，长片段会因模长大被系统性排到前面，召回莫名其妙地差。库向量和查询向量两侧都要归一化。回看 career-sprint#B5。
   <!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》L2 归一化与余弦等价 / pack:rag-vectordb-pack-->
3. **意图识别/文本分类默认甩文本相似度匹配**。课程实测文本匹配仅 79%，有标注数据用 Embedding 特征 + XGB/融合能到 98%~100%。回看 career-sprint#B5。
   <!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》意图识别方法评分表 / pack:rag-vectordb-pack-->
4. **拿 CLIP 的绝对分数判检索好坏**。CLIP 文搜图正常就 0.24 左右、中文更低，看排序不看绝对值，中文必上两阶段 rerank，查询别超 77 token。回看 career-sprint#B4。
   <!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》多模态分数速查 / pack:rag-advanced-pack-->
5. **混合检索把顺序做反**。铁律：召回放大 → RRF 融合 → 精排全量 → 截断 `rerank_top_k` → 最后做阈值过滤。顺序反了（比如先阈值过滤）会把正确候选提前杀光。回看 career-sprint#B4、career-sprint#B6。
   <!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》混合检索顺序铁律 / pack:rag-advanced-pack-->
6. **Precision 高 Recall 低却去加 rerank**。rerank 治的是噪声多（低 Precision）；召回不足要调 top_k/threshold/换 Embedding，方向搞反越调越差。回看 career-sprint#B6。
   <!--src:A; ref:讲次《企业级 RAG 技术优化 & 评估》优化处方 / pack:rag-basics-pack-->
7. **LangGraph 加了 checkpointer 忘传 thread_id**。记忆会丢、会话会串。thread_id 是会话隔离的唯一标识，缺它断点也不生效。回看 career-sprint#B2。
   <!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》thread_id 铁律 / pack:langgraph-pack-->
8. **拿 deepseek-r1 当 GraphRAG 的文本模型、多 CSV 不统一 text 列**。前者 index 直接失败；后者缺列文件会静默加载为 None、丢整批信息还不报错。回看 career-sprint#B7。
   <!--src:A; ref:讲次《GraphRAG 应用实战》索引配置红线与静默坑 / pack:rag-graphrag-pack-->

---

## 三、迁移挑战

**挑战 1** 你在 B3/B5 手写过 FAISS 检索。现在要把纯向量检索升级成"BM25 + 稠密向量 + Reranker"的三路混合检索（B4/B6 的进阶栈）。列出正确的执行顺序和几个关键参数基线。

【参考答案】顺序：每路召回放大（`recall_top_k=30`）→ RRF 倒数排名融合（`reciprocal_rerank`，按排名而非原始分数合并）→ Reranker 精排全量召回（`top_n = recall_top_k`，不是只精排 top5）→ 截断到 `rerank_top_k=5` → 最后阈值过滤。度量用 IP（内积），必须配 embedding 的 L2 归一化，多路向量统一维度（课程基线 512）。这条链路把 B5 的"同模型 + L2 归一化"红线直接带进了进阶场景。
【自查要点】你能解释"为什么精排要 top_n=recall_top_k 而不是 top5"（源码明写：只精排 top5 会漏掉被 BM25 召回但向量排名靠后的正确候选）。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》三路召回融合与两阶段精排 / pack:rag-advanced-pack-->

**挑战 2** 你在 B1 用 `create_agent` 搭了会调工具的 Agent。现在要把 B3 搭的那条 RAG 问答接进这个 Agent，让它按需检索（Agentic RAG）。怎么接？和把检索硬塞进每轮 Prompt 有什么区别？

【参考答案】把整条 RAG 检索链路封装成一个 Tool 交给 `create_agent`，Agent 自己判断"要不要查、查什么"再调用（Agentic RAG）。区别于 Native RAG 的每轮都强行检索：Agentic 方式让模型对简单寒暄不检索、对知识型问题才检索，还能多轮改写查询。接进去后 B2 的多智能体编排里，这个 RAG Tool 可以作为其中一个数字员工的能力。
【自查要点】你把 RAG 当成"一个可调用的工具"而非"固定前置步骤"，并知道这为 E1 编译式 RAG 知识工厂、E2 客服中台埋了伏笔。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》Agentic RAG 封装为 Tool / pack:langchain-agent-pack-->

---

## 四、排错挑战

下面这段 RAG 检索脚本"能跑但答非所问"，问题不止一处。找出症状、根因、修法。

```python
# build_index.py —— 离线索引
from langchain_community.embeddings import HuggingFaceEmbeddings
import faiss, numpy as np

embed = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh")
doc_vecs = np.array([embed.embed_query(t) for t in docs]).astype("float32")
index = faiss.IndexFlatIP(doc_vecs.shape[1])
index.add(doc_vecs)                      # 直接入库

# query.py —— 在线查询（另一个文件，另一位同事写的）
from langchain_openai import OpenAIEmbeddings
q_embed = OpenAIEmbeddings(model="text-embedding-3-small")
q_vec = np.array(q_embed.embed_query(question)).astype("float32").reshape(1, -1)
scores, ids = index.search(q_vec, k=5)   # 分数看着挺正常，答案却对不上
```

【症状】检索能返回 5 条、分数也在正常区间（没有报错），但召回的片段和问题基本不相关，答案答非所问；越是长文档越容易被排到前面。

【根因】两处红线同时被踩。① 索引端用 `bge-small-zh`（维度 512）、查询端用 `text-embedding-3-small`（维度 1536），索引与查询不是同一个 Embedding 模型——向量落在不同语义空间，跨空间算相似度结果无意义，且维度都对不上；这是入门期最高频、最难查的坑，因为分数"看着正常"。② 建了 `IndexFlatIP` 却全程没做 `faiss.normalize_L2`，内积没归一化就不等价余弦，长文本因模长大被系统性高估、排到前面。

【修法】索引端和查询端换成同一个 Embedding 模型（同模型同版本同维度），换模型要重建整个索引不能只改查询端；入库前对库向量、检索前对查询向量都调 `faiss.normalize_L2`。修完跑 `check_index_consistency.py` 比对 index 与 query 的 embed_model / dim 与 `normalized` 标志，退出码非 0 不许交付。

```python
# 修正后：两端统一模型 + 两侧都归一化
embed = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh")   # 索引/查询同一个
doc_vecs = np.array([embed.embed_query(t) for t in docs]).astype("float32")
faiss.normalize_L2(doc_vecs)                 # 库向量归一化
index = faiss.IndexFlatIP(doc_vecs.shape[1])
index.add(doc_vecs)
# 查询端
q_vec = np.array(embed.embed_query(question)).astype("float32").reshape(1, -1)
faiss.normalize_L2(q_vec)                     # 查询向量也归一化
scores, ids = index.search(q_vec, k=5)
```
<!--src:A; ref:讲次《大模型 RAG 入门》数据流一致性 · L2 归一化与 FAISS 索引 / pack:rag-basics-pack-->

---

## 五、面试自测（15 题，每题两层追问）

> 这 15 题按当下大模型应用/Agent 岗真实考点排，答完自己过一遍两层追问。JD 高频关键词已对齐："熟练掌握 LangChain/LangGraph/LlamaIndex""有 RAG/Agent 实战并能工程优化""RAG 评估与检索优化"。

**面试 1** LangChain 从 0.x 到 1.0，Agent 的写法变了什么？为什么说是"大一统"？

【考察点】是否真用过 1.0、能否讲清架构收敛而非只会背 API。
【答题要点】0.x 按场景给不同构造器（`create_tool_calling_agent`+`AgentExecutor`、`create_react_agent`、`create_json_agent`、`initialize_agent`），1.0 全收敛到单一 `create_agent`，直接返回可调用体、ReAct 循环内建、结构化输出走 `response_format`。运行时角色也反转：LangGraph 从"外挂"变成底座。
【追问①】迁移时最容易报的参数错是什么？→ 0.x 的 prompt 里要留 `("placeholder","{agent_scratchpad}")`，1.0 不再需要，忘删会报参数错；入参也从 `{"input":...}` 变成 `{"messages":[...]}`。
【追问②】记忆怎么迁？→ 0.x 的 Memory 类换成 Checkpointer + thread_id，记忆由运行时按线程管，不再手动拼历史。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》0.x→1.0 迁移映射 / pack:langchain-agent-pack-->

**面试 2** LangChain 1.0 的中间件有几个生命周期钩子？为什么装配顺序是"安全优先"，洋葱模型怎么理解？

【考察点】是否理解中间件执行模型，能否讲出为什么顺序影响安全。
【答题要点】6 个钩子：`before_agent`、`before_model`、`wrap_model_call`、`after_model`、`wrap_tool_call`、`after_agent`。洋葱模型：`before_*` 先注册先执行，`after_*` 先注册后执行（先进后出）。安全优先指 PII 脱敏这类必须最先经过，鉴权放 `before_agent`，响应校验/HITL 放 `after_model`。
【追问①】`before_model` 和 `wrap_model_call` 都在模型调用附近，区别是什么？→ `before_model` 每次模型调用前跑（改上下文/压缩/脱敏），`wrap_model_call` 是洋葱包裹整次调用（动态换模型、动态提示词、故障回退）。
【追问②】限制工具调用次数放哪个钩子？→ `ToolCallLimitMiddleware`，中间件类型是 `wrap_tool_call`（工具调用包装）；`tool_name=None` 限所有工具、给定 `tool_name` 限单个工具，配额参数 `thread_limit`/`run_limit`。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解》6 钩子与编排黄金法则 / pack:langchain-agent-pack-->

**面试 3** LangGraph 里 State 的 Reducer 是干什么的？`operator.add` 和 `add_messages` 有什么区别？

【考察点】是否理解 LangGraph 状态合并机制这一底层原语。
【答题要点】节点只返回"要更新的键"，LangGraph 按每个 key 各自的 Reducer 合并进全局 State。不挂 Reducer = 新值覆盖旧值；`operator.add` = 无脑追加（列表拼接）；`add_messages` = 按消息 ID 合并（同 ID 替换、新 ID 追加）。
【追问①】为什么对话消息要用 add_messages 而不是 operator.add？→ operator.add 只能追加到末尾、改不了已存在的消息，人工要更新某条历史消息就必须 add_messages 的按 ID 合并。
【追问②】`MessageGraph` 是什么？→ 是 `StateGraph` 用 `Annotated[list[AnyMessage], add_messages]` 初始化的子类，专用于以消息为中心的流程。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》State 与 Reducer 对照 / pack:langgraph-pack-->

**面试 4** LangGraph 的检查点（Checkpointer）和 thread_id 是什么关系？生产该选哪种 Checkpointer？

【考察点】记忆与持久化的工程落地，是否踩过会话串线的坑。
【答题要点】加了 checkpointer 就必须传 thread_id（`config={"configurable":{"thread_id":...}}`），thread_id 是会话隔离唯一标识，换 thread_id = 换独立会话。四类选型：MemorySaver（实验，进程结束即失）、SqliteSaver（轻量本地，`from_conn_string` 要在 `with` 里用）、PostgresSaver（生产，支持并发规模化）、自定义（实现 BaseCheckpointSaver）。
【追问①】短期记忆和长期记忆在 LangGraph 里分别靠什么？→ 短期记忆靠 Checkpointer + thread_id（单会话内）；长期跨会话记忆走 Store（`InMemoryStore` + `namespace` + `put/search`）。
【追问②】断点（HITL）为什么会"不生效"？→ `compile(checkpointer=..., interrupt_before=[...])` 两个参数缺一断点就不生效，恢复要用 `astream(None, config)`，第一个参数 None 表示从断点继续而非从头重跑。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》Checkpointer 四类 · HITL / pack:langgraph-pack-->

**面试 5** 要编排一套多智能体系统，Supervisor、Network、Hierarchical 怎么选？

【考察点】多代理架构选型判断，直接对上 E2 客服中台、E3 数字员工工厂。
【答题要点】Supervisor（唯一 supervisor 节点中央路由，扩展性好）是大多数工作流编排首选；Network（每个代理都能决定调谁、都能 END）灵活但不随代理数扩展，适合少量代理；Hierarchical（分层多 supervisor）适合代理很多需分组。关键区别：Supervisor 是唯一决策者，Network 每个子代理都能自行 END。
【追问①】Supervisor 的关键代码件有哪些？→ `class AgentState(MessagesState): next: str` + `Router` TypedDict（`next: Literal[*options]`）+ `FINISH`（路由到 END）。
【追问②】用了 Subgraph，父图流式看不到子图内部输出怎么办？→ 子图 state 独立于父图，看子图内部要 `astream(..., subgraphs=True)`；state 有重叠要在调用前把父图输入转子图、返回前转回。
<!--src:A; ref:讲次《自主 Agent 开发框架 LangGraph》多代理五架构 / pack:langgraph-pack-->

**面试 6** 讲一下 RAG 的完整链路，离线和在线分别做什么？六大核心组件是什么？

【考察点】RAG 基本功，能否把两阶段数据流讲清而不是只说"检索加生成"。
【答题要点】离线索引：加载→切分→向量化→建 FAISS 索引→持久化；在线查询：查询向量化→Top-K 检索→阈值过滤→Token 预算拼接→组 RAG Prompt→生成。对应六大组件（文档加载、切分、Embedding、向量库、检索器、生成）。工程数据在各阶段以结构化对象传递。
【追问①】RAG Prompt 少写约束会怎样？→ 必须带三约束（只用参考资料/引用来源/资料不足就说不知道），缺一模型会用先验知识编造，RAG 退化成普通对话。
【追问②】上下文怎么拼才不炸窗口？→ 按相似度降序贪心累加逼近 `max_tokens` 就停，宁少拼低分片段不无限拼接。
<!--src:A; ref:讲次《大模型 RAG 入门》两阶段数据流与六大组件 / pack:rag-basics-pack-->

**面试 7**（累积复习必答）不借任何工具口述：为什么索引和查询必须用同一个 Embedding 模型？`IndexFlatIP` 前为什么要 L2 归一化？

【考察点】这是进入 C 阶段前的 gate 复习点，考的是"能不能从原理讲透两条最高频红线"。
【答题要点】① 不同 Embedding 模型产出的向量落在不同语义空间，跨空间做相似度检索结果无意义（分数看着正常但召回全错），换模型必须重建整个索引；② FAISS `IndexFlatIP` 算内积不是余弦，只有向量 L2 归一化（模长=1）后内积才数值等价余弦相似度，两侧都要归一化，漏一侧长文本因模长大被系统性高估、排序错乱。
【追问①】只把查询端换成别的模型省事，会发生什么？→ 向量空间不同，检索全错且分数正常极难发现，这是红线 1 头号坑。
【追问②】怎么机检这两条？→ 跑 `check_index_consistency.py` 比对 index/query 的 embed_model、dim、normalized 标志与样本向量实际模长是否≈1。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》数据流一致性 · L2 归一化 / pack:rag-vectordb-pack-->

**面试 8** 文档切分该怎么选切分器、`chunk_size` 和 `overlap` 怎么定？切太大或太小各有什么问题？

【考察点】切分是 RAG 效果的隐形天花板，考工程调参经验。
【答题要点】按内容形态选切分器（Token/句子/代码/Markdown/语义/句子窗口/结构切分器），`chunk_size` 太大则一个 chunk 混多个主题、检索命中噪声多、拼接吃 token；太小则语义被切断、上下文不足。`overlap` 用来避免句子在边界被切断。参数按数据形态调，不是固定值（GraphRAG CSV 示例 `size=300, overlap=100`）。切分粒度存疑时回切分步调参、重建索引后重评。
【追问①】切分前一定要做什么？→ 先清洗（去页眉页脚/水印/乱码/规整空白），垃圾进垃圾出，用 `repr()` 抽查有无隐藏空白。
【追问②】改了 chunk_size 后怎么确认更好了？→ 回 B6 双层评估重跑指标对照，别凭"看起来能答"。
<!--src:A; ref:讲次《大模型 RAG 入门》文档内容切分实战 / pack:rag-basics-pack-->

**面试 9** 做 Text-to-SQL，模型经常生成不存在的表名列名。生产上你怎么加固？

【考察点】结构化检索落地，直接对上 E4 数据分析智能体的 NL2SQL。
【答题要点】四层加固：① Schema 增强（注入字段业务含义与规则，消列名幻觉，性价比最高）；② Few-shot（3–5 个问题→SQL 示范）；③ SQL 校验器（校验→修复→重试环，模糊匹配修复表名列名，如 product→products，cutoff 0.6）；④ 查询缓存（高频查询走缓存，会变的表设 TTL）。核心：LLM 生成 SQL 后先过校验器再执行，不直接跑。
【追问①】校验器具体怎么修表名？→ 别名映射 / 模糊匹配（cutoff 0.6）/ 列名智能匹配 / 多表 JOIN 上下文推断。
【追问②】数据量小、就是要精确聚合，还用 Text-to-SQL 吗？→ 小数据可全量入内存用 PandasQueryEngine 做精确聚合排序；中大且已在数据库才用 Text-to-SQL。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》Text-to-SQL 四层加固 / pack:rag-advanced-pack-->

**面试 10** 混合检索（BM25 + 向量 + Reranker）的正确执行顺序是什么？为什么精排要"全量再截断"？

【考察点】进阶检索工程，考顺序意识和对 Reranker 的理解。
【答题要点】顺序铁律：召回放大（`recall_top_k=30`）→ RRF 倒数排名融合（`reciprocal_rerank`，按排名合并）→ Reranker 精排全量召回（`top_n=recall_top_k`）→ 截断 `rerank_top_k=5` → 最后阈值过滤。精排全量是因为只精排 top5 会漏掉被 BM25 召回但向量排名靠后的正确候选。度量用 IP，配 embedding L2 归一化，多路统一维度 512。
【追问①】Reranker 为什么值这份延迟？→ 它是 cross-encoder，把 query 和文档联合打分（bi-encoder 是各自独立编码），精度明显更高但更慢，是准确率换延迟的取舍，中文尤其不能省。
【追问②】顺序反了（先阈值过滤再融合）会怎样？→ 会把正确候选提前杀光，因为阈值在融合/精排前误杀低分但相关的片段。
<!--src:B; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》混合检索顺序 + cross-encoder 精排原理（一手：Reranker 为 cross-encoder，联合编码高精度高延迟）-->

**面试 11** 用 CLIP 做文搜图，检索分数很低，怎么判断是不是坏了？中文场景要注意什么？

【考察点】跨模态检索的经验坑，考"绝对分数不可信"这条反直觉红线。
【答题要点】CLIP 文搜图正常分数就偏低（约 0.24），看排序不看绝对值；中文查询更低（约 0.26–0.28）、区分度差，中文必上两阶段 rerank；CLIP 查询 token 上限 77，超长被截断，长问题要小搜大问解耦。生产走 CLIP/Qwen3-VL 粗召回 + Reranker 精排两阶段。
【追问①】服务端文搜图阈值怎么定？→ 按实测分位定（服务端观测约 0.43），别拍脑袋定 0.5。
【追问②】为什么不用 ColPali 这类多向量方案？→ ColPali 存储是 Qwen3-VL 的 30–250 倍，存储膨胀太重，故选稠密 + Reranker。
<!--src:A; ref:讲次《大模型 RAG 进阶讲解（LlamaIndex）》多模态模型与分数速查 / pack:rag-advanced-pack-->

**面试 12** 做意图识别，产品催着上线，你打算怎么选方法？有哪些方案、准确率大概什么水平？

【考察点】是否有"用数据压直觉"的选型意识，别一上来甩文本匹配。
【答题要点】按有没有标注数据分流：有标注 → Embedding 特征 + XGBoost/LightGBM，课程实测 98%~100%（XGB 100%、投票融合 100%）；无标注零样本 → Function calling 优先 GPT-4（98%，GPT-3.5 90%）；Function calling 误判 → Few-shot 错误样本修正（GPT-3.5 从 90% 提到 99%）。别默认用文本匹配检索，实测仅 79% 是最低。
【追问①】为什么文本匹配这么低还有人用？→ 它简单、零训练，只适合做快速基线或辅助信号，不该当主力。
【追问②】机器学习方法里 Embedding 起什么作用？→ 把每条文本的 Embedding 向量当特征喂给 RF/LGBM/XGB 训练，有标注数据时最强。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》意图识别方法评分表 / code:03_RAG基础 & 向量数据库应用/4. 借助Embedding提升用户意图识别准确率.ipynb-->

**面试 13** 怎么评估一个 RAG 系统好不好？只看"答得顺不顺"行不行？

【考察点】RAG 评估体系，2026 岗位高频（对应业界 RAG Triad）。
【答题要点】必须分两层量化，不能拍脑袋。检索层：Hit Rate、Recall@k、Precision@k、MRR、NDCG；生成层：忠实度 Faithfulness、答案相关性 Answer Relevancy、事实依据性 Groundedness。忠实度是 RAG 生命线，低忠实度=幻觉严重。这与业界常说的 RAG 三元组（忠实度/答案相关性/上下文相关性）一致。
【追问①】Precision 高 Recall 低怎么开处方？→ 召回太保守，调大 top_k、降 threshold、换更强 Embedding、加关键词召回；千万别去加 rerank（那治的是 Precision 低）。
【追问②】没人工标注怎么评？→ 用 `generate_question_context_pairs` 让 LLM 基于每个 chunk 反向生成问题得 golden set，半自动评。
<!--src:B; ref:讲次《企业级 RAG 技术优化 & 评估》双层评估指标（一手：RAG Triad = Faithfulness/Answer Relevance/Context Relevance）/ pack:rag-basics-pack-->

**面试 14** GraphRAG 相比普通向量 RAG 解决什么问题？部署有哪些坑？local/global/drift 怎么选？

【考察点】图谱式 RAG 的适用边界与工程细节，硬核加分项。
【答题要点】GraphRAG 把私域文档建成"实体-关系-社区"知识图谱，擅长需要跨文档全局归纳、多跳关系的问题，普通向量 RAG 只做片段相似召回答不好这类。部署坑：`settings.yaml` 要配 `default_chat_model` + `default_embedding_model` 双模型，deepseek-r1 不可作文本模型，输入默认仅 .txt/.csv，多 CSV 要统一 text 列否则静默丢数据。检索选型：问具体实体→local，问全局主题→global，探索→drift，向量对照→basic。
【追问①】index 跑完怎么验收？→ 日志出 `All workflows completed successfully`，`output/` 下五类核心 parquet（entities/relationships/communities/community_reports/text_units）齐全，local 和 global 各冒烟一次。
【追问②】增量更新为什么"改了内容不生效"？→ GraphRAG 按 `title`（=input 文件名）识别文档，同名文档改内容不触发更新，要时间戳命名（如 `merged_review_20250403.csv`）。
<!--src:A; ref:讲次《GraphRAG 应用实战》双模型配置 · 产物验收 · 检索选型 / pack:rag-graphrag-pack-->

**面试 15** 给海量文本批量做 Embedding，怎么控 token 成本？有哪些输入侧的坑？

【考察点】Embedding 工程与成本意识，规模化落地常被追问。
【答题要点】成本控制：数组一次批量调用、用 `tiktoken` 预估 token 与费用、超大数据先抽样试、`encoding_format=float`。输入红线：input 非空且 ≤8192 tokens、`\n` 换行必须清洗成空格（否则污染向量质量）、索引/查询同模型同维度、`cl100k_base` 编码不可改。大数据入库前先跑成本预估脚本，不要直接全量甩 API。
【追问①】不清洗换行会有什么后果？→ 换行符会降低 Embedding 向量质量，是最高频最难查的输入坑之一，入库前必清洗。
【追问②】选 Embedding 模型看哪些维度？→ 维度（越高越稳但存储/计算越贵）、最大序列长度（影响切分覆盖）、中英能力、价格、是否要本地化合规；中文入门可用 bge-small-zh / m3e-base，进阶 BGE-M3。
<!--src:A; ref:讲次《RAG 基础 & 向量数据库应用》Embedding 成本控制与输入红线 / pack:rag-vectordb-pack-->

---

## 六、模拟面试官指令

把下面这段整段发给一个大模型，让它当面试官，你只用语音或文字口述作答，不许翻手册、不许让它替你写代码。

```
你现在是某公司大模型应用开发岗的技术面试官，我是候选人，目标岗位关键词：LangChain/LangGraph、RAG 检索、向量库、RAG 评估、GraphRAG。
规则：
1. 一次只问一题，等我答完再追问，不要一次抛全部。
2. 先从这 7 个主题各抽一题：LangChain 1.0 create_agent 与迁移、LangGraph State/Reducer 与 thread_id、RAG 两阶段链路、Embedding 索引一致性与 L2 归一化、Text-to-SQL 加固、RAG 双层评估、GraphRAG 检索选型。
3. 每题我答完，你至少追问两层，专挑我讲得含糊的地方深挖（比如我只说"用同一个 Embedding 模型"，就追问"为什么、不同会怎样、怎么机检"）。
4. 如果我给出的数字或字段名不对（例如把文本匹配意图识别准确率说成 90%、把 IndexFlatIP 说成算余弦），当场指出并要我改正。
5. 全部问完给一份评估：哪些点答得扎实、哪些是背书没理解、哪些直接答错，并按"能跟做/能独立做/能教别人"给我每个主题打一档。
先说"准备好了吗"，我回复后开始第一题。
```

自评基线：能扛住每题两层追问且数字字段不被纠错，才算这一阶段过关；被面试官频繁纠正数字，回看对应节点。
<!--src:A; ref:career-sprint#B1 至 career-sprint#B7 节点验收口径-->

---

## 七、能力自评表

对每个技能点，诚实勾一档：能跟做（照着能包/课件能跑通）/ 能独立做（关掉能力包自己写得出）/ 能教别人（能讲清原理并纠别人的错）。gate 项必须到"能独立做"才算过 B5。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| `create_agent` 搭会调工具 + 带记忆的 Agent（B1） | ☐ | ☐ | ☐ |
| 中间件 6 钩子按安全优先编排、洋葱模型讲得清（B1） | ☐ | ☐ | ☐ |
| LangGraph State/Reducer 选型（add_messages vs operator.add）（B2） | ☐ | ☐ | ☐ |
| Checkpointer + thread_id 记忆隔离、HITL 断点恢复（B2） | ☐ | ☐ | ☐ |
| 多代理 Supervisor/Network 选型与编排（B2） | ☐ | ☐ | ☐ |
| 从零搭"加载→切分→向量化→检索→生成"RAG 闭环（B3） | ☐ | ☐ | ☐ |
| **【gate】关包手写 Embedding→检索脚本 + 算 Recall@k（B5）** | ☐ | ☐ | ☐ |
| 索引/查询同模型 + IndexFlatIP L2 归一化，能讲透原理（B3/B5） | ☐ | ☐ | ☐ |
| Text-to-SQL 四层加固、校验器兜列名幻觉（B4） | ☐ | ☐ | ☐ |
| 三路混合检索 + RRF 融合 + Reranker 全量精排顺序（B4/B6） | ☐ | ☐ | ☐ |
| CLIP 跨模态检索红线（77 token / 绝对分不可信 / 中文 rerank）（B4） | ☐ | ☐ | ☐ |
| RAG 检索层 + 生成层双层评估、按症状开处方（B6） | ☐ | ☐ | ☐ |
| GraphRAG 双模型配置、五类 parquet 验收、local/global 选型（B7） | ☐ | ☐ | ☐ |

<!--src:A; ref:career-sprint Stage B 各节点验收（deliverable/verify）-->

---

## 八、落地对照（JD 条目映射）

下面把当下大模型应用/Agent 岗 JD 里的高频条目，对到你这一阶段能拿出的产出和面试话术。左边是 JD 常见原话样式，右边是你怎么接。

**JD 条目 1**｜"熟练掌握 LangChain、LangGraph、LlamaIndex 等 Agent/RAG 框架"
→ 你的接法：B1 用 `create_agent` 搭过会调工具带记忆的 Agent、B2 用 LangGraph 搭过带检查点的 ReAct 与多代理、B4 用 LlamaIndex 做过 Text-to-SQL 与多模态检索。面试直接讲 0.x→1.0 迁移收敛、State/Reducer、Supervisor 编排，证明是用过不是听过。
<!--src:A; ref:career-sprint#B1 / career-sprint#B2 / career-sprint#B4-->

**JD 条目 2**｜"有知识库（RAG）实战经验，了解技术原理并能工程优化"
→ 你的接法：B3 从零搭端到端 RAG，B6 出双层评估报告 + 至少一条按指标落地的优化。面试讲两阶段数据流、六大组件、按"Precision 高 Recall 低"开处方，把"工程优化"落到具体指标变化，不空谈。
<!--src:A; ref:career-sprint#B3 / career-sprint#B6-->

**JD 条目 3**｜"熟悉向量数据库与 Embedding 检索，能做检索质量优化"
→ 你的接法：B5 深潜 gate 关包手写 Embedding→检索脚本并跑出 Recall@k，能讲透索引/查询同模型、IndexFlatIP L2 归一化两条红线。这是最能区分"真做过 vs 背过"的点，面试官一追原理就见真章。
<!--src:A; ref:career-sprint#B5-->

**JD 条目 4**｜"负责 NL2SQL / 数据问答 / 结构化检索相关能力"
→ 你的接法：B4 把 Text-to-SQL 用四层加固（Schema 增强 + Few-shot + 校验器 + 缓存）到能上生产，讲得出校验器怎么修表名列名幻觉。这条直通 career-sprint 后面的 E4 工业级数据分析智能体。
<!--src:A; ref:career-sprint#B4-->

**JD 条目 5**｜"了解 GraphRAG / 知识图谱问答，有多跳/全局归纳检索经验"（加分项）
→ 你的接法：B7 部署过 Microsoft GraphRAG、建过知识图谱索引、按问题类型选 local/global/drift，讲得清双模型配置红线和五类 parquet 产物。多数候选只会向量 RAG，这条是差异化加分。
<!--src:A; ref:career-sprint#B7-->

**JD 条目 6**｜"掌握 RAG 评估体系，能量化检索与生成质量"
→ 你的接法：B6 能背检索层（Hit Rate/Recall/Precision/MRR/NDCG）与生成层（忠实度/答案相关性/事实依据性）两套指标，对得上业界 RAG 三元组，并能半自动构建 golden set。用"我怎么证明我的 RAG 靠谱"这套话术压过只会说"效果不错"的候选人。
<!--src:B; ref:career-sprint#B6（一手：RAG Triad 忠实度/答案相关性/上下文相关性为业界通行评估口径）-->

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
