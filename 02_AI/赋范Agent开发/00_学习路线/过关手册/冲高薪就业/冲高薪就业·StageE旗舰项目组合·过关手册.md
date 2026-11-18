# 冲高薪就业 · Stage E 旗舰项目组合 · 过关手册

> 收官阶段。你要做的不是"看懂"四个项目，而是把它们做成能写进简历、扛得住技术面追问的作品集。
> 本册按求职向出题：每道面试题都配两层追问链，帮你预演真面试里"层层递进"的压力；答案后的隐藏注释是给你核对来源用的。
>
> 覆盖节点：career-sprint#E1（编译式 RAG 企业知识工厂 / Fufan_Chat）、career-sprint#E2（企业 AI 客服中台 / AssistGen）、career-sprint#E3（OPC 数字员工工厂 / FuFanManus，内容更新中）、career-sprint#E4（工业级数据分析智能体 / NL2SQL 可视化 Agent）。

---

## 一、随堂练习

每个旗舰项目 3 题，边跑项目边做。做完对照【参考答案】，再用【自查要点】确认自己是真会还是背下来了。

### E1 编译式 RAG 企业知识工厂（Fufan_Chat）

**练习 1｜跑通问答后回填红线自查**
把知识库问答跑通后，写出你在"加载 → 切分 → 向量化 → 检索 → 拼接 → 生成"这条链路里，实际守住了入门的哪几条工程红线，尤其是索引和查询这一环。

- 【参考答案】最该守住的一条：索引阶段和查询阶段必须用同一个 Embedding 模型，否则两边向量不在一个空间里，检索必然答非所问；其次是 chunk_size 别切太大（太大召回噪声多、命中率降）。检索答非所问时先查这两点，再谈要不要上重排。
- 【自查要点】你能说清"为什么必须同模型"到向量空间层面，而不是只背结论。
<!--src:A; ref:career-sprint#B3 checkpoint 与 blockers（索引/查询同 Embedding 模型、chunk_size）-->

**练习 2｜Milvus 选型理由**
交付方要问你"为什么用 Milvus 存向量，不直接用 Faiss？"用一句话答清。

- 【参考答案】Milvus 是 ZILLIZ 开源的向量数据库，底层索引/量化思路和 Faiss 相似，但可维护性更强、能做分布式部署，适合企业级知识库这种要长期运维、数据量会涨的场景；Faiss 更像单机库。Milvus 跑在 Docker 容器里，靠 docker compose 起服务。
- 【自查要点】你答的是工程可运维性差异，不是"Milvus 更快"这种含糊话。
<!--src:A; ref:career-sprint#E1《Ubuntu 系统安装部署 Milvus 向量数据库》（ZILLIZ 开源、可分布式、Docker 部署）-->

**练习 3｜交付前的评估报告**
知识工厂要交付，简历里也要写"附评估报告"。列出报告至少要覆盖的两层指标。

- 【参考答案】检索层：命中率、召回、精确、MRR；生成层：忠实度、相关性。别只看生成结果拍脑袋说好不好，必须落到这两层客观数值，优化前后要能对照出变化。
- 【自查要点】你能指出"生成层忠实度低"和"检索层召回低"要开的是不同处方。
<!--src:A; ref:career-sprint#B6 checkpoint（检索层命中率/召回/精确/MRR + 生成层忠实度/相关性）-->

### E2 企业 AI 客服中台（AssistGen）

**练习 1｜验证 DeepSeek 提示缓存**
用 DeepSeek 在线 API 做两轮相同前缀的对话，说出从返回里看哪两个字段判断缓存命中，以及命中规则。

- 【参考答案】看 usage 里的 `prompt_cache_hit_tokens` 和 `prompt_cache_miss_tokens`，分别是命中和未命中的 token 数。命中规则：只有两个请求的前缀内容从第 0 个 token 开始相同才算重复，中间开始的重复不能被命中。硬盘缓存自动运行，无需改代码，缓存有效期是几小时到几天。
- 【自查要点】你知道 DeepSeek 的 `/chat/completions` 是无状态接口，多轮要自己把历史拼全再传。
<!--src:A; ref:career-sprint#E2《DeepSeek 提示缓存功能介绍》（prompt_cache_hit/miss_tokens、前缀命中规则）-->

**练习 2｜给客服接联网检索（Function Calling）**
客服要能查实时信息，你打算用 Function Calling。说出 DeepSeek 两类模型里哪个能用、哪个不能，Ollama 本地模型呢。

- 【参考答案】DeepSeek-chat（V3）支持 Function Calling，DeepSeek-reasoner（R1）不支持。Ollama 起的 deepseek-r1 不支持 tools，但 qwen2.5、llama3.3 这类主流对话模型经 OpenAI 兼容接口是支持的。Function Calling 是三步手动流程：模型返回要调哪个工具和参数 → 开发者执行拿结果 → 把结果和原问题一起回喂模型出终答。
- 【自查要点】你没把"支持 Function Calling"误解成模型全自动调用，清楚执行那步在开发者手里。
<!--src:A; ref:career-sprint#E2《DeepSeek Function Calling 功能详解》（chat 支持/reasoner 不支持、Ollama deepseek-r1 不支持 tools）-->

**练习 3｜多轮记忆隔离**
客服中台多用户并发，Agent 要记住各自上文。用 LangGraph 做，必须配什么才能让会话之间不串。

- 【参考答案】必须给每个会话配独立的 `thread_id`，并挂上 Checkpointer；检查点按线程隔离，缺了 thread_id 就会记忆丢失或线程串了。
- 【自查要点】你能说清 Checkpointer 存的是短期记忆、按 thread_id 分线程这件事。
<!--src:A; ref:career-sprint#B2 blockers（thread_id 隔离检查点）+ langgraph-pack SKILL（checkpointer 必带 thread_id）-->

### E3 OPC 一人公司数字员工工厂（FuFanManus，内容更新中）

**练习 1｜项目来历**
面试官会问"这个数字员工工厂是你从零写的吗？"如实说清 FuFanManus 的技术来历。

- 【参考答案】FuFanManus 基于开源项目 Suna（kortix-ai/suna，Apache-2.0 许可）的后端重构 + 前端二次开发落地。为适配国内私有化部署，做了三处关键替换：移除 Supabase 云数据库，改本地 PostgreSQL；移除 Daytona 沙箱，换成国内可用的 Agent 沙箱；弃用原生大模型接口，由 Google ADK 框架统一接管模型服务。前端是 Next.js，做了汉化。
- 【自查要点】你讲得清"为什么要重构"（云依赖强、国内难落地），而不是只背替换清单。
<!--src:A; ref:career-sprint#E3《FuFanManus 系统架构及本地部署》Part 1（基于 Suna 重构、三处替换、Google ADK）-->

**练习 2｜多智能体先算账**
数字员工工厂要拆成多个数字员工协作。立项第一步先算什么账，默认选哪种协作模式。

- 【参考答案】先算 token 账：多智能体相比单 Agent 大约是 15 倍的 token 开销，这笔账立项时就要算清。协作模式拿不准默认 Supervisor（中央派单、约七成生产部署用它），再按任务性质向 Workflow / Hierarchical 特化，Swarm 留给"对话跟着人走"的窄场景。层级要设两层封顶并配护栏。
- 【自查要点】你知道 Supervisor 派单要收回全局控制权、Swarm 交出去不收回，这条控制权语义是命门。
<!--src:A; ref:career-sprint#E3 multi-agent-collaboration-pack SKILL（约 15 倍 token、默认 Supervisor 约七成、两层封顶）-->

**练习 3｜数字员工能干什么活**
说出 FuFanManus 预构建了哪些外部工具，以及怎么接自定义工具。

- 【参考答案】预构建 web search、computer use、browser use、code interpreter 等工具，跑在 Agent 沙箱里；扩展走自定义 MCP 服务接入或自定义外部工具服务接入。整套还含用户认证、大模型接入（在线 DeepSeek-chat/Qwen3/GPT + 本地 vLLM/Ollama）、Agent 沙箱管理几个模块。
- 【自查要点】你能把"工具在沙箱里执行"说清，而不是以为工具直接在主进程跑。
<!--src:A; ref:career-sprint#E3《FuFanManus 系统架构及本地部署》Part 1（功能模块与预构建工具清单）-->

### E4 工业级数据分析智能体（NL2SQL 可视化 Agent）

**练习 1｜Agent 挂了哪几把工具**
打开 `backend/vanna/src/Improve/agent/nl2sql_agent.py`，写出 create_agent 挂了哪 5 个工具，以及系统提示里强制的第一步是什么。

- 【参考答案】5 个工具：`get_all_tables_info`、`get_table_schema`、`check_mysql_version`、`validate_sql_syntax`、`execute_sql`。强制工作流第一步必须调 `check_mysql_version()` 确认版本，否则不知道能不能用 CTE / 窗口函数。用的是 LangChain 1.0 的 `create_agent(llm, tools, system_prompt, middleware)`。
- 【自查要点】你能说清"为什么第一步是探测版本"，联系到 5.7 和 8.0 的语法差异。
<!--src:A; ref:code:backend/vanna/src/Improve/agent/nl2sql_agent.py（5 工具 + create_agent）与 config/prompts.py（强制先 check_mysql_version）-->

**练习 2｜SQL 安全闸**
读 `database_tools.py` 的 `validate_sql_syntax`，写出它拦哪些危险关键字、为什么这是只读工作台的必要设计。

- 【参考答案】拦 INSERT、UPDATE、DELETE、DROP、ALTER、TRUNCATE、CREATE、GRANT、REVOKE，命中即报"安全风险"。还要求 SQL 必须以 SELECT 开头、括号配对。数据分析场景只该读不该写，这道闸挡住大模型误生成的写操作，避免动到业务库。`execute_sql` 里另有互斥锁强制串行执行，一次只跑一条，多条按分号自动拆。
- 【自查要点】你能指出"验证通过"只是基础语法检查（关键字/SELECT/括号），不等于 SQL 语义正确。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/database_tools.py（dangerous_keywords 九个、必须 SELECT、括号匹配、_sql_execution_lock 串行）-->

**练习 3｜RAG 三集合**
读 `rag_tools.py`，写出 Milvus 三个集合各存什么、检索时各取几条。

- 【参考答案】三集合：`vannaddl` 存表结构 DDL、`vannadoc` 存业务文档、`vannasql` 存问题-SQL 对。检索时 DDL 取 3 条（n_results=3）、文档取 3 条、历史相似 SQL 取 2 条（n_results=2），拼成上下文喂给模型生成 SQL。
- 【自查要点】你清楚这是"用 RAG 把库结构和历史 SQL 喂给模型"来提升 NL2SQL 准确度，不是靠模型凭空猜表名。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/rag_tools.py（vannaddl/vannadoc/vannasql、n_results=3/3/2）-->

---

## 二、误区自查

八条最容易在技术面被戳穿的错觉。逐条问自己"我是不是也这么想的"。

1. **"项目跑不通是环境玄学"**。多数跑不通是某个基础没扎实：检索答非所问回到 career-sprint#B3/B5，多轮记忆丢失回到 career-sprint#B2，SQL 报错回到 career-sprint#B4。Stage E 是复检场，不是新知识场。
<!--src:A; ref:career-sprint#E narrative（每站技能在这里被复检）-->

2. **"检索不准就是模型不行"**。先查索引和查询是否用了同一 Embedding 模型、chunk_size 是否过大，再谈重排，别急着甩锅模型。
<!--src:A; ref:career-sprint#B3 blockers-->

3. **"DeepSeek-R1 更强，用它做客服的工具调用"**。R1（reasoner）不支持 Function Calling，Ollama 起的 deepseek-r1 也不支持 tools；要工具调用用 DeepSeek-chat（V3）或 qwen2.5 这类模型。
<!--src:A; ref:career-sprint#E2《DeepSeek Function Calling 功能详解》-->

4. **"上提示缓存得改代码、中间重复也能命中"**。DeepSeek 硬盘缓存自动运行无需改代码，且只有前缀从第 0 个 token 起相同才命中，中间开始的重复命不中。
<!--src:A; ref:career-sprint#E2《DeepSeek 提示缓存功能介绍》-->

5. **"大模型能直接生成正确 SQL，不用校验"**。NL2SQL 项目里模型先探测 MySQL 版本、再过语法/安全校验、才执行；跳过校验极易踩 5.7 不支持的 CTE、或误生成写操作。
<!--src:A; ref:code:backend/vanna/src/Improve/config/prompts.py（强制工作流）-->

6. **"validate_sql_syntax 通过就说明 SQL 对了"**。它只做基础检查：危险关键字、是否 SELECT 开头、括号是否配对，不校验表名列名和语义，执行阶段还可能报表名错。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/database_tools.py（validate_sql_syntax 只基础验证）-->

7. **"多智能体越多越智能"**。多智能体大约 15 倍 token 成本，层级要两层封顶配护栏；拿不准默认单 Supervisor，不是拆得越碎越好。
<!--src:A; ref:career-sprint#E3 multi-agent-collaboration-pack SKILL-->

8. **"MySQL 都支持 WITH 和窗口函数"**。只有 8.0+ 支持 CTE（WITH）、RANK/ROW_NUMBER 窗口函数；5.7 得改子查询或用户变量模拟。项目里版本探测失败会默认按 5.7 保守处理。
<!--src:B; ref:MySQL 官方文档：CTE 自 8.0 起支持，窗口函数自 8.0 起支持；与课件 check_mysql_version 判据一致-->

---

## 三、迁移挑战

把一个项目里学到的做法，搬到另一个场景。写出方案要点即可，不必全实现。

**迁移 1｜把 NL2SQL 的"先探测再生成"搬到新数据库方言**
E4 里对 MySQL 做了"先 check_mysql_version、按版本裁剪语法"的设计。现在业务库换成 PostgreSQL，你要怎么改这套 Agent 才能安全生成 SQL？

- 【参考思路】新增一个探测 PostgreSQL 版本/能力的工具替换 check_mysql_version，把系统提示里的 5.7/8.0 语法约束替换成 PG 的方言约束（PG 支持 CTE 和窗口函数，但字符串拼接、LIMIT/OFFSET、类型转换语法与 MySQL 不同）；validate_sql_syntax 的危险关键字闸和"必须 SELECT 开头"可复用，串行执行锁可复用。RAG 三集合里的 DDL/历史 SQL 要换成 PG 的样本重新灌库。
- 【自查要点】你抓住了"方言约束在系统提示 + 探测工具，安全闸和 RAG 结构可复用"这条分层，而不是推倒重写。
<!--src:A; ref:code:backend/vanna/src/Improve/config/prompts.py + tools/database_tools.py（版本探测与语法约束分层）-->

**迁移 2｜把客服的语义缓存搬到数据分析智能体**
E2 客服中台用"Redis + 向量相似度阈值"做语义缓存，命中相似问题就不再调模型。E4 数据分析里高频出现相同/相似的问数问题，你要怎么把这套缓存搬过去、缓存什么？

- 【参考思路】把用户中文问题向量化后在缓存里搜相似问题，相似度超阈值（课件示例用 0.95）就直接返回缓存里的 SQL 和结果，不再走 Agent 全流程；miss 才执行 NL2SQL 链路并把"问题→SQL→结果"回写缓存。要注意数据库数据会变，缓存的是 SQL 结果要设失效时间或按表变更清理，缓存 SQL 语句本身则更耐用。
- 【自查要点】你意识到"缓存 SQL 文本"和"缓存查询结果"过期风险不同，没有把会变的结果无脑长期缓存。
<!--src:A; ref:career-sprint#E2《DeepSeek 提示缓存功能介绍》（Redis 语义缓存、相似度阈值 0.95）-->

---

## 四、排错挑战

一份完整排错。故障已埋进真实链路，按【症状】【根因】【修法】三段走。

**场景**：NL2SQL 可视化 Agent 刚部署完，`GET /health` 返回里 `agent_initialized` 是 true，服务能起。但实际提问时，一是 Agent 每次都回"未找到相关信息"，二是它生成的 SQL 频繁报表名/列名不存在的错。

- 【症状】
  1. 前端提问，事件流里 `get_table_schema` 这一步返回空或"未找到相关信息"。
  2. 模型只能靠猜表名写 SQL，`execute_sql` 阶段频繁报表不存在。
  3. `/health` 正常、LLM 能连通，看起来"服务是好的"，所以容易误判成模型能力问题。

- 【根因】Milvus 三个集合（vannaddl / vannadoc / vannasql）是空的——训练数据没灌。`get_table_schema` 走 `get_related_ddl`、`get_related_documentation`、`get_similar_question_sql` 去检索，集合为空自然全部返回空，模型拿不到表结构和历史 SQL，只能瞎猜。`agent_initialized: true` 只说明 Agent 装配成功，不代表 RAG 库里有数据，这是最迷惑人的地方。

- 【修法】用内置的 `nl2sql_training_data.py` 或训练数据接口（`POST /api/v1/training/add`，data_type 取 sql / ddl / documentation）把业务库的 DDL、业务文档、若干问题-SQL 示例灌进 Milvus 三集合；灌完再问，`get_table_schema` 能返回相关 DDL 和历史 SQL，模型据此生成的 SQL 表名列名就对得上了。核对方式：不是看接口 EXIT 正常，而是看事件流里 `get_table_schema` 真的返回了表结构。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/rag_tools.py + api_server.py（三集合检索、/api/v1/training/add、health agent_initialized）与 README 常见问题-->

---

## 五、面试自测

15 题，每题配【考察点】【答题要点】和两层【追问】链（面试官会顺着你的答案往下追）。先自己讲一遍，再对答案。

**面试 1｜说说你这个 NL2SQL 数据分析 Agent 的整体链路**
- 【考察点】能否把端到端链路讲成一个连贯故事，而不是罗列名词。
- 【答题要点】自然语言问题进来，Agent（LangChain 1.0 create_agent）先探测 MySQL 版本，再取全表信息、走 RAG 从 Milvus 三集合检索 DDL/文档/历史 SQL，模型据此在推理里直接写 SQL，过 validate_sql_syntax 校验，execute_sql 串行执行，结果回前端渲染成表格/图表；全程通过中间件把工具调用过程以 UI 事件流透明展示。后端 FastAPI 跑 8000，前端 React 跑 3000。
- 【追问一】为什么 SQL 让模型"在推理里直接写"，而不是再调一个"生成 SQL 的工具"？→ 系统提示明确要求模型参考 get_table_schema 返回的示例 SQL 直接在回复里写 SQL，没有单独的生成工具；多一层工具反而增加往返和出错面。
- 【追问二】这条链路里哪一步最容易成为准确率瓶颈？→ RAG 检索质量：三集合里 DDL 和历史 SQL 样本是否齐、是否覆盖了当前问题涉及的表，直接决定模型能不能写对表名列名。
<!--src:A; ref:code:backend/vanna/src/Improve/agent/nl2sql_agent.py + config/prompts.py + tools/rag_tools.py-->

**面试 2｜LangChain 1.0 的 create_agent 你怎么用的，和旧版 AgentExecutor 差别在哪**
- 【考察点】是否跟上了 LangChain 1.0 的写法，不是停在 0.x。
- 【答题要点】1.0 用 `create_agent(llm, tools, system_prompt, middleware)` 一步装配会调工具的 Agent，中间件按顺序挂（项目里挂了 trace 和 UI 事件两类）；旧版 0.x 的 AgentExecutor 写法已被这套大一统 API 取代。
- 【追问一】中间件的顺序有讲究吗？→ 有，项目里 UI 事件中间件放在 trace 之后，好利用 trace 的输出；中间件按列表顺序执行。
- 【追问二】要一行切换模型厂商怎么做？→ 用 init_chat_model 按厂商/模型名切换，Agent 组装代码不动。
<!--src:A; ref:code:backend/vanna/src/Improve/agent/nl2sql_agent.py（create_agent + middleware 顺序）与 career-sprint#B1-->

**面试 3｜你怎么防止大模型生成有破坏性的 SQL**
- 【考察点】安全意识和工程落地，而不是嘴上说"会注意"。
- 【答题要点】validate_sql_syntax 里维护一张危险关键字黑名单：INSERT、UPDATE、DELETE、DROP、ALTER、TRUNCATE、CREATE、GRANT、REVOKE，命中即拒；再强制 SQL 必须 SELECT 开头、括号配对。数据分析工作台只读不写，这道闸是硬约束。
- 【追问一】黑名单会不会漏？比如注释绕过？→ 系统提示里禁止 SQL 带注释，execute_sql 会先移除注释再按分号拆分执行，减少注释注入面；但黑名单本质是防御性的，生产还应叠加数据库只读账号权限做兜底。
- 【追问二】并发提问会不会把数据库连接搞坏？→ execute_sql 用互斥锁强制串行执行，一次只跑一条，多条按分号自动逐条执行。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/database_tools.py（validate_sql_syntax 黑名单 + _sql_execution_lock）-->

**面试 4｜MySQL 版本适配你是怎么处理的**
- 【考察点】是否理解"生成式 SQL 必须适配目标库能力"这件工程事。
- 【答题要点】强制工作流第一步调 check_mysql_version：主版本 ≥8 支持 CTE（WITH）和窗口函数（RANK、ROW_NUMBER），<8 则禁用，改子查询、GROUP BY + LIMIT 或用户变量模拟排名；系统提示把 5.7 禁用项写死。版本探测失败时保守按 5.7 处理，避免用了不支持的语法整条查询失败。
- 【追问一】为什么探测失败要"假设 5.7"而不是"假设 8.0"？→ 保守取能力更弱的一档，宁可不用高级语法也不生成会 100% 失败的 SQL。
- 【追问二】5.7 下要做 Top-N 排名怎么写？→ 用用户变量模拟（SET @rank，SELECT @rank:=@rank+1）或子查询，不能用窗口函数。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/database_tools.py（check_mysql_version）+ config/prompts.py-->

**面试 5｜企业知识库 RAG，索引和查询要注意什么最关键**
- 【考察点】RAG 最底层也最常被忽视的一致性问题。
- 【答题要点】索引和查询必须用同一个 Embedding 模型，否则向量不同空间、检索必错；chunk_size 别过大；检索答非所问先查这两点再谈重排。交付要带检索层（命中率/召回/精确/MRR）+ 生成层（忠实度/相关性）双层评估。
- 【追问一】纯文本相似度匹配意图够用吗？→ 课程实测纯文本匹配约 79%，换成 Embedding 特征 + ML/融合可到 98%~100%，别默认甩文本匹配。
- 【追问二】用 FAISS 的 IndexFlatIP 前要做什么？→ 做 L2 归一化，让内积等价余弦相似度，否则相似度失真。
<!--src:A; ref:career-sprint#B3/B5 blockers（同 Embedding 模型、文本匹配 79% vs 98~100%、IndexFlatIP 前 L2 归一化）-->

**面试 6｜多轮客服的会话记忆你怎么做的**
- 【考察点】LangGraph 记忆机制的实操理解。
- 【答题要点】用 Checkpointer 存短期记忆，每个会话配独立 thread_id，检查点按线程隔离；这样多用户并发不串、中断后能从检查点恢复。
- 【追问一】记忆丢了/线程串了最常见原因？→ 没给会话传 thread_id，或多会话共用了同一个。
- 【追问二】要跨会话记住用户长期偏好呢？→ 短期记忆用 Checkpointer + thread_id，长期记忆用 Store 按 namespace 存取，两者分开。
<!--src:A; ref:career-sprint#B2 blockers + langgraph-pack SKILL（Checkpointer + thread_id 短期、Store 长期）-->

**面试 7｜提示缓存能帮客服中台省什么，怎么用**
- 【考察点】成本/延迟优化的真实理解，不是背概念。
- 【答题要点】DeepSeek 硬盘提示缓存降低首 token 延迟和推理成本，自动运行无需改代码，按实际命中计费；从 usage 的 prompt_cache_hit_tokens / miss_tokens 看命中。前缀从第 0 token 相同才命中，所以 system + 稳定历史放前面最划算。
- 【追问一】多轮对话怎么最大化命中？→ 保持 system 和历史前缀稳定，把变化的新问题追加在最后，别打乱前缀。
- 【追问二】除了模型层缓存还能怎么省？→ 加应用层语义缓存（Redis + 向量相似度阈值），相似问题直接返缓存答案，不再调模型。
<!--src:A; ref:career-sprint#E2《DeepSeek 提示缓存功能介绍》（硬盘缓存、命中字段、语义缓存）-->

**面试 8｜Function Calling 是模型自动调工具吗**
- 【考察点】是否清楚"模型只给意图、执行在开发者手里"这条正确认识。
- 【答题要点】不是全自动。模型返回"要调哪个工具 + 参数"，开发者手动执行拿结果，再把结果和原问题回喂模型出终答，三步。DeepSeek-chat 支持，DeepSeek-reasoner 不支持。
- 【追问一】本地 Ollama 模型都能用吗？→ deepseek-r1 不支持 tools，qwen2.5、llama3.3 等经 OpenAI 兼容接口支持。
- 【追问二】和 MCP 是什么关系？→ MCP 把外部工具标准化接入，Function Calling 是模型发起调用的机制；MCP 让同一批工具能被不同 Agent 复用。
<!--src:A; ref:career-sprint#E2《DeepSeek Function Calling 功能详解》 + career-sprint#A7（MCP）-->

**面试 9｜数字员工工厂为什么不从零写，基于 Suna 重构合理吗**
- 【考察点】工程取舍判断：复用成熟开源 vs 从零造。
- 【答题要点】Suna 后端架构成熟、前端精美，但强依赖 Supabase、Daytona 等云服务，国内私有化难落地。重构做三处替换：Supabase→本地 PostgreSQL、Daytona→国内 Agent 沙箱、原生模型接口→Google ADK 统一接管，再把前端汉化二开。复用成熟骨架 + 替换不可落地的依赖，是合理的工程决策。
- 【追问一】为什么把模型接入交给 Google ADK 而不留原生接口？→ 原生接口手动处理对话/函数调用流程代码繁多、学习和维护成本高，ADK 统一接管更省。
- 【追问二】选开源底座要看什么？→ 许可证（Suna 是 Apache-2.0，允许商用/修改/分发）、架构可替换性、社区活跃度。
<!--src:A; ref:career-sprint#E3《FuFanManus 系统架构及本地部署》Part 1-->

**面试 10｜什么时候该拆多智能体，拆了怎么控成本**
- 【考察点】多智能体的选型判断和成本敏感度。
- 【答题要点】不是越多越好：多智能体约 15 倍 token 成本。拿不准默认 Supervisor（中央派单、约七成生产部署），任务是固定流程用 Workflow，要分层用 Hierarchical 但两层封顶配护栏，对话跟人走才用 Swarm。控制权语义是命门：Supervisor 派单收回全局，Swarm 交出不收回。
- 【追问一】Agent 命名有什么坑？→ 交接工具/Agent.name 一律用英文，撞名会导致 handoff 崩。
- 【追问二】handoff 没真正转过去怎么排查？→ 确认 handoff 是真的工具调用而非口头转接，active_agent 要跨轮持久化，否则多轮会丢当前接待者。
<!--src:A; ref:career-sprint#E3 multi-agent-collaboration-pack SKILL（15 倍 token、四模式选型、控制权语义、命名英文）-->

**面试 11｜把 Agent 从 notebook demo 变成能上线的服务，你做了哪些**
- 【考察点】服务化/工程化落地能力（岗位高频）。
- 【答题要点】用 FastAPI 把 Agent 封 HTTP 服务，设计接口（NL2SQL 项目里有 /api/v1/chat、/chat/stream 流式、训练数据增查删、数据源测试/连接），加 /health 健康检查和 /docs 接口文档；再容器化（Docker + compose），评估用 LangSmith 做 trace。
- 【追问一】/health 返回 agent_initialized: true 就代表能用了吗？→ 不代表，它只说明 Agent 装配成功，RAG 库空、数据库不通照样答不出，得看真实链路是否返回数据。
- 【追问二】流式接口对客服体验的意义？→ SSE 流式让用户更快看到第一个字，配合提示缓存进一步压低首 token 延迟。
<!--src:A; ref:code:backend/vanna/api_server.py（/api/v1/chat、/chat/stream、/health）与 career-sprint#D1/D2-->

**面试 12｜你简历里最能打的差异化是什么**
- 【考察点】能否讲清自己相对"只会调 API"的候选人的稀缺点。
- 【答题要点】微调实战记录：用 Llama-Factory / Unsloth 跑通 LoRA/QLoRA 全流程，含显存估算表、方法选型理由、训练配置、领域评测指标和过拟合排查。大多数候选人只会调 API，会把开源模型微调到能用的人稀缺。
- 【追问一】7B 模型 QLoRA INT4 大概要多少显存，你怎么算的？→ 按课程显存表：INT4 量化后权重约 3.5GB 量级，加上 LoRA 适配器、激活和优化器状态，单卡消费级（如 24GB）可跑；实际以手算显存表为准，全量微调则显存需求高一个量级。
- 【追问二】SFT 和 PPO/DPO/GRPO 分别学什么？→ 学知识用 SFT，学偏好/对齐用 PPO/DPO/GRPO；先定目标再选方法。
<!--src:A; ref:career-sprint#C1/C5 checkpoint（显存估算、方法选型、SFT vs 强化微调）-->

**面试 13｜你这套 RAG 怎么证明"做得好"，不是拍脑袋**
- 【考察点】评估驱动而非感觉驱动。
- 【答题要点】双层评估：检索层看命中率、召回、精确、MRR，生成层看忠实度、相关性；按症状开处方，比如 Precision 高 Recall 低说明召回不足要扩检索或降阈值。优化前后指标要能对照出变化，报告随项目交付。
- 【追问一】召回高但忠实度低说明什么？→ 检索到的内容够，但生成没老实基于检索、有幻觉，要改提示约束"只依据检索内容作答"或加引用。
- 【追问二】怎么给检索层建"已知答案集"来算 Recall@k？→ 准备一批问题和它们应命中的文档标注，跑检索看前 k 个是否包含标注文档。
<!--src:A; ref:career-sprint#B5/B6（Recall@k、双层评估指标与处方）-->

**面试 14｜结合当下岗位：招聘要求常写"Text2SQL / NL2SQL 经验"，你怎么证明**
- 【考察点】岗位校准——把 JD 关键词对应到可展示的证据。
- 【答题要点】拿 E4 项目说：不只是跑通脚手架，我在脚手架之外亲手实现了 NL2SQL→执行→可视化的核心链路，能讲清版本探测、RAG 增强表结构、SQL 安全校验、串行执行这几处工程决策，并通过项目自检和 git 提交。这比"用过某低代码 Text2SQL 工具"更能扛追问。
- 【追问一】面试官问"准确率多少"你怎么答？→ 说清准确率取决于 RAG 样本覆盖和评估口径，给出你用的评估方式（标注问题-SQL 集比对执行结果），而不是报一个没有出处的数字。
- 【追问二】上生产还差什么？→ 数据库只读账号、查询超时/行数限制、结果脱敏、慢查询监控，这些安全和治理项。
<!--src:C; ref:external（岗位校准题，课件无对应章节；技术锚点见 code:backend/vanna 与 career-sprint#E4）-->

**面试 15｜结合当下岗位：JD 常要求"LangChain/LangGraph 实战经验"，你怎么把项目讲成经验**
- 【考察点】岗位校准——把框架关键词落到具体项目细节。
- 【答题要点】LangChain 1.0：用 create_agent 装配带中间件的 NL2SQL Agent（E4）；LangGraph：客服中台多轮用 Checkpointer + thread_id 做会话隔离与断点恢复（E2）、数字员工工厂用图编排做多智能体协作（E3）。讲的时候把"我在哪个项目用它解决了什么具体问题"说清，而不是罗列 API。
- 【追问一】LangChain 和 LangGraph 什么时候各用哪个？→ 单 Agent 应用层用 LangChain create_agent；要有状态图、多节点路由、断点、多智能体编排用 LangGraph 的 StateGraph。
- 【追问二】面试官要你现场画 Supervisor 多智能体图？→ AgentState 带 next 字段，Supervisor 节点用 Router（Literal 含各 worker 和 FINISH）决定路由，条件边按 next 分发，worker 执行完回 Supervisor 收回控制权。
<!--src:C; ref:external（岗位校准题，课件无对应章节；技术锚点见 langgraph-pack SKILL 与 career-sprint#B1/B2）-->

---

## 六、模拟面试官指令

把下面这段发给你的 Agent，让它扮演技术面试官，对你做一轮真实压力面。你只用嘴答（或语音转写），不许查资料、不许让它替你写代码。

```text
你现在是一家做企业 AI 应用公司的资深面试官，我在面大模型 Agent 开发岗。
请基于我做过的四个项目（编译式 RAG 企业知识工厂 Fufan_Chat、AI 客服中台 AssistGen、
OPC 数字员工工厂 FuFanManus、NL2SQL 数据分析 Agent）对我做一轮 30 分钟技术面：

规则：
1. 一次只问一个问题，等我答完再追问，不要一次抛一堆。
2. 每个主问题至少追问两层，专戳我讲得含糊或有漏洞的地方
   （比如我说"用了 RAG"，你就追"索引和查询用的同一个 Embedding 模型吗？为什么必须同一个？"）。
3. 覆盖这些必考点：RAG 索引/查询一致性与双层评估、LangChain 1.0 create_agent 与中间件、
   LangGraph 的 thread_id 会话隔离、NL2SQL 的版本探测与 SQL 安全校验、
   Function Calling 哪些 DeepSeek 模型支持、提示缓存命中规则、多智能体成本与协作模式选型、微调差异化。
4. 如果我报了一个数字（显存、准确率、召回），追问出处和口径，没出处就指出来。
5. 面完给我一张评分表：每个考点标"能跟做/能独立做/能教别人"三档，并指出最该回炉的两个点，
   对应到本路径哪个节点（用 career-sprint#节点id 标注）。

现在从第一个问题开始。
```

用完让它按第 5 条给你回炉清单，对照下面自评表校准。

---

## 七、能力自评表

逐行打勾。三档：能跟做（照着课件/Agent 能跑通）、能独立做（关掉参考自己能做出）、能教别人（能讲清原理并纠错）。凡是停在"能跟做"的行，就是你面试最容易被追问穿的地方。

| 技能点 | 能跟做 | 能独立做 | 能教别人 | 回炉锚点 |
|---|:---:|:---:|:---:|---|
| 部署跑通企业知识库 RAG 问答并出评估报告 | ☐ | ☐ | ☐ | career-sprint#E1 |
| 讲清索引/查询同 Embedding 模型的必要性 | ☐ | ☐ | ☐ | career-sprint#B3 |
| 检索层 + 生成层双层评估与按症状开处方 | ☐ | ☐ | ☐ | career-sprint#B6 |
| 部署跑通客服中台多轮对话 + 工单流转并服务化 | ☐ | ☐ | ☐ | career-sprint#E2 |
| DeepSeek 提示缓存命中规则与语义缓存落地 | ☐ | ☐ | ☐ | career-sprint#E2 |
| 说清哪些 DeepSeek/Ollama 模型支持 Function Calling | ☐ | ☐ | ☐ | career-sprint#E2 |
| LangGraph 用 thread_id + Checkpointer 隔离会话 | ☐ | ☐ | ☐ | career-sprint#B2 |
| 讲清 FuFanManus 基于 Suna 重构的三处替换 | ☐ | ☐ | ☐ | career-sprint#E3 |
| 多智能体成本账与四模式选型（默认 Supervisor） | ☐ | ☐ | ☐ | career-sprint#E3 |
| create_agent 装配带中间件的 Agent（LangChain 1.0） | ☐ | ☐ | ☐ | career-sprint#B1 |
| NL2SQL 的版本探测 + RAG 三集合 + SQL 安全校验 | ☐ | ☐ | ☐ | career-sprint#E4 |
| 脚手架之外亲手实现 NL2SQL→执行→可视化核心链路 | ☐ | ☐ | ☐ | career-sprint#E4 |
| 把 Agent 用 FastAPI 服务化并容器化上线 | ☐ | ☐ | ☐ | career-sprint#D1 |
| 讲清微调差异化（显存账、方法选型、领域指标） | ☐ | ☐ | ☐ | career-sprint#C1 |

---

## 八、落地对照

把你的作品集对准招聘要求。左列是 JD 里常见的条目措辞，中列是你能拿出的证据，右列是被追问时该讲的深度。面试前照着过一遍，确保每条 JD 你都有项目 + 细节接得住。

**1. "熟悉 RAG / 向量数据库，有企业知识库落地经验"**
→ 证据：编译式 RAG 企业知识工厂（Fufan_Chat），Milvus 存向量、附检索/生成双层评估报告。
→ 讲深度：索引/查询同 Embedding 模型、chunk 选型、Milvus 相比 Faiss 的可运维/分布式优势、双层指标与处方。
<!--src:A; ref:career-sprint#E1 + B3/B5/B6-->

**2. "掌握 LangChain / LangGraph，能搭生产级 Agent"**
→ 证据：NL2SQL Agent 用 LangChain 1.0 create_agent + 中间件（E4）；客服中台用 LangGraph 多轮记忆（E2）。
→ 讲深度：create_agent 与中间件顺序、thread_id 会话隔离与断点恢复、图编排多智能体。
<!--src:A; ref:code:backend/vanna/src/Improve/agent/nl2sql_agent.py + career-sprint#B1/B2-->

**3. "有 Text2SQL / NL2SQL 或数据分析智能体经验"**
→ 证据：工业级数据分析智能体，脚手架之外亲手实现 NL2SQL→执行→可视化核心链路，通过自检 + git 提交。
→ 讲深度：MySQL 版本探测与语法适配、RAG 三集合增强、危险关键字安全闸、串行执行锁。
<!--src:A; ref:code:backend/vanna/src/Improve/tools/database_tools.py + rag_tools.py（career-sprint#E4）-->

**4. "了解 Agent 服务化 / 容器化部署"**
→ 证据：Agent 用 FastAPI 封 HTTP 服务（/health、/docs、流式接口），项目可容器化交付。
→ 讲深度：健康检查语义、SSE 流式与首 token 延迟、Docker + compose 交付。
<!--src:A; ref:code:backend/vanna/api_server.py + career-sprint#D1/D2/D3-->

**5. "有大模型微调经验（LoRA / QLoRA 优先）"——差异化重点**
→ 证据：微调实战记录，Llama-Factory / Unsloth 跑通 LoRA/QLoRA，含显存估算表、领域评测指标、过拟合排查。
→ 讲深度：手算显存、方法选型（全量/LoRA/QLoRA INT4）、SFT vs PPO/DPO/GRPO 的适用。
<!--src:A; ref:career-sprint#C1/C3/C5-->

**6. "熟悉多智能体编排 / 通用智能体（Manus 类）"**
→ 证据：OPC 数字员工工厂（FuFanManus），基于 Suna 私有化重构，一句话调度数字员工。
→ 讲深度：多智能体成本账、四模式选型与控制权语义、沙箱化工具执行、Google ADK 统一接管模型。
<!--src:A; ref:career-sprint#E3 + multi-agent-collaboration-pack SKILL-->

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
