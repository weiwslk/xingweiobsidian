# 工程师转型 · Stage C 进阶能力 · 过关手册（练习向）

> 路径：工程师转型（engineer-transform） ｜ 阶段：C 七大进阶能力（C1–C3） ｜ 风味：练习向，闭卷自测加重
>
> 这一站给你已经会搭的通用 Agent 加内功：上下文工程（C1 必修，决定 Agent 能不能长上下文稳定工作）、记忆管理（C2 选修）、驾驭工程 Harness（C3 选修）。
> 本手册不讲新知识，只帮你确认两件事：这三块内功你到底掌握没有、能不能扛住技术面的追问。
>
> 用法：先做「随堂练习」确认动手过关；再用「误区自查 / 迁移挑战 / 排错挑战」查漏；
> 最后把「面试自测」当口试反复刷，用「模拟面试官指令」找人（或 Agent）给你上一场压力面。
> 答案全在手册里，不依赖任何外部资料。

**阶段 gate 提醒**：C1 是深潜专属徽章 gate。关掉 Agent，独立写出一份上下文工程策略文档，把选择 / 压缩 / 隔离 / 注入四类策略各用在哪讲清楚，并给同一任务优化前后的 token 数与至少一个效果指标的实测对比表。产物文件提交才算过关。C2 / C3 按你的方向选修，不强求。

**本阶段来源索引**（题目均锚课件与真实源码 / 能力包，标注见每题）：
- C1《上下文工程》基础入门 + 组合编排进阶，配套 context-engineering-pack
- C2《智能体长短期记忆管理》基础入门 + mem0 进阶实战，配套 agent-memory-pack
- C3《Harness Engineering 驾驭工程实战》第一节原理与概念、第二节手搓 Mini Harness，配套 harness-engineering-pack

---

## 一、随堂练习（每节点动手小题）

> 每题先自己做，再对参考答案。做不动就回看对应课件，不要跳过 C1 的 gate。

### C1 · 上下文工程（含策略 gate）

**练习 C1-1（gate 核心）**：不看课件，写出 Anthropic 五大上下文管理策略的名字，并各用一句话说清机制。

- 参考答案：Write（写入持久化，把信息写到上下文窗口之外的持久化存储，如 Scratchpad / 文件 / 数据库）、Select（运行时按需检索，Just-in-time 只加载当前任务相关的子集，如 RAG、Glob+Grep、动态工具加载）、Compress（上下文压缩，保留关键信息同时缩减 token，如工具结果清除、观察遮蔽、LLM 摘要）、Isolate（上下文隔离，用子 Agent 各自的窗口防止无关上下文互相干扰）、Cache（提示缓存，复用不变的前缀避免重复计算）。自查要点：这五类不是互斥选项，而是组合叠加，生产级系统通常同时用 3 到 5 种。gate 文档里你要把这五类各落到一个具体优化点上。
<!--src:A; ref:《上下文工程》基础入门 五大策略总览; look:engineer-transform#C1-->

**练习 C1-2**：你的模型宣传窗口是 1M token，能不能把 1M token 的信息直接全塞进去？在多少 token 处应该开始主动压缩，为什么？

- 参考答案：不能。课件的 Context Rot（上下文腐化）结论是，随上下文变长，模型准确召回和利用信息的能力会逐步下降，NVIDIA RULER 的系统测试显示有效窗口大约只有宣传窗口的 50% 到 65%。行业实践是即使 1M 窗口也应在 256K token 以下就启动主动压缩，不要等 API 报错，那时推理质量早就退化了。自查要点：买 1M 窗口 ≠ 可以塞 1M token 的信息，这是新手最常踩的坑，Agent 在长对话里忘记早期指令、重复生成、工具调用失败，但 API 并不报错，所以找不到原因。
<!--src:A; ref:《上下文工程》基础入门 Context Rot 与有效窗口表; look:engineer-transform#C1-->

**练习 C1-3**：三种压缩技术「工具结果清除 / 观察遮蔽 / LLM 摘要」各在什么场景优先用？

- 参考答案：工具结果清除（调用完成后把冗长原始输出用 LLM 压成一行结论摘要，成本削减 20% 到 40%、几乎零延迟、仅保留结论）是所有场景首先采用的轻量策略；观察遮蔽（保留 Action + Reasoning，把旧的 Observation 也就是 ToolMessage 内容替换成占位符，成本削减约 52%、几乎零延迟）优先用在工具密集的编程 / SE Agent；LLM 摘要（独立模型压缩全部交互，削减 50% 到 60%，但要 1 到 5 秒延迟、可能丢细节）用在需要理论上无限回合的通用 Agent。自查要点：观察遮蔽的结论不能推广到对话型 Agent，因为对话里的「观察」就是用户的话，遮蔽用户的话会灾难性丢信息；对话型 Agent 优先 trim_messages 加 LLM 摘要。
<!--src:A; ref:《上下文工程》基础入门 三种压缩技术对比表; look:engineer-transform#C1-->

### C2 · 记忆管理

**练习 C2-1**：短期记忆的三大机制是什么？教学版实现里压缩阈值和截断阈值为什么要分开设，各设多少？

- 参考答案：三大机制是存储、截断、压缩摘要。教学版用 `COMPRESS_TRIGGER = 20`、`MAX_HISTORY = 30`：累计到 20 条消息优先尝试压缩摘要（把较早对话用 LLM 压成一段摘要存入 `compressed_context`），只有压缩失败且历史继续涨到 30 条以上，才在构造 LLM 输入时裁掉最旧消息。自查要点：两个阈值若都设 20，压缩刚触发时截断也同时生效，没有降级空间；截断不是日常清理策略，而是压缩失败时保护上下文窗口的最后防线。这里计数单位是「消息条数」不是「对话轮数」，一次 user + assistant 往返是两条，20 条约等于 10 轮。
<!--src:A; ref:《记忆管理》基础入门 第三章 截断与压缩阈值; look:engineer-transform#C2-->

**练习 C2-2**：说出课件里四条记忆层实现路线，各自的代表方案。

- 参考答案：文件系统记忆路线（OpenClaw / mini-OpenClaw，记忆存成人类可读的 Markdown，`MEMORY.md` 存长期事实、会话存 JSON，默认全文注入，超阈值降级为 BM25 加向量混合检索）；多后端存储路线（LangChain 1.x + LangGraph，短期用 Checkpointer、长期用 Store，切后端即可从原型到生产）；多源召回路线（LlamaIndex，ChatMemoryBuffer + VectorMemory + SimpleComposableMemory 可组合，RAG 密集场景强）；分层自管理路线（Letta / 原 MemGPT，Core / Archival / Recall 三层加 sleep-time agent 后台整理，自主性最强但工程最重）。自查要点：四条路线没有绝对优劣，选型看场景，个人和教学项目优先文件系统路线，透明易调试。
<!--src:A; ref:《记忆管理》基础入门 第二章 四种记忆层路线; look:engineer-transform#C2-->

**练习 C2-3**：mini-OpenClaw 的会话 JSON 有哪两个核心字段？MemoryManager 的「可降级」体现在哪？

- 参考答案：核心两字段是 `messages`（最近对话原文）和 `compressed_context`（压缩摘要），写文件时 `ensure_ascii=False` 保证中文不被转义成 `\u` 序列。可降级体现在：长期记忆写入的 `_is_worth_memorizing` 判断包在 try/except 里，写入失败（网络超时 / API 限流 / JSON 解析错）就静默跳过，不中断主对话，短期 session 保存不受影响；短期压缩失败时 `_maybe_compress()` 捕获错误、保留全部原消息，再靠 `max_history` 截断兜底。自查要点：`MEMORY.md` 不存在、索引构建失败、网络超时，任何单点异常都不该让 Agent 崩，每个模块独立失败其他继续工作。
<!--src:A; ref:《记忆管理》基础入门 第六章 MemoryManager 三原则; look:engineer-transform#C2-->

### C3 · 驾驭工程 Harness

**练习 C3-1**：把 naive agent 的八大故障模式尽量列全（记住这张问题清单）。

- 参考答案：①循环失控（缺刹车，`while True` 无终止判断）②context 溢出（`messages.append` 无限积累）③cache miss（system prompt 或前缀在循环内被改动）④tool 错误吞（`except: result = ""` 或 subprocess returncode 被丢，以为成功继续做）⑤状态丢失（没有 progress 文件，进程一关全丢）⑥缺权限闸 ⑦缺自动化评审 ⑧成本失控（无 token budget）。自查要点：这张清单是驾驭工程的问题轴，八大机制每一个都对应清单里某一行。其中 ⑥ 和 ⑧ 在第一节课只做部分覆盖，主解由第二节课的 Permission Gate 权限闸和 Token Budget 成本预算交付。
<!--src:A; ref:《驾驭工程》第一节 表5-1 八大故障模式; look:engineer-transform#C3-->

**练习 C3-2**：把 Agent Loop / Tool Use / Progress Tracking / Context Management 四个机制各对应它主解的故障编号，并说清核心做法。

- 参考答案：Agent Loop 四相循环（Gather 收集 → Action 执行 → Verify 验证 → Iterate 迭代，用 `for i in range(MAX_ITER)` 取代 `while True` 加显式终止判断）解 ① 循环失控；Tool Use 工具编排（失败时返回 `{"status": "error", "error": "..."}` 结构化回传，而不是空串或吞异常）解 ④ tool 错误吞；Progress Tracking 进度追踪（进度写磁盘 + git commit 双 checkpoint）解 ⑤ 状态丢失；Context Management 上下文管理（监控 token，超阈值压缩或重置，同时保持 system prompt 前缀稳定）一次解 ② context 溢出和 ③ cache miss。自查要点：Agent 不该完全信任模型的自我终止判断，系统必须有一个强制出口，Anthropic 官方叫「structured loop with guaranteed exit」。
<!--src:A; ref:《驾驭工程》第一节 6.1-6.4 八大机制; look:engineer-transform#C3-->

**练习 C3-3**：Generator-Evaluator 三角色架构里，为什么 Evaluator 必须「不看生成过程」？

- 参考答案：这个架构借鉴 GAN 思路，把任务拆成 Planner（拆任务）、Generator（写代码）、Evaluator（独立评审）三个独立角色的 Agent，解 ⑦ 缺自动化评审。Evaluator 只看最终产出、不看 Generator 的中间过程，以「怀疑者」视角打分，不通过就打回。原因是：如果让同一个 LLM 既做生成又做评审，它会倾向为自己辩护；把评审单独抽成一个 Agent、给它怀疑者角色的 prompt，它才会认真挑刺。自查要点：这是通过架构性约束制造对抗，而不是靠单 Agent 的自我反思，类比论文评审制度，作者、审稿人、编辑三个角色分离才能保证质量。
<!--src:A; ref:《驾驭工程》第一节 6.8 Generator-Evaluator; look:engineer-transform#C3-->

---

## 二、误区自查

> 逐条对照。命中任一条，说明对应知识点还没掌握，按提示回看。

1. **如果你以为「模型窗口够大就不用管上下文，全塞进去就行」** → 你还没理解 Context Rot。有效窗口只有宣传窗口的 50% 到 65%，塞满会让模型准确召回能力退化，且 API 不报错，问题最难查。回看 C1《上下文工程》基础入门 Context Rot 一节。
<!--src:A; ref:《上下文工程》基础入门 Context Rot; look:engineer-transform#C1-->
2. **如果你以为「上下文工程取代了提示词工程和记忆管理」** → 你还没理解三代工程是同心圆式的包含关系，不是替代关系。写 prompt 时不会突然不需要上下文，管上下文时也不会突然不需要 prompt，每一代把前一代吸纳进来再加一层。回看 C3《驾驭工程》第一节 3.1。
<!--src:A; ref:《驾驭工程》第一节 3.1 三代工程包含关系; look:engineer-transform#C3-->
3. **如果你以为「Cache 就是把请求结果缓存起来」** → 你混淆了前缀缓存和响应缓存。Prompt Caching 是服务端缓存不变前缀的 KV 计算，仍会调 API、仍生成新回复，省的是前缀的重复推理成本；响应缓存才是完全相同输入直接返回旧结果。回看 C1《上下文工程》基础入门 3.1.5。
<!--src:A; ref:《上下文工程》基础入门 3.1.5 Cache 策略; look:engineer-transform#C1-->
4. **如果你以为「上下文窗口够大就等于 Agent 有记忆」** → 你还没分清上下文窗口和记忆系统。上下文随进程结束消失、每轮携带完整历史成本线性增长、超上限会报错，跨会话持久记忆必须写入存储介质。回看 C2《记忆管理》基础入门第一章。
<!--src:A; ref:《记忆管理》基础入门 第一章 上下文窗口≠记忆系统; look:engineer-transform#C2-->
5. **如果你以为「工具描述不占多少上下文」** → 你低估了工具上下文这个水龙头。Anthropic 实测，一个装了 100 个工具的 Agent，30% 到 40% 的窗口被工具描述占据，还没开始对话有效窗口就缩水近一半。回看 C1《上下文工程》基础入门 2.5 工具上下文管理层。
<!--src:A; ref:《上下文工程》基础入门 2.5 工具上下文; look:engineer-transform#C1-->
6. **如果你以为「Agent 说自己做完了就是做完了」** → 你还没建立验证闭环意识。驾驭工程要求用 Playwright / Puppeteer / pytest 真机验证，通过才标记完成，且不可为了让流程过就删改测试。回看 C3《驾驭工程》第一节 6.6 验证闭环。
<!--src:A; ref:《驾驭工程》第一节 6.6 Verification Loop; look:engineer-transform#C3-->

---

## 三、迁移挑战

> 换个场景，用同一套方法。能迁移才算真掌握。

**迁移 1（把五策略用到你自己的 Agent）**：你在 B2 做过一个通用 Agent，跑长了就变慢变贵。请按 Cache → Isolate → Compress 的落地优先级，给它排出前三步优化动作，并说明为什么 Cache 排第一。
- 参考答案：第一步 Cache，把 system prompt 和工具描述放在消息列表最前面、保持逐字节稳定，让 DeepSeek / OpenAI 的前缀缓存自动命中，缓存命中价仅标准价的 10%，也就是这部分输入成本直降约 90%，且零代码改动；这就是它排第一的理由，ROI 最高、零架构变更。第二步按需引入 Compress（先上工具结果清除这种零延迟策略），第三步对可并行的子任务上 Isolate 子 Agent。核心逻辑是零成本策略先用、复杂策略按需叠加，别一上来就堆满栈。自查：造新 Agent 前应先按 Agent 类型查决策矩阵定最小组合。
<!--src:A; ref:《上下文工程》基础入门 3.3 决策优先级 + pack:context-engineering-pack; look:engineer-transform#C1-->

**迁移 2（把验证闭环从编程 Agent 迁到别的场景）**：你在编程 Agent 上用 pytest 做过验证闭环。现在要给一个「自动填表并提交网页表单」的 Agent 加验证，你的验证工具、验证信号、失败处理分别怎么设计？
- 参考答案：验证工具从 pytest 换成 Playwright / Puppeteer 这类真机浏览器工具，实际驱动页面提交并读回结果；验证信号从「测试全绿」换成「提交后页面出现成功态 / 目标记录真的写进去了」这种可观测事实，而不是听 Agent 自己说「我觉得填好了」；失败处理沿用同一套，验证不过就回退重做、不标记完成，通过才置为 passes=true。方法链不变（每个子任务做完必过真机验证），换的只是验证工具和信号口径。自查：不可为了让流程通过就放宽或删掉验证。
<!--src:A; ref:《驾驭工程》第一节 6.6 验证闭环 + pack:harness-engineering-pack; look:engineer-transform#C3-->

---

## 四、排错挑战

> 下面是一段真实感的 naive agent 主循环（很多人第一版就这么写），跑起来又贵又不稳还会假成功。
> 请先自己找出所有毛病，再看答案。

```python
# run_agent.py —— 目标：让 Agent 读测试文件、修 bug、跑 pytest 确认通过
SYSTEM = "你是编程助手。"

def run_agent(task):
    messages = []
    while True:                                   # (a)
        sys_prompt = SYSTEM + f"\n当前时间：{now()}\n任务：{task}"  # (b)
        resp = call_llm(system=sys_prompt, messages=messages)
        messages.append(resp)                     # (c) 只增不减
        for call in resp.tool_calls:
            try:
                out = run_tool(call)              # run_pytest 只返回 stdout
            except Exception:
                out = ""                          # (d)
            messages.append({"role": "tool", "content": out})
        if resp.says_done:                        # 全凭模型自己说做完了
            return
    # 进程被 kill 后，下次从零开始
```

**参考答案**（每条按【症状】→【根因】→【修法】三段拆解）：

1. **【症状】**(a) 处 Agent 有时任务没真做完就一直空转，烧光 API 额度。**【根因】**故障①循环失控：`while True` 没有硬性出口，完全信任模型的 `says_done` 自我判断。**【修法】**换成 `for i in range(MAX_ITER)`，并在每轮加显式 verify 判断能不能停，做成 Gather → Action → Verify → Iterate 四相循环，保证有出口。
2. **【症状】**(b) 处每轮请求延迟和费用都翻倍，`cached_tokens` 始终是 0。**【根因】**故障③ cache miss：把 `now()` 时间戳拼进了 system prompt，前缀每轮都在变，prompt cache 永远命不中。**【修法】**把变动内容（时间、任务态）移出 system prompt，保持前缀逐字节稳定，动态内容放消息后半段；靠 API 返回的 `cached_tokens` 确认命中。
3. **【症状】**(c) 处对话跑几十轮后直接超 context window 报错，或 token 费用失控。**【根因】**故障② context 溢出：`messages` 只 append 不清理，无限积累。**【修法】**加 Context Management，监控 token，超阈值触发压缩（compaction）或重置，把大任务用 Feature List 拆成单次只做一件事，从源头控制膨胀。
4. **【症状】**(d) 处 pytest 明明失败，Agent 却以为成功、继续在错代码上改，最后交一份「假通过」。**【根因】**故障④ tool 错误吞：`except: out = ""` 把异常吞成空串，且 `run_pytest` 只返回 stdout、丢掉了 returncode。**【修法】**工具失败返回 `{"status": "error", "error": str(e)}` 结构化回传，pytest 要读 returncode 判定真实成败；再叠一层验证闭环，用真机 pytest 结果决定是否标记完成。
5. **【症状】**进程被 kill 后第二天完全从零重来。**【根因】**故障⑤ 状态丢失：没有任何 progress 文件。**【修法】**加 Progress Tracking，进度写磁盘 + git commit 双 checkpoint，支持断点续传。

修正后的骨架关键点：`for i in range(MAX_ITER)` 取代 `while True`、system prompt 前缀稳定不含时间戳、每轮监控并压缩 context、工具错误结构化回传且读 returncode、progress 文件持久化。（综合《驾驭工程》第一节 表5-1 八大故障模式与 6.1–6.6 机制）
<!--src:A; ref:《驾驭工程》第一节 5.1 naive agent + 八大故障模式; look:engineer-transform#C3-->

---

## 五、面试自测（10 题 · 每题一层追问）

> 每题给「考察点 / 答题要点 / 追问」。追问是面试官会往深挖的第一层，你要能接得住才算真掌握。
> 刷题时先捂住答案自己讲一遍。

**Q1 · 提示词工程、上下文工程、驾驭工程是什么关系？**
- 考察点：能不能识破「谁取代谁」的标题党，理解包含关系。
- 答题要点：三代工程是同心圆式的包含关系，不是替代。提示词工程（2022–2023）管单轮对话里怎么把问题问好；上下文工程（2024–2025）管多轮里喂给 LLM 的信息如何精准，含 RAG、tool schema、对话历史管理、缓存；驾驭工程（2025–2026）管多轮工具调用加长时间运行时整个系统怎么不失败，含循环刹车、状态持久化、验证闭环、子代理分治。同一个 Claude Code 会话里三代机制同时在跑，没有哪一层被替代。（C3《驾驭工程》第一节 3.1）
- 追问：那 Context Engineering 和 Memory Management 又是什么关系？→ 记忆管理是「供给侧」负责存取（对应 mem0.add / search），上下文工程是「消费侧」负责在有限窗口里策展放什么，上下文工程包含记忆但远不止记忆，窗口里的系统指令、工具定义、RAG 文档等都要它管。
<!--src:A; ref:《驾驭工程》第一节 3.1 + 《上下文工程》基础入门 1.1 供需模型; look:engineer-transform#C1-->

**Q2 · 什么是 Context Rot？对工程有什么直接含义？**
- 考察点：长上下文的真实容量认知，不是背窗口数字。
- 答题要点：Context Rot（上下文腐化）指随上下文变长，模型准确召回和利用信息的能力逐步下降，是所有大语言模型的普遍现象。NVIDIA RULER 的结论是有效窗口大约只有宣传窗口的 50% 到 65%。工程含义：买 1M 窗口不等于能塞 1M token，行业实践是即使 1M 窗口也应在 256K 以下就启动主动压缩，别等 API 报错。（C1《上下文工程》基础入门 1.2）
- 追问：Agent 在长对话里表现变差，你怎么定位是不是 Context Rot？→ 先检查当前上下文 token 数是否已超过有效窗口（量的问题），排除后再逐一排查中毒 / 干扰 / 混淆 / 冲突四种质量问题；Context Rot 的坑在于 API 不报错，只能靠主动测量 token 定位。
<!--src:A; ref:《上下文工程》基础入门 1.2 Context Rot; look:engineer-transform#C1-->

**Q3 · Anthropic 五大上下文策略是哪五个？为什么 Cache 排在落地优先级第一位？**
- 考察点：策略框架 + 工程优先级判断。
- 答题要点：Write（写入持久化）、Select（运行时按需检索）、Compress（压缩）、Isolate（隔离）、Cache（提示缓存）。落地优先级 Cache 第一，因为它零代码改动、零架构变更，只要保持请求前缀稳定就能省 85% 以上的静态前缀成本，缓存命中价仅标准价的 10%，ROI 最高，应在上线首日启用。之后按需叠 Compress、Isolate 等更重的策略。（C1《上下文工程》基础入门 3.1、3.3）
- 追问：Cache 的最小缓存单元是多少，前缀被破坏会怎样？→ 触发门槛因厂商而异：OpenAI 自动前缀缓存对 ≥1024 tokens 的前缀才生效；DeepSeek 服务端前缀缓存以 64 token 为最小分块单元、默认开启、无固定 token 下限；Anthropic 需用 cache_control 显式标记，不同模型最小可缓存前缀在 1024 到 4096 tokens 之间（老模型 1024、Sonnet 2048、Opus/Haiku 4096），最多 4 个断点。system prompt 里若混入时间戳、随机 ID 这类变量，前缀每轮都变，缓存全部失效，延迟和费用双双翻倍。
<!--src:A; ref:《上下文工程》基础入门 3.1/3.3 五策略与优先级; look:engineer-transform#C1-->

**Q4 · Prompt Caching 是怎么工作的？为什么系统提示要放最前面？**
- 考察点：缓存机制的底层理解，不是「缓存结果」这种误解。
- 答题要点：Agent 每轮调用 LLM 时请求前缀（系统提示 + 工具描述）几乎完全相同，Prompt Caching 缓存这些不变前缀的 KV 计算，后续请求前缀匹配就直接复用、跳过重复计算，仍会调 API、仍生成新回复。缓存匹配从请求头部开始，所以系统提示和工具描述必须在消息列表最前面、动态内容放后面，顺序颠倒缓存直接失效；缓存字段覆盖 system、messages、tools。（C1《上下文工程》基础入门 3.1.5）
- 追问：你观察到多轮循环里 `cache_hit_tokens` 一直是 0，怎么排查？→ 说明前缀被改动了。查是不是有 middleware 改了消息中段、或 system prompt 里混了每轮变化的内容（时间戳 / task_id）；把变动项移出前缀、保持逐字节稳定即可命中。
<!--src:A; ref:《上下文工程》基础入门 3.1.5 + pack:context-engineering-pack cache 红线; look:engineer-transform#C1-->

**Q5 · 工具结果清除、观察遮蔽、LLM 摘要三种压缩怎么选？**
- 考察点：压缩策略族的选型直觉，能给量级。
- 答题要点：工具结果清除（调用完把冗长输出压成一行结论，削减 20% 到 40%、几乎零延迟、仅保留结论）是所有场景首先采用；观察遮蔽（保留 Action 加 Reasoning、把旧 Observation 也就是 ToolMessage 内容换成占位符，削减约 52%、几乎零延迟）优先用在工具密集的 SE / 编程 Agent；LLM 摘要（独立模型压缩全部交互，削减 50% 到 60%、要 1 到 5 秒延迟、可能丢细节）用在需要理论上无限回合的通用 Agent。（C1《上下文工程》基础入门 3.1.3）
- 追问：观察遮蔽这么省，为什么不能无脑用在客服 Agent？→ 客服 / 教学这类对话型 Agent 里，「观察」就是用户说的话，遮蔽用户的话会造成灾难性信息丢失；对话型应优先 trim_messages 加 LLM 摘要，观察遮蔽只适合观察结果高度结构化的工具密集场景。
<!--src:A; ref:《上下文工程》基础入门 3.1.3 三种压缩对比; look:engineer-transform#C1-->

**Q6 · 上下文变「脏」有哪四种失败模式？和 Context Rot 有什么区别？**
- 考察点：上下文质量诊断框架，排查 Agent 异常的核心尺子。
- 答题要点：四种失败模式（Drew Breunig 总结，被 LangChain 引用）是 Context Poisoning 上下文中毒（幻觉 / 错误信息进入上下文污染后续推理，症状是基于错误前提做看似合理的推理，修法加验证步骤 / 交叉检查）、Context Distraction 干扰（信息量过大淹没模型固有能力，简单问题也答错，修法压缩裁剪）、Context Confusion 混淆（无关上下文影响输出，修法相关性过滤）、Context Clash 冲突（上下文信息互相矛盾，症状自相矛盾 / 反复摇摆，修法矛盾检测 / 时间感知更新 / 去重）。区别：Rot 是「量」的问题（token 太多注意力衰减），这四种是「质」的问题（信息本身有毒 / 有噪 / 有冲突）。（C1《上下文工程》基础入门 3.4）
- 追问：这四种里你认为最危险的是哪个，为什么？→ Context Poisoning，因为症状看起来像「模型在正常推理」，只是前提错了，最难发现；常见来源是前序工具返回幻觉数据、摘要时引入不存在的细节、子 Agent 错误结论被主 Agent 当事实。
<!--src:A; ref:《上下文工程》基础入门 3.4 四种失败模式; look:engineer-transform#C1-->

**Q7 · 短期记忆和长期记忆有什么本质区别？长期记忆为什么可以用文件而不是数据库？**
- 考察点：记忆系统的分层认知，学知识 vs 记事实的通道划分。
- 答题要点：短期记忆生命周期是单次会话内、存 messages 列表 / 会话 JSON、直接传入上下文、token 随轮次线性增长要截断压缩，装的是当前对话流水；长期记忆跨会话永久、存 MEMORY.md / 向量库、全文注入或 RAG 检索、成本相对固定，装的是用户画像和稳定事实。文件系统路线（mini-OpenClaw）之所以成立，是因为它有三个优势：透明可读（Markdown 随时打开看 Agent 记住了什么）、热更新（运行期直接改文件下轮生效）、人机共同维护（人可随时纠错）。局限是文件涨到几万字超上下文容量时全文注入失效，要降级为 RAG 语义检索。（C2《记忆管理》基础入门 2.3、2.4）
- 追问：那什么时候必须从文件切到向量库 / RAG？→ 当 MEMORY.md 体积超过上下文容量、全文注入装不下时，就要引入 RAG 做语义检索，OpenClaw 的做法是超阈值自动降级为 BM25 加向量混合检索。
<!--src:A; ref:《记忆管理》基础入门 2.3/2.4 短期vs长期与文件路线; look:engineer-transform#C2-->

**Q8 · 课件讲了四条记忆层实现路线，各自代表方案和适用场景？**
- 考察点：记忆方案选型的全景视野。
- 答题要点：文件系统路线（OpenClaw / mini-OpenClaw，Markdown + JSON，透明易调试，适合个人 Agent 和教学）；多后端存储路线（LangChain + LangGraph，短期 Checkpointer 长期 Store，切后端从原型到生产，适合要快速落地且有成熟生态）；多源召回路线（LlamaIndex，ChatMemoryBuffer + VectorMemory + SimpleComposableMemory 可组合，适合 RAG 密集和企业多源）；分层自管理路线（Letta / MemGPT，Core / Archival / Recall 三层加 sleep-time agent 后台整理，自主性最强、工程最重，多用于研究型长自主对话）。（C2《记忆管理》基础入门 2.2）
- 追问：长期记忆存储又分几类，怎么按检索需求选？→ 课件第四章分向量数据库、KV 存储、图数据库、关系型数据库四类；语义相似检索用向量库，精确键值查用 KV，关系推理用图库，结构化条件查用关系型，按检索需求选合适存储。
<!--src:A; ref:《记忆管理》基础入门 2.2 四路线 + 第四章 四种存储类型; look:engineer-transform#C2-->

**Q9 · 为什么说 Agent = Model + Harness？有没有硬数据支撑？**
- 考察点：驾驭工程的核心命题，能不能拿出实证反击「换皮论」。
- 答题要点：同样是前沿旗舰模型，裸 API 调用和 Claude Code 的产出差一个数量级，差距不在模型权重里而在「调用模型的那套系统」也就是 harness。硬数据：LangChain 2026-02-17 用同一个 gpt-5.2-codex 只改 harness、不动模型，Terminal Bench 2.0 从 52.8% 提到 66.5%，提升 13.7 个百分点；OpenAI 官方博文称用 harness 五个月约百万行代码、约 1500 个 PR、全程零人工编码。这证明 harness 是可 benchmark 量化、能承担生产级强度的工程变量。（C3《驾驭工程》第一节 1.2、2.2）
- 追问：那 naive agent 具体会以哪些方式失效？→ 八大故障模式：循环失控、context 溢出、cache miss、tool 错误吞、状态丢失、缺权限闸、缺自动化评审、成本失控，每一条都对应一个 harness 机制来解。
<!--src:A; ref:《驾驭工程》第一节 1.2/2.2 公式与硬数据; look:engineer-transform#C3-->

**Q10 · 驾驭工程八大机制怎么和故障对应？三支柱视角又是什么？**
- 考察点：机制清单 + 工程重心两套坐标系。
- 答题要点：机制对故障：Agent Loop 四相循环解①循环失控、Tool Use 结构化错误解④错误吞、Progress Tracking 解⑤状态丢失、Context Management 解②③溢出与 cache miss、Feature List 解②的源头、Verification Loop 解④的下游、Subagents 解②的另一路、Generator-Evaluator 解⑦缺评审。三支柱是从工程维度切的第二套坐标：Context Engineering（Agent 能看见什么）、Architectural Constraints（能做什么，设红线拉电闸）、Garbage Collection（能跑多久，容错与动态收敛）。机制视角用来盘点做了什么，支柱视角用来判断工程投资重心在哪。（C3《驾驭工程》第一节 6.9、7.1）
- 追问：为什么第一节课的 Garbage Collection 这一柱基本是空的？→ 不是漏讲，GC 支柱主要由 Token Budget / Cost Cap 这类成本控制机制承担，它们连同 Permission Gate 是第二节课的核心，也正是八大故障里 ⑥⑧ 留到下节主解的原因。
<!--src:A; ref:《驾驭工程》第一节 6.9/7.1 机制矩阵与三支柱; look:engineer-transform#C3-->

---

## 六、模拟面试官指令（压力面模式）

> 把下面整段复制给一个 Agent（或找同学扮演），让它给你上一场 Agent 进阶能力的压力面。
> 这是「考官」不是「教学助手」，它只考不教，追问到你答不出为止，最后逐题评价。

```
你现在是一位资深 AI Agent 工程师面试官，正在面试我，方向是 Agent 上下文工程、记忆管理与驾驭工程。请严格进入「压力面」模式，全程遵守：

1. 角色纪律：你只做考官，不做辅导。面试进行中绝不解释知识点、绝不给正确答案、绝不提示，
   只提问、只追问、只记录。我答错或答不全，你不纠正，继续往下挖或换题。
2. 出题范围（从这几类里出，覆盖到位）：上下文工程五策略与落地优先级、Context Rot 与有效窗口、
   Prompt Caching 机制与前缀稳定、三种压缩技术选型、四种上下文失败模式、短期与长期记忆区别、
   四条记忆层路线与选型、Agent=Model+Harness 与八大故障模式、八大机制对应故障、三支柱视角。
3. 压力面规则：每道主问题至少追问一到两层，往原理深处挖（例如问完五策略，追问 Cache 为何排第一、
   再追问 cache_hit_tokens=0 怎么排查）。我答得上就继续加深，直到我答不出或明显卡壳，再转下一题。
   语气专业、简洁、不寒暄、不鼓励。一次只发一个问题，等我回答后再发下一个。
4. 时长：共出 8 到 10 道主问题（含追问）。中途不要总结、不要给答案。
5. 收尾（全部问完后一次性输出）：
   - 逐题打分（0–5 分）并指出我每题在第几层追问卡住、暴露了哪个知识盲区；
   - 给一份按薄弱点排序的复习清单，只指出该回看哪个主题（如「观察遮蔽适用边界」「Prompt Cache 前缀稳定」），
     不展开讲解知识点；
   - 给一个总体判断：以当前水平能不能过 Agent 进阶能力技术面，还差哪几块。

现在开始，直接发第一个问题，不要复述这些规则。
```

---

## 七、能力自评表

> 每个技能点，诚实勾选你在哪一档。C1 是必修，核心项要能「教别人」（讲清原理与取舍）。

| 技能点 | 能跟做（照着课件 / Agent 走） | 能独立做（关掉 Agent 自己完成） | 能教别人（讲清原理与取舍） |
|---|---|---|---|
| 五策略框架（Write/Select/Compress/Isolate/Cache）落地 | ☐ | ☐ | ☐ |
| Context Rot 与有效窗口判断（何时启动压缩） | ☐ | ☐ | ☐ |
| Prompt Caching 机制与前缀稳定（cache_hit 排查） | ☐ | ☐ | ☐ |
| 三种压缩技术选型（工具结果清除 / 观察遮蔽 / 摘要） | ☐ | ☐ | ☐ |
| 四种上下文失败模式诊断（中毒 / 干扰 / 混淆 / 冲突） | ☐ | ☐ | ☐ |
| 上下文工程策略文档 + 优化前后 token 实测（C1 gate） | ☐ | ☐ | ☐ |
| 短期记忆三机制（存储 / 截断 / 压缩摘要） | ☐ | ☐ | ☐ |
| 记忆方案选型（四路线 + 四种长期存储） | ☐ | ☐ | ☐ |
| 给 Agent 接可读写的长短期记忆并验证跨会话 | ☐ | ☐ | ☐ |
| 八大故障模式与八大机制配对 | ☐ | ☐ | ☐ |
| 给 Agent 加一层 harness（约束 / 校验 / 重试） | ☐ | ☐ | ☐ |
| 验证闭环设计（真机验证，拒绝自我声称） | ☐ | ☐ | ☐ |

自评红线：只要「五策略落地」「Context Rot 判断」「上下文工程策略文档」这三项还停在「能跟做」，就说明 C1 必修没过，别急着进 D 阶段，这三块是上下文工程能不能长上下文稳定工作的地基。

---

## 八、落地对照（这批技能对应岗位里的哪些要求）

> 把你 Stage C 练下来的本事，对齐到当下 Agent / 大模型应用岗真实出现的要求条目。

| 你在 Stage C 掌握的 | 对应的典型岗位要求条目 |
|---|---|
| 上下文工程五策略 + token 预算优化（C1） | 「负责 Agent 上下文管理与成本优化，能在有限窗口下保证长任务稳定运行」 |
| Prompt Caching 与前缀稳定、压缩策略选型（C1） | 「熟悉大模型推理成本优化，掌握 Prompt Caching、上下文压缩等降本手段」 |
| 四种上下文失败模式诊断（C1） | 「能定位 Agent 在长对话中表现退化的根因并给出修复方案」 |
| 短期 / 长期记忆架构与选型（C2） | 「设计 Agent 记忆系统，实现跨会话的用户偏好与事实记忆」 |
| mem0 集成与记忆质量把控（C2） | 「熟悉 mem0 / 向量库等记忆方案，能做记忆写入去重与检索相关性优化」 |
| 驾驭工程八大机制与 harness 设计（C3） | 「为 Agent 设计 harness，覆盖循环刹车、状态持久化、验证闭环，保证生产级稳定」 |
| 验证闭环与结构化错误处理（C3） | 「建立 Agent 结果的自动验证机制，用真机测试替代模型自我声称」 |

**给简历 / 面试的一句话打法**：多数人只会写「用某框架搭了个 Agent」，你要能写「为长任务 Agent 设计上下文工程策略，把静态前缀成本降约 90%、在有效窗口 80% 处启动压缩，并加了一层 harness（四相循环 + 结构化错误 + 进度持久化 + 真机验证闭环）让它连续跑不崩」。这套「让 Agent 稳定跑长任务」的内功，正是从会搭 Agent 到能交付生产级 Agent 的分界。

---

*出品：赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套*
