# 产品/业务岗 AI 全栈 · Stage D《项目落地》· 过关手册

> 练习向 · 面向你本人使用
>
> 这一站你要跨过的两道关：
> - **D1 部署工业项目 #1：FF-Coworker AI 产品经理**，完整走一遍「Agent 读 AGENTS.md → 要凭证 → 装配起 → 自检」，并看懂它的三层降级（LIVE / 中途兜底 / DEMO）。
> - **D2 部署工业项目 #2（二选一）**：私域运营「autopilot」或 AI HR「小慧」，用 D1 的方法无提示独立再拿下一个，逐项自检全勾。
>
> 到这里你要证明的不是会写代码，而是：拿到任何带 AGENTS.md 的工业项目，你都能指挥 Agent 部署跑通、逐项过验收标准，还能自己讲清它的降级与安全边界。这就是「拿到别人做好的项目也能落地」的能力。
>
> 用法：先做「随堂练习」把部署与自检的动作走熟；再用「误区自查」照镜子；「迁移挑战」看你能不能把这套方法搬到别的项目；「排错挑战」练你独立止血；「面试自测 + 模拟面试官」帮你把项目讲清楚；最后用「能力自评表」和「落地对照」判断自己能接什么活。
>
> 本手册所有题目都取自 FF-Coworker、autopilot、小慧三个项目的真实源码与配套文档（README / AGENTS.md / docs/PRD.md），数字与字段名逐字核对，答案自包含，代码引用一律用仓库相对路径。

---

## 一、随堂练习（动手小题）

### D1 · 部署 FF-Coworker AI 产品经理
> 来源：`AGENTS.md` 部署完成的验证标准 / `package.json` / `src/lib/runtime-config.ts` / `src/app/api/agent/[id]/route.ts`

**练习 D1-1｜起服务并对 SSE 接口做一次自检**
你刚把 FF-Coworker 拉到本机。写出安装、启动的命令，以及启动后用来确认 SSE 接口活着的那条 `curl`，并说出你要在返回里盯的首帧与末帧。

- 【参考答案】
  ```bash
  pnpm install
  pnpm dev            # 固定监听 3210
  # 对某个员工的 Agent 接口发 SSE 请求（-N 关闭缓冲，才能看到流）
  curl -N http://localhost:3210/api/agent/alex
  ```
  首帧要含 `"t":"meta"`（带 agent / mode / startedAt 等元信息），末帧应为 `{"t":"end"}`。中间会依次流出 boot / phase / mechanism / tool / token / log / metric / artifact / done 等事件。
- 【自查要点】端口 `3210` 在 `package.json` 的 `dev` / `start` 脚本里硬编码（`next dev -p 3210`），不读环境变量；被占用就得释放端口或改脚本。`-N` 不能省，否则 `curl` 会把流缓冲住看不到逐帧。
  <!--src:A; ref:code:AGENTS.md 部署完成的验证标准; package.json; src/app/api/agent/[id]/route.ts-->

**练习 D1-2｜背下六条部署验证标准**
FF-Coworker 的 `AGENTS.md` 给了一份「部署完成的验证标准（Agent 自检清单）」共六条。凭记忆写出至少四条。

- 【参考答案】任写四条即可：
  1. `pnpm install` 无报错，`pnpm dev` 成功监听 `3210`；
  2. 打开首页正常渲染（Hero + 等距办公室 + 8 机制条 + 5 场景卡）；
  3. 访问 `/employee/alex`，点「派发任务」后思考流流式输出、工具调用栈增长、最终出现 DONE 总结；
  4. `curl -N /api/agent/alex` 返回 SSE 流，首帧含 `"t":"meta"`、末帧为 `{"t":"end"}`；
  5. 未配 Key 时徽章为 🟡 DEMO；配了有效 `sk-or-` Key 时为 🟢 LIVE，且出现「真实模式 · 调用 …」日志；
  6. `curl -s /api/agent/nope` 返回 HTTP 404 且体含 `agent_not_found`。
- 【自查要点】D1 的徽章要求是你**亲手逐条勾**这份清单（不让 Agent 代填），把 `curl` 输出存成产物文件，再手写一段解释三层降级为何这样设计。每一条都能用命令或界面直接看到结果，不靠感觉。
  <!--src:A; ref:code:AGENTS.md 部署完成的验证标准-->

**练习 D1-3｜说清 LIVE / DEMO 徽章是怎么判定的**
员工页右上角有个 🟢 LIVE 或 🟡 DEMO 徽章。它由后端哪个判定决定？判定只看什么？

- 【参考答案】由 `detectMode()` 决定：`getApiKey() ? "live" : "demo"`。`getApiKey()` 读服务端环境变量 `OPENROUTER_API_KEY`，先 `trim`，为空返回 `undefined`；**且必须以 `sk-or-` 开头**，否则也当作无效返回 `undefined`。所以只有「有值且以 `sk-or-` 开头」才进 LIVE，其余一律 DEMO。前端在 `meta` 事件里读到 `mode` 后切徽章文案。
- 【自查要点】判定只认 `sk-or-` 前缀，不真的去 ping OpenRouter。你就算填了别家的 key（比如 `sk-ant-` 开头的 Anthropic key），前缀不符也会被判成无效、退回 DEMO。默认模型是 `anthropic/claude-sonnet-4.6`（`AGENT_MODEL` 可覆盖）。
  <!--src:A; ref:code:src/lib/runtime-config.ts getApiKey/detectMode-->

**练习 D1-4｜画出三层降级在接口里的走法**
`GET /api/agent/[id]` 号称三层降级。请按代码把三条路径写清楚：什么时候走 L1、什么时候落 L2、什么时候进 L3。

- 【参考答案】先发一个 `meta` 事件，然后：
  - **L1 LIVE**：`mode==="live"` 且该员工有 persona，调 `runWithOpenRouter` 真跑 LLM，把解析出的事件逐条转发。
  - **L2 中途兜底**：L1 抛 `LLMRunFailed` 时发一条 warn 日志；若 `err.partial` 为真（已经有真事件流出去了），**不重放剧本**，只补一个 `done` 再 `end` 收尾；若是启动就失败、0 事件，则回落完整 Mock 剧本。
  - **L3 DEMO**：`mode` 不是 live（没有效 Key）时，直接按 `agent-scripts.ts` 里该员工的 Mock 剧本逐步 emit。
  最后统一发 `{"t":"end"}` 关闭流。
- 【自查要点】L2 的关键分叉是 `partial`：`runWithOpenRouter` 里 `emittedCount>0` 才把 `partial` 置真。已经吐过真事件就不重放整段剧本（否则观众会看到重复的开头），只补收尾。persona 与剧本都没有的 id 直接 404。
  <!--src:A; ref:code:src/app/api/agent/[id]/route.ts; src/lib/openrouter-runner.ts-->

### D2 · 部署第二个项目（autopilot 或 小慧，二选一）
> 来源：autopilot `AGENTS.md` / `src/app/api/health/route.ts` / `src/app/api/mode/route.ts`；小慧 `AGENTS.md` / `run.sh` / `backend/src/server.ts`

**练习 D2-1｜autopilot：起服务、健康、模式三连自检**
你选了 autopilot 做第二个项目。写出起服务的命令，以及确认「后端活着」和「当前跑哪个模式」的两条 `curl`，并说出未配 key 时你期望看到什么。

- 【参考答案】
  ```bash
  pnpm install
  pnpm dev                                    # 硬编码监听 3333
  curl -s http://localhost:3333/api/health    # 期望含 "ok":true, "service":"ff-autopilot"
  curl -s http://localhost:3333/api/mode       # 未配 key 期望 "mode":"mock"
  ```
  未配任何 key 时 `/api/mode` 返回 `"mode":"mock"`、`"provider":null`、`"reason":"未配 key · 走合成 mock"`；配了 key 才是 `"real"`。
- 【自查要点】autopilot 是 **mock-first**：不配任何 key 也能完整跑通（走合成事件流），所以「只做演示」时根本不用向用户要凭证，直接装起来。端口 `3333` 同样在 `package.json` 硬编码、不读环境变量。
  <!--src:A; ref:code:src/app/api/health/route.ts; src/app/api/mode/route.ts; AGENTS.md-->

**练习 D2-2｜autopilot：说清 mock / real 怎么切、数据为什么刷新就没**
autopilot 配了 key 就会真调大模型吗？它的数据刷新一下为什么就清空了？

- 【参考答案】切 real 的前提是 `buildClient()` 能建出客户端：它取 `OPENROUTER_API_KEY ?? ANTHROPIC_API_KEY`，**key 必须以 `sk-` 开头**，否则返回 `null` 回落 mock。有 key 时关键步骤（意图分类 / 尺码 / 起草回复、以及目标计划与重规划、周报）会真调 Claude，但时间轴巡航等仍走脚本编排，`/api/mode` 的 badge 只是切成 real。数据刷新就没，是因为 `src/lib/db.ts` 是**进程内内存 store**（模仿 better-sqlite3 接口），重启即清零，属设计使然。
- 【自查要点】`sk-or-` 走 OpenRouter（`baseURL` 指 openrouter），`sk-ant-` 走 Anthropic 原生（`baseURL` 留空）。想要落盘持久化，按 `db.ts` 注释把它换成 better-sqlite3 实现即可，其他代码零改动。之所以不默认上 better-sqlite3：Node 新版下 native 绑定要额外 rebuild，对教学 demo 不划算，内存 store 反而让演示节奏干净。
  <!--src:A; ref:code:src/lib/agent/runner-input.ts buildClient; src/lib/db.ts-->

**练习 D2-3｜小慧：一键起 + 现场演示权限拦截**
如果第二个你选的是小慧（AI HR），写出把它跑起来最省事的命令，并说清怎么在界面上现场演示「权限拦截」。

- 【参考答案】
  ```bash
  ./run.sh                                     # 一键起前后端，自动扫空闲端口
  curl http://localhost:8788/api/health         # 期望 ok:true, service:"ff-xiaohui-hr-backend", mode:"mock"
  ```
  `run.sh` 会从 8788（后端）/ 5190（前端）向上扫第一个空闲端口、绑 `0.0.0.0`、注入局域网地址，启动后终端打印「本机访问」与「局域网访问」两个 URL。演示权限：进 `/app` 总控调度台，用「普通员工」发「导出全公司花名册」会被拦截，切「HR 主管」再发同一句就放行。
- 【自查要点】权限判定就是一句「当前身份在不在这条任务的允许身份名单里」（`task.allow.includes(role)`），花名册导出只允许 `hr`，依据是《个人信息保护法》最小必要原则。手动单起前端时要保证 `frontend/.env` 的 `VITE_API_BASE` 端口和后端一致（默认 8788），用 `./run.sh` 则自动注入、不会对不上。
  <!--src:A; ref:code:run.sh; backend/src/server.ts; AGENTS.md 部署完成的验证标准-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句话，说明你还没真正掌握右边那件事，去回看对应节点。

1. **如果你以为「部署跑通就等于让 Agent 跑一遍 `pnpm dev` 就完事」**，说明你没抓住 D1 的验收在哪：徽章要求是你亲手逐条勾 AGENTS.md 的六条验证标准、把 `curl` 输出存成产物、再手写一段三层降级机制说明，不是让 Agent 代填。回看 product-fullstack#D1 的深潜验收说明。

2. **如果你以为「FF-Coworker 默认就在真调大模型」**，说明你没搞清它的默认态：没有效 Key 时 `detectMode()` 返回 `demo`，走 `agent-scripts.ts` 的 Mock 剧本，零外部依赖；只有 `OPENROUTER_API_KEY` 有值且以 `sk-or-` 开头才进 LIVE。回看 product-fullstack#D1，重点看 `src/lib/runtime-config.ts` 的 `getApiKey`。

3. **如果你以为「只要配了 Key 就一定是 LIVE 在真跑」**，说明你漏了中途兜底这层：Key 有效进了 L1，但 OpenRouter 非 2xx 或超时（默认 45s）会抛 `LLMRunFailed`，接口发 warn 日志后降级到 Mock。想定位就看页面「系统日志」里的 warn 行是哪个 stage 失败。回看 product-fullstack#D1，看 `src/lib/openrouter-runner.ts`。

4. **如果你以为「员工调的那些 `gitee.create_pr`、`aliyun.ecs_status` 是真在外呼」**，说明你没读安全边界：工具调用是叙事编排，`tool.result` 由 LLM 或 Mock 剧本编写以保证故事连贯，并不真连 Gitee / 阿里云 / 小红书。回看 product-fullstack#D1，看 `src/lib/employee-personas.ts` 顶部注释与 docs/PRD.md 第 5 章范围之外。

5. **如果你以为「autopilot 配了 key 就全程在真调 Claude」**，说明你把 badge 当成了全链路真跑：`buildClient()` 要 key 以 `sk-` 开头才建客户端，而且本版本只有关键步骤（意图分类、计划、重规划、周报）真调，时间轴巡航仍走脚本，`/api/mode` 只是把 badge 切成 real。回看 product-fullstack#D2，看 `src/lib/agent/runner-input.ts`。

6. **如果你以为「autopilot 或小慧刷新页面后数据还在」**，说明你没注意它们没有真数据库：autopilot 的 `src/lib/db.ts` 是进程内内存 store，重启即清零；小慧也是只读回放录得的 JSON。要持久化得自己换实现。回看 product-fullstack#D2，看 `src/lib/db.ts` 注释。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜把「Agent 读 AGENTS.md 部署」变成拿到任何陌生项目都能走的标准动作**
D1 和 D2 你都用同一句话让 Agent 部署。请把这套流程抽成一条你拿到任意带 AGENTS.md 的开源项目都能复用的动作线：从对方把项目发你，到你交付「能打开的地址 + 自检结果」，中间分几步、每步交付什么。

- 【怎么算过关】能给出一条可复用的动作线：先确认项目根有没有 `AGENTS.md` / `CLAUDE.md`、README 有没有「外部服务准备」表和一份可逐项核对的验证标准 → 让 Agent 读手册、查环境 → 向对方要齐外部凭证（且提醒凭证只进 `.env.local`、不进代码不进 git）→ 一键起 → 逐条过验证标准 → 把自检输出和访问地址交付。能指出「先走 mock / DEMO 那条零外部依赖最稳、任何环境都能演示」，说明你抓住了三个项目共有的设计取舍。
  <!--src:A; ref:code:AGENTS.md; README.md-->

**迁移挑战 2｜给你自己的演示项目加一层「无 Key 也能跑」的降级**
FF 用三层降级、autopilot 用 mock-first，共同点是「没配外部服务也能完整演示」。假设你有一个自己写的、要真调某个外部 API 的小工具，请照这个思路给它设计降级：默认跑什么、配了凭证跑什么、凭证失效或超时怎么办、前端怎么让人一眼看出当前在哪个模式。

- 【怎么算过关】① 默认态不依赖任何外部服务，用一份录得 / 合成数据就能把主链路走完；② 有有效凭证时才切真调，且凭证只从服务端读、不透传前端；③ 真调失败或超时能自动回落到默认态，不把整个服务搞崩；④ 给一个像 FF 的 🟢/🟡 徽章或 autopilot 的 `/api/mode` 这样的显式信号，让人一眼看出真调还是演示。能说清「为什么默认要选最不依赖外部的那条」，就说明你把「课堂 / 直播零翻车」这条设计迁移过来了。
  <!--src:A; ref:code:src/lib/runtime-config.ts; src/app/api/mode/route.ts-->

---

## 四、排错挑战（给你一段带病的配置，修好它）

一位同学部署 FF-Coworker，想走 LIVE 真调大模型。他手上有一个 Anthropic 控制台创建的 key，就直接填进了项目根的 `.env.local`：

```bash
# .env.local
OPENROUTER_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxx
AGENT_MODEL=anthropic/claude-sonnet-4.6
```

他重启了 `pnpm dev`，但员工页右上角**一直是 🟡 DEMO**，系统日志也只有「演示模式 · 预设剧本」，从没出现「真实模式 · 调用 …」。请找出问题并修好。

- 【症状】配了 key、也重启了服务，但徽章始终 🟡 DEMO，走的是 Mock 剧本，从未真调 LLM；`curl -N /api/agent/alex` 的首帧 `meta.mode` 是 `"demo"` 而不是 `"live"`。
- 【根因】`getApiKey()` 对 key 做了前缀校验：`if (!k.startsWith("sk-or-")) return undefined`。他填的是 Anthropic 的 `sk-ant-` 开头 key，前缀不是 `sk-or-`，被判成无效返回 `undefined`，于是 `detectMode()` 落到 `demo`。FF-Coworker 的真调只走 OpenRouter（`src/lib/openrouter-runner.ts` 打的是 OpenRouter 端点），并不直连 Anthropic，所以 Anthropic 的 key 在这里根本用不上。
- 【修法】去 openrouter.ai 的 Keys 页创建一个 `sk-or-` 开头的 key，替换进 `.env.local`：
  ```bash
  # .env.local
  OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxx
  AGENT_MODEL=anthropic/claude-sonnet-4.6
  ```
  重启 `pnpm dev` 即可看到 🟢 LIVE 与「真实模式 · 调用 …」日志。（注意：`sk-ant-` 前缀在另一个项目 autopilot 里是有效的，因为它 `buildClient()` 只要求 `sk-` 前缀并支持 Anthropic 直连；两个项目的校验口径不同，别搞混。）
- 【一句话总结】FF-Coworker 的 LIVE 判定只认 `sk-or-` 前缀、只真调 OpenRouter；填了别家格式的 key 会被静默判无效、退回 DEMO。
  <!--src:A; ref:code:src/lib/runtime-config.ts getApiKey; src/lib/openrouter-runner.ts; AGENTS.md 外部服务与凭证-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。
> 说明：以下题目按 2026 年 AI 应用 / Agent 工程岗真实高频方向校准，考点集中在 AGENTS.md 部署规范、SSE 事件流、三层降级 / mock-first、凭证与安全边界、上下文工程机制、部署验收工程。

**面试 1｜AGENTS.md 是什么？它和 README.md、CLAUDE.md 怎么分工？**
- 考察点：对「给 AI Agent 读的操作手册」这一开放标准的理解。
- 答题要点：AGENTS.md 是给 AI 编程 Agent（Claude Code / Codex / Cursor 等）读的操作手册，命令优先、精确到 flag，让 Agent 能自动完成环境检查 → 要凭证 → 安装配置 → 启动 → 逐项自检。README.md 是给人读的产品与上手说明。CLAUDE.md 是 Claude Code 的入口，三个项目里内容通常只有一行 `@AGENTS.md`，把 Claude Code 转去读 AGENTS.md。三者内容对齐、受众不同。
- 追问：「为什么把给 Agent 的手册单独拆一份，而不都塞进 README？」，因为 Agent 要的是可直接执行的命令与验证标准，人要的是背景与产品说明；拆开后 Agent 读起来歧义少、执行更稳，人读 README 也不被一堆命令淹没。
  <!--src:A; ref:code:AGENTS.md; CLAUDE.md; README.md-->

**面试 2｜FF-Coworker 的三层降级是怎么设计的？为什么默认 DEMO？**
- 考察点：演示系统在「课堂可靠」与「真实接入」之间的取舍。
- 答题要点：L1 LIVE 有效 `sk-or-` Key 时真调 OpenRouter；L2 中途兜底，LIVE 调用失败时按是否已吐过真事件（`partial`）决定「只补 done 收尾」还是「回落完整 Mock 剧本」；L3 DEMO 无 Key 时直接跑 `agent-scripts.ts` 的确定性 Mock 剧本。默认 DEMO 是为了任何人零配置、零外部依赖就能看到完整链路，直播不翻车。
- 追问：「LIVE 中途失败、且已经吐过几条真事件了，为什么不干脆从头重放剧本？」，因为重放会让观众看到重复的开头、破坏叙事连贯；已有部分真事件时只补一个 done 收尾更干净。
  <!--src:A; ref:code:src/app/api/agent/[id]/route.ts; src/lib/runtime-config.ts-->

**面试 3｜LIVE 判定为什么要做 `sk-or-` 前缀校验？配了 Key 仍是 DEMO 怎么排查？**
- 考察点：凭证校验与降级判定的排障能力。
- 答题要点：`getApiKey()` 先 `trim`，为空返回 `undefined`，再要求以 `sk-or-` 开头，否则也返回 `undefined`；`detectMode()` 据此判 live/demo。前缀校验是为了快速挡掉「填了别家格式 key」这类无效值，不必真发一次请求就能判无效。配了仍 DEMO，先确认 `.env.local` 里 `OPENROUTER_API_KEY` 确实以 `sk-or-` 开头、没有多余空格；若前缀对但仍降级，多半是 OpenRouter 非 2xx 或超时，看页面系统日志的 warn 行定位 stage。
- 追问：「填了 Anthropic 的 `sk-ant-` key 会怎样？」，会被判无效退回 DEMO，因为 FF-Coworker 只真调 OpenRouter、不直连 Anthropic；要真调得换 `sk-or-` 开头的 OpenRouter key。
  <!--src:A; ref:code:src/lib/runtime-config.ts getApiKey; AGENTS.md 常见故障-->

**面试 4｜这个 SSE 接口为什么必须跑在 Node runtime？它的事件契约长什么样？**
- 考察点：SSE 流式接口的运行时约束与协议设计。
- 答题要点：`GET /api/agent/[id]` 声明 `runtime="nodejs"`、`dynamic="force-dynamic"`。跑在 Edge runtime 会导致 SSE 无输出，所以要保持 Node runtime。响应头是 `text/event-stream`，每帧 `data: {json}\n\n`。首帧是 `meta`（含 agent / mode / model / totalMs / steps / startedAt），末帧是 `{"t":"end"}`，中间流出 boot / phase / mechanism / tool / token / log / metric / artifact / done。persona 与剧本都不存在的 id 返回 HTTP 404、体为 `{"error":"agent_not_found"}`。
- 追问：「`curl` 看不到逐帧输出、只在最后一次性打印，为什么？」，多半没加 `-N`，`curl` 默认缓冲；SSE 要用 `curl -N` 关掉缓冲才能实时看到每帧。
  <!--src:A; ref:code:src/app/api/agent/[id]/route.ts; AGENTS.md 常见故障-->

**面试 5｜真调 OpenRouter 时，怎么把 LLM 的输出变成前端能消费的 SSE 事件？**
- 考察点：流式解析里「两层缓冲」的理解。
- 答题要点：`runWithOpenRouter` 让 LLM 被 system prompt 约束输出 NDJSON（每行一个事件），POST OpenRouter 的 `chat/completions`（`stream:true`、`temperature:0.7`、`max_tokens:3500`）。解析有两层缓冲：外层按 OpenAI 的 SSE 帧（以 `\n\n` 分隔、取 `data:` 行）累积 delta 拼出 LLM 文本；内层把拼出的文本按 `\n` 切成一行行 NDJSON，逐行 `JSON.parse`，校验是含字符串 `t` 字段的对象后 `send` 给前端。单行解析失败静默丢弃、不打断流。
- 追问：「LLM 流跑完但一条合法事件都没解析出来会怎样？」，`emittedCount===0` 时抛 `LLMRunFailed("parse", …)` 且 `partial=false`，接口据此回落完整 Mock 剧本，保证前端仍有完整演示。
  <!--src:A; ref:code:src/lib/openrouter-runner.ts; src/lib/employee-personas.ts-->

**面试 6｜这个演示站要讲清「上下文工程 8 大机制」，是哪 8 个？在演示里怎么被看见？**
- 考察点：对 Anthropic 上下文工程机制的理解与可视化落地。
- 答题要点：8 机制出自 Anthropic《Effective context engineering for AI agents》（2025-09-29）：① 系统提示、② 工具定义、③ 范例提示（Few-Shot）、④ 消息历史、⑤ 按需检索（Just-in-Time Retrieval）、⑥ 上下文压缩（Compaction）、⑦ 结构化笔记、⑧ 子 Agent 架构。演示里每个员工的任务会流出 `mechanism` 事件（带 `id` 和估算 `tokens`），前端把对应机制条点亮、显示 token 占用，让你直观看到一次任务激活了哪几条机制。
- 追问：「每个员工不是把 8 条全用一遍吧？」，不是，每个 persona 有自己的 `mechanismFocus`（偏向 3-4 条），比如 CTO 陈昊偏系统提示 / 按需检索 / 范例提示 / 结构化笔记，演示时只重点点亮这几条。
  <!--src:B; ref:code:src/lib/mechanisms.ts; src/lib/employee-personas.ts；一手来源 Anthropic《Effective context engineering for AI agents》2025-09-29-->

**面试 7｜员工调的那些工具是真外呼吗？这样设计的安全边界在哪？**
- 考察点：演示系统的边界诚实与凭证安全。
- 答题要点：不是真外呼。`tool.result` 由 LLM 或 Mock 剧本编写以保证叙事连贯，并不真连 Gitee / 阿里云 / 小红书，`employee-personas.ts` 顶部与 docs/PRD.md 第 5 章都写明了。安全红线上：`OPENROUTER_API_KEY` 只从服务端 `process.env` 读、绝不透传前端或客户端组件；凭证只进 `.env.local`、不硬编码不提交 git；任何改动都要保留三层降级，保证无 Key 时仍能跑 Mock 剧本。
- 追问：「设置页里还有个 `ANTHROPIC_API_KEY` 输入框，是不是能直连 Anthropic？」，不能，设置页是展示态、输入不落库，运行时只读 `OPENROUTER_API_KEY`，那个输入项属规划叙事而非已实现能力（docs/PRD.md 第 13 章有记录）。
  <!--src:A; ref:code:src/lib/employee-personas.ts; src/lib/runtime-config.ts; docs/PRD.md 第5章/第13章-->

**面试 8｜autopilot 的 mock-first / real-on-key 是怎么判定的？跟 FF 的校验有什么不同？**
- 考察点：不同项目的凭证判定口径对比。
- 答题要点：autopilot 默认 mock-first，不配 key 也能跑合成事件流。`/api/mode` 判定 `OPENROUTER_API_KEY` 或 `ANTHROPIC_API_KEY` 是否以 `sk-` 开头，有则 `real`、无则 `mock`。`buildClient()` 取两者之一，key 必须 `sk-` 开头否则返回 `null` 回落 mock；`sk-or-` 走 OpenRouter，`sk-ant-` 走 Anthropic 直连。跟 FF 的区别：FF 只认 `sk-or-`、只走 OpenRouter；autopilot 认 `sk-` 前缀、两家都支持。
- 追问：「autopilot 显示 real 就代表整个流程都在真调吗？」，不代表，本版本只有意图分类、计划、重规划、周报等关键步骤真调 Claude，时间轴巡航仍走脚本，`/api/mode` 只是把 badge 切成 real。
  <!--src:A; ref:code:src/app/api/mode/route.ts; src/lib/agent/runner-input.ts buildClient-->

**面试 9｜autopilot 没用数据库、用内存 store，代价是什么？为什么这么选？**
- 考察点：演示项目的数据层取舍与事件溯源。
- 答题要点：`src/lib/db.ts` 是进程内内存 store，接口模仿 better-sqlite3（prepare/get/all/run/exec），事件全部 event-sourced（时间轴可回放）。代价是重启即清零、无并发写与事务。之所以这么选：Node 新版下 better-sqlite3 的 native 绑定要额外 rebuild，对教学 demo 不划算，内存 store 让演示节奏更干净；要落盘只需按注释把这一个文件换成 better-sqlite3 实现，其他代码零改动。
- 追问：「HITL 人工审批那步说是持久暂停，重启会不会丢？」，事件与暂停状态都记在同一个内存 store 里，进程重启就清空；要做到真正重启不丢，得先把 store 换成落盘实现。
  <!--src:A; ref:code:src/lib/db.ts; src/lib/agent/goal-runner.ts-->

**面试 10｜让你验收一次「Agent 部署」，你怎么证明它真部署成功、而不是 Agent 嘴上说成功？**
- 考察点：把自检清单变成可复现证据的工程习惯。
- 答题要点：不听 Agent 自述，逐条跑 AGENTS.md 的验证标准拿证据。以 FF 为例：`curl -N /api/agent/alex` 看首帧 `meta`、末帧 `end`；`curl /api/agent/nope` 看 404 与 `agent_not_found`；浏览器进 `/employee/alex` 看思考流 / 工具栈 / DONE；看徽章 DEMO/LIVE 是否与是否配 Key 一致。autopilot 则看 `/api/health` 的 `ok:true`、`/api/mode` 的 mock/real。每条存下输出，再手写一段机制说明。这样才叫「你验收它」。
- 追问：「Agent 报告全部通过，但你 `curl /api/mode` 拿到 `real` 而你根本没配 key，说明什么？」，说明环境和你以为的不一致，得停下来核对 `.env.local` 里到底有没有 `sk-` 开头的 key，别急着签收。这也是把 `release-gate-acceptance`（发布门：测试全绿 + 无孤儿需求 + 无破坏性契约变更）那种「机械判据、不靠感觉」的习惯用到部署验收上。
  <!--src:A; ref:code:AGENTS.md 部署完成的验证标准; src/app/api/mode/route.ts; pack:ai-product-dev-pack-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请只做考官、不做辅导。

```
请你扮演一名技术面试官，围绕「拿一个带 AGENTS.md 的工业项目、指挥 Agent 部署并逐项验收」这一阶段考我。规则如下：

【角色纪律】请只做考官、不做辅导。在我明确说「结束，请点评」之前，你不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，且尽量贴近真实岗位：
1. AGENTS.md：它是给谁读的、和 README / CLAUDE.md 的分工、为什么单独拆一份给 Agent。
2. 部署与自检：一键起服务的命令、健康检查看哪些字段、如何逐条过「部署完成的验证标准」、SSE 为什么要 curl -N。
3. FF-Coworker 三层降级：LIVE / 中途兜底 / DEMO 各在什么条件、为什么默认 DEMO、sk-or- 前缀校验、配了 Key 仍 DEMO 怎么排。
4. SSE 与解析：接口为什么要 Node runtime、meta/end 事件契约、404 agent_not_found、真调时 OpenAI SSE 帧与 NDJSON 两层缓冲怎么解析。
5. autopilot mock-first / real-on-key：/api/mode 判定、buildClient 的 sk- 前缀、real 是否全程真调、内存 store 为什么刷新清零。
6. 安全与边界：凭证只进 .env.local 不透传前端、工具调用是叙事编排非真外呼、上下文工程 8 机制。
7. 工程判断：如何证明 Agent 真部署成功而不是嘴上说成功。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 如果我答「不会」，记录后直接下一题，不要讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现评价（答对 / 部分 / 未答）、暴露的薄弱点、以及建议我回看这一阶段的哪个环节（D1 FF-Coworker / D2 autopilot 或小慧）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件 / Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 让 Agent 读 AGENTS.md，走完「查环境 → 要凭证 → 装配起 → 自检」全流程 | ☐ | ☐ | ☐ |
| 逐条核对 FF-Coworker 六条验证标准并手写机制说明 | ☐ | ☐ | ☐ |
| 用 `curl -N /api/agent/alex` 看懂 SSE 首帧 meta、末帧 end | ☐ | ☐ | ☐ |
| 讲清 FF 三层降级 LIVE / 中途兜底 / DEMO 各自条件与 partial 分叉 | ☐ | ☐ | ☐ |
| 说清 LIVE 判定为何要 `sk-or-` 前缀校验、配了 Key 仍 DEMO 怎么排 | ☐ | ☐ | ☐ |
| 讲清 SSE 接口为何要 Node runtime、404 agent_not_found 的触发条件 | ☐ | ☐ | ☐ |
| 讲清真调时 OpenAI SSE 帧与 NDJSON 两层缓冲的解析 | ☐ | ☐ | ☐ |
| 说出上下文工程 8 大机制并解释演示里怎么被点亮 | ☐ | ☐ | ☐ |
| 说清工具调用是叙事编排非真外呼、凭证只进 `.env.local` 不透传前端 | ☐ | ☐ | ☐ |
| autopilot：起服务 + `/api/health` + `/api/mode` 三连自检 | ☐ | ☐ | ☐ |
| 讲清 autopilot mock-first / real-on-key 判定与内存 store 刷新清零 | ☐ | ☐ | ☐ |
| 小慧：`./run.sh` 一键起 + 现场演示三档权限拦截 | ☐ | ☐ | ☐ |
| 排掉「填了 `sk-ant-` key 却一直 DEMO」这类凭证判定故障 | ☐ | ☐ | ☐ |
| 拿到一个陌生项目，判断它能否走「Agent 读 AGENTS.md 部署」这条路 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」，其中「逐条核对验证标准并手写机制说明」「讲清三层降级」这两项能到「能教别人」，你才算真正拿下 D 阶段徽章，因为徽章要求就是你不靠 Agent 全代跑、亲手逐条 `curl` 自检并讲清三层降级为何这样设计。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的活儿，描述自包含。它们是你作品集和现金流的第一步。

1. **帮别人把带 AGENTS.md 的开源 / 工业项目在他电脑上部署跑通。**
   很多人有「想用某个 AI 项目但装不明白」的需求。你会让 Agent 读 AGENTS.md、查环境、要凭证、一键起，再逐条过验证标准，交付「能打开的地址 + 自检输出」。默认走 DEMO / mock 那条零外部依赖最稳，交付成功率高。

2. **做「AI 项目部署 + 验收」的陪跑服务。**
   面向不敢自己动手的人：你带着他把一个工业项目部署起来，逐条过验证标准，并教他用 `/api/health`、`/api/mode`、`curl -N` 这些自检手段确认真的跑通，而不是 Agent 嘴上说通。交付物是一份可复现的部署与自检记录。

3. **给一个要真调外部 API 的演示项目加一层「无 Key 也能跑」的降级。**
   你已经拆透了 FF 的三层降级和 autopilot 的 mock-first：默认合成数据、有凭证才真调、失败自动回落、前端显式显示当前模式。这套「直播零翻车」的降级正是很多演示 / 路演项目缺的，你能帮别人补上。

4. **给只读回放的演示系统做数据脱敏与替换。**
   小慧的数据是 `backend/data/*.json`、autopilot 是内存 store 里的合成事件。你懂这套「录得 / 合成数据回放」的结构，可以帮别人把演示系统里的真实数据换成脱敏样例，做一个能对外展示又不泄露真实信息的版本。

5. **把一个跑通的项目整理成「命令优先、带验证标准」的部署手册。**
   你自己部署一遍后，能照 AGENTS.md 的写法给别人的项目补一份 AGENTS.md / 验证清单，让他们的项目也能被 Agent 一键读懂部署。这本身就是一单可交付的文档活，也顺带把项目变成你作品集里能讲清楚的一项。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
