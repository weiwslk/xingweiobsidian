# 零基础超级个体 · Stage C《上手基座智能体》· 过关手册

> 练习向 · 面向你本人
>
> 这一阶段你要跨过的三道关：
> - **C1 Claude Code 快速入门（含 Fufan-CC Flow 可视化）**：会用 Claude Code 完成「对话 → 改码 → 提交」的完整循环
> - **C2 给你的 Agent 装上课程全家桶**：一键装入能力包，让 Agent 从此按课堂工程方法干活
> - **C3 Cursor 进阶 + MCP：做出数据分析助理（并行支线）**：会写 Cursor Rules、接 MCP、做出自然语言查库的助理
>
> 用法：先做「随堂练习」确认手熟；再用「误区自查」照镜子；「迁移挑战」看你会不会换场景用同一套方法；「排错挑战」练你独立止血；「面试自测 + 模拟面试官」帮你把嘴皮子练顺；最后用「能力自评表」和「落地对照」判断自己站到哪一格、能接什么活。
>
> 本手册题目取自 Stage C 实际课件与项目源码，答案自包含，不依赖任何外部链接或视频回看即可对照。每题答案后有一行给机器读的来源标注（你看不见），用来核验题目有据可查。
>
> 提醒：C 阶段徽章的过关点在 C3 的深潜验收上——闭卷手写一条 Cursor Rule（`.mdc`）、手配一个最小 MCP stdio 连接，全程不让 Agent 代做。下面的 C3 练习就是照它设计的。

---

## 一、随堂练习（动手小题）

### C1 · Claude Code 快速入门（含 Fufan-CC Flow 可视化）
> 来源：《Claude Code 快速入门公开课（Fufan-CC Flow）》及配套 fufan-cc-flow 项目源码

**练习 C1-1｜跑通 Claude Code 的「对话 → 改码 → 提交」一圈**
在一个真实项目目录里，用 Claude Code 完成一个小任务（比如给已有页面加一个按钮），写出这一圈里的关键动作，以及这一站过关要交出的两个凭据。

- 【参考答案】关键动作：在项目目录起 Claude Code → 用自然语言把任务讲清 → 它读相关文件、改代码 → 你审一遍它改了什么 → 让它自己跑一次验证 → 用 git 提交（让它 commit 或你手动 commit 都行）。过关凭据两个：**任务产物在**、**git 提交在**，并附上这次的 commit。
- 【自查要点】Claude Code 不是「更聪明的补全」，它会自己读多个文件、动手改、跑命令、再提交，是一个完整循环。这一圈能自己转起来，才算真正从「用 AI 编辑器」升级到「指挥 Agent 干活」。
<!--src:A; ref:zero-to-builder C1 目标与过关点（Claude Code 完成一个真实小任务，产物+git提交在，附 commit）-->

**练习 C1-2｜Fufan-CC Flow 把命令行的哪些能力搬上了 Web**
不看代码，凭课件项目的功能总览，至少写出 Fufan-CC Flow 这个 Web 界面把 Claude Code CLI 的哪四项能力可视化了，并说出它前后端各用了什么。

- 【参考答案】任举其四：实时对话流（Token 级流式，带 Thinking 折叠）／工具调用可视化（每次 Tool Call 以卡片展示名称、输入、结果）／HIL 权限确认（危险操作弹框，支持一次性批准、永久批准、拒绝）／会话管理与分支 Fork／上下文压缩感知（实时显示 Token 用量）／模型切换（Claude Opus / Sonnet / Haiku 及国产基座）／MCP 管理（图形化增删 stdio / HTTP / SSE 的 MCP Server）／Memory 管理（Auto Memory 与 CLAUDE.md 双体系）／Sub-Agent 树。技术栈：前端 React 19 + Vite + TypeScript + Tailwind CSS v4 + Zustand；后端 Node.js + Express + WebSocket；AI 集成用 `@anthropic-ai/claude-agent-sdk`。
- 【自查要点】界面上的「HIL 权限确认」不是装饰，它对应 CLI 里真正的危险操作拦截；你在网页上点「批准」，才放行到 CLI 那层执行。看懂这一点，你就理解了这个 Web 前端到底在包装什么。
<!--src:A; ref:code:fufan-cc-flow/fufan-cc-flow-src/README.md 功能亮点+技术栈表-->

**练习 C1-3｜装 CLI 与验收口径**
写出安装 Claude Code 命令行工具的命令，以及「这一圈算跑通了」的最小验收标准。

- 【参考答案】安装：`npm install -g @anthropic-ai/claude-code`。验收：拿它完成一个真实小任务后，**任务产物和 git 提交都在**（附 commit）即算过关，不需要复杂交付。
- 【自查要点】把「提交」纳入验收，是逼你从第一天就带上工程习惯：Agent 改完不落 git，改动就是悬空的，无法追溯也无法回退。
<!--src:A; ref:code:fufan-cc-flow/fufan-cc-flow-src/README.md 环境要求（CLI 安装命令）; zero-to-builder C1 过关点-->

---

### C2 · 给你的 Agent 装上课程全家桶
> 来源：数字资产《赋范 Skills 全家桶 · 使用说明》

**练习 C2-1｜装到全局 vs 只装当前项目，装完怎么确认生效**
写出全家桶「装到用户级（全局生效）」和「只装当前项目」两条命令的区别，以及重启后如何确认它真的加载了。

- 【参考答案】解压 `赋范Skills全家桶.zip` 后进解压目录：`./install.sh` 装到用户级（全局生效）；`./install.sh --project` 只装当前项目。装完**重启 Claude Code / Codex 会话**才生效。确认方式：问 Agent 一个 Cursor Rules 的问题，看它是否按课程 Cursor 包的方法来回答（附对话截图）。
- 【自查要点】正常提问即可自动触发，不用手动调用某个命令。Windows 下没有 `install.sh`，要手动把「Claude Code 版」文件夹复制进 `%USERPROFILE%\.claude\skills\`、「Codex 版」复制进 `%USERPROFILE%\.agents\skills\`。
<!--src:A; ref:《赋范 Skills 全家桶 · 使用说明》安装/包清单节-->

**练习 C2-2｜为什么装完全家桶，后面几站才开「快车道」**
用一句话说清：C2 装好之后，C3、D、E 各会用到全家桶里的哪一类包。

- 【参考答案】能力包装入后，Agent 就按课堂工程方法干活：C3 会调用 Cursor 与 MCP 两个包，D 会调用写能力包的那个包，E 部署时会调用相关的部署包。所以 C2 是后续所有「课程包快车道」的开关。
- 【自查要点】装好后 Claude Code 版的包落在 `.claude/skills`，Codex 版落在 `.agents/skills`——两个流派各有各的目录，别装错地方。
<!--src:A; ref:zero-to-builder C2 目标与复习锚点（C3 调 cursor/mcp 包、D 调 authoring 包、E 调部署包）-->

---

### C3 · Cursor 进阶 + MCP：做出数据分析助理
> 来源：《Cursor 应用进阶与 MCP 集成实战》《MCP 快速入门实战》，能力包 cursor-vibe-coding-pack / mcp-dev-pack，配套 NL2SQLAgent 项目源码

**练习 C3-1｜Cursor Rules 四类型各归其位**
写出 Cursor Rules 的四种类型、各自 Frontmatter 里的关键字段、以及各自适用的场景。规则文件放在哪个目录、什么后缀？

- 【参考答案】文件放 `.cursor/rules/` 下的 `.mdc` 文件，靠 Frontmatter 控制生效方式：
  - **Always（全局强约束）**：`alwaysApply: true`，每次都注入，只放 1–5 条团队铁律（如回复用中文）；
  - **Auto Attached（按文件注入）**：用 `globs:` 指定文件类型（如 `*.py`），命中才激活，企业里用得最多；
  - **Agent Requested（按语义触发）**：写清 `description:`，AI 按语义判断是否加载（如「我要写单元测试」→ 自动挂测试规范）；
  - **Manual（手动触发）**：无自动条件，靠 `@rules` 显式调用（如 `@format-sql`）。
- 【自查要点】选类型不是比「哪个好」，而是问「这条规则该在什么上下文注入最合适」。只对 `*.py` 成立的规范别塞进 Always，用 Auto Attached 的 `globs` 限定范围；Always 塞太多会互相打架、还挤占上下文。
<!--src:A; ref:pack:cursor-vibe-coding-pack Cursor Rules 编写（四类型对照表）-->

**练习 C3-2｜MCP 三种传输怎么选**
写出 MCP 的三种传输方式、一句话判据、各自适用场景。

- 【参考答案】判据一句话：服务器和客户端在不在同一台机器、要不要并发。
  - **stdio**：双向但仅限本地，靠 stdin/stdout 做进程间通信，服务器作为客户端子进程运行，`mcp.run(transport='stdio')` 一行搞定，是本地调试的默认起点；
  - **SSE**：基于 HTTP 的单向流，并发中等，长连接不稳、无多通道并发，撑不起企业级，是过渡选项；
  - **StreamableHttp**：双向、长连接、高并发，异地/企业级首选，SDK v1.8.0 起内置 `StreamableHTTPSessionManager`。
- 【自查要点】要异地企业级并发就直接上 StreamableHttp，别再从 SSE 起步；只在本机调试就用 stdio，别为它徒增长连接复杂度。stdio 没有网络端口，异地 HTTP 永远直连不上。
<!--src:A; ref:pack:mcp-dev-pack MCP 传输方式选型（stdio/SSE/StreamableHttp）-->

**练习 C3-3｜徽章 gate 同款：手写一条 Rule + 手搭一个最小 stdio MCP（闭卷）**
不让 Agent 生成配置，自己对照文档：① 写一个最小的 FastMCP stdio 服务器骨架；② 写一条 Always 类型的 Cursor Rule（`.mdc`）。

- 【参考答案】
  ```python
  # server.py —— 最小 stdio MCP 工具
  from mcp.server.fastmcp import FastMCP
  mcp = FastMCP("WeatherServer")

  @mcp.tool()
  async def query_weather(city: str) -> str:
      """输入指定城市的英文名称，返回今日天气查询结果。"""  # docstring 就是工具契约
      data = await fetch_weather(city)   # fetch_weather 内做 try/except，返回 {"error": ...}
      return format_weather(data)        # 返回给模型读的文本，不是原始 JSON

  if __name__ == "__main__":
      mcp.run(transport='stdio')
  ```
  ```markdown
  ---
  alwaysApply: true
  ---
  - 所有回复用中文。
  - 不要在源码里硬编码任何 API Key，一律走环境变量注入。
  ```
- 【自查要点】决定工具能不能被模型正确调用的三件事：函数 **docstring**（客户端据此识别功能与参数）、参数类型标注（生成 inputSchema）、返回值是给模型读的**文本**。这一站的验收是交出：Rule 文件 + mcp 配置文件 + 一次工具调用的连通日志。
<!--src:A; ref:pack:mcp-dev-pack MCP Server 搭建（FastMCP stdio 骨架）; pack:cursor-vibe-coding-pack Cursor Rules（Always 模板）; zero-to-builder C3 深潜验收-->

**练习 C3-4｜NL2SQL 助理的六步链路 + 安全红线 + 关键数字**
照 NL2SQLAgent 项目，写出这个数据分析助理的六步链路、三条安全红线、以及三个关键数字。它用 SSE 往前端推哪些事件类型？

- 【参考答案】六步（写在系统提示里）：① 理解问题 → ② 用 `sql_db_list_tables` + `sql_db_schema` 看表结构（schema-first）→ ③ 写 SQL → ④ 用 `sql_db_query_checker` 校验语法 → ⑤ 用 `sql_db_query` 执行 → ⑥ 分析结果并用中文回答。三条红线：只执行 SELECT，禁止 INSERT/UPDATE/DELETE/DROP；结果限 `top_k` 条以内；优先选相关列、避免 `SELECT *`。三个数字：`max_iterations=6`（迭代上限）、`top_k=10`（限行）、记忆窗口 `window_size=10`（保留最近 10 轮）；模型是 `qwen3-max`。SSE 事件类型：`thinking / text / sql / data / chart / error / done`。
- 【自查要点】schema-first 是防幻觉的地基：不先摸清真实表名列名，LLM 凭空编的 SQL 必错。校验这一关（`sql_db_query_checker`）在执行前必过；出错就「分析原因 → 重写 → 再校验」，用 `max_iterations` 给这个自愈循环设上限，别无限打转。
<!--src:A; ref:code:NL2SQLAgent/backend/app/core/agent.py（max_iterations=6/top_k=10）; core/llm.py（qwen3-max/SQL_AGENT_SYSTEM_PROMPT 六步）; core/memory.py（window_size=10）; schemas/chat.py（SSEEventType）-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句想法，说明你还没真正掌握右边那件事，去回看对应的一站。

1. **如果你以为「Claude Code 就是个更聪明的代码补全」** → 说明你还没抓住它的本质：它会自己读多个文件、动手改、跑命令、再提交，是一个完整循环。回看 zero-to-builder#C1，先只跟着走通一次「对话 → 改码 → 提交」。

2. **如果你以为「Fufan-CC Flow 网页上点的按钮只是界面效果」** → 说明你没看懂它在包装什么。它把 CLI 的能力可视化，HIL 权限确认对应的是 CLI 里真正的危险操作拦截，你点「批准」才放行执行。回看 zero-to-builder#C1 的项目功能总览。

3. **如果你以为「全家桶装好就完事、不用重启也不用验证」** → 说明你漏了两步：装完要重启会话才生效，还要问 Agent 一个 Cursor Rules 问题、确认它按课程包的方法回答。回看 zero-to-builder#C2 的安装与确认。

4. **如果你以为「Cursor Rules 写得越多、都塞进 Always 越保险」** → 说明你没理解「在什么上下文注入」才是关键。Always 只放 1–5 条铁律，规则越多越互相打架、还挤占上下文；只对某类文件成立的规范用 Auto Attached 的 `globs`。回看 zero-to-builder#C3 的 Rules 四类型。

5. **如果你以为「异地用 MCP 也先从 stdio 起步」** → 说明你没记住 stdio 没有网络端口、只能作为本地子进程通信。要异地、要企业级并发就直接上 StreamableHttp。回看 zero-to-builder#C3 的传输选型。

6. **如果你以为「让 Agent 帮我把 Cursor Rule 和 MCP 配置生成了，也算过 C 阶段」** → 说明你没看清徽章过关点是闭卷手写：自己对照文档写 `.mdc`、手填 mcp 配置并连通一次，交 Rule 文件 + 配置文件 + 连通日志。让 Agent 代做等于没拿到徽章。回看 zero-to-builder#C3 的深潜验收。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜把 NL2SQL 的安全链路迁移到另一个业务库**
课程的 NL2SQLAgent 查的是示例库。现在你要把同一套助理接到一个电商订单库上（表结构完全不同），做「用中文查订单、出图表」。请只描述你要保留和调整的地方，不用真跑。

- 【怎么算过关】能说清：① 六步链路一步不少（先 `sql_db_list_tables` + `sql_db_schema` 摸清新库结构，再写 SQL）；② 三条安全红线照写进系统提示（只 SELECT、限 `top_k`、避免 `SELECT *`）；③ 换库基本只动数据库连接串（`DATABASE_URL`）和让 Agent 重新读一遍 schema，六步逻辑、`max_iterations`、校验环几乎不用改。能讲清「为什么改动这么小」，就说明你抓住了 schema-first 的意义：Agent 每次都现查表结构，不写死表名列名。
<!--src:A; ref:code:NL2SQLAgent/backend/app/core/agent.py + core/llm.py（六步/只SELECT/top_k）; pack:cursor-vibe-coding-pack NL2SQL 助理搭建-->

**迁移挑战 2｜把本机 stdio 的天气工具改成异地可调**
你按课程搭了一个 stdio 的天气 MCP 工具，本机 Inspector 能连。现在要让异地的同事也能调它，你要改哪几处？

- 【怎么算过关】能指出：① 传输从 `stdio` 换成 `StreamableHttp`；② `host` 从 `127.0.0.1` 改成 `0.0.0.0`（并想清楚谁能连、要不要鉴权）；③ 客户端连的是 HTTP 地址加 `/mcp` 端点（课程天气服务器用 3000 端口）；④ 启动顺序变成「先起服务器、再起 Inspector」；⑤ key 走环境变量注入、别硬编码。能说清「为什么 stdio 直接搬过去不行」= stdio 没有网络端口、只能本地子进程通信，异地必然连不上。
<!--src:A; ref:pack:mcp-dev-pack 传输选型 + 凭证红线 + 连不上排查（host/启动顺序/3000·/mcp）-->

---

## 四、排错挑战（给你一段带病的代码，修好它）

一位同学写了一个 stdio 的 MCP 天气工具，反馈：**「模型死死不肯调用我这个工具；更糟的是同事说在我公开的仓库里翻到了我的真 API Key。」** 请找出问题并修好。

```python
# server.py（有问题的版本）
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("WeatherServer")

API_KEY = "sk-1234真实密钥就这么写死在这里"   # ①

@mcp.tool()
async def query_weather(city: str) -> str:
    # ②（没有 docstring）
    url = f"https://api.example.com/weather?city={city}&key={API_KEY}"
    resp = httpx.get(url)
    return resp.text        # ③ 直接把原始 JSON 扔回去

if __name__ == "__main__":
    mcp.run(transport='stdio')
```

- 【症状】① 客户端里这个工具形同虚设，模型不知道什么时候该调、要传什么参，于是干脆不调；② 这份代码一 `push` 到公开仓库，真 Key 就随仓库泄露，会被爬虫扫走刷爆配额。
- 【根因】① 工具函数**没写 docstring**——docstring 是客户端识别工具功能与参数的唯一依据，空着模型就无从判断；② **API Key 硬编码进源码**并提交，违反 MCP 凭证红线「key 走注入、绝不硬编码」；③ 返回的是原始 JSON 而不是给模型读的人话文本，模型理解成本高。
- 【修法】
  ```python
  # server.py（修好的版本）
  import argparse
  import httpx
  from mcp.server.fastmcp import FastMCP

  mcp = FastMCP("WeatherServer")
  API_KEY = None   # 占位，main() 里从参数/环境变量注入

  @mcp.tool()
  async def query_weather(city: str) -> str:
      """输入指定城市的英文名称，返回今日天气查询结果。"""   # ← 补上工具契约
      try:
          url = f"https://api.example.com/weather?city={city}&key={API_KEY}"
          async with httpx.AsyncClient() as client:
              resp = await client.get(url)
              resp.raise_for_status()
          return format_weather(resp.json())    # ← 转成人话文本再返回
      except Exception as e:
          return f"查询失败：{e}"                 # ← 不裸抛，让模型能读懂并重试

  if __name__ == "__main__":
      parser = argparse.ArgumentParser()
      parser.add_argument("--api_key", required=True)   # ← key 从参数注入
      API_KEY = parser.parse_args().api_key
      mcp.run(transport='stdio')
  ```
  再补两件：把 `*.env`、含真 key 的配置写进 `.gitignore`；发布/提交前跑一遍凭证扫描，命中硬编码 key 先改再发。
- 【一句话总结】工具要被模型调用，靠 **docstring 当契约**；Key 要活命，靠 **注入而非硬编码 + `.gitignore` 排除**。这两条是 MCP 开发全程都在的红线。
<!--src:A; ref:pack:mcp-dev-pack MCP Server 搭建（docstring/返回文本）+ 凭证与环境变量红线（注入/.gitignore/扫描）-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。
> 说明：以下题目按 2026 年 AI 应用 / Agent 开发岗方向校准，考点集中在基座 Agent 上手、Cursor Rules 与模式、MCP 传输与调试、NL2SQL 安全链路。

**面试 1｜用 Claude Code 这类基座 Agent 干活，和用「AI 代码补全」有什么本质区别？**
- 考察点：对「指挥 Agent」与「用编辑器插件」的分界理解。
- 答题要点：补全只在你光标处给建议，你仍是主操作者；基座 Agent 会自己读多个文件、动手改、跑命令、再提交，是一个「对话 → 改码 → 验证 → 提交」的完整循环，你的角色从写代码变成描述任务加审阅。
- 追问：「怎么算这一圈真做完了？」→ 任务产物在、git 提交也在（附 commit），改动落了 git 才可追溯可回退。
<!--src:A; ref:zero-to-builder C1 目标与过关点; code:fufan-cc-flow/fufan-cc-flow-src/README.md-->

**面试 2｜MCP 有哪几种传输方式？分别什么时候用？**
- 考察点：MCP 传输选型（Agent 岗高频）。
- 答题要点：三种。stdio：仅限本地、靠标准输入输出做进程间通信、服务器是客户端的子进程，本地调试默认起点；SSE：基于 HTTP 的单向流，并发中等、长连接不稳，是过渡选项；StreamableHttp：双向长连接高并发，异地企业级首选，SDK v1.8.0 起内置会话管理器。判据就一句：在不在同一台机器、要不要并发。
- 追问：「异地为什么别从 SSE 起步？」→ SSE 单向、无多通道并发、长连接不稳，撑不起企业级；要异地直接上 StreamableHttp。
<!--src:A; ref:pack:mcp-dev-pack MCP 传输方式选型-->

**面试 3｜Cursor Rules 有哪几种类型？怎么决定一条规则该用哪种？**
- 考察点：Rules 的上下文注入模型。
- 答题要点：四种，靠 Frontmatter 区分。Always（`alwaysApply: true`，每次都注入，只放 1–5 条铁律）；Auto Attached（`globs` 指定文件类型，命中才激活）；Agent Requested（写 `description`，AI 按语义判断加载）；Manual（`@` 显式调用）。选哪种不是比好坏，而是问「这条规则该在什么上下文注入最合适」。文件放 `.cursor/rules/` 下的 `.mdc`。
- 追问：「只对 `*.py` 成立的规范放哪？」→ Auto Attached，用 `globs: *.py` 限定，别塞进 Always 变成全局。
<!--src:A; ref:pack:cursor-vibe-coding-pack Cursor Rules 编写（四类型）-->

**面试 4｜在 Cursor 这类 AI 编辑器里，几种问答模式该怎么选？**
- 考察点：按任务意图选模式、别习惯性开自主执行。
- 答题要点：先问「这一步会不会改代码」。只想读懂、绝不改 → 只读问答模式（Ask/Chat）；要改但跨多文件、需求没拆清 → 先进规划模式（Plan）出方案再执行；需求清楚且范围明确 → 自主执行模式（Agent/Composer），但先确认它会动哪些文件；单文件一处小改 → 行内模式。越危险的模式越要先看它的行动计划。
- 追问：「复现得了、但读代码看不出原因的 Bug 用哪种？」→ 调试模式，让它靠运行时证据（日志/断点/复现）定位，而不是凭读代码猜。
<!--src:A; ref:pack:cursor-vibe-coding-pack Cursor 问答模式选型-->

**面试 5｜MCP 服务器「起来了但连不上」，你按什么顺序排查？**
- 考察点：MCP 调试分诊（用官方 Inspector）。
- 答题要点：Inspector 是官方通用客户端，用来测你写的 server。常见几类：调工具超时，多半是 Inspector 默认 10 秒响应窗口太短，去 Configuration 调大；Streamable HTTP 连不上、List Tools 空，先查启动顺序和 host 绑定；云主机本地打不开，是端口没走 SSH 隧道映射（Inspector 用 6274、其内建 server 用 6277、天气 HTTP 服务器用 3000）；stdio「单独跑没反应」，是它必须作为客户端子进程被拉起、单独跑没端口可连。
- 追问：「Streamable HTTP 和 stdio 的启动顺序谁先？」→ Streamable HTTP 先起服务器再起 Inspector；stdio 相反，启 Inspector 时会同步拉起 server 子进程。
<!--src:A; ref:pack:mcp-dev-pack MCP 服务器调试与连不上排查（10秒/6274/6277/3000/启动顺序）-->

**面试 6｜做自然语言查数据库的助理，为什么必须 schema-first、只读、先校验？**
- 考察点：NL2SQL 的安全与防幻觉链路。
- 答题要点：schema-first 是先用 `sql_db_list_tables` + `sql_db_schema` 摸清真实表结构再写 SQL，不这样 LLM 会凭空编表名列名，必错；只读是系统提示里只允许 SELECT、禁止增删改删表，防止助理破坏数据；先校验是执行前必过 `sql_db_query_checker`，语法错就重写再校验。出错走「分析原因 → 重写 → 再校验」的自愈环，用 `max_iterations`（课程取 6）设上限。
- 追问：「除了系统提示，还怎么兜底防止执行写操作？」→ 执行前用一个 SQL 安全检查脚本拦截非 SELECT 语句，双保险。
<!--src:A; ref:pack:cursor-vibe-coding-pack NL2SQL 助理搭建; code:NL2SQLAgent/backend/app/core/llm.py 系统提示（只SELECT/校验）-->

**面试 7｜这个数据分析助理是怎么把中间过程流式推给前端的？**
- 考察点：SSE 事件分型与流式响应。
- 答题要点：后端用 FastAPI 的 `StreamingResponse`、媒体类型 `text/event-stream`，把 Agent 每一步按事件类型分开推：`thinking`（思考）、`text`（文本）、`sql`（生成的 SQL）、`data`（查询结果）、`chart`（图表配置）、`error`、`done`。前端按事件类型分别渲染，所以你能看到「先想、再出 SQL、再出结果和图」这样的过程。
- 追问：「响应头里为什么要加 `X-Accel-Buffering: no`？」→ 让 Nginx 之类的反向代理别缓冲，否则流式会被攒成一整块，逐步输出的效果就没了。
<!--src:A; ref:code:NL2SQLAgent/backend/app/api/chat.py（StreamingResponse/text/event-stream/X-Accel-Buffering）; schemas/chat.py（SSEEventType）-->

**面试 8｜开发 MCP Server 有哪些凭证红线？**
- 考察点：密钥安全（发布类服务尤其被问）。
- 答题要点：四条。key 走注入绝不硬编码（stdio 用 argparse 参数、HTTP 用环境变量）；发布包用 `.gitignore`/`.npmignore` 排除 `*.env`、`__pycache__`；官方能力的 key 与中转 key 不能混用；host 暴露面要管，本地绑 `127.0.0.1`、异地才绑 `0.0.0.0` 且要想清鉴权。因为 MCP Server 一旦发到 pypi/npm，硬编码的 key 会随公开包泄露。
- 追问：「OpenAI Agents / WebSearch 这类官方能力能用中转 key 吗？」→ 不能，必须用官方 API Key，中转 key 会静默失败、不报错但结果为空，排查很费劲。
<!--src:A; ref:pack:mcp-dev-pack MCP 凭证与环境变量红线（四条）-->

**面试 9｜课程能力包装到 Agent 上是怎么做的？怎么确认装好了？**
- 考察点：能力包安装与验证。
- 答题要点：解压全家桶后进目录，`./install.sh` 装到用户级全局、`./install.sh --project` 只装当前项目，装完重启会话生效，正常提问就会自动触发、不用手动调用。确认方式：问 Agent 一个 Cursor Rules 之类的问题，看它是否按课程包的方法来回答。
- 追问：「Claude Code 版和 Codex 版分别装到哪个目录？」→ Claude Code 版进 `.claude/skills`，Codex 版进 `.agents/skills`，两个流派目录不同。
<!--src:A; ref:《赋范 Skills 全家桶 · 使用说明》安装节; zero-to-builder C2 复习锚点-->

**面试 10｜C 阶段的徽章为什么要「闭卷手写 Rule + 手配 MCP」，而不是让 Agent 代做？**
- 考察点：对「会指挥 Agent」与「真懂底层」的分界认知。
- 答题要点：快车道让你拿到产出，但徽章过关点偏偏考你能不能不靠 Agent、自己对照文档写出一条 `.mdc` Rule、手填一个最小 MCP stdio 配置并连通。因为只有亲手写过，你才真懂 Rules 的 Frontmatter 字段、MCP 的传输与启动方式，日后 Agent 配错了你才看得出来。
- 追问：「这一站手配 stdio 连通的验收凭据是什么？」→ 交出 Rule 文件 + mcp 配置文件 + 一次工具调用的连通日志，三样齐了才算过。
<!--src:A; ref:zero-to-builder C3 深潜验收（闭卷手写 Rule + 手配 stdio，交 Rule/配置/日志）-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请它只做考官、不做辅导。

```
请你扮演一名技术面试官，围绕「上手基座智能体：Claude Code + Cursor + MCP」这一阶段考我。规则如下：

【角色纪律】请只做考官、不做辅导。在我明确说「结束，请点评」之前，你不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，尽量贴近真实岗位：
1. 基座 Agent 上手：Claude Code 的「对话→改码→提交」循环、和 AI 补全的区别、验收（产物+git 提交）。
2. Fufan-CC Flow：它把 CLI 的哪些能力搬上 Web（HIL 权限、会话 Fork、MCP 管理、Sub-Agent 树）。
3. 课程全家桶：全局 vs 项目安装、装完怎么验证、Claude Code 版与 Codex 版目录。
4. Cursor Rules：四类型（Always/Auto Attached/Agent Requested/Manual）与各自 Frontmatter 字段。
5. Cursor 问答模式：按任务意图选模式，别习惯性开自主执行。
6. MCP：stdio/SSE/StreamableHttp 传输选型、Inspector 排查连不上（超时窗口、启动顺序、端口隧道、host）。
7. NL2SQL 助理：schema-first→校验→只读执行的六步链路，只 SELECT、限 top_k、避免 SELECT*。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 如果我答「不会」，记录后直接下一题，不要讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现（答对/部分/未答）、暴露的薄弱点、以及建议我回看 Stage C 的哪一站（C1 Claude Code 入门 / C2 装全家桶 / C3 Cursor 进阶+MCP）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件或 Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 用 Claude Code 跑通一次「对话→改码→提交」的真实小任务（带 git 提交） | ☐ | ☐ | ☐ |
| 说清 Fufan-CC Flow 把 CLI 的哪些能力搬上 Web（HIL / Fork / MCP / Sub-Agent） | ☐ | ☐ | ☐ |
| 装好课程全家桶（全局 / 项目），并验证 Agent 按课程包方法回答 | ☐ | ☐ | ☐ |
| 写对 Cursor Rules 四类型、选对 Frontmatter 字段 | ☐ | ☐ | ☐ |
| 为任务选对 Cursor 的问答模式（只读 / 规划 / 自主 / 调试 / 行内） | ☐ | ☐ | ☐ |
| 说清 MCP 的 stdio / SSE / StreamableHttp 选型 | ☐ | ☐ | ☐ |
| 从零搭一个 FastMCP stdio 工具（docstring 即契约、key 注入不硬编码） | ☐ | ☐ | ☐ |
| 用 Inspector 排 MCP 连不上（超时窗口 / 启动顺序 / 端口隧道 / host） | ☐ | ☐ | ☐ |
| 搭一个 NL2SQL 助理（schema-first / 只 SELECT / 先校验 / 限行） | ☐ | ☐ | ☐ |
| 闭卷手写一条 Cursor Rule + 手配一个最小 MCP stdio 连接 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」；其中「手写 Rule + 手配 stdio」「MCP 传输选型」这两项能到「能教别人」，你才真正拿下 C 阶段徽章，因为徽章过关点就是闭卷手写 Rule 加手配 MCP。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的入门到进阶的活儿，描述自包含。它们是你作品集和「超级个体」现金流的第一步。

1. **帮个人 / 小团队装好基座 Agent 工作台并跑通首个带提交的小任务。**
   很多人装不明白 Claude Code、也不敢让它动自己的代码。你会装 CLI、会带他走一圈「对话→改码→提交」、会用 git 兜底回退，这就是一单「远程装机 + 首任务陪跑」的服务。

2. **给一个现有代码库配一套 Cursor Rules。**
   帮团队把 AI 协作风格统一：几条 Always 铁律（回复语言、命名约定）、按文件类型的 Auto Attached 规范（`*.py` 守 PEP8）、场景化的 Agent Requested 规则。交付物是一份 `.cursor/rules/` 目录，让全组的 AI 输出不再各写各的。

3. **给商家 / 运营做一个「用中文查数据库」的看板。**
   拿 NL2SQL 助理的链路裁剪：schema-first + 只 SELECT + 限行 + 图表可视化，把「不会写 SQL 的人也能问数据」交付成一个网页。安全红线你已经会守，正好是这类活的核心。

4. **给别人封一个最小 MCP 工具接进他的 Cursor / Claude Code。**
   把「查天气 / 查某个内部接口 / 读某类文件」封成一个 stdio MCP 工具，docstring 写清、key 走注入，帮他在编辑器里调通。是把「会接 MCP」直接变现的最短路径。

5. **接「MCP 连不上」这类疑难的排障服务。**
   很多人卡在「本机能跑、异地连不上 / Inspector 超时 / 云主机打不开」。你手里有一棵排查树（超时窗口、启动顺序、端口隧道 6274/6277/3000、host 绑定），能快速定位根因，这本身就是一门可收费的手艺。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
