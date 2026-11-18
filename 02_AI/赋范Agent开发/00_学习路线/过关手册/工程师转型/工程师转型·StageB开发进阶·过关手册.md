# 工程师转型 · Stage B《开发进阶》· 过关手册

> 这一站你要走完一条完整的手艺进阶线：先补齐大模型 Agent 的通用底座（B1），再选一条运行时框架深走（B2 的 LangChain 或 LangGraph），最后关掉框架从零手搓一个 DeepResearch 内核（B3）。只有手搓过，你才真懂框架替你做了什么。
> 本册练习向，重心压在「面试自测」10 道题上，每题都带一层追问，逼你把"会用"讲成"讲得清"。关底徽章卡在 B3 的深潜验收：不 import 任何框架、不让 Agent 代写主循环，自己把 think-act-observe 写出来。下面每一节都在为那道关做准备。

---

## 一、随堂练习

### B1｜练习 1：写出工具定义三要素，并讲清 LLM 到底执行了什么

要求：为一个"查城市天气"的工具写出符合 OpenAI/DeepSeek 规范的 JSON Schema 定义，三要素齐全；再用一句话说清 Function Calling 里 LLM 做了什么、没做什么。

【参考答案】

```python
weather_tool = {
    "type": "function",
    "function": {
        "name": "get_weather",                    # 三要素之一：唯一标识，小写+下划线+动词开头
        "description": (                           # 三要素之二：最关键，决定调用命运
            "获取指定城市的当前天气信息，包括气温、天气状况和湿度。"
            "当用户询问某个城市的天气、气温、是否需要带伞时调用此工具。"
            "目前仅支持中国大陆主要城市。"
        ),
        "parameters": {                            # 三要素之三：JSON Schema 参数定义
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "要查询天气的城市名称，例如：北京"}
            },
            "required": ["city"]
        }
    }
}
```

一句话：LLM 从头到尾没有执行任何函数，它只根据三要素生成一段结构化的调用指令（JSON），真正的执行由你的代码完成。

【自查要点】三个字段名逐字写对：`name` / `description` / `parameters`；`parameters` 内层必须有 `type: "object"`、`properties`、`required`。`description` 有没有写全三段，功能说明 + 触发条件 + 能力边界？课件里最容易被忽视的一句：工具描述是写给 LLM 看的，不是写给人看的；Agent 经常调错工具，第一个排查方向就是 `description`。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》第8章 工具定义三要素-->

### B1｜练习 2：补全 Function Calling 第三阶段的拦截执行代码

要求：模型第一次返回已带 `tool_calls`，你要写本地代码接管：解析函数名与参数、从注册表找到真实函数执行、把结果按规范拼回消息历史。填出关键三步。

【参考答案】

```python
if response_message.tool_calls:
    messages.append(response_message)                      # 先把"请求调用"这条也放进历史
    for tool_call in response_message.tool_calls:
        function_name = tool_call.function.name            # 提取字符串名字，如 "get_weather"
        function_args = json.loads(tool_call.function.arguments)  # JSON 字符串→字典
        function_to_call = available_functions.get(function_name) # 字典派发，找到真实函数
        result = function_to_call(**function_args)         # 本地真正执行
        messages.append({
            "role": "tool",                                # 角色必须是 tool
            "tool_call_id": tool_call.id,                  # 必须与请求 id 完全一致
            "content": str(result),
        })
```

【自查要点】三个"必须"有没有踩到：`available_functions` 是连接"字符串名字"和"内存里真实函数"的唯一桥梁，课件用 `.get()` 取而不是 `[]`，找不到时不崩；回传消息 `role` 必须为 `"tool"`；`tool_call_id` 必须与模型返回的 `id` 严格一致，否则模型无法把结果和请求对上，后续推理出错。最后别忘了带着结果二次请求模型，它才能"看着真实结果说人话"。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》第7章/第10章 完整生命周期-->

### B2｜练习 3：用 LangChain 1.0 的 create_agent 搭一个最小多步 Agent

要求：用课程 1.0 的写法，10 行以内搭出一个能调工具的 Agent，说清它的三要素分别是什么。

【参考答案】

```python
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent

model = init_chat_model("deepseek-chat", model_provider="deepseek")
agent = create_agent(
    model=model,                                  # 要素一：模型
    tools=[get_weather, calculate],               # 要素二：工具
    system_prompt="你是一个多功能助手，可以查询天气和进行数学计算。"  # 要素三：系统提示词
)
result = agent.invoke({"messages": [{"role": "user", "content": "北京天气怎么样？"}]})
```

【自查要点】`create_agent` 的三要素是模型 / 工具 / 系统提示词，另有三扩展（中间件 / 记忆 / 状态）留给生产级需求。有没有用错旧版写法？1.0 已经淘汰 `AgentExecutor` + `LLMChain`，统一入口是 `create_agent`，它建立在 LangGraph 之上。注意 `system_prompt` 在 1.0 里不支持直接嵌 `{variable}` 占位符，要动态内容得走 `dynamic_prompt` 中间件。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解·LangChain 1.0 框架基础入门实战》1.3 底层运行架构 / 搭建Agent应用实战 create_agent 参数表-->

### B2｜练习 4：用 LangGraph 的 StateGraph 定义状态、节点、边并跑通

要求：定义一个带 `question`/`answer` 的状态，加一个 LLM 节点，用 START/END 连起来编译执行。

【参考答案】

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class OverallState(TypedDict):
    question: str
    answer: str

def llm_node(state: OverallState):
    # 节点是一个函数：接收当前 State，返回要更新的 State 通道
    return {"answer": f"收到问题：{state['question']}"}

builder = StateGraph(OverallState)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", END)
graph = builder.compile()
graph.invoke({"question": "你好"})
```

【自查要点】几个课件反复强调的点：状态用 `TypedDict` 定义、由每个节点的返回值动态更新；`START` 和 `END` 是两个特殊节点，分别表示用户输入入口与终止；节点函数第一个位置参数固定是 `state`；图定义完必须 `compile()` 才能跑，编译时会做孤立节点等基本检查。`StateGraph` 继承自 `Graph` 基类。
<!--src:A; ref:讲次《自主Agent开发框架LangGraph·底层原理解析与基础应用入门》2.2 GraphState / 2.3 Nodes / 2.4 Edges-->

### B2｜练习 5：让 LangChain Agent 输出结构化结果

要求：让 create_agent 直接返回一个 Pydantic 结构（比如天气预报），而不是自由文本。

【参考答案】

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent

class WeatherForecast(BaseModel):
    """天气预报结构化输出"""
    city: str = Field(description="城市名")
    temperature: float = Field(description="气温（摄氏度）")

agent = create_agent(
    model=model,
    tools=[get_weather],
    response_format=WeatherForecast     # 指定结构化输出格式
)
```

单模型场景则用 `structured_llm = llm.with_structured_output(WeatherForecast)`。

【自查要点】记住 1.0 的两条口径：Agent 里用 `response_format` 参数指定 Pydantic 模型；裸模型用 `with_structured_output()`。有一条"json 铁律"，用 `with_structured_output` 这条路时，提示词里需要出现 "json" 这个词，否则可能触发校验问题。字段一律用 `Field(..., description="...")` 给语义。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解·框架基础入门实战》3.7 结构化输出解析 / with_structured_output-->

### B3｜练习 6：手写一个带硬上限的最小 agent loop 骨架

要求：不 import 任何框架，写一个 while 循环：模型决定是否调工具、执行工具、回填观察，直到模型不再调工具或到达轮数上限。

【参考答案】

```python
def run_agent_loop(user_input, tools_schema, tool_registry, max_turns=8):
    messages = [{"role": "user", "content": user_input}]
    for _ in range(max_turns):                       # 硬上限：绝不写裸 while True
        resp = client.chat.completions.create(
            model="deepseek-chat", messages=messages, tools=tools_schema
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:                        # 停止条件：模型不再调工具
            return msg.content
        messages.append(msg.model_dump())
        for tc in msg.tool_calls:
            fn = tool_registry.get(tc.function.name)  # .get 兜底，找不到不崩
            try:
                result = fn(**json.loads(tc.function.arguments))
            except Exception as e:
                result = f"工具执行失败：{e}"          # 异常包成 Observation，不中断循环
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
    return "已达最大轮数上限，停止。"
```

【自查要点】手搓包的四条红线你占了几条：`max_turns` 硬上限（不写裸 `while True`）、工具名用 `.get` 兜底不崩、异常包成 Observation 回填而不是让整个 Agent 崩、上下文守预算。停止条件必须显式，模型不再返回 `tool_calls` 才算收敛，光靠模型自觉停不下来会一直烧钱。
<!--src:A; ref:pack:agent-loop-scratch-pack 递归控制与执行红线 / 讲次《从零手搓DeepResearch》-->

### B3｜练习 7：说清 Mini DeepResearch 的三 Agent 分工

要求：不看笔记，写出课程 DeepResearch 的三个 Agent 各自的职责、输入、输出结构。

【参考答案】

| Agent | 职责 | 输入 | 输出结构 |
|-------|------|------|---------|
| `planner_agent` | 把研究主题拆成一组搜索关键词与理由 | 研究主题 | `WebSearchPlan`（含 `searches: list[WebSearchItem]`，每项有 `reason` + `query`） |
| `search_agent` | 对单个关键词联网搜索并压成简短摘要 | 单个关键词 | 文本摘要（2-3 段、<300 词，只留信息不带评论） |
| `writer_agent` | 汇总所有摘要写成长报告 | 主题 + 全部摘要 | `ReportData`（`short_summary` / `markdown_report` / `follow_up_questions`） |

三者由 `ResearchManager` 串起来：`_plan_searches` → `_perform_searches` → `_write_report`。

【自查要点】三个结构化输出类的字段名有没有记准：`WebSearchItem` 是 `reason` + `query`，`ReportData` 是 `short_summary` + `markdown_report` + `follow_up_questions`。都靠 `output_type=` 约束模型必须输出这个结构。`search_agent` 挂的是 `WebSearchTool`，且 `tool_choice="required"` 强制它一定要真搜，不许凭空编。
<!--src:A; ref:讲次《从零手搓DeepResearch》三、从零到一搭建 Mini DeepResearch-->

### B3｜练习 8：把串行搜索改成并行 fan-out，并过滤单点失败

要求：`planner` 给了一串关键词，要对它们并行搜索，任何一条失败不能拖垮整体。

【参考答案】

```python
async def _perform_searches(self, search_plan):
    tasks = [asyncio.create_task(self._search(item)) for item in search_plan.searches]
    results = []
    for task in asyncio.as_completed(tasks):       # 谁先完成先收谁
        result = await task
        if result is not None:                     # 过滤失败项
            results.append(result)
    return results

async def _search(self, item):
    try:
        result = await Runner.run(search_agent, input=f"Search term: {item.query}")
        return str(result.final_output)
    except Exception:
        return None                                # 单点失败返回 None，不抛出
```

【自查要点】并行的关键是 `asyncio.create_task` 建任务、`as_completed` 收结果；单点失败靠 `_search` 里 `try/except` 返回 `None`、上层用 `if result is not None` 过滤掉。这正是"plan → 并行 search fan-out → synthesize"三段式：规划一次、搜索并发、综合一次。
<!--src:A; ref:讲次《从零手搓DeepResearch》ResearchManager 主类 / pack:agent-loop-scratch-pack DeepResearch 流水线编排-->

---

## 二、误区自查

以下六条，每条先判断对错，再看纠正。

1. 「Function Calling 就是 LLM 自己执行了那个函数。」错。这是最普遍的误解。LLM 只生成结构化的调用指令（JSON），真正的执行完全由你的代码完成，LangChain、AutoGen 这些框架只是帮你封装了"拦截 tool_calls、查表、执行、拼回结果"这段 if/for 代码。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》7.1 核心误解纠正-->

2. 「工具定义里 `name` 写清楚就行，`description` 随便写。」错。`description` 才是决定工具命运的最重要字段，它直接决定 LLM 是否在正确时机调用。好描述要回答三问：做什么、什么时候调、不能做什么。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》8.1.2 description-->

3. 「`tool_choice="required"` 更稳，干脆全程强制调用工具。」错。`required` 会逼模型"为了调用而调用"，连"Python 里 list 和 tuple 有什么区别"这种不该调工具的问题它也会硬调。通用场景默认用 `auto`，`required` 只在"必须执行某操作"时用。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》9.1 tool_choice 四种模式-->

4. 「LangGraph 是全新框架，和 LangChain 没关系。」错。LangGraph 是以 LangChain 的表达式语言（LCEL）为基础构建的，它用"图"取代了 LangChain 的 `AgentExecutor` 来管理代理生命周期；它能脱离 LangChain 独立运行，但血缘就在那里。LangChain 1.0 的 `create_agent` 反过来也建立在 LangGraph 之上。
<!--src:A; ref:讲次《自主Agent开发框架LangGraph·底层原理解析与基础应用入门》1. LangGraph 底层原理-->

5. 「用了框架就不用管循环会不会停。」错。手搓时循环是你自己写的，必须显式设停止条件（轮数上限 + 模型不再调工具）；框架只是把这段停止逻辑封装掉了。手搓 loop 停不下来无限检索，就是没设 `max_turns` 硬上限。
<!--src:A; ref:pack:agent-loop-scratch-pack 递归控制与执行红线 / 讲次《从零手搓DeepResearch》-->

6. 「什么任务都用 Agent 最省事。」错。Agent 灵活但执行路径不可预测、调试难、Token 贵、易出错。路径在设计时就能画死的（如每日定时报告、表单确认邮件、ETL）用 Workflow 更好；只有需要动态决策、依赖中间结果的任务才用 Agent。常见架构是"Workflow 作骨架，Agent 作关键节点"。
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》第6章 Agent vs Workflow-->

---

## 三、迁移挑战

### 挑战 1：把 B1 手写的 Function Calling 管线迁到 LangChain create_agent

你在 B1 用原生 API 手写了完整的六步管线（构造 messages → 请求带 tools → 拦截 tool_calls → 查表执行 → 回填 role=tool → 二次请求）。现在改用 LangChain 1.0 的 `create_agent` 实现同样功能。要改哪些、框架替你省了什么？

【参考答案】核心逻辑压成三行：

```python
model = init_chat_model("deepseek-chat", model_provider="deepseek")
agent = create_agent(model=model, tools=[get_weather, calculate],
                     system_prompt="你是能查天气和做计算的助手。")
agent.invoke({"messages": [{"role": "user", "content": "北京天气怎么样？"}]})
```

框架替你省掉的正是 B1 手写的那些样板：不用自己写 `if response.tool_calls` 判断、不用自己维护 `available_functions` 查表、不用自己 `json.loads` 解析参数、不用自己拼 `role="tool"` + `tool_call_id` 消息、不用自己写二次请求循环。工具用 `@tool` 装饰普通函数即可注册。你换来的是简洁，代价是这套拦截-查表-回填的机制被隐藏了，所以先手搓一遍，你才知道框架背后在做什么。
<!--src:A; ref:讲次《LangChain 1.0 体系讲解·框架基础入门实战》2. 核心依赖包 / create_agent-->

### 挑战 2：给 LangGraph 的 ReAct 代理换成多工具自治循环

你原来用 LangGraph 搭了个只能单次调用的 Tool Calling Agent，现在任务要求"依次用天气工具、入库工具、查库工具"，单次调用满足不了。怎么改？

【参考答案】换成预构建的自治循环代理 `create_react_agent`：

```python
from langgraph.prebuilt import create_react_agent
tools = [fetch_real_time_info, get_weather, insert_weather_to_db, query_weather_from_db]
graph = create_react_agent(llm, tools=tools)
```

它把大模型放进一个 `while` 循环：每一步模型自主决定调哪些工具、执行、把输出作为 `ToolMessage` 观察结果反馈回去，直到某一步模型不再返回 `tool_calls`，循环终止、输出最终结果。这解决了 Tool Calling Agent"每次只能调一次"的局限，多步骤决策 + 多工具访问正是自治循环（Full Autonomous）架构的两个扩展点。它的状态模式必须含 `messages` 和 `is_last_step` 两个键。
<!--src:A; ref:讲次《自主Agent开发框架LangGraph·实现自治循环代理（ReAct）及事件流的应用》1. ReAct 构建原理 / create_react_agent-->

---

## 四、排错挑战

【症状】你按加餐课的思路手搓 DeepResearch 内核，写了个自动多轮检索的循环。跑起来它一直在搜、越搜越多、Token 账单飙升，永远出不来最终报告：

```python
def deep_research(query, tool_registry):
    messages = [{"role": "user", "content": query}]
    while True:                                        # 这里
        resp = client.chat.completions.create(
            model="deepseek-chat", messages=messages, tools=SEARCH_TOOLS
        )
        msg = resp.choices[0].message
        messages.append(msg.model_dump())
        for tc in msg.tool_calls:                      # 还有这里
            result = tool_registry[tc.function.name](**json.loads(tc.function.arguments))
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
```

【根因】两处致命问题。第一，`while True` 没有任何终止条件，既没有轮数硬上限，也没有"模型不再调工具就返回"的出口，只要模型每轮都想再搜一次，循环就永不结束。第二，`for tc in msg.tool_calls` 直接遍历，可当模型判断信息够了、这一轮不调工具时 `tool_calls` 是 `None`，遍历会直接抛错崩掉，于是要么无限循环、要么异常退出，两条死路。DeepResearch 的本质是"带停止条件的多轮 ReAct + 证据聚合"，停止条件缺了内核就散了。

【修法】补齐显式停止逻辑：轮数上限 + 充分性判断。

```python
def deep_research(query, tool_registry, max_turns=6):
    messages = [{"role": "user", "content": query}]
    for _ in range(max_turns):                         # 修法1：轮数硬上限
        resp = client.chat.completions.create(
            model="deepseek-chat", messages=messages, tools=SEARCH_TOOLS
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:                          # 修法2：模型不再调工具→信息够了→收敛
            return msg.content
        messages.append(msg.model_dump())
        for tc in msg.tool_calls:
            fn = tool_registry.get(tc.function.name)     # 修法3：.get 兜底，未知工具不崩
            try:
                result = fn(**json.loads(tc.function.arguments))
            except Exception as e:
                result = f"检索失败：{e}"                  # 异常包成 Observation
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
    return "已达检索轮数上限，据现有信息作答。"
```

排这类"停不下来"的通用动作：先看有没有硬上限，再看有没有正常出口，最后确认工具调度有兜底和异常包装。别只靠模型自觉停，课件明确要求设计显式停止条件（轮数上限 + 充分性判断）。
<!--src:A; ref:pack:agent-loop-scratch-pack 症状"loop 停不下来" / 讲次《从零手搓DeepResearch》停止条件设计-->

---

## 五、面试自测

### 1. 什么是 TAO 循环？为什么说 Agent 是闭环系统而聊天机器人是开环系统？

【考察点】对 Agent 核心运行机制的理解。
【答题要点】TAO 循环即 Think（思考，分析状态决定下一步）→ Act（行动，调工具或生成回答）→ Observe（观察，收集结果纳入上下文），三环形成闭环循环往复直到任务完成。它是所有 Agent 架构的共同基础，ReAct、Plan-and-Execute 都是它的变体。传统聊天机器人是开环（输入→输出，没有反馈），Agent 是带反馈的控制回路（输入→执行→观察→调整→再执行），所以能根据中间结果动态调整策略、处理复杂任务。
【追问】TAO 循环怎么决定跑几轮？（答：靠自终止性，每轮 Think 阶段都判断"任务是否已完成"，完成就输出并退出，简单任务一轮结束，复杂任务可能十轮以上。）
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》第3章 TAO 循环-->

### 2. Function Calling 里，LLM 到底做了什么、没做什么？

【考察点】工具调用的分工本质，Agent 体系的基石。
【答题要点】LLM 只做一件事：接收用户请求和工具说明书，判断要不要调工具，若要则生成一段结构化 JSON 调用指令（含函数名和参数）。它从不真正执行函数。执行由你的本地代码完成：拦截 `tool_calls`、用字典把字符串函数名映射到内存里的真实函数、`json.loads` 解析参数、调用、把结果拼成 `role="tool"` 的消息回传。这套机制像 RPC 的变种：LLM 是大脑/调度器，你的代码是执行节点，`available_functions` 字典是连接文本世界和真实代码的唯一桥梁。
【追问】把结果回传时哪个字段绝不能错？（答：`tool_call_id`，必须与模型返回的调用 `id` 完全一致，否则模型无法把结果和请求对上，后续推理出错；同时 `role` 必须是 `"tool"`。）
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》7.1 核心误解纠正 / 第10章 完整生命周期（步骤五 回传结果）/ 12.3 tool_call_id 精确匹配-->

### 3. 工具定义的三要素是什么？为什么 description 最重要？

【考察点】能不能抓住工具调用成败的关键。
【答题要点】三要素是 `name`（唯一标识，小写+下划线+动词开头）、`description`（工具描述）、`parameters`（JSON Schema 参数定义）。`description` 最重要，因为它直接决定 LLM 是否在正确时机调用，很多工具调用失败根源不在代码而在描述写得差。好描述要回答三问：做什么（功能说明）、什么时候调（触发条件）、不能做什么（能力边界）。它是写给 LLM 看的决策依据，不是写给人看的功能概述。
【追问】`parameters` 用什么标准写、为什么？（答：JSON Schema 标准，因为 LLM 训练时见过大量 JSON Schema 样本，能精准理解；`enum` 限定可选值、`required` 标必填、`default` 给默认值。）
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》8.1 工具定义三要素-->

### 4. tool_choice 有哪四种模式？required 有什么坑？

【考察点】对工具调用行为控制的掌握。
【答题要点】四种：`auto`（模型自主判断是否调用，最常用）、`required`（必须调至少一个工具，不能直接回答）、`none`（禁止调用只能生成文本）、指定工具（`{"type":"function","function":{"name":"xxx"}}` 强制调指定工具）。`required` 的坑：对本不需要工具的问题（如"list 和 tuple 的区别"），它会逼模型"为了调用而调用"，产生奇怪行为。所以通用场景默认 `auto`，`required` 只用在"必须执行某操作"的场景（如每次对话都记日志）。
【追问】`none` 模式适合什么场景？（答：需要纯文本输出、不希望模型中途调工具的场景，如生成报告时。）
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》9.1 tool_choice 四种模式-->

### 5. 一个任务该用 Agent 还是 Workflow，你怎么判断？

【考察点】工程选型判断力，避免"什么都用 Agent"。
【答题要点】三个问题快速判断：一，执行路径设计时能不能完全画死？能就用 Workflow。二，后续步骤的选择是否依赖前面步骤的结果、且无法穷举？是就要 Agent。三，是否需要执行中动态切换计划？需要就用 Agent。Workflow 路径固定、成本低、可靠性高，适合定时报告、表单确认、ETL、CI/CD；Agent 适合调研、比价、复杂投诉这类路径不确定、依赖中间结果的任务。代价是 Agent 路径不可预测、调试难、Token 贵。
【追问】实际项目里两者怎么结合？（答：最常见是"Workflow 作骨架、Agent 作关键节点"，用 Workflow 控整体流程，在需要动态决策的节点嵌 Agent，兼顾可靠性和灵活性。）
<!--src:A; ref:讲次《大模型Agent开发基础·大模型 Agent 基础入门与实战》6.2 选型决策树 / 6.3 场景对比-->

### 6. LangChain 1.0 的 create_agent 是什么抽象？它建立在什么之上？

【考察点】对运行时框架分层的理解。
【答题要点】`create_agent` 是 1.0 的核心 Agent 接口，用"三要素 + 三扩展"重构开发范式：三要素是模型、工具、系统提示词（决定 Agent 能思考什么、能做什么、行为边界）；三扩展是中间件、记忆、状态（撑起生产级的可靠性与可观测性）。10 行代码就能搭出封装了"模型调用→工具选择→执行→结束"闭环的基础 Agent。它建立在 LangGraph 之上，LangGraph 是底层的有状态 Agent 运行时（Runtime），负责控制流。1.0 已淘汰旧版 `AgentExecutor`。
【追问】想让 Agent 直接返回结构化对象而不是自由文本，用哪个参数？（答：`response_format`，传入一个 Pydantic 模型；裸模型场景则用 `with_structured_output()`。）
<!--src:A; ref:讲次《LangChain 1.0 体系讲解·框架基础入门实战》1. 1.0 架构 / 搭建Agent应用实战 create_agent-->

### 7. LangGraph 为什么用"图"取代 AgentExecutor？State 和 Reducer 各是什么？

【考察点】对 LangGraph 底层设计动机的理解。
【答题要点】LCEL 构建的是线性 DAG（有向无环图），处理需要循环、复杂条件交互的智能系统时局限明显。LangGraph 用循环图解决这个问题：用图取代 AgentExecutor 管理代理生命周期，把中间状态作为消息跟踪，支持循环、分支、持久化、人机交互。State 是随图计算维护更新的共享状态（上下文/记忆），每个节点读写它、由节点输出动态更新，是节点间通信的通道；节点靠写入 State 通道来传递信息。Reducer（如 `add_messages`）决定新值怎么并入状态，是覆盖还是追加。
【追问】图里 `START` 和 `END` 是什么？条件边靠什么决定走哪条路？（答：START/END 是两个特殊节点，分别是用户输入入口和终止节点；条件边用一个类似 if-else 的 Router 决定路径，可由提示工程、输出解析器或工具调用三种方式实现。）
<!--src:A; ref:讲次《自主Agent开发框架LangGraph·底层原理解析与基础应用入门》1-2. 原理与源码 / State-->

### 8. LangGraph 预构建的 create_react_agent，它的 while 循环什么时候终止？

【考察点】对自治循环代理运行机制的掌握。
【答题要点】`create_react_agent` 把大模型放进一个 `while` 循环：Agent 节点用消息列表调用大模型，若生成的 `AIMessage` 含 `tool_calls`，就转到 tools 节点执行（每个 tool_call 执行一个工具），把响应作为 `ToolMessage` 加回消息列表，然后 Agent 节点再次调用大模型。重复直到某轮响应里不再有 `tool_calls`，循环终止，由 Agent 节点把完整消息列表作为含 `messages` 键的字典返回。它支持工具调用、记忆、多步规划三种能力。
【追问】它的状态模式有什么硬性要求？（答：必须含 `messages` 和 `is_last_step` 两个键，默认就是定义这两键的 AgentState。）
<!--src:A; ref:讲次《自主Agent开发框架LangGraph·实现自治循环代理（ReAct）及事件流的应用》1. ReAct 构建原理-->

### 9. 课程 Mini DeepResearch 是怎样的架构？为什么 search_agent 要用 tool_choice="required"？

【考察点】对多 Agent 研究流水线设计的理解。
【答题要点】三 Agent 架构：`planner_agent` 接收研究主题、生成一份结构化搜索计划 `WebSearchPlan`（每条含 `reason` + `query`）；`search_agent` 对单个关键词用 `WebSearchTool` 联网搜索、压成 2-3 段 <300 词的纯信息摘要；`writer_agent` 汇总所有摘要写成长报告 `ReportData`（`short_summary` + `markdown_report` + `follow_up_questions`）。由 `ResearchManager` 按 plan → 并行 search → write 三步串起来。`search_agent` 用 `tool_choice="required"` 是强制它一定调用搜索工具、不许凭空回答或胡编搜索内容，保证信息真实。
【追问】多个关键词怎么高效搜、某条失败了怎么办？（答：用 `asyncio.create_task` + `as_completed` 并行 fan-out；`_search` 里 `try/except` 失败返回 `None`，上层用 `if result is not None` 过滤，单点失败不拖垮整体。）
<!--src:A; ref:讲次《从零手搓DeepResearch》三、Mini DeepResearch / ResearchManager-->

### 10. 手搓 agent loop 为什么必须设停止条件？怎么设才稳？

【考察点】手搓内核的工程安全意识，关底能力。
【答题要点】因为手搓时循环是你自己写的，只靠模型自觉判断"够了"停不下来，它每轮都可能想再调一次工具，导致无限检索、Token 烧爆。稳的做法是双保险：一是轮数硬上限（用 `for _ in range(max_turns)`，绝不写裸 `while True`）；二是正常收敛出口（模型某轮不再返回 `tool_calls` 就 return），即"轮数上限 + 充分性判断"。配套还要工具名用 `.get` 兜底不崩、工具异常包成 Observation 回填而不中断循环、上下文守 token 预算。DeepResearch 的本质就是"带停止条件的多轮 ReAct + 证据聚合"。
【追问】上下文越滚越长撑爆窗口怎么办？（答：按手搓包的做法做上下文预算，messages 滑动窗口 + 用 tiktoken 计数，检索文本做 token 截断（如 ≤12000 token），保住关键事实与工具结果、压缩历史。）
<!--src:A; ref:pack:agent-loop-scratch-pack 递归控制与执行红线 / 上下文预算 / 讲次《从零手搓DeepResearch》-->

---

## 六、模拟面试官指令

把下面这段话贴给一个 Agent，让它扮演面试官考你。规则是闭卷：你先自己答，它只在你答错或答漏时点破，不许替你把答案讲一遍。

```
你现在是大模型 Agent 开发岗的面试官，我是候选人。请就"Agent 开发进阶（通用底座 + 运行时框架 + 手搓内核）"
这一主题，按由浅入深的顺序，一次问我一道题，等我答完再追问一层，然后才进入下一题。
必考这几点，答不到位就追到我答对为止：
1. TAO 循环三环，以及为什么 Agent 是闭环、聊天机器人是开环；追问自终止性怎么实现。
2. Function Calling 里 LLM 到底做没做函数执行；追问回传结果时 tool_call_id 和 role 的硬要求。
3. 工具定义三要素，为什么 description 最重要（功能/触发/边界三段）。
4. tool_choice 四模式，required 为什么会"为调用而调用"。
5. Agent vs Workflow 三问选型，以及"Workflow 作骨架 Agent 作节点"的混合架构。
6. LangChain 1.0 create_agent 三要素三扩展、它建立在 LangGraph 之上；追问 response_format 与 with_structured_output。
7. LangGraph 为什么用图取代 AgentExecutor，State 与 add_messages reducer 的作用。
8. create_react_agent 的 while 循环何时终止，状态模式必须含哪两个键。
9. Mini DeepResearch 三 Agent 分工与三个结构化输出类字段；search_agent 为什么 tool_choice=required。
10. 手搓 loop 为什么必须设停止条件、怎么设（轮数上限 + 充分性判断 + 兜底 + 异常包装）。
全程不要替我回答，只判定对错并追问；最后给我一个通过/不通过的结论和薄弱项清单。
```

用完把它给你标出的薄弱项，对照本册相应小节回炉。
<!--src:A; ref:讲次《大模型Agent开发基础》《LangChain 1.0 体系讲解》《自主Agent开发框架LangGraph》《从零手搓DeepResearch》全篇考点汇总-->

---

## 七、能力自评表

对每个技能点，诚实勾选你现在的档位。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|--------|--------|----------|----------|
| 讲清 TAO 循环三环与闭环/开环区别，说出自终止性 | ☐ | ☐ | ☐ |
| 写出工具定义三要素，并说清 LLM 不执行函数只生成指令 | ☐ | ☐ | ☐ |
| 手写 Function Calling 拦截执行（查表 + role=tool + tool_call_id） | ☐ | ☐ | ☐ |
| 说清 tool_choice 四模式与 required 的坑 | ☐ | ☐ | ☐ |
| 用三问判断一个任务该上 Agent 还是 Workflow | ☐ | ☐ | ☐ |
| 用 create_agent 搭最小多步 Agent，讲清三要素三扩展 | ☐ | ☐ | ☐ |
| 用 StateGraph 定义状态/节点/边并 compile 跑通 | ☐ | ☐ | ☐ |
| 讲清 LangGraph 用图取代 AgentExecutor、State 与 Reducer | ☐ | ☐ | ☐ |
| 说清 create_react_agent 的 while 何时终止、状态模式两键 | ☐ | ☐ | ☐ |
| 讲清 Mini DeepResearch 三 Agent 分工与结构化输出字段 | ☐ | ☐ | ☐ |
| 不 import 框架手写带停止条件的 agent loop，并做 token 预算 | ☐ | ☐ | ☐ |

三个技能点以上还停在"能跟做"，就别急着过关，回到对应练习补齐。最后一行（手搓 loop）是 B 阶段徽章的闭卷题，必须能脱稿写出。

---

## 八、落地对照

拿这一站真正要交出的东西对照，一条条打勾（回看锚点：engineer-transform#B1、engineer-transform#B2、engineer-transform#B3）。

1. 交得出一个手写的最小 ReAct Agent：会规划、会调一个工具、会观察结果，脚本跑通至少一轮 think-act-observe（对应 B1 主线验收，回看 engineer-transform#B1）。
2. 交得出用所选框架搭的多步 Agent：含 ≥2 个工具调用 + 一次状态流转，代码与运行日志留档（对应 B2 验收，回看 engineer-transform#B2）。
3. 能只改核心三行，把 B1 手写的 Function Calling 管线迁到 create_agent，并说出框架替你省掉的样板。
4. 能用 StateGraph 或 create_react_agent 搭出带条件路由/自治循环的图，讲清 State 怎么在节点间流转。
5. 交得出一个纯手写、不 import 任何框架的 DeepResearch 内核：自写 loop + 检索工具 + 多轮综合，源码里不出现 langchain/langgraph 等框架 import（这是 B 阶段徽章的闭卷题，回看 engineer-transform#B3）。
6. 手搓内核对一个研究型问题能跑出多轮检索并产出带引用的综合报告，每条结论可追溯到检索来源，日志显示 ≥2 轮且能自行判断信息够了才停。
7. 能脱稿讲清"框架帮你省掉了哪些样板"：状态管理、重试、工具路由、拦截回填，核心决策仍是你写的。

做完这一站，你手里握的不再是"会用某个框架搭 Agent"，而是"给我任意运行时框架我能上手，关掉框架我也能从零把内核写出来"。带着这份底气去 C 阶段给 Agent 加内功（engineer-transform#C1）。

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
