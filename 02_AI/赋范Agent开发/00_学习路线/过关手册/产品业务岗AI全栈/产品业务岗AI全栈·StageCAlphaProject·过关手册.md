# 产品/业务岗 AI 全栈 · Stage C《AlphaProject 三部曲》· 过关手册

> 练习向 · 面向你本人
>
> 这一阶段你要跨过的三道关，就是完整主导一个 MVP 的三段路：
> - **C1 上篇 · 深度调研与生产级 PRD**：会用 AI 协同做行业与用户调研，产出一份结构完整、可交付的生产级 PRD（含 6 篇调研笔记）
> - **C2 中篇 · 架构选型、UI 设计与环境配置**：会做前后端技术栈选型、把 Spec-Kit 装进项目、把 PRD 的 6 大功能规范化为 spec/plan/tasks，并导出 UI 设计参考
> - **C3 下篇 · 全栈测试体系**：会用「后端单测 / 前端交互 / 全栈切片 / 全链路」四层测试把 MVP 打磨稳定，用 run-feature 收口特性
>
> 用法：先做「随堂练习」确认手熟；再用「误区自查」照镜子；「迁移挑战」看你会不会换个产品用同一套方法；「排错挑战」练你独立止血；「面试自测 + 模拟面试官」帮你把嘴皮子练顺；最后用「能力自评表」和「落地对照」判断自己站到哪一格、能接什么活。
>
> 本手册题目取自 AlphaProject（A 股自动盯盘 AI 助手）实际课件、真实 PRD 与项目源码，数字与字段名逐字核对。每题答案后有一行给机器读的来源标注（你看不见），用来核验题目有据可查。
>
> 提醒：C 阶段徽章的过关点在 C1 的深潜验收上：关掉能力包代笔，自己亲笔主导 PRD 的「问题定义 / 目标用户 / 核心功能范围与非目标」三节，并为每个功能标注它来自哪篇调研。下面的 C1 练习就是照它设计的。

---

## 一、随堂练习（动手小题）

### C1 · 上篇：深度调研与生产级 PRD
> 来源：《AlphaProject 上篇：深度调研与产品需求文档生成》，能力包 ai-product-dev-pack，配套 AlphaProject 项目 specs/ 产出

**练习 C1-1｜数清 6 篇调研笔记 + 双路调研为什么是写死规范**
不看项目，凭方法论写出立项调研「4 路并行 + 1 路收敛」分别产出哪几个文件，以及每个 sub-agent 必须同时用哪两路检索、为什么。

- 【参考答案】4 路并行产出 `specs/research/01-产品形态.md`、`02-数据来源.md`、`03-开源项目.md`、`04-实现方案.md`（4 主题之间无依赖，可 4 个 sub-agent 同时跑），最后串行收敛出 `05-决策汇总.md`；若做了对抗选型再加一份 `06-架构基线决策.md`。双路：**路 A** 用 Claude 内置 `WebSearch` / `WebFetch`（偏海外 / 英文 / 官方 / GitHub），**路 B** 用国内合规检索源 `web_search` / `web_fetch` / `web_map`（偏中文 / 国内）。写死是因为中国市场场景单靠 WebSearch 会漏大量中文信源，双路交叉验证是规范不是可选项。
- 【自查要点】最终真正的交付物不是 5 份文件，而是一份完整的**并行调研提示词**：用户复制一次就能启动 4-agent 并行，里面要含项目定位、输出路径、4 主题、汇总、质量要求、双路规范、并行说明、下一步衔接。
<!--src:A; ref:pack:ai-product-dev-pack 立项并行调研（4 路并行 + 双路规范）; code:AlphaProject/specs/research/01-05.md-->

**练习 C1-2｜复述架构基线决策 06 的最终基线与 A/B 双轨人日**
照 AlphaProject 的 `06-架构基线决策.md`，写出这个项目最终锁定的架构基线是什么组合，A 轨 / B 轨各约多少人日、差在哪。

- 【参考答案】架构基线 = **TradingAgents-CN（后端内核）+ TauricResearch/TradingAgents（上游同步）后端复合 + 前端自主构建**（不 fork stock-dashboard，它降级为「视觉参考」看不抄代码）。**A 轨**（首选，依赖 CN 商授通过）：fork CN 含闭源后端商授使用 + 前端自建 + 月度 cherry-pick 上游，预估 **~30 人日**；**B 轨**（兜底，商授失败时）：只用 CN 开源内核 `tradingagents/`（Apache-2.0）+ 上游同步 + 自建 FastAPI 轻量后端，预估 **~40 人日**。差别就在「后端能不能直接商授用 CN 的 `app/`」。
- 【自查要点】被否的都是「单 fork」方案：go-stock 单 fork（GPLv3 商业自杀，owner 自承）、ai-hedge-fund 单 fork（美股化改造反涨到 46 人日）、全自建（62 人日）。抓住一句：能复用别人已工程化的中文 prompt 与 A 股 dataflows，就别从零写。
- 【自查要点补】这份决策是 5 角色法庭辩论后 Lead 终审写的，只基于「幸存论点」，不基于「谁先说 / 谁写得长」。
<!--src:A; ref:code:AlphaProject/specs/research/06-架构基线决策.md（基线组合 + A轨30/B轨40人日 + 被否方案）-->

**练习 C1-3｜PRD 的 What/How 边界 + Non-Goal**
给你三条要写进文档的话：①「首屏加载 ≤ 2 秒」②「用 React + Vite 实现 Dashboard」③「不做下单 / 交易执行」。判断各自进 PRD 还是搬 TRD，并说出判断法。写出 AlphaProject PRD 里至少 3 条 Non-Goal。

- 【参考答案】①进 PRD（What：换一种实现仍要满足）；②搬 TRD（How：指定了技术栈，工程师换实现就不成立，污染 What/How 边界）；③进 PRD 的 Non-Goal（明确不做的范围）。判断法一句话：**工程师能换一种实现仍满足该条 → 留 PRD；不能换实现 → 那是技术决定，搬 TRD**。AlphaProject 真实 Non-Goal 任举三条：不做下单 / 交易执行（未持牌避监管）、不对外开放 / 不收费 / 不做 SaaS（单用户本地工具）、不做 Tick / Level-2 实时行情（分钟级足够）。
- 【自查要点】PRD 是决议书不是探讨稿。没有 Out-of-Scope，范围一定会爆；把技术栈写进功能描述，就把实现绑死了。AlphaProject PRD 结尾专门有一张「反模式自检」清单，写完必逐条勾。
<!--src:A; ref:pack:ai-product-dev-pack 生产级 PRD（What/How 边界 + 反模式）; code:AlphaProject/specs/prd.md §2.3 非目标-->

---

### C2 · 中篇：架构选型、UI 设计与环境配置
> 来源：《AlphaProject 中篇：技术架构选型·UI设计与环境配置》，能力包 ai-product-dev-pack / spec-driven-dev-pack，配套 AlphaProject 项目 .specify/ 与 specs/001-006

**练习 C2-1｜6 大功能规范化后长什么样**
写出 AlphaProject 把 PRD 的 6 个 P0 功能规范化后，`specs/` 下的 6 个功能目录名，以及每个目录里必有的三件套文件。

- 【参考答案】6 个功能目录：`001-watchlist-crud`（自选股 CRUD）、`002-feishu-push-channel`（飞书推送通道）、`003-watchlist-dashboard`（自选股看板）、`004-llm-anomaly-explain`（LLM 异动解读）、`005-anomaly-detection-push`（异动检测推送）、`006-morning-briefing`（晨报）。每个目录必有三件套：`spec.md`（只写行为 What/Why，不含技术选型）、`plan.md`（技术方案）、`tasks.md`（可执行任务拆解）。Spec-Kit 装好后落在项目 `.specify/`，含 `memory/constitution.md`（宪法）、workflows、templates、scripts。
- 【自查要点】`spec.md` 顶部会写清「本文档仅描述 What & Why，不含任何技术选型（存储引擎、框架、库、表结构均留给 plan.md）」，这就是 spec 与 plan 的分工：spec 是唯一事实源，plan 才落实现。
<!--src:A; ref:code:AlphaProject/specs/001-watchlist-crud ~ 006-morning-briefing（三件套 + .specify/）-->

**练习 C2-2｜Spec-Kit 命令链 + 为什么不能整份 PRD 喂给 /specify**
写出 Spec-Kit 从宪法到实现的命令链（8 步），以及为什么要按 Must-have 逐条拆、不能把整份 PRD 一次喂给 `/specify`。clarify 阶段的纪律是什么？

- 【参考答案】命令链：`constitution → specify → clarify → plan → tasks → analyze → checklist → implement`。按 Must-have 拆条各跑一次，是因为整份 PRD 一次喂给 `/specify` 会把上下文撑爆，而且 `/specify` 自己会从 `specs/prd.md` 取材料，不用你手动整份灌。clarify 纪律：按 11 类扫描、最多问 5 个、一次只问一个、答案回填直到 `NEEDS CLARIFICATION` 标记清零。
- 【自查要点】跳过 PRD 直接 `/specify`，spec 会缺业务上下文、AI 就瞎补默认假设；这跟「想法→直接写代码」是同一类头号返工。tasks 会分 Setup / Tests / Core / Integration / Polish 几个 Phase，能并行的标 `[P]`。
<!--src:A; ref:pack:spec-driven-dev-pack Spec-Kit 命令链（8 命令 + clarify 11类/最多5问）-->

**练习 C2-3｜加了 UI 设计要不要重跑 /specify + CLAUDE.md 为什么 ≤200 行**
上篇 PRD 和 spec/plan/tasks 都跑完了，现在补了一套 UI 设计（DESIGN.md + 原型），你要不要重新跑一遍 `/specify`？项目根的 `CLAUDE.md` 为什么控制在 200 行以内、不抄整份 PRD？

- 【参考答案】不重跑 `/specify`。业务需求没变，只是加了设计，重跑等于把已做的工作推倒重来、纯浪费。正确做法是设计注入：更新宪法（constitution injection）+ 只选择性重跑「触及 UI 的那几个 feature」的 plan/tasks，不推倒重来。`CLAUDE.md` 是一张 ≤200 行的项目记忆路由图（Karpathy 4 原则），抄整份 PRD 会撑破 200 行、让指令遵循度断崖式下降，它该指路，不该复述内容。
- 【自查要点】AlphaProject 的 UI 走的是自主构建（Vite + React + TS + shadcn/ui），`stock-dashboard` 只当视觉参考，导出物落在 `specs/design-reference/stitch-export/`（screen PNG + HTML + DESIGN.md），看不抄代码。
<!--src:A; ref:pack:ai-product-dev-pack 设计注入 Spec-Kit + CLAUDE.md 项目记忆（≤200行/不推倒重来）; code:AlphaProject/specs/design-reference/stitch-export/-->

---

### C3 · 下篇：全栈测试体系
> 来源：《AlphaProject 下篇：全栈测试体系》，能力包 ai-product-dev-pack / superpowers-workflow-pack，配套 AlphaProject backend/tests 与 frontend 真实测试

**练习 C3-1｜四层测试 + 一个改动怎么路由**
写出四层测试体系是哪四层，以及给你一处改动时按什么信号判它该写哪层测试。

- 【参考答案】四层：**后端层 / 前端层 / 切片层（局部前后端）/ 全链路层**。判类信号与路由：只触后端单元（`[BE]` 风格）→ 后端层；只落前端不联调真后端（`[FE]`）→ 前端层；「打通 X 到 Y」、feature 内端到端（`[INT]`）→ 切片层；依赖图 A→B→C 跨 feature + 该旅程的验收 → 全链路层；输出是另一模块输入且有契约声明 → 跨模块契约（接口接缝，漂移高发区）。
- 【自查要点】测试跟着「行为」走不跟「模块」走。别习惯性给每处改动都写全链路。条件命中才升层：涉 DB 写 / 约束 / 迁移才标真库，多用户 / 隔离 / 特权才标越权，共享资源 / 限频 / 配额才标并发，调外部依赖才标韧性。纯逻辑纯读的简单后端可以一项都不命中。
<!--src:A; ref:pack:ai-product-dev-pack 测试场景路由（判类信号 × 四层执行器）-->

**练习 C3-2｜风险分级 P0-P3 + 异动规则真实阈值**
写出测试风险分级 P0-P3 各是什么、权限行为（越权）默认定几级。再写出 AlphaProject 异动检测的真实阈值：主板 / 创业板科创板 / ST 的涨跌停幅度、振幅异常、量比异常各是多少。

- 【参考答案】按「出事的代价 × 概率」给**每条行为**定级（不是给模块）：**P0** 失败即不可逆（数据损坏 / 丢失、权限越界、资金额度错算、安全边界击穿），必测、发布门硬输入；**P1** 核心流程不可用但可恢复；**P2** 边缘 / 非关键可延后；**P3** 无逻辑可不测。权限行为（BOLA/BFLA）默认 **P0**。异动真实阈值（`backend/app/config.py`）：涨跌停 `ANOMALY_LIMIT_MAIN = 10.0`（主板）、`ANOMALY_LIMIT_STARTUP = 20.0`（创业板 300* / 科创板 688*）、`ANOMALY_LIMIT_ST = 5.0`（ST）；振幅异常 `ANOMALY_AMPLITUDE_PCT = 8.0`；量比异常 `ANOMALY_VOLUME_RATIO = 3.0`。
- 【自查要点】同一个模块里「扣减额度 / 破上限」是 P0、「返回提示文案」可能就是 P2。判板块靠 `board_kind()`：名称含 ST 归 ST、代码 `300`/`688` 前缀归创业板科创板、其余归主板。
<!--src:A; ref:pack:ai-product-dev-pack 测试风险分级 P0-P3; code:AlphaProject/backend/app/config.py（ANOMALY_LIMIT_MAIN=10/STARTUP=20/ST=5/AMPLITUDE=8/VOLUME_RATIO=3）; app/anomaly/price_rules.py board_kind-->

**练习 C3-3｜补测五步 + 自愈护栏 + 一个真实缺口**
写出结构性缺口的补测五步、自动补测前必守的自愈护栏（至少 3 条），并举 AlphaProject 里一个真实被补的 P0 缺口。

- 【参考答案】补测五步：**检测缺口 → 生成测试（先红复现缺口）→ 跑 → 修产品码到绿 → 固化为回归（进常驻套件）**。补测前先分类：已被开发期 TDD / 契约覆盖的跳过，从未被任何层测到的真实风险才是目标；不固化回归，同一 bug 会再回来。自愈护栏（任何自动生成 / 修复测试前必守）：①只写 `tests/` 目录 ②断言不可弱化 ③禁伪造修复 ④有界重试 ⑤产 PR 交人审。真实缺口 G-01：自选股 30 只 / 持仓 5 只上限校验是 read→check→write 两段无锁，两个并发线程各自通过 `len(items) < 30` 后都 INSERT，最终能加到 31 只（P0，阻断发布）；修法是 service 层加锁串行化。
- 【自查要点】这个 race 的测试是 `backend/tests/test_gap_concurrency_caps.py`，登记在 `tests/GAPS.md` 里，可追溯到 `F5 / 001-watchlist-crud / cap=30`。反过来 G-09（BOLA/BFLA 越权）标「不命中、永久不做」，因为 PRD §7.3 是单用户 Tailscale 内网、无 auth 无多租户，不命中的维度就别硬测。
<!--src:A; ref:code:AlphaProject/backend/tests/GAPS.md（G-01 caps race P0 / G-09 BOLA不命中）; tests/test_gap_concurrency_caps.py-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句想法，说明你还没真正掌握右边那件事，去回看对应的一站。

1. **如果你以为「调研做完就能直接写 PRD」** → 说明你没看清调研只解决「市面上能做什么、有什么资源」，不解决「我到底做哪个版本」。从调研直跳 PRD = 在所有可能性里抓阄，得先用脑暴把方向收敛到一个明确 MVP。回看 product-fullstack#C1 的链路衔接。

2. **如果你以为「把技术栈写进 PRD 才够具体」** → 说明你没掌握 What/How 边界。工程师能换一种实现仍满足的才留 PRD，指定了框架 / 库就该搬去 TRD，否则污染边界、绑死实现。回看 product-fullstack#C1 的 PRD 编写。

3. **如果你以为「架构选型让一个 Agent 给个推荐就行」** → 说明你没理解对抗选型为什么要 5 角色法庭 + 致命缺陷自述。单个视角有代言人偏见，对抗发生在 paper 层、每份 paper 强制自曝 3 个缺陷，才逼出真实取舍。回看 product-fullstack#C1 的对抗选型。

4. **如果你以为「6 大功能可以把整份 PRD 一次喂给 /specify」** → 说明你会把上下文撑爆。要按 Must-have 逐条拆、各跑一次，`/specify` 自己会从 `specs/prd.md` 取材料。回看 product-fullstack#C2 的 Spec-Kit 拆条。

5. **如果你以为「加了 UI 设计要把 /specify 重跑一遍」** → 说明你没抓住业务没变就不重跑的原则。正确做法是设计注入：更新宪法 + 只选择性重跑触及 UI 的那几个 feature 的 plan/tasks，不推倒重来。回看 product-fullstack#C2 的设计注入。

6. **如果你以为「每处改动都写一条全链路测试最保险」** → 说明你没理解「缺陷应在最便宜的那层被抓住」。全链路是补网、不是首次发现 bug 的地方；bug 首现在全链路 = 下层漏测，要回补下层。回看 product-fullstack#C3 的三层节奏与路由。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜把这套「调研→PRD→拆条」搬到一个「个人记账 App」**
你不做投研了，改做一个给自己用的记账 App。请只描述你要保留和调整的地方，不用真跑。

- 【怎么算过关】能说清：①立项调研仍是 4 路并行 + 1 收敛，但「主题 2 数据来源」对纯本地记账可裁剪或替换（骨架固定、填什么由你的答案决定）；②仍走脑暴收敛到明确 MVP 再写 PRD，PRD 仍守 What/How 边界、必写 Non-Goal 与验收（Given/When/Then）；③把 Must-have 功能逐条 `/specify → clarify → plan → tasks`，不整份灌；④`CLAUDE.md` 仍 ≤200 行只当路由图。能讲清「为什么方法一模一样、只有填充内容变了」，就说明你抓住的是链路纪律不是投研业务。
<!--src:A; ref:pack:ai-product-dev-pack 全链路方法（调研4主题可裁剪 + PRD边界 + 逐条拆条）-->

**迁移挑战 2｜给一个「优惠券额度扣减」功能定测试层与风险级**
某电商功能：用户下单时扣减优惠券额度，多人可能同时用同一批共享券。请说清你会给哪些行为定什么风险级、路由到哪层测试。

- 【怎么算过关】能指出：①「扣减额度 / 不能扣成负数 / 不能超发」是 **P0**（资金额度错算、失败不可逆），必测、进发布门；「返回的提示文案」是 P2。②共享券 + 并发扣减命中「并发 / 竞态 / 限频」维度，要写并发测试（就像 AlphaProject 的 30/5 caps race：read→check→write 无锁会超发，修法是加锁串行化）。③涉 DB 写与约束就标真库集成，路由到后端层 + 必要时切片层。能说清「为什么这里必须测并发」= 共享资源上的 read-then-write 两段操作不加锁，天然会超卖。
<!--src:A; ref:pack:ai-product-dev-pack 风险分级 P0 + 并发命中条件; code:AlphaProject/backend/tests/test_gap_concurrency_caps.py（caps race 同型）-->

---

## 四、排错挑战（给你一段带病的代码，修好它）

AlphaProject 的自选股服务要守两条上限：总数 ≤ 30 只、持仓标记 ≤ 5 只。一位同学写完后反馈：**「单人点点没问题，可我压测脚本一并发，自选股居然加到了 31 只，上限形同虚设。」** 请找出问题并修好。

```python
# app/services/watchlist_service.py（有问题的版本）
class WatchlistService:
    def __init__(self, repo, max_total=30, max_holding=5):
        self.repo = repo
        self.max_total = max_total
        self.max_holding = max_holding

    def add(self, *, code: str, name: str) -> None:
        items = self.repo.list_active()          # ① 先读
        if len(items) >= self.max_total:          # ② 再判
            raise WatchlistError("自选股上限 30")
        self.repo.insert(code=code, name=name)    # ③ 后写
```

- 【症状】单线程顺序调用一切正常；一旦两个（或多个）线程并发调 `add`，最终自选股数可能达到 31 只，突破 30 上限。持仓 5 只上限同理会被击穿。
- 【根因】`add` 是 **read → check → write 三段、中间无锁**。已有 29 只时，两个线程几乎同时执行 ② 都读到 `len == 29`、都通过 `< 30` 判断，然后各自 ③ INSERT，结果落库 31 只。这是典型的 read-then-write 竞态，属结构性缺口 G-01，风险级 P0（数据完整性，阻断发布）。
- 【修法】把「读→判→写」这段临界区用锁串行化：

  ```python
  import threading

  class WatchlistService:
      def __init__(self, repo, max_total=30, max_holding=5):
          self.repo = repo
          self.max_total = max_total
          self.max_holding = max_holding
          self._lock = threading.Lock()          # ← 一把锁护住临界区

      def add(self, *, code: str, name: str) -> None:
          with self._lock:                        # ← 读+判+写整体串行
              items = self.repo.list_active()
              if len(items) >= self.max_total:
                  raise WatchlistError("自选股上限 30")
              self.repo.insert(code=code, name=name)
  ```

  修完必须补一条并发回归测试固化：用 `threading.Barrier(2)` 让两个线程在第 30 个槽位上抢跑，断言最终 `len(list_active()) == 30`、恰好一个线程拿到 `WatchlistError`（对应 `tests/test_gap_concurrency_caps.py`）。测回归时禁用 `sleep` 制造时序，用 barrier / fake clock 这类确定性手段。
- 【一句话总结】上限校验只要是「先查再写」两段，并发下就会被击穿；护住临界区靠**加锁串行化 + 一条能复现 race 的回归测试**，二者缺一，同一个超卖 bug 迟早回来。
<!--src:A; ref:code:AlphaProject/backend/tests/test_gap_concurrency_caps.py + tests/GAPS.md（G-01 caps race / P0 / service层加锁）; app/services/watchlist_service.py-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。
> 说明：以下题目按 2026 年 AI 产品 / AI 全栈交付方向校准，考点集中在「从调研到测试自己主导一个 MVP」的方法论与真实取舍。

**面试 1｜调研做完了，为什么不能直接写 PRD，中间非要插一步脑暴？**
- 考察点：对「不跳步」链路纪律的理解。
- 答题要点：调研只回答「市面上能做什么、有什么资源、有哪些开源可复用」，不回答「我到底做哪个版本」。从调研直跳 PRD 等于在所有可能性里抓阄，PRD 会写得发散。中间的脑暴用苏格拉底式追问把方向收敛到一个明确 MVP，PRD 才是一份沉淀已收敛结论的决议书，而不是探讨稿。
- 追问：「那对抗选型选出的架构基线，脑暴时能不能推翻重议？」→ 不能。`06-架构基线决策.md` 是 5 角色法庭辩论后终审的，脑暴不重开此案；但 06 里的 Open Questions 必须全部进脑暴议程逐个回答。
<!--src:A; ref:pack:ai-product-dev-pack Vibe Coding 链路（调研→脑暴→PRD 不跳步）-->

**面试 2｜PRD 的 What/How 边界怎么划？技术栈到底能不能写进 PRD？**
- 考察点：PRD 与 TRD 的分工。
- 答题要点：判断法一句话：工程师能换一种实现仍满足该条，就留 PRD（What）；不能换实现，那是技术决定，搬去 TRD（How）。「首屏 ≤ 2 秒」「数据缺失时展示占位并提示重试」进 PRD；「用 React + Vite」「用 Redis 缓存热点 key」搬 TRD。技术栈不写进 PRD 功能描述，非功能需求也要用数字不用形容词。
- 追问：「那 PRD 里完全不提架构基线吗？」→ 提，但只在依赖 / 约束里引用已锁定的 `06-架构基线决策.md`，不重开选型、不写具体框架类名；技术细节留 `plan.md`。
<!--src:A; ref:pack:ai-product-dev-pack PRD 编写（What/How 一刀切边界）; code:AlphaProject/specs/prd.md §12 约束-->

**面试 3｜法庭式对抗架构选型是哪 5 个角色？为什么对抗发生在 paper 层而不是 teammate 层？**
- 考察点：对抗选型协议与反偏见设计。
- 答题要点：固定 5 个角色，即 3 个代言人 + 1 个红队（魔鬼代言人）+ 1 个集成评估师，主 Claude 当 Lead 法官。三阶段：Phase 1 各自独立写 position paper（禁互通），Phase 2 papers 打乱进 mailbox、红队对每份发 3 条质疑、代言人逐条回应 ≤200 字，Phase 3 Lead 综合终审写 `06`。对抗发生在 paper 层级：总质疑数 = 3N 随候选数线性扩展，而 teammate 数固定 5，不会因候选多就爆角色。
- 追问：「防代言人偏见最核心的一条硬约束是什么？」→ 每份 paper 必含「致命缺陷自述」，强制自曝本方案最大的 3 个缺陷；另配随机打乱顺序、回应限 200 字、Lead 辩论期全程禁表态。
<!--src:A; ref:pack:ai-product-dev-pack 架构对抗选型（5角色/3阶段/反偏见5条）-->

**面试 4｜一份生产级 PRD 至少要有哪些「必备章节」，Non-Goal 为什么不能省？**
- 考察点：PRD 结构完整性与反模式。
- 答题要点：14 章里必备的包含文档信息、背景与目标（Why）、目标用户与画像（含反画像）、用户故事、功能范围（含 Out-of-Scope）、详细功能描述（触发 / 输入 / 流程 / 输出 / 边界五元组）、非功能需求（用数字）、验收标准（Given/When/Then）、优先级 / MVP、度量指标（含关停线）、风险与开放问题；小项目可精简到 9 章。Non-Goal 不能省是因为没有 Out-of-Scope 范围一定会爆，它把「明确不做」写死，挡住需求蔓延。
- 追问：「MVP 的 P0 功能数有没有上限？」→ 有反模式经验值，AlphaProject 明确 MVP P0 = 6 个并注明「满足 ≤ 8 个的要求」，太多说明没收敛。
<!--src:A; ref:pack:ai-product-dev-pack PRD 14章/9章 + 反模式; code:AlphaProject/specs/prd.md §10（P0=6）-->

**面试 5｜Spec-Kit 从宪法到实现的命令链是哪几步？为什么不能整份 PRD 喂给 /specify？**
- 考察点：Spec-Kit 命令链与拆条纪律。
- 答题要点：命令链 `constitution → specify → clarify → plan → tasks → analyze → checklist → implement`。要按 Must-have 逐条拆、各跑一次：整份 PRD 一次喂会把上下文撑爆，而且 `/specify` 会自己从 `specs/prd.md` 取材料，不用手动整份灌。tasks 分 Setup / Tests / Core / Integration / Polish 几个 Phase，能并行的标 `[P]`。
- 追问：「clarify 阶段的纪律是什么？」→ 按 11 类扫描、最多问 5 个、一次只问一个，答案回填直到 `NEEDS CLARIFICATION` 标记清零，才放行到 plan。
<!--src:A; ref:pack:spec-driven-dev-pack Spec-Kit 命令链（8命令 + Must-have逐条拆 + clarify纪律）-->

**面试 6｜跑完 spec/plan/tasks 后补了一套 UI 设计，要不要重跑 /specify？**
- 考察点：设计注入 vs 推倒重来。
- 答题要点：不重跑。业务需求没变，重跑 `/specify` 等于把已做工作推倒重来、纯浪费。正确做法是设计注入：更新宪法（constitution injection）+ 只选择性重跑「触及 UI 的那几个 feature」的 plan/tasks，不动没被设计影响的 feature。
- 追问：「AlphaProject 前端最后是 fork 现成看板还是自建？」→ 自主构建（Vite + React + TS + shadcn/ui），`stock-dashboard` 降级为视觉参考（看不抄代码），导出物放 `specs/design-reference/stitch-export/`。
<!--src:A; ref:pack:ai-product-dev-pack 设计系统注入 Spec-Kit（选择性重跑/不推倒重来）; code:AlphaProject/specs/research/06-架构基线决策.md 前端自主构建修订-->

**面试 7｜项目根的 CLAUDE.md 为什么要控制在 200 行以内，不能抄整份 PRD？**
- 考察点：项目记忆引导的取舍。
- 答题要点：`CLAUDE.md` 是给 Agent 看的项目记忆路由图，遵循 Karpathy 4 原则，作用是「指路」：告诉 Agent 关键文档在哪、宪法红线是什么、约定长什么样。抄整份 PRD 会撑破 200 行，让指令遵循度断崖式下降，反而稀释了真正该被记住的路由信息。它读已有文档产出一张精简路由图，不复述内容。
- 追问：「怎么机检 CLAUDE.md 合不合格？」→ 用包里的脚本跑行数 / `@path` 引用 / 强标记校验，只看输出，不合格回修。
<!--src:A; ref:pack:ai-product-dev-pack 项目记忆引导 CLAUDE.md（≤200行/Karpathy 4原则/linter）-->

**面试 8｜四层测试体系是哪四层？给你一个改动，你怎么判它该写哪层？**
- 考察点：测试场景判类与路由。
- 答题要点：四层是后端层 / 前端层 / 切片层（局部前后端）/ 全链路层。判类靠信号：只触后端单元 → 后端层；只落前端不联调真后端 → 前端层；「打通 X 到 Y」、feature 内端到端 → 切片层；依赖图跨 feature + 该旅程验收 → 全链路层。而且条件命中才升维，涉 DB 写才标真库、多用户 / 特权才标越权、共享资源 / 限频才标并发、调外部依赖才标韧性，纯逻辑纯读可一项不命中。
- 追问：「四层里哪一层的难点不是写断言？」→ 局部前后端（切片层）的新难点是「起真栈」的环境编排，即把前端、后端、依赖真实同起，这是第一道坎；全链路层则是被测对象要先被挖掘出来（依赖图 + 契约 + 验收半自动枚举再人确认）。
<!--src:A; ref:pack:ai-product-dev-pack 测试路由矩阵 + 四层执行器结构性缺口-->

**面试 9｜三层测试节奏 L1/L2/L3 各在什么时机、测什么？bug 首次出现在 L3 说明什么？**
- 考察点：测试节奏与「缺陷在最便宜层被抓」的原则。
- 答题要点：L1 在每个 task（TDD 红绿），测单元 + 契约，缺陷在产生那刻抓最便宜；L2 在 feature 全绿后，做 feature 内集成冒烟（真库 / 真迁移跑主路径），绑 feature 边界不绑项目边界；L3 在全部完成后，做跨 feature 端到端 + 跨模块一致性，它是补网、不是首次发现 bug 的地方。bug 首现在 L3 = 下层漏测，要回补对应的下层测试，而不是就地在 L3 打补丁。
- 追问：「全链路测试里为什么禁止用 sleep 等异步？」→ sleep 靠时间赌时序、既慢又 flaky；要用 fake clock、手动触发 job、poll-retry 这类确定性手段贯穿异步 / 跨通道。
<!--src:A; ref:pack:ai-product-dev-pack 三层测试节奏 L1/L2/L3 + 全链路禁sleep-->

**面试 10｜合并 / 发布前的 go/no-go 门有哪三道机械判据？任一不满足会怎样？**
- 考察点：发布门验收标准。
- 答题要点：三门，任一不满足即 no-go：①测试全绿（至少 P0/P1，无 skip 掩盖、无 flaky 放行）；②无孤儿需求（每条验收标准 AC 有稳定 ID、被测试引用，可追溯校验通过，有 AC 无测试叫漏测、测试引用不存在的 AC 叫幽灵）；③无破坏性契约变更（变更须兼容或已被消费方确认）。这是机械判据不靠人情，且强制落到 CI / git hook / pre-commit，蓝图只给标准。
- 追问：「AC 的 ID 发布后能重排或复用吗？」→ 不能。像 `AC-007.3` 这类 ID 发布后不重排不复用，否则可追溯链会错位，孤儿 / 幽灵校验就失效了。
<!--src:A; ref:pack:ai-product-dev-pack 发布门 go/no-go（三门机械判据 + 需求↔测试可追溯）-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请它只做考官、不做辅导。

```
请你扮演一名技术面试官，围绕「AlphaProject 三部曲：从调研到测试自己主导一个 MVP」这一阶段考我。规则如下：

【角色纪律】请只做考官、不做辅导。在我明确说「结束，请点评」之前，你不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，尽量贴近真实岗位：
1. 立项调研：4 路并行 + 1 收敛的 5 份产出、双路调研（WebSearch 路 A + 国内 web_search 路 B）为什么写死。
2. PRD：What/How 边界、Non-Goal 为什么必写、14 章 / 9 章必备项、MVP P0 数量反模式。
3. 对抗选型：5 角色（3 代言人 + 红队 + 集成评估师）、对抗在 paper 层、致命缺陷自述等反偏见约束、架构基线 A/B 双轨。
4. Spec-Kit：命令链 constitution→specify→clarify→plan→tasks→analyze→checklist→implement、为什么按 Must-have 逐条拆、clarify 11 类最多 5 问。
5. 设计注入与 CLAUDE.md：加设计不重跑 /specify、CLAUDE.md ≤200 行不抄 PRD。
6. 四层测试与三层节奏：判类路由、L1/L2/L3 时机、bug 首现在 L3 = 下层漏测。
7. 风险分级与发布门：P0-P3、权限行为默认 P0、go/no-go 三门、AC 可追溯（无孤儿 / 无幽灵）。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 如果我答「不会」，记录后直接下一题，不要讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现（答对 / 部分 / 未答）、暴露的薄弱点、以及建议我回看 Stage C 的哪一站（C1 调研与 PRD / C2 架构选型与 Spec-Kit / C3 全栈测试）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件或 Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 用 4 路并行 + 1 收敛做立项调研，产出 5-6 份调研笔记 + 一份并行调研提示词 | ☐ | ☐ | ☐ |
| 写一份守住 What/How 边界、含 Non-Goal 与 Given/When/Then 验收的生产级 PRD | ☐ | ☐ | ☐ |
| 用 5 角色法庭式对抗选出架构基线，说清 A/B 双轨取舍 | ☐ | ☐ | ☐ |
| 把 Spec-Kit 装进项目、按 Must-have 逐条把功能拆成 spec/plan/tasks | ☐ | ☐ | ☐ |
| 做设计注入（更新宪法 + 选择性重跑），而不是推倒重跑 /specify | ☐ | ☐ | ☐ |
| 写一份 ≤200 行、只当路由图的 CLAUDE.md | ☐ | ☐ | ☐ |
| 给一处改动判类并路由到四层测试的正确一层 | ☐ | ☐ | ☐ |
| 给每条行为定 P0-P3 风险级、按 L1/L2/L3 三层节奏安排测试 | ☐ | ☐ | ☐ |
| 补一个结构性缺口：先红复现 → 修产品码到绿 → 固化回归 | ☐ | ☐ | ☐ |
| 过发布门 go/no-go（测试全绿 + 无孤儿 + 无破坏性契约），并守可追溯 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」；其中「守 What/How 边界写 PRD」「四层测试判类路由」这两项能到「能教别人」，你才真正拿下 C 阶段徽章——因为徽章过关点就是关掉代笔、亲笔主导 PRD 核心三节并标注调研依据，而这背后正是「你真懂产品交付链路每一步为什么这么走」。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的入门到进阶的活儿，描述自包含。它们是你作品集和「能把想法带到 MVP」的第一步。

1. **给一个创业想法做立项调研 + 生产级 PRD。**
   很多人有想法但写不出一份能给工程 / 投资人看的需求文档。你会用 4 路并行调研摸清竞品与开源生态，再产出一份守 What/How 边界、含 Non-Goal 与 Given/When/Then 验收的 PRD。交付物是 `specs/research/` 6 份调研 + 一份 PRD。

2. **给一个「要不要 fork 哪个开源项目」的决定做对抗选型。**
   团队常在几个候选之间拍脑袋。你能用 5 角色法庭式对抗（含致命缺陷自述）逼出真实取舍，产出一份带复用矩阵、改造人日、A/B 双轨与被否理由的架构基线决策，让选型有据可查。

3. **把一份 PRD 规范化成可交付的 spec/plan/tasks。**
   帮别人把「一堆想法」落成工程可接的功能包：按 Must-have 逐条 `/specify → clarify → plan → tasks`，装好 Spec-Kit 与项目宪法。交付物是 `specs/00X-*/` 各含三件套 + 一份 ≤200 行的 CLAUDE.md。

4. **给一个已跑通但不稳的 MVP 补测试体系。**
   很多 demo 能跑但一上真数据就崩。你会按风险分级 P0-P3 + 三层节奏，把该测的行为路由到后端 / 前端 / 切片 / 全链路四层，补上并发、真库、韧性这些结构性缺口并固化回归。交付物是可追溯的测试套件 + 一份缺口登记表。

5. **给一个项目搭「发布门」把关。**
   帮团队立一道 go/no-go 门：测试全绿 + 无孤儿需求 + 无破坏性契约变更，落到 CI / git hook。让「差不多了就上线」变成「过了三道机械判据才放行」，这本身就是一门能收费的工程手艺。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
