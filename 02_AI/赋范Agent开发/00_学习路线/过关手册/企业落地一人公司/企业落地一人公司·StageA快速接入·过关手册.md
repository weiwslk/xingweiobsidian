# 企业落地 / 一人公司 · Stage A《快速接入：把工作台架起来》· 过关手册

> 练习向 · 面向本人自测 · 面试自测 10 题带一层追问
>
> 本阶段你要跨过的两道关：
> - **A1 Claude Code 快速入门 + 可视化工作台（Fufan-CC Flow）** ｜ 在浏览器里指挥 Claude Code 干活，并能关掉 Agent 自己讲清前端 / 后端 / Agent 三层各干什么
> - **A2 给你的 Agent 装上课程全家桶** ｜ 一键把课程能力包装进对应目录，让后面每一站的快车道都有课程包驱动
>
> 用法：先做「随堂练习」确认手熟；再拿「误区自查」照镜子；「迁移挑战」检验你会不会把方法搬到自己的场景；「排错挑战」练你独立止血；「面试自测 + 模拟面试官」把你的嘴皮子练顺；最后用「能力自评表」和「落地对照」判断你站到哪一格、能接什么活。
>
> 所有题目取自 Stage A 实际课件与 Fufan-CC Flow 真实源码，端口 / 版本号 / 目录名逐字核对；答案自包含，不依赖任何外部链接或回看视频即可对照。这一阶段不追求懂原理，追求你能顺畅地「派活给 Agent、自己验收」。

---

## 一、随堂练习（动手小题）

### A1 · Claude Code 快速入门 + 可视化工作台（Fufan-CC Flow）
> 依据：Fufan-CC Flow 项目源码 + 课件《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》

**练习 A1-1｜口述前端 / 后端 / Agent 三层各干什么**
这是 A1 徽章要你关掉 Agent 自己讲的那道关。请对着工作台把三层各自的职责、用的技术、跑在哪个端口讲清楚。

- 【参考答案】
  - **前端（浏览器层）**：`React 19` + `Vite` + `Tailwind CSS v4`，由 15 个独立 `Zustand` Store 驱动状态，经典三栏布局（左栏 Sidebar / 中栏 ChatPanel / 右栏 RightPanel）。只负责渲染画面和收发消息，跑在浏览器 `localhost:5173`。
  - **后端（Node 服务层）**：`Node.js` + `Express` + `ws`，跑在 `:3001`。对外有 2 路 WebSocket（`/ws/chat` 走对话、`/ws/terminal` 走终端 I/O）+ 约 15 路 REST（走配置管理）。职责是把 WebSocket 消息翻译成 SDK / PTY 调用、再把输出翻译回 WebSocket 事件；路由层只做参数校验和转发，业务逻辑下沉到 `services/`。
  - **Agent（进程层）**：后端通过 `@anthropic-ai/claude-agent-sdk` 的 `query()` `spawn` 出一个 Claude Code CLI 子进程，CLI 自带模型调用、工具执行、文件读写、MCP 集成等完整能力。真正干活的是这个被拉起来的子进程。
- 【自查要点】一句话记住「前端只管画和收发，后端管翻译和调度，真正干活的是被 spawn 出来的 CLI 子进程」。这道深潜关不许 Agent 替你讲，你得自己看懂工作台在做什么。
<!--src:A; ref:code:server/src/index.ts + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.2 双泳道 5 层架构-->

**练习 A1-2｜双泳道：中栏聊天和右栏终端各走哪条链**
工作台中栏能聊天、右栏能开真终端。写出这两条链各自走什么后端处理、底层用什么 spawn 出子进程，以及它们的关系。

- 【参考答案】两条并行、互不感知的进程链。**Chat/Agent 泳道**：`chatHandler.ts → claudeAgentService.ts`，底层用 `claude-agent-sdk` 的 `query()` spawn 出 Claude Code CLI 子进程。**Terminal 泳道**：`terminalHandler.ts → ptyService.ts`，底层用 `node-pty` spawn 出 `bash` / `zsh` shell 子进程。两个进程互不知道对方存在，应用层也不强行同步它们的状态。
- 【自查要点】为什么要分两条？因为对话是流式 token + 工具调用，终端是原生 shell I/O，两类进程的职责和数据格式完全不同，塞进一条链会互相干扰。你在右栏敲 `vim foo.py` 改的是真文件，跟中栏对话是两码事。
<!--src:A; ref:code:server/src/websocket/ + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.2-->

**练习 A1-3｜一条消息的飞行路径，你派的活是怎么跑完的**
在中栏输入「你好」回车。请把这条消息在 5 层之间走的 8 步大致复述一遍，重点是你验收时该盯哪几步。

- 【参考答案】
  1. `ChatPanel` 捕获输入，经 WebSocket 发出 `send_message`；
  2. `chatHandler` 收到走 `send_message` 分支（顺带查是不是 Plan Mode）；
  3. 调 `claudeAgentService.start()`，从 WebSocket 层进入 Node 服务层；
  4. `start()` 内调 `query()` spawn 出 CLI 子进程，返回一个 `AsyncIterable` 流；
  5. CLI 把 prompt 发模型，模型说要调工具 → 经 `canUseTool` 回调请权限（HIL 入口）；
  6. CLI 执行工具，同时把 transcript 追加写到 `~/.claude/projects/<hash>/<sessionId>.jsonl`；
  7. `consumeStream` 用 `for await` 消费流，经 EventEmitter `emit` 出各种事件；
  8. `chatHandler` 监听事件，`forward` 经 WebSocket 推给浏览器渲染。
- 【自查要点】你要能对着 UI 说清「我看到的流式打字和工具卡片，是第 7-8 步把 CLI 的流式输出翻译成 WebSocket 事件推上来的」。派活给 Agent、自己验收，本质就是能判断这条链断在哪一步。
<!--src:A; ref:《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.3 八步飞行路径-->

---

### A2 · 给你的 Agent 装上课程全家桶
> 依据：课件《赋范 Skills 全家桶 · 使用说明》+ 全家桶 `清单.md` + `install.sh`

**练习 A2-1｜全家桶怎么装，双流派各进哪个目录**
写出用户级安装与项目级安装的命令，以及两个流派各装进哪个目录、Windows 手动怎么放。

- 【参考答案】解压后进入解压目录执行 `./install.sh` 装到用户级（`~/.claude/skills` 与 `~/.agents/skills`），或 `./install.sh --project` 只装到当前项目（`./.claude/skills` 与 `./.agents/skills`），加 `--force` 覆盖同名包（默认跳过已存在）。Windows 手动装：`Claude Code 版/` 下所有文件夹复制进 `%USERPROFILE%\.claude\skills\`，`Codex 版/` 复制进 `%USERPROFILE%\.agents\skills\`。装完**重启 Claude Code / Codex 会话**即生效，正常提问自动触发，无需手动调用。
- 【自查要点】记住两个目标目录不一样：Claude Code 流派进 `.claude/skills`，Codex 流派进 `.agents/skills`，`install.sh` 两个流派各装一遍。装错流派目录等于没装。
<!--src:A; ref:《赋范 Skills 全家桶 · 使用说明》安装 + install.sh-->

**练习 A2-2｜全家桶里装了些什么，规模多大**
写出全家桶的包数与能力数、按哪几个方向分组，以及你后面每一站快车道各要用到里面哪个包。

- 【参考答案】全家桶共 **36 个场景包 / 290 项能力**（双流派），按 6 个方向分组：AI 编程、Multi-Agent、RAG、工程规范、模型微调、部署运维。你后面每一站快车道要用的包都在里面：B2 打样用 `rag-advanced-pack`，C1 接口部署用 `agent-api-deployment-pack`，C2 容器用 `container-orchestration-pack`，D1 评估用 `agent-evaluation-pack`，D2 驾驭工程用 `harness-engineering-pack`。
- 【自查要点】验收不是数文件在不在，是让 Agent 列出它现在多了哪些领域（RAG / 部署上线 / 容器编排 / 评估 / 驾驭工程）的能力，对照清单逐项核。
<!--src:A; ref:全家桶 清单.md-->

**练习 A2-3｜装完 Agent 说没有对应能力怎么办，这一站还要提前备什么**
按常见问题写出先排查哪两件事；再写出这一站要提前备好的下一阶段外部依赖。

- 【参考答案】① 确认包确实放进了对应 skills 目录（Claude Code 版在 `.claude/skills`、Codex 版在 `.agents/skills`）且目录名没改；② **重开会话**再触发一次，能力包在会话启动时加载，不重启就识别不到。两件事都做了还不行，再逐个确认包是否解压完整。另外这一站要提前备好下一阶段打样要用的模型 API Key：DeepSeek 与阿里云百炼各备一个，B2 打样按你选的场景至少用到其一，提前备好免打断。
- 【自查要点】最常见的坑就是「放对目录但没重启会话」。提前备 Key 是为了 B2 打样时不被中途卡住去申请账号。
<!--src:A; ref:《赋范 Skills 全家桶 · 使用说明》安装 + business-landing#A2 准备-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句 X，说明你还没真正掌握 Y，去回看 Z。

1. **如果你以为「Fufan-CC 不就是又一个套壳聊天框」** → 说明你还没看清它在做什么。它把 Claude Code CLI 的**全部**能力（工具调用可视化、HIL 权限确认、Session 管理、终端集成）搬进了 Web，前端只是壳，真正干活的是后端 spawn 出来的 CLI 子进程。回看 business-landing#A1 的三层架构。

2. **如果你以为「UI 上看到啥，SDK 就有啥」** → 说明你还没建立「产品层 ≠ SDK 层」的判断。课件用 4 类后端转译（prompt 改写 / 后端调度 / 后端加工 / UI 聚合）点破：10 行 UI ↔ SDK 映射表里只有 4 行是纯 SDK。比如 Plan Mode 其实是后端往 prompt 前面拼一段「【PLAN MODE】…」字符串，SDK 根本没有 `planMode` 这个参数。回看 business-landing#A1 的四类后端转译。

3. **如果你以为「前端有个按钮，后端就一定接了」** → 说明你漏了「按钮真假生效判断」。前端定义了 `default` / `plan` / `edit` 三档模式，但后端只写了 `if (runMode === "plan")` 一个分支，`edit` 被静默丢弃，跑起来跟 `default` 一模一样。看到任何 Web 控件，先 grep 一下后端有没有对应 handler。回看 business-landing#A1 的按钮真假生效判断。

4. **如果你以为「工作台自己存了我的对话数据」** → 说明你还没理解「零侵入」。Fufan-CC 不自建存储，直接读 Claude Code CLI 自己落盘的 `~/.claude/projects/<hash>/*.jsonl` 事实源，跟 CLI 共享同一份数据。所以你在工作台和在命令行的会话可以互相接力。回看 business-landing#A1 的零侵入设计。

5. **如果你以为「把开关写进 settings.json 就能开 Agent Teams」** → 说明你踩了「env 快照」这坑。后端判定 Teams 是否启用只读 `process.env`，**不读** `~/.claude/settings.json`（那个文件只给 `claude` CLI 自己用）；而且 Node 进程的环境变量是启动那一刻的快照，你必须在启动 `pnpm dev` 的**同一个 shell** 里先 export。回看 business-landing#A1 的环境变量准备。

6. **如果你以为「装完全家桶，Agent 就自动会用课程包了」** → 说明你漏了「重启会话」这一步，还可能把两个流派的目录搞混。能力包在会话启动时加载，放对目录也要重启；Claude Code 版进 `.claude/skills`、Codex 版进 `.agents/skills`，装错流派目录等于没装。回看 business-landing#A2 的安装与验收。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜把「4 类后端转译」搬到另一个 CLI 集成 Web 的工具**
A1 课件给你一把判断工具：任何「把命令行 AI Agent 集成进 Web」的项目（Cursor / aider / goose 都在走这条路），UI 上的交互都能归进 5 类之一——纯 SDK / prompt 改写 / 后端调度 / 后端加工 / UI 聚合。挑一个你在用的 AI 编程工具的 Web 版，对它 3 个 UI 交互做归类。

- 【怎么算过关】① 对至少 3 个 UI 交互各给出归类；② 每个都能说出判断依据，例如「这个实时 token 进度条 SDK 没有对应回调，一定是后端从 `usage` 字段挖数据自己 emit 的后端加工」；③ 能指出哪些交互只是前端占位、后端可能还没接。
- 【提示】迁移骨架的第一步永远是先找到「事实源」（数据最终落在哪）；再看前端每个控件的值最终去哪：进 SDK 字段、被 prompt 包装、还是作环境变量注入。
<!--src:A; ref:《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第四章 4.1 四类后端转译 / 第五章 迁移骨架-->

**迁移挑战 2｜把「按钮真假生效判断」搬到你 B2 要打样的服务**
假设你在 B2 打样出的 MVP 前端上加了个开关（比如「严格模式」）。用 A1 教的方法验证它后端是不是真接了，而不是只改了个图标。

- 【怎么算过关】能说清两个动作：① 在前端 Network 面板查请求 / WebSocket 帧，确认这个字段真的发出去了；② 到后端 grep 这个字段名，看有没有对应的 handler 分支处理它。两者都有才算真生效，只有前端 = UI 占位。
- 【提示】这正是 Edit 模式被静默丢弃那个坑的通用形态。到你自己的场景服务里，「前端 emit 了字段 ≠ 后端有 handler」这条纪律照样适用。
<!--src:A; ref:《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第四章 4.1 按钮真假生效判断-->

---

## 四、排错挑战（给你一段带病的配置，修好它）

你想在 Fufan-CC 工作台里试试右栏的 Agent Teams，可 Teams Tab 一直显示「未启用」。你按网上说法把开关写进了 `~/.claude/settings.json`，然后**新开了一个终端**跑 `pnpm dev`，结果 Teams Tab 还是「未启用」，也没任何报错。下面是你写进 settings.json 的内容，请找出问题并修好。

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

- 【症状】`settings.json` 里明明写了开关 = `"1"`，也重启了工作台，右栏 Teams Tab 依旧显示「未启用」，全程不报任何错。
- 【根因】两层错叠在一起。① Fufan-CC 后端判定 Teams 是否启用只有一行（`teamService.ts:46`）：`process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS === "1"`——它只读**进程环境变量**，压根不读 `~/.claude/settings.json`（那个文件只给 `claude` CLI 自己用，Express 后端进程访问不到它）。② 就算你改用环境变量，Node 进程的 env 是**启动那一刻的快照**，你在**另一个**终端里设置、再回原终端看，或写完文件不重开启动 shell，后端都读不到。
- 【修法】在准备跑 `pnpm dev` 的那个终端里先 export，再在**同一个终端**启动：
  ```bash
  echo 'export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1' >> ~/.zshrc
  source ~/.zshrc
  echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS   # 应输出 1
  pnpm dev                                     # 必须在能 echo 出 1 的同一个终端里跑
  ```
  Windows：用 `[Environment]::SetEnvironmentVariable("CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS","1","User")` 写入后**关掉当前 PowerShell 窗口、重开新窗口**再启动，不能在同一窗口内立刻 `pnpm dev`。
- 【一句话总结】环境变量类开关认的是「启动进程的那个 shell 的 env 快照」，不是配置文件。排查口诀：先确认后端到底读 env 还是读文件，再确认你是不是在同一个 shell 里启动的。
<!--src:A; ref:code:server/src/services/teamService.ts:46 + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第一章 Agent Teams 环境变量-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。

**面试 1｜Fufan-CC Flow 到底解决什么问题？它跟「应用直连大模型 API」的 RAG / LangChain 工作流有什么本质不同？**
- 考察点：对「CLI 集成进 Web」这条工程范式的理解。
- 答题要点：RAG / LangChain 是「应用直连大模型 API」——一个 HTTP 请求过去拿回答。Fufan-CC 走另一条路：应用层**不直接调模型，而是包装一个本地 CLI 进程**（Claude Code CLI）。它要解决的是「把已经设计好的 CLI 智能体（带工具调用、文件系统访问、MCP）搬到浏览器里给用户用」。Cursor / aider / goose 都在走这条路。具体做法 = `claude-agent-sdk` + `node-pty` + `WebSocket` 把 CLI 包成 Web UI。
- 追问：「那模型 API Key 填在哪？」→ 后端在 `process.env` 里读 `ANTHROPIC_API_KEY`（国产基座则填自定义 `baseURL`），通过环境变量传给 spawn 出来的 CLI 子进程，由 CLI 自己去调模型；Key 只存本地、不写日志。
<!--src:A; ref:code:server/src/services/claudeAgentService.ts + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》引言-->

**面试 2｜让你口述 Fufan-CC 的三层架构，你怎么讲？**
- 考察点：A1 徽章关，能不能关掉 Agent 自己讲清架构。
- 答题要点：**前端（浏览器）**：`React 19` + `Vite` + `Tailwind CSS v4` + 15 个 `Zustand` Store，三栏 UI，只管渲染和收发，跑在 `:5173`。**后端（Node 服务层）**：`Express` + `ws` 在 `:3001`，2 路 WebSocket（`/ws/chat` 对话、`/ws/terminal` 终端）+ 约 15 路 REST（配置），职责是把 WebSocket 消息翻译成 SDK / PTY 调用再翻译回来，路由只做校验转发、业务逻辑在 `services/`。**Agent（进程层）**：后端用 `claude-agent-sdk` 的 `query()` spawn 出 Claude Code CLI 子进程，CLI 自带模型调用 / 工具 / 文件 / MCP。
- 追问：「前端和后端为什么用 WebSocket 而不是全用 REST？」→ 因为 CLI 输出是流式的（一个 token 一个 token 吐、实时穿插工具调用），HTTP 长轮询撑不住这种密度；非实时的配置操作才走 REST。
<!--src:A; ref:code:server/src/index.ts + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章-->

**面试 3｜双泳道架构是什么？为什么要两条泳道？**
- 考察点：Chat 泳道 vs Terminal 泳道的分工。
- 答题要点：后端不是一条流水线，而是两条并行、互不感知的进程链。**Chat/Agent 泳道**：`chatHandler.ts → claudeAgentService.ts`，底层 `claude-agent-sdk` 的 `query()` spawn CLI 子进程。**Terminal 泳道**：`terminalHandler.ts → ptyService.ts`，底层 `node-pty` spawn `bash` / `zsh` 子进程。分两条是因为对话（流式 token + 工具调用）和终端（原生 shell I/O）是两类完全不同的进程和数据格式，应用层也不强行同步它们的状态。
- 追问：「这两条泳道各锁死了什么版本约束？」→ Chat 泳道 SDK 是 `@anthropic-ai/claude-agent-sdk` `0.2.63`（npm 端）；Terminal 泳道 `node-pty` `1.1` 的预编译二进制只覆盖到 Node 22，所以整个项目必须用 **Node 22.x LTS**，Node 26 会直接编译失败。
<!--src:A; ref:code:server/package.json + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.2-->

**面试 4｜一条消息从浏览器发出到显示回复，大致经过哪几跳？**
- 考察点：8 步飞行路径。
- 答题要点：8 步贯穿 5 层：① `ChatPanel` 经 WebSocket 发 `send_message`；② `chatHandler` 收到走分支（查 Plan Mode）；③ 调 `claudeAgentService.start()`；④ `start()` 内调 `query()` spawn CLI，返回 `AsyncIterable`；⑤ CLI 把 prompt 发模型，模型要调工具 → `canUseTool` 请权限；⑥ CLI 执行工具，同时把 transcript 追加到 `~/.claude/projects/<hash>/<sessionId>.jsonl`；⑦ `consumeStream` 用 `for await` 消费流、经 EventEmitter emit 事件；⑧ `chatHandler` 监听事件 `forward` 经 WebSocket 推给浏览器渲染。
- 追问：「第 6 步那个 jsonl 文件是谁写的？」→ 是 CLI 子进程自己写的（一行一条消息的 JSON Lines），Fufan-CC 不写只读，这就是零侵入的根基。
<!--src:A; ref:《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.3-->

**面试 5｜HIL 权限确认是怎么工作的？自动放行的白名单是 SDK 给的吗？**
- 考察点：`canUseTool` 异步桥接 + 白名单是后端自造。
- 答题要点：SDK 给一个 `canUseTool` 回调（Python 侧 `can_use_tool`），语义要求「调用 → 拿到 PermissionResult → 工具才继续」的阻塞流程；而用户决策走 WebSocket 推弹窗 + 点击的真异步路径，中间用 Promise + Map 做异步桥接。后端把 `permissionMode` 设为 `bypassPermissions` 让 CLI 内部不弹提示，全交给前端确认。自动放行的白名单**不是** SDK 给的——SDK 只给 `canUseTool` 这一个回调，「哪些工具自动放行」完全是后端自己定的 `AUTO_APPROVE_TOOLS` 集合（`Read` / `Glob` / `Grep` / `WebSearch` / `WebFetch` / `Task` / `Agent` / `TodoWrite` / `NotebookRead` / `LS` 等只读、低风险工具），命中就直接 allow 不弹窗，没命中才转发给前端等你点。
- 追问：「权限弹窗一直不点会怎样？」→ 60 秒超时自动拒绝（`claudeAgentService` 里的 `60_000` 定时器），返回 deny 给 SDK，对话继续。
<!--src:A; ref:code:server/src/websocket/chatHandler.ts + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第二章 2.3-->

**面试 6｜「UI 上看到啥 SDK 就有啥」这句话错在哪？用 4 类后端转译讲。**
- 考察点：4 类后端转译。
- 答题要点：10 行 UI ↔ SDK 映射表里只有 4 行是纯 SDK（能在官方文档查到的标准参数），其余 6 行涉及后端转译，归成 4 类：**prompt 改写**（Plan Mode 其实是往 prompt 前面拼「【PLAN MODE】请先分析需求并制定详细的执行计划，不要修改任何文件…」字符串，SDK 没有 `planMode` 参数）、**后端调度**（`AUTO_APPROVE_TOOLS` 白名单是后端自己挑的）、**后端加工**（token 进度条的 `context_usage` 是 Fufan-CC 自造的 WS 事件，后端从 `message.usage` 挖 token 数自己 emit，SDK 没有 `onContextUsage` 回调）、**UI 聚合**（Checkpoint 时间线 = SDK 的 `enableFileCheckpointing` 标志 + 流方法 `rewindFiles` + `sessionManager` fallback 三合一）。
- 追问：「举一个『后端加工』最容易被误判的例子。」→ 实时 token 进度条。很多人以为 SDK 自带回调，其实 `context_usage` 是自造事件，后端在消费流时从 `usage` 字段挖 token 数、在 3 个时机（消息完成 / 流开始 / 流中途增量）各 emit 一次，才做出「实时」刷新的效果。
<!--src:A; ref:《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第四章 4.1-->

**面试 7｜Fufan-CC 的「零侵入哲学」指什么？会话数据存在哪？**
- 考察点：零侵入 + 事实源。
- 答题要点：Fufan-CC 不自建数据存储，直接读 Claude Code CLI 自己落盘的 `~/.claude/projects/<hash>/*.jsonl`（JSON Lines，一行一条消息）作为事实源，跟 CLI 共享同一份数据。`sessionManager` 在事实源之上做 4 件加工（路径哈希 / 索引解析 / 内部消息过滤 / file-history-snapshot 解析），把原始 transcript 翻译成 UI 能渲染的状态。
- 追问：「零侵入带来什么好处？」→ 你在工作台里的会话和你直接用 `claude` CLI 的会话是同一份，History 能列出 CLI 跑过的历史 session、能 Resume 复用 `session_id` 继续上下文；换句话说工作台和命令行随时可以互相接力。
<!--src:A; ref:code:server/src/services/sessionManager.ts + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第三章 3.1 第 6 层 / 第五章-->

**面试 8｜你在一个 Claude Code 会话里去启动 Fufan-CC 后端，结果 CLI 起不来，可能是什么？**
- 考察点：`delete process.env.CLAUDECODE` 嵌套会话检测。
- 答题要点：Claude Code CLI 检测到 `CLAUDECODE` 环境变量会拒绝启动（防止嵌套会话）。当 Fufan-CC 的 server 由一个 Claude Code 会话启动时，会继承这个变量，导致它 spawn 的 CLI 子进程起不来。项目在 `claudeAgentService.ts` 顶部用一行 `delete process.env.CLAUDECODE` 先清掉这个嵌套标记，再去 spawn CLI。
- 追问：「还有个相关坑：发消息报 `executable not found at cli.js` 是什么？」→ 后端构造给 CLI 的 env 没有 spread `process.env`，子进程丢了 PATH 找不到 `cli.js`；排查锚点在 `claudeAgentService.ts` 的 env 构造处。
<!--src:A; ref:code:server/src/services/claudeAgentService.ts:29 + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第二章 故障速查表-->

**面试 9｜Fufan-CC 跑起来要哪些环境？端口是多少？为什么 Node 版本不能随便选？**
- 考察点：环境 / 端口 / 版本约束（落地实操）。
- 答题要点：它是 Monorepo（`pnpm workspace` 用根 `pnpm-workspace.yaml` 串 `client/` 和 `server/`）。需要 Node.js 22.x LTS、pnpm 8.x+、Claude Code CLI、Python + C++ 工具链（`node-pty` 要本机编译）。`pnpm dev` 用 `concurrently` 同时起前后端：后端 Express + WS 在 `:3001`，前端 Vite 在 `:5173`，浏览器开 `localhost:5173`。Node 版本必须 22.x，是因为 `node-pty 1.1` 的预编译二进制只覆盖到 Node 22，Node 26 会直接编译失败。
- 追问：「Windows 上还要额外装什么？」→ Git for Windows（含 Git Bash，Claude Code CLI 在 Windows 上跑工具调用必需），装 Node 时勾「Automatically install the necessary tools」一次性把 Python + VS Build Tools 装齐；Fufan-CC 会自动检测 Git Bash 路径，也可用项目根 `.env` 里的 `CLAUDE_CODE_GIT_BASH_PATH` 覆盖。
<!--src:A; ref:code:README.md + 《Fufan-CC Flow 项目：CLI 集成进 Web 的范式拆解》第一章-->

**面试 10｜课程全家桶怎么装？双流派、规模、生效条件各是什么？**
- 考察点：A2 安装与验收。
- 答题要点：全家桶是双流派（Claude Code 版 / Codex 版）合集，共 36 个场景包 / 290 项能力，按 6 个方向分组（AI 编程 / Multi-Agent / RAG / 工程规范 / 模型微调 / 部署运维）。装法：解压后 `./install.sh` 装用户级（`~/.claude/skills` 与 `~/.agents/skills`），或 `./install.sh --project` 装当前项目，`--force` 覆盖同名包（默认跳过）。Windows 手动：Claude Code 版进 `%USERPROFILE%\.claude\skills\`，Codex 版进 `%USERPROFILE%\.agents\skills\`。生效条件 = 放进对应 skills 目录 + 重启会话，之后正常提问自动触发、无需手动调用。
- 追问：「怎么验收真的装上了，而不只是文件躺在目录里？」→ 让 Agent 列出它现在多了哪些领域（RAG / 部署上线 / 容器编排 / 评估 / 驾驭工程）的能力，对照清单核；或直接问一个该领域问题看它是不是按课程包的方法答。装完发现没识别到，九成是放对目录但没重启会话。
<!--src:A; ref:《赋范 Skills 全家桶 · 使用说明》+ 全家桶 清单.md + install.sh-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请只做考官、不做辅导。

```
请你扮演一名技术面试官，围绕「快速接入：Claude Code 可视化工作台 Fufan-CC Flow + 课程全家桶」这一阶段考我。规则如下：

【角色纪律】只做考官、不做辅导。在我明确说「结束，请点评」之前，不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，尽量贴近真实岗位：
1. 三层架构：前端（React/Zustand，:5173）/ 后端（Express/ws，:3001，2 路 WebSocket + 15 路 REST）/ Agent（claude-agent-sdk query() spawn CLI 子进程）各干什么。
2. 双泳道：Chat 泳道走 Agent SDK、Terminal 泳道走 node-pty，为什么分两条，各锁什么版本（SDK 0.2.63 / node-pty 1.1 / Node 22）。
3. 飞行路径：一条消息 8 步怎么走，事实源 jsonl 是谁写的。
4. HIL 与转译：canUseTool 异步桥接、AUTO_APPROVE 白名单是后端自造、4 类后端转译（prompt 改写 / 后端调度 / 后端加工 / UI 聚合）、按钮真假生效判断。
5. 零侵入：读 ~/.claude/projects/*.jsonl、sessionManager 4 件加工、工作台与命令行互相接力。
6. 环境与全家桶：端口 / Node 版本约束 / delete process.env.CLAUDECODE 嵌套会话 / Teams 开关认 env 快照；全家桶双流派目录、36 包 290 能力、重启生效与验收。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 我答「不会」就记录后直接下一题，不讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现评价（答对 / 部分 / 未答）、暴露的薄弱点、以及建议我回看 Stage A 的哪个环节（A1 工作台架构 / A2 全家桶）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件或 Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 关掉 Agent 口述前端 / 后端 / Agent 三层各干什么 | ☐ | ☐ | ☐ |
| 说清双泳道：Chat 走 Agent SDK、Terminal 走 node-pty，为什么分两条 | ☐ | ☐ | ☐ |
| 复述一条消息的 8 步飞行路径、指出事实源 jsonl 是谁写的 | ☐ | ☐ | ☐ |
| 讲清 HIL 的 canUseTool 异步桥接与 60 秒超时 | ☐ | ☐ | ☐ |
| 说清 AUTO_APPROVE 白名单是后端自造、不是 SDK 给的 | ☐ | ☐ | ☐ |
| 用 4 类后端转译判断 UI 交互属于哪一类（含 Plan Mode 的 prompt 改写） | ☐ | ☐ | ☐ |
| 会做「按钮真假生效判断」（前端 emit ≠ 后端有 handler） | ☐ | ☐ | ☐ |
| 讲清零侵入：读 ~/.claude/projects/*.jsonl、工作台与命令行互相接力 | ☐ | ☐ | ☐ |
| 记住端口（3001/5173）与 Node 22 版本约束、会排 Teams 开关的 env 快照坑 | ☐ | ☐ | ☐ |
| 会 delete process.env.CLAUDECODE 解释嵌套会话导致 CLI 起不来 | ☐ | ☐ | ☐ |
| 双流派装好课程全家桶并验收 Agent 真按课程包方法答题 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」，其中「三层架构口述」这一项要到「能教别人」，你才真正拿下 A 阶段徽章——因为这道深潜关就是闭卷、不许 Agent 代做地在工作台里手动跑通一次真实小改动并讲清三层。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的活儿，描述自包含。它们也是你落地方法的第一批可复制件。

1. **帮团队 / 客户把 Fufan-CC 可视化工作台架起来。**
   装好 Node 22 + pnpm + Claude Code CLI，`pnpm dev` 起前后端（`:3001` / `:5173`），配齐 Settings 四件套（环境检测、API Key 或国产基座 baseURL、模型、工作目录）。交付物 = 一台能在浏览器里指挥 Claude Code 干活的工作台。

2. **给一个想落地的人一键装好课程全家桶。**
   双流派 `install.sh` 装到对应目录（Claude Code 版进 `.claude/skills`、Codex 版进 `.agents/skills`），重启会话后让 Agent 列出多了哪些领域能力、对照清单验收。交付物 = 装好且验收通过的 Agent 环境。

3. **排查「工作台连不上 / 开关不生效」这类环境故障。**
   你会分层判断断在前端 WebSocket、后端服务、还是 CLI 子进程哪一层，会认「env 快照」类坑（Teams 开关要在启动 shell 里 export、嵌套会话的 `CLAUDECODE` 要清、Node 版本要锁 22）。这是一单实打实的「环境搭建 / 链路排障」小服务。

4. **给团队讲清「UI 上看到啥不一定 SDK 就有啥」。**
   用 4 类后端转译 + 按钮真假生效判断，帮他们评估任意「CLI 集成进 Web」工具里哪些能力是 SDK 真给的、哪些是后端转译或纯前端占位，少走弯路少踩「改错层」的坑。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
