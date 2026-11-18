# 产品/业务岗 AI 全栈 · Stage A《Vibe 基础：会用 AI 编辑器做出能跑的东西》· 过关手册

> 练习向 · 给你自测用
>
> 本阶段你要跨过的三道关：
> - **A1 导学：环境配置与工具准备**：搭好 Cursor + Git + 网络，完成人生第一次代码提交
> - **A2 Vibe Coding 范式与快速上手（ChatGPT 克隆实战）**：跟做出一个前后端分离、能流式对话的应用，并自己改一处代码
> - **A3 Cursor 进阶 + MCP：做出数据分析助理**：会写 Cursor Rules、接 MCP、做出自然语言查数据库（NL2SQL）的助理
>
> 用法：先做「随堂练习」确认手熟；再用「误区自查」照镜子；「迁移挑战」检验你会不会换场景用；「排错挑战」练你独立止血；「面试自测 + 模拟面试官」帮你把话讲顺；最后用「能力自评表」和「落地对照」判断自己站到哪一格、能接什么活。
>
> 你不必先啃语法：这一阶段的关键是把工具链搭好、体验一次「AI 替你写、你来验收」的完整循环。所有题目取自 Stage A 实际课件与配套源码（chatgpt-clone、NL2SQLAgent、cursor-vibe-coding-pack），答案自包含，不依赖任何外部链接或视频回看即可对照。

---

## 一、随堂练习（动手小题）

### A1 · 导学：环境配置与工具准备
> 来源讲次：《导学：环境配置与工具准备》导学3 Git 安装与 GitHub 配置

**练习 A1-1｜第一次提交推送三连**
在本机建一个空目录，用命令行完成：初始化仓库 → 配置用户名和邮箱 → 新建一个 `hello.txt` → 提交 → 关联远程 GitHub 仓库并推送。把用到的命令按顺序写出来。

- 【参考答案】
  ```bash
  git init
  git config user.name "你的名字"
  git config user.email "你的邮箱"
  git add hello.txt
  git commit -m "first commit"
  git branch -M main
  git remote add origin <你的仓库地址>
  git push -u origin main
  ```
- 【自查要点】`add` 之后才能 `commit`（暂存区的概念）；`commit` 只动本地、`push` 才同步到远程；默认分支要用 `main` 不是过时的 `master`（课件安装选项里明确写「Default Branch → Override, use "main"」，并注明「新仓库默认分支为 master（已过时命名）」）。
  <!--src:A; ref:导学《环境配置与工具准备》导学3 Default Branch override main / Git 四大核心操作-->

**练习 A1-2｜说清四大核心操作 + Git 的四重职责**
课件把 Git 归纳为四个核心操作：`git commit` / `git branch` / `git push` / `git pull`。用一句话分别写出它们「在做什么」，再答出课件强调的 Git「四重职责」。

- 【参考答案】
  - `commit`：把变更固化为一个可追溯的版本节点；
  - `branch`：创建独立开发线，隔离不同功能的变更；
  - `push`：把本地变更同步到远程仓库（才能触发 CI/CD 或供团队 review）；
  - `pull`：从远程拉取最新代码并合并（协作时先 pull 再开发，避免冲突）。
  四重职责：**团队协作协议 / 交付审计链路 / 变更追踪系统 / CI-CD 触发源**。
- 【自查要点】课件反复强调的一句：Git 不是「代码备份」，而是工程协作的底层协议，是「AI 代码进入真实工程体系的唯一入口」。能背出四重职责，说明你理解到了它为什么排在环境配置第一站。
  <!--src:A; ref:导学《环境配置与工具准备》导学3 Git 四重职责 / 四大核心操作对应表-->

**练习 A1-3｜推送为什么老要输密码 / 认证失败**
你按课件装好了 Git，但每次 `push` 都被要求输账号密码，或直接认证失败。课件给的正解是配置什么？在 GitHub 那侧要先生成什么？

- 【参考答案】配置 **Git Credential Manager（GCM，凭证管理器）** 安全存储访问令牌，避免每次 push 都手输 Token；在 GitHub 侧先创建一个 **Personal Access Token（PAT，个人访问令牌）**，用它代替密码。
- 【自查要点】现在 GitHub 已不支持账号密码直接 push，必须用 PAT / GCM。这正是 A1 常见卡点「GitHub 推送失败 → 重配凭据」的根因。
  <!--src:A; ref:导学《环境配置与工具准备》导学3 GitHub Token 配置 / Credential Manager；product-fullstack#A1-->

---

### A2 · Vibe Coding 范式与快速上手（ChatGPT 克隆实战）
> 来源讲次：《Vibe Coding AI 编程范式与快速上手实践》及配套 chatgpt-clone 项目

**练习 A2-1｜画出这个项目的技术栈两张清单**
不看代码，凭课件的技术栈总览，分别写出前端和后端各用了哪些主要技术。

- 【参考答案】
  - 前端：React 19 + Vite 7 + TypeScript + Tailwind CSS 4 + Zustand（状态管理）+ react-markdown（Markdown 渲染）+ rehype-highlight（代码高亮）+ lucide-react（图标）。
  - 后端：FastAPI + Uvicorn + openai（Python SDK，调 DeepSeek）+ SQLAlchemy（异步）+ aiosqlite/SQLite + python-dotenv + sse-starlette。
- 【自查要点】「前后端分离」= 前端一个工程、后端一个工程，各自独立启动、用 HTTP 接口通信。前端 Vite 默认跑在 5173，后端 FastAPI 跑在 8000。记住这两个端口，排错时用得上。
  <!--src:A; ref:code:chatgpt-clone/frontend/package.json + backend/requirements.txt-->

**练习 A2-2｜后端起不来，先查哪一行配置**
你把前端起来了，界面能打开，但一发消息后端就 500。按 A2 课件，九成是什么没配好？具体是哪个文件里的哪一项？服务初始化会抛什么错？

- 【参考答案】九成是 `.env` 里的 **API Key 没配**。后端 `DeepSeekService.__init__` 读的是环境变量 `DEEPSEEK_API_KEY`，客户端 `base_url` 指向 DeepSeek 接口（host 为 `api.deepseek.com`），固定模型 `deepseek-chat`。Key 缺失时初始化直接 `raise ValueError("DEEPSEEK_API_KEY not found in environment variables")`。
- 【自查要点】环境变量（`.env`）不进 git（在 `.gitignore` 里），别人拉了仓库也要自己配一份。密钥属于后端机密，前端永远看不到它。
  <!--src:A; ref:code:chatgpt-clone/backend/app/services/deepseek_service.py DeepSeekService.__init__-->

**练习 A2-3｜跑通自检三条 + 两种模型**
写出「chatgpt-clone 本机跑通」的三条验收标准。另外，`/api/models` 接口返回了哪两个模型？它们的区别是什么？

- 【参考答案】三条验收：① 打开页面能对话；② 回复是**流式**逐字出现，不是一次性蹦出来；③ 代码块能正常高亮。附一张运行截图作凭据。两个模型：`deepseek-chat`（名字 DeepSeek V3，`thinking_enabled=false`）与 `deepseek-chat-thinking`（名字 DeepSeek R1，`thinking_enabled=true`，会展示思考过程）。
- 【自查要点】开启思考模式时，后端给请求加了 `extra_body={"thinking": {"type": "enabled"}}`；思考内容走 `reasoning_content`、正式回答走 `content`，前端靠每块的 `type` 字段（reasoning / content）区分要不要放进「思考」折叠块。
  <!--src:A; ref:code:chatgpt-clone/backend/app/main.py /api/models + services/deepseek_service.py stream_chat-->

**练习 A2-4｜手加一个「清空对话」按钮（深潜专属验收同款）**
A2 的徽章关卡是：关掉 Agent 代写，自己给页面加一个「清空对话」按钮并跑通，再用一句话说清数据如何从前端→后端→模型→前端流转。请先写出这条数据流。

- 【参考答案】你在页面点发送 → 前端 `streamChat()` 用 `fetch` 把 `POST /api/chat`（body 带 `messages`）发给后端 → 后端路由 `chat()` 返回 `StreamingResponse(generate_sse_stream(...), media_type="text/event-stream")` → `generate_sse_stream` 逐块调用 `deepseek_service.stream_chat()` → `DeepSeekService` 用 `AsyncOpenAI(base_url=api.deepseek.com)` 打到 DeepSeek，每收到一个 chunk 就 `yield {"type":"content","data":那一小块}` → 上层拼成 `data: {json}\n\n` 边生成边发回 → 前端 `response.body.getReader()` 逐块读、按 `data: ` 拆行、`JSON.parse` 后把新字拼到界面，于是你看到逐字蹦字。
- 【自查要点】徽章是**闭卷**：让 Agent 替你画链路、答代码追踪题不算过关。能不看代码讲清「前端发起 → 后端中转（藏密钥、落库）→ 模型服务封装真正调 API」这三层，才拿到 A 阶段徽章。
  <!--src:A; ref:code:chatgpt-clone/backend/app/api/chat.py + frontend/src/services/api.ts；product-fullstack#A2-->

---

### A3 · Cursor 进阶 + MCP：做出数据分析助理
> 来源讲次：《Cursor 应用进阶与 MCP 集成实战》及配套 NL2SQLAgent 项目 / cursor-vibe-coding-pack

**练习 A3-1｜Cursor 四大问答模式各是干嘛的**
课件速查表把 Cursor 归纳为四大问答模式，写出它们的名字和一句话用途，并说清「越危险越要」怎么用。

- 【参考答案】Ask/Chat（只读探索，搜库+答疑、不改任何代码）/ Plan（先规划后执行，写代码前先出实现计划让你审阅）/ Agent/Composer（自主执行，搜库+改文件+跑命令）/ Debug（拿运行时证据排错，不靠猜）。补充：Inline/Edit（光标处小改）、Composer（多文件协同）是编辑形态。最坏后果对照：只读问答顶多多花时间、可放心用；自主执行可能改错一批文件，越危险越要**先看它要动哪些文件再放行**。
- 【自查要点】速查表开头那句要记住：「名称随版本变化，认能力别认名字」。你要抓的是「会不会改代码 / 要不要先规划 / 要不要运行时证据」这四种能力，别被具体名称带偏。
  <!--src:A; ref:pack:cursor-vibe-coding-pack references/cursor-modes-cheatsheet.md 四大问答模式-->

**练习 A3-2｜Cursor Rules 四类型选型**
课件把 Cursor Rules 分成四类。写出四类的名字、生效方式、关键 Frontmatter 字段，并指出企业开发里「用得最多」的是哪一类。

- 【参考答案】
  - Always（全局强约束）：每次都注入，`alwaysApply: true`，只留 1–5 条核心原则；
  - Auto Attached（按文件注入）：命中 `globs`（如 `*.py`）的文件才激活，**企业开发用得最多、最合理**；
  - Agent Requested（按语义触发）：AI 靠 `description` 判断是否加载；
  - Manual（手动触发）：`@rule` 显式调用，无自动条件。
  规则文件放 `.cursor/rules/` 目录、`.mdc` 格式。
- 【自查要点】课件的核心认知：Rule Type 不是「好不好」，而是「在什么上下文注入最合适」。选错类型不是内容错，是注入时机错。还有一条纪律：等 Agent 反复犯同样问题时再写这条规则，别一开始就堆一堆。
  <!--src:A; ref:pack:cursor-vibe-coding-pack references/cursor-rules-types-reference.md 四类型总览-->

**练习 A3-3｜接 MCP 前必守的三条红线**
你要在 Cursor 里接一个 MCP Server。课件（cursor-vibe-coding-pack 的 MCP 集成约束）要求动手前守住哪三条红线？一句核心口径是什么？

- 【参考答案】三条：① **省 Token**，不盲目挂 MCP，每次外部调用都会把结果灌进上下文烧 Token，只接当前项目真正要用的 Server；② **审授权**，连第三方服务要走 Connect → OAuth 授权 → Authorize，授权即放权，先想清它能读写什么；③ **配 Rules**，用 `.cursor/rules` 框定「何时允许 Agent 调 MCP 工具」，别让它随意调；外加凭证走配置/环境变量、不硬编码。核心口径：**MCP + Rules = 高效可控的 Agent 工作流**（MCP 给能力，Rules 给约束）。
- 【自查要点】MCP = 模型上下文协议，把外部工具/数据源暴露给 Agent 调用，配置一般落在 `mcp.json`（或 `.cursor/mcp.json`）。接不对的三个后果：烧 Token、放权过大、上下文污染。
  <!--src:A; ref:pack:cursor-vibe-coding-pack capabilities/cursor-mcp-integration-guarding.md 红线清单-->

**练习 A3-4｜NL2SQL 六步链路与安全红线**
NL2SQLAgent 让 LLM 通过工具调用把自然语言变成 SQL。写出它的六步标准链路，以及必须写进系统提示的安全红线。

- 【参考答案】六步：① 理解问题 → ② 查看表结构（schema-first：先 `sql_db_list_tables` + `sql_db_schema`，不凭空猜表名列名）→ ③ 编写 SQL → ④ 校验 SQL（`sql_db_query_checker`，执行前必过）→ ⑤ 执行查询（`sql_db_query`）→ ⑥ 分析结果（中文回答并说明思路）。安全红线：**只执行 SELECT，禁止 INSERT/UPDATE/DELETE/DROP**；结果限制在 `top_k` 条（课件取 10）以内；优先选相关列、避免 `SELECT *`。出错走「分析原因 → 重写 SQL → 再校验 → 再执行」，用 `max_iterations`（课件取 6）设上限避免无限打转。
- 【自查要点】会话记忆用滑动窗口：按会话 ID 存历史，保留最近 `window_size` 轮（课件取 10，一问一答为一轮），既有多轮上下文又不无限膨胀。示例库有两张业务表：`sales`（product_name/category/quantity/price/sale_date/region）和 `employees`（name/department/position/salary/hire_date）。
  <!--src:A; ref:code:NL2SQLAgent/backend/app/core/agent.py + llm.py SQL_AGENT_SYSTEM_PROMPT；product-fullstack#A3-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句 X，说明你还没真正掌握 Y，去回看 Z。

1. **如果你以为「AI 写的代码不用看懂，能跑就行」** → 说明你还没意识到 A2 徽章关卡就是闭卷讲链路。快车道只让你拿产出，深潜徽章偏偏要你关掉 Agent 代写、自己改一处代码并口述数据流。回看 A2 chatgpt-clone 后端，先只跟一条 `/api/chat` 路由走一遍。

2. **如果你以为「Git 就是个代码备份工具，用到再学」** → 说明你还没掌握 Git 的四重职责（团队协作协议 / 交付审计链路 / 变更追踪系统 / CI-CD 触发源）。回看 A1 导学3「Git 不是代码备份，而是工程协作的底层协议」一节。

3. **如果你以为「API Key 写在前端方便点」** → 说明你还没理解「后端中转」的意义：密钥必须藏在后端 `.env`（且不进 git），前端只跟自己的后端说话，绝不把密钥送到浏览器。回看 A2 项目结构与 `DeepSeekService` 初始化。

4. **如果你以为「流式回复就是前端加载动画做得好看」** → 说明你还没掌握流式的本质：后端用 `StreamingResponse` 边生成边发（`yield f"data: {json}\n\n"`），前端 `getReader()` 边收边拼。回看 A2 后端 `generate_sse_stream` 与前端 `streamChat`。

5. **如果你以为「MCP 看到有用的就全挂上」** → 说明你还没守住 MCP 三红线。盲目挂会烧 Token、放权过大、污染上下文。回看 A3 MCP 集成约束：只接当前要用的、授权看清范围、用 Rules 框定调用时机。

6. **如果你以为「让数据助理能跑任意 SQL 才够强」** → 说明你把安全红线当成了限制。NL2SQL 一律「只 SELECT + 先校验 + 限行」，能写库的助理是隐患不是本事。回看 A3 系统提示里的安全红线与六步链路。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜用同样的 Vibe Coding 流程做一个待办清单应用**
课件里附带了一个纯前端的 `todo-app`（`index.html` + `app.js` + `styles.css`）。请**不照抄**，用 A2 学到的 Vibe Coding 流程从零指挥编辑器做一个待办清单：能添加、勾选完成、删除任务，刷新后数据还在（用浏览器 localStorage）。先让 Agent 陪你跑起来，再自己讲清「点击添加」时数据怎么进列表、怎么渲染回页面。

- 【怎么算过关】① 三个功能都能用；② 你能像 A2 那样口述一次「输入文字 → 点添加 → 出现在列表」的数据流；③ 用 Git 提交至少两次（初版 + 一次改进），说明你把 A1 的提交习惯迁移过来了。
- 【提示】这题故意去掉后端。先体会「纯前端也能是完整应用」，再对比 A2 的对话应用为什么一定要后端（藏密钥、落库、统一调模型）。
  <!--src:A; ref:code:chatgpt-clone 同级 todo-app（index.html/app.js/styles.css）；product-fullstack#A2-->

**迁移挑战 2｜把 NL2SQL 助理换一个业务库**
NL2SQLAgent 默认示例库有 `sales` 和 `employees` 两张表。假设你要把它接到自己的一张业务表（比如「订单表 orders」），你需要改动哪些地方、又有哪些**几乎不用动**？只描述改动点，不用真跑。

- 【怎么算过关】能指出：换 `DATABASE_URL` 指向你的库即可，`sql_db_list_tables` / `sql_db_schema` 会自动摸新表结构（schema-first 的好处就在这，Agent 不靠你手填表名列名）；系统提示里的安全红线、六步链路、`top_k`、`max_iterations` 全都不用动。能说清「为什么改动这么小」= 因为 Agent 是先查库结构再写 SQL，而不是把表结构写死在代码里。
  <!--src:A; ref:code:NL2SQLAgent/backend/app/db/connection.py + core/agent.py schema-first；product-fullstack#A3-->

---

## 四、排错挑战（给你一段带病的代码，修好它）

下面是一位同学写的 NL2SQL 助理的**系统提示 + 执行环节**简化版。他反馈：**「大部分查询没问题，但有一次我随口说『把库存为 0 的商品清掉』，助理真的生成并执行了 DELETE，把示例库的几行数据删没了。」** 请找出问题并修好。

```python
# 有问题的版本：系统提示没有只读红线，执行前也不校验
SYSTEM_PROMPT = """你是一个 SQL 数据库分析助手。
请根据用户的自然语言，直接生成 SQL 并执行，然后把结果告诉用户。
数据库方言: {dialect}
"""

async def handle(user_input):
    sql = await llm_generate_sql(user_input)   # 直接让模型生成 SQL
    result = db.run(sql)                        # 拿到就执行，什么都跑
    return result
```

- 【症状】助理会执行写操作（DELETE / UPDATE / DROP 等），本该只读的分析助理把示例库改坏了；且没有「先校验语法再执行」的步骤，偶尔还会因 SQL 写错直接把报错抛给用户。
- 【根因】两处缺失叠加：其一，**系统提示里没有「只执行 SELECT，禁止 INSERT/UPDATE/DELETE/DROP」这条安全红线**，模型被诱导就会生成写操作；其二，**生成的 SQL 不经 `sql_db_query_checker` 校验就直接执行**，跳过了六步链路里「校验 SQL」这一关，也没有脚本兜底拦截危险语句。
- 【修法】
  ```python
  # 修好的版本：红线写进系统提示 + 校验后再执行 + 脚本兜底
  SYSTEM_PROMPT = """你是一个专业的 SQL 数据库分析助手。
  请按步骤操作：查看表结构(sql_db_list_tables/sql_db_schema) → 编写 SQL →
  校验(sql_db_query_checker) → 执行(sql_db_query) → 用中文分析结果。

  数据库方言: {dialect}
  注意事项：
  - 只执行 SELECT 查询，禁止 INSERT/UPDATE/DELETE/DROP 等写操作
  - 查询结果限制在 {top_k} 条以内，除非用户明确要求更多
  - 优先选择相关列，避免 SELECT *
  - 查询出错就分析原因、重写 SQL、再校验再执行
  """

  async def handle(user_input):
      sql = await llm_generate_sql(user_input)
      # 执行前兜底：只放行 SELECT（对应 pack 的 scripts/sql_safety_check.py）
      if not is_select_only(sql):
          return "只支持只读查询（SELECT），已拦截写操作。"
      checked = db.run("sql_db_query_checker", sql)   # 先过语法校验
      if checked.has_error:
          sql = await llm_fix_sql(user_input, checked.error)  # 自愈重写
      return db.run("sql_db_query", sql)              # 校验通过再执行
  ```
- 【一句话总结】数据分析助理的安全靠两道闸：**系统提示里写死「只 SELECT + 限行 + 先校验」**，再加**执行前用安全脚本兜底拦截**。少一道，模型早晚会被一句话诱导去写库。这与 A3 课件的 NL2SQL 安全红线、六步链路一致。
  <!--src:A; ref:code:NL2SQLAgent/backend/app/core/llm.py SQL_AGENT_SYSTEM_PROMPT + pack:cursor-vibe-coding-pack cursor-nl2sql-assistant-building.md-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。
> 说明：题目按 2026 年产品/AI 应用岗真实高频方向校准，考点集中在 Git 基础、前后端分离、SSE 流式、密钥安全、Cursor 模式与 Rules、MCP、NL2SQL。

**面试 1｜Git 里 `add` / `commit` / `push` 分别在做什么？三者顺序能颠倒吗？**
- 考察点：对 Git 工作区 / 暂存区 / 本地仓库 / 远程四区的基本理解。
- 答题要点：`add` 把改动放进**暂存区**；`commit` 把暂存区固化成一个本地版本节点；`push` 把本地节点同步到**远程**。顺序不能颠倒：没 add 就没东西可 commit，没 commit 就没东西可 push。commit 只动本地，push 才上远程。
- 追问：「commit 完发现信息写错了、还没 push，怎么改上一条 commit 信息？」→ `git commit --amend`。
  <!--src:B; ref:git 官方四区模型（工作区/暂存区/本地仓库/远程）；product-fullstack#A1-->

**面试 2｜什么是前后端分离？本课项目是怎么解决跨域的？**
- 考察点：前后端分离架构直觉 + 跨域处理。
- 答题要点：前端（浏览器界面）和后端（提供数据/接口的服务）是两个独立工程，各自部署、通过 HTTP 通信。chatgpt-clone 就是前端 React（5173）+ 后端 FastAPI（8000）。跨域这里有两层解法：开发期 Vite 配了 `proxy`，把前端的 `/api` 请求转发到后端 8000，浏览器看到的是同源、不触发跨域；同时后端也配了 CORS 中间件，`allow_origins` 列了 `localhost:5173` 等来源，供直连时放行。
- 追问：「为什么前端 `API_BASE` 只写 `/api` 而不写后端完整地址？」→ 因为走 Vite 代理，`/api` 会被开发服务器转发到后端，既避免跨域、又让前端代码不写死后端地址，换环境只改代理配置。
  <!--src:A; ref:code:chatgpt-clone/frontend/vite.config.ts proxy + backend/app/main.py CORS；product-fullstack#A2-->

**面试 3｜SSE 是什么？为什么大模型对话应用都用它做流式输出？**
- 考察点：流式响应的协议选择（2026 年 AI 岗高频）。
- 答题要点：SSE（Server-Sent Events）是基于普通 HTTP 的**服务端单向推送**：客户端发一次请求，服务端保持连接、持续往回推数据块，格式是一行行 `data: ...\n\n`。大模型逐 token 生成、只需服务端往客户端单向推，正好命中 SSE；而且它是普通 HTTP，天然兼容代理/负载均衡，比 WebSocket 好扩展。本课后端就是 `StreamingResponse(..., media_type="text/event-stream")`，还加了 `X-Accel-Buffering: no` 关掉 Nginx 缓冲保证不攒着发。
- 追问：「本课前端是用浏览器的 `EventSource` 收 SSE 吗？」→ 不是。前端用 `fetch` + `response.body.getReader()` 手动读流，所以断线不会像 `EventSource` 那样自动重连，需要时得自己写重试。
  <!--src:B; ref:MDN Server-sent events；code:chatgpt-clone/backend/app/api/chat.py；product-fullstack#A2-->

**面试 4｜后端为什么要「中转」模型请求，而不是前端直接调模型 API？**
- 考察点：密钥安全与后端职责。
- 答题要点：前端直接调，API 密钥就会打包进浏览器代码，任何人打开开发者工具就能偷走盗刷。所以密钥必须藏在**后端** `.env`（且写进 `.gitignore` 不进 git）；前端只跟自己的后端说话，后端拿密钥去调模型再把结果流回。顺带后端还能统一落库对话、加权限、算用量。chatgpt-clone 的 `DeepSeekService` 就在初始化时读 `DEEPSEEK_API_KEY`，缺失直接抛错。
- 追问：「密钥怎样才算安全存放？」→ 放后端环境变量 / `.env` 且写进 `.gitignore`，绝不硬编码进代码、绝不提交仓库、绝不下发前端。
  <!--src:A; ref:code:chatgpt-clone/backend/app/services/deepseek_service.py（密钥仅后端）；product-fullstack#A2-->

**面试 5｜前端拿到 SSE 流后，是怎么一块块读出来并显示的？为什么要留 buffer？**
- 考察点：流式前端的实现细节（能答出细节是加分项）。
- 答题要点：`fetch` 拿到响应后用 `response.body.getReader()` 得到 reader，循环 `reader.read()` 直到 `done`；每块用 `TextDecoder` 解码，累加到 `buffer`，按 `\n` 切行、`buffer = lines.pop()` 把最后可能不完整的一行留着，以 `data: ` 开头的行 `JSON.parse(line.slice(6))` 后按 `type`（reasoning/content/done/error）分发，把新字拼到界面。
- 追问：「为什么要保留 buffer 里没处理完的最后一行？」→ 网络分块到达时一条 `data:` 行可能被从中间切断，直接解析会报错；留半行、等下一块拼全再解析才安全。
  <!--src:A; ref:code:chatgpt-clone/frontend/src/services/api.ts streamChat；product-fullstack#A2-->

**面试 6｜大模型 API 是按什么计费的？「token」是什么？**
- 考察点：token 与计费的基本认知（后续阶段会深挖，A 阶段要能答概念）。
- 答题要点：大模型按 **token** 计费，token 是模型处理文本的最小单位（约等于一个词/几个字），一次调用费用 ≈（输入 token + 输出 token）× 单价，输入输出通常单价不同。所以 prompt 越长、回复越长越贵。流式输出不改变计费，只是把同样的 token 边生成边发。
- 追问：「同样一句话，为什么中文和英文消耗的 token 数可能不一样？」→ 不同语言、不同分词器切出来的 token 数不同，中文常常一个字就占一个甚至多个 token。
  <!--src:B; ref:大模型 API token 计费通识（DeepSeek/OpenAI 定价页口径）；product-fullstack#A2-->

**面试 7｜Cursor 有哪几种问答模式？需求明确、要一次改多个文件时该用哪种？为什么不能无脑用自主执行？**
- 考察点：Cursor 模式选型直觉。
- 答题要点：四大能力是只读问答（Ask/Chat）、规划（Plan）、自主执行（Agent/Composer）、调试（Debug）；另有 Inline/Edit 行内小改、Composer 多文件协同。需求明确、要一次改多个文件用**自主执行/Composer**。不能无脑用自主执行，是因为它会搜库、改多文件、跑命令，最坏后果是改错一批文件，所以越危险越要先看它要动哪些文件再放行。
- 追问：「一个你能复现但读代码看不出原因的 Bug，该用哪种模式？」→ Debug 模式，核心是依赖运行时证据而不是凭猜去修。
  <!--src:A; ref:pack:cursor-vibe-coding-pack cursor-modes-cheatsheet.md；product-fullstack#A3-->

**面试 8｜Cursor Rules 有哪四种类型？企业里用得最多的是哪种？为什么 Always 要克制？**
- 考察点：Rules 四类型与注入时机。
- 答题要点：Always（每次注入，`alwaysApply: true`）、Auto Attached（命中 `globs` 文件才激活）、Agent Requested（AI 靠 `description` 语义触发）、Manual（`@rule` 手动）。企业里用得最多的是 **Auto Attached**，因为规则只对某一类文件成立、不污染其他上下文。Always 要克制（只留 1–5 条核心原则）是因为它每次都占上下文，写越多越挤 Token、越容易自相矛盾。
- 追问：「Rule Type 选错，错在哪？」→ 错在「注入时机」而非内容：同一条规则放错类型，要么每次都占上下文，要么该生效时没生效。
  <!--src:A; ref:pack:cursor-vibe-coding-pack cursor-rules-types-reference.md；product-fullstack#A3-->

**面试 9｜MCP 是什么？接一个 MCP Server 前你会守住哪几条红线？**
- 考察点：MCP 概念 + 集成安全。
- 答题要点：MCP（Model Context Protocol，模型上下文协议）通过协议把外部工具/数据源暴露给 Agent 调用，配置一般在 `mcp.json`（或 `.cursor/mcp.json`）。接入前三条红线：省 Token（只接当前项目真正要用的，别盲目全挂）、审授权（走 Connect → OAuth → Authorize，授权即放权，先看清它能读写什么）、配 Rules（用 `.cursor/rules` 框定何时允许调用）；凭证走配置/环境变量、不硬编码。核心口径：MCP 给能力、Rules 给约束，配套才可控。
- 追问：「接了 MCP 却不写 Rules 会怎样？」→ Agent 会随意调外部工具，既烧 Token 又放权失控；MCP 必须配 Rules 才可控。
  <!--src:A; ref:pack:cursor-vibe-coding-pack cursor-mcp-integration-guarding.md；product-fullstack#A3-->

**面试 10｜做一个自然语言查数据库（NL2SQL）的助理，你怎么保证它「查得准、不闯祸」？**
- 考察点：NL2SQL 的正确链路与安全设计。
- 答题要点：查得准靠 **schema-first**：先用 `sql_db_list_tables` + `sql_db_schema` 摸清真实表结构再写 SQL，不凭空猜表名列名；写完先 `sql_db_query_checker` 校验语法再 `sql_db_query` 执行，出错就分析原因、重写、再校验（用 `max_iterations` 设上限，课件取 6）。不闯祸靠安全红线写进系统提示：只执行 SELECT、禁止写操作、结果限 `top_k`（课件取 10）、避免 `SELECT *`，并用安全脚本在执行前兜底拦截。
- 追问：「用户说『帮我删掉过期数据』，一个合格的分析助理应该怎么回应？」→ 拒绝并说明只支持只读查询（SELECT），因为写操作被红线和执行前兜底拦截了。
  <!--src:A; ref:code:NL2SQLAgent/backend/app/core/agent.py + llm.py SQL_AGENT_SYSTEM_PROMPT；product-fullstack#A3-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请**只做考官、不做辅导**。

```
请你扮演一名技术面试官，围绕「用 AI 编辑器做出能跑的全栈应用与数据分析助理」这一阶段考我。规则如下：

【角色纪律】请只做考官、不做辅导。在我明确说「结束，请点评」之前，你不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，尽量贴近真实岗位：
1. Git 基础：add/commit/push/pull 的区别与顺序、本地 vs 远程、PAT/凭证管理、默认分支 main、四重职责。
2. 前后端分离：前端后端各管什么、端口(5173/8000)、Vite proxy 与 CORS、为什么密钥只能放后端。
3. SSE 流式：SSE 是什么、为什么大模型用它、StreamingResponse 与 text/event-stream、前端如何用 getReader 逐块读、buffer 为什么留半行。
4. Cursor 进阶：四大问答模式选型、Cursor Rules 四类型与注入时机、Always 为什么要克制。
5. MCP 集成：MCP 是什么、接入三红线(省Token/审授权/配Rules)、为什么必须配 Rules。
6. NL2SQL 助理：schema-first 六步链路、只读安全红线、top_k 与 max_iterations、如何拒绝写操作。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 如果我答「不会」，记录后直接下一题，不要讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现评价（答对/部分/未答）、暴露的薄弱点、以及建议我回看 Stage A 的哪一站（A1 环境与 Git / A2 跑通 ChatGPT 克隆 / A3 Cursor 进阶+MCP+NL2SQL）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件/Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 配好 Cursor + Git + 网络，完成首次提交推送 | ☐ | ☐ | ☐ |
| 说清 Git add/commit/push/pull 与本地-远程关系、四重职责 | ☐ | ☐ | ☐ |
| 用 PAT / 凭证管理器解决推送认证失败 | ☐ | ☐ | ☐ |
| 把 chatgpt-clone 在本机跑通（能流式对话、代码高亮） | ☐ | ☐ | ☐ |
| 讲清前后端分离：端口、Vite proxy 与 CORS、密钥为什么只放后端 | ☐ | ☐ | ☐ |
| 讲清 SSE 流式：StreamingResponse 边产边发、前端 getReader 逐块读 | ☐ | ☐ | ☐ |
| 关掉 Agent 代写，自己给 chatgpt-clone 加一处改动并口述数据流 | ☐ | ☐ | ☐ |
| 区分 Cursor 四种问答模式并按场景选对 | ☐ | ☐ | ☐ |
| 说清 Cursor Rules 四类型与各自的注入时机 | ☐ | ☐ | ☐ |
| 接 MCP 前守住三红线（省 Token / 审授权 / 配 Rules） | ☐ | ☐ | ☐ |
| 搭出 NL2SQL 助理：schema-first 六步链路 + 只读安全红线 | ☐ | ☐ | ☐ |
| 解释 token 计费的基本口径 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」，其中「口述数据流」「NL2SQL 六步链路与安全红线」这两项能到「能教别人」，你才真正拿下 A 阶段徽章，因为徽章关卡就是闭卷讲链路。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的入门级活儿，描述自包含。别小看，它们是你作品集和现金流的第一步。

1. **给自己/朋友做一个单页工具站。**
   一个纯前端小工具（单位换算、随机点名、简易记账、番茄钟），用 A2 的 Vibe Coding 流程指挥编辑器做出来，用 A1 的 Git 技能推到 GitHub Pages 上线。交付物 = 一个能打开的网址，也是你第一个可展示作品。

2. **给个人/小团队搭一个「专属对话助手」页面。**
   拿 A2 的 chatgpt-clone 改造：换欢迎语、换模型、加一句系统提示词（比如「你是我的英语陪练」），本机或小服务器跑起来给身边人用。密钥藏后端、流式回复、落库这套你已经会。

3. **做一个带流式打字机效果的展示 Demo。**
   很多人想要「像 ChatGPT 那样一个字一个字蹦出来」的效果但不会做。你懂 SSE 流式的前后端两端，可以把它封成一个可复用的小组件/小页面拿去接活。

4. **给业务方/小商家做一个「说人话查数据」的看板助理。**
   拿 A3 的 NL2SQLAgent 接他们的一张业务表，业务同事用中文问「上个月华东卖得最好的三个品类」，返回 SQL 结果和图表。你会 schema-first、只读安全红线、限行这套，正好是这类活的核心，也是产品/业务岗最能直接变现的能力。

5. **帮不懂技术的人把一个现成开源项目在他电脑上跑起来。**
   很多人「想用某个开源小工具但装不明白」。你会 Git 拉代码、看 README、配 `.env`、排「后端 500 多半是 Key 没配」这类常见坑，这就是一单「远程装机/环境搭建」的小服务。

6. **给小团队接一个内部工具的 MCP，让 AI 编辑器直接查内部系统。**
   你懂 MCP 三红线（省 Token、审授权、配 Rules），可以帮小团队把一个内部数据源接成 MCP，配好 `.cursor/rules` 限定调用时机，让他们在 Cursor 里安全地用自然语言问内部数据。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
