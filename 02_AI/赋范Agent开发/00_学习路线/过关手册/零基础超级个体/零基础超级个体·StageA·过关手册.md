# 零基础超级个体 · Stage A《会用 AI 写第一行代码》· 过关手册

> 练习向 · 给你自测用
>
> 本阶段你要跨过的四道关：
> - **A1 导学：环境配置与工具准备**：搭好 Cursor + Git + 网络，完成人生第一次代码提交
> - **A2 跑通 ChatGPT 克隆（Vibe Coding 范式）**：先跑起来，做出一个前后端分离、能流式对话的应用
> - **A3 大模型必备 Python 基础**：补足读懂后端所需的最小 Python（变量 / 函数 / 类 / 请求）
> - **A4 读懂 ChatGPT 克隆的后端（深潜专属验收）**：不靠 Agent 代工，亲口讲清一次请求的完整链路
>
> 用法：先做「随堂练习」确认手熟；再用「误区自查」照镜子；「迁移挑战」检验你是不是真会迁移；「排错挑战」练你独立止血的能力；「面试自测 + 模拟面试官」帮你把嘴皮子练顺；最后用「能力自评表」和「落地对照」判断自己站到哪一格、能接什么活。
>
> 本手册所有题目都取自 Stage A 实际课件，答案自包含，不依赖任何外部链接或视频回看即可对照。

---

## 一、随堂练习（动手小题）

### A1 · 导学：环境配置与工具准备
> 来源讲次：《导学：环境配置与工具准备》导学2 Cursor 安装订阅 / 导学3 Git 安装与 GitHub 配置

**练习 A1-1｜第一次提交推送三连**
在本机建一个空目录，用命令行完成：初始化仓库 → 配置你的用户名和邮箱 → 新建一个 `hello.txt` → 提交 → 关联远程 GitHub 仓库并推送。把用到的命令按顺序写出来。

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
- 【自查要点】`add` 之后才能 `commit`（暂存区的概念）；`commit` 只动本地、`push` 才同步到远程；默认分支要用 `main` 不是过时的 `master`（课件「Default Branch → Override, use main」）。
  <!--src:A; ref:导学《环境配置与工具准备》导学3 Git 四大核心操作 / Default Branch override main-->

**练习 A1-2｜说清四大核心操作的工程含义**
课件把 Git 归纳为四个核心操作：`git commit` / `git branch` / `git push` / `git pull`。用一句话分别写出它们「在做什么」。

- 【参考答案】
  - `commit`：把这次改动固化成一个可追溯的版本节点；
  - `branch`：开一条独立开发线，把不同功能的改动隔离开；
  - `push`：把本地改动同步到远程仓库（才能触发 CI/CD 或供别人 review）；
  - `pull`：从远程拉最新代码并合并（协作时先 pull 再开发，避免冲突）。
- 【自查要点】课件反复强调的一句话：Git 不是「代码备份」，而是**工程协作的底层协议**（团队协作协议 / 交付审计链路 / 变更追踪系统 / CI-CD 触发源 四重职责）。能背出「四重职责」说明理解到位。
  <!--src:A; ref:导学《环境配置与工具准备》导学3 Git 四重职责-->

**练习 A1-3｜推送为什么老是要输密码 / 认证失败**
你按课件装好了 Git，但每次 `push` 都被要求输入账号密码，或直接认证失败。课件给的正解是配置什么？在 GitHub 那侧要先生成什么？

- 【参考答案】配置 **Git Credential Manager（GCM，凭证管理器）** 安全存储访问令牌，避免每次 push 都手输；在 GitHub 侧先创建一个 **Personal Access Token（PAT，个人访问令牌）**，用它代替密码。
- 【自查要点】现在 GitHub 已不支持用账号密码直接 push，必须用 PAT / GCM。这正是 A1 常见卡点里「GitHub 推送失败 → 重配凭据」的根因。
  <!--src:B; ref:GitHub 官方 PAT/GCM 认证机制（导学3 提及）；zero-to-builder#A1-->

**练习 A1-4｜Cursor 的四大问答模式各是干嘛的**
课件速查表把 Cursor 归纳为四大问答模式，写出它们的名字和一句话用途。

- 【参考答案】Ask（只读探索，只回答与代码咨询、不动你的文件）/ Agent（自主执行，会自己动手改多个文件并运行）/ Plan（先规划后执行，先列步骤再动手）/ Debug（拿运行时证据排错）。补充：Inline（行内补全）和 Composer（多文件编辑）是两种编辑形态，不算问答模式。提醒一句：认能力别认名字。这些模式的叫法会随 Cursor 版本变化，你要抓住的是「只读 / 自主执行 / 先规划 / 拿运行时证据」这四种能力，别被具体名称带偏。
- 【自查要点】能区分「Ask 只回答、Agent 会自己动手改多个文件并执行」，就抓住了 Cursor 相对传统「编辑器 + 插件」的本质差异：AI 从旁观者，变成了会动手的参与者。
  <!--src:A; ref:pack:cursor-vibe-coding-pack references/cursor-modes-cheatsheet.md 四大问答模式-->

---

### A2 · 跑通 ChatGPT 克隆（Vibe Coding 范式 · 先跑起来）
> 来源讲次：《Vibe Coding AI 编程范式与快速上手实践》及配套 chatgpt-clone 项目

**练习 A2-1｜画出这个项目的技术栈两张清单**
不看代码，凭课件的技术栈总览，分别写出前端和后端各用了哪些主要技术。

- 【参考答案】
  - 前端：React 19 + Vite + TypeScript + Tailwind CSS + Zustand（状态管理）+ react-markdown（Markdown 渲染）+ rehype-highlight（代码高亮，作为 react-markdown 的 rehype 插件，底层基于 highlight.js/lowlight）。
  - 后端：FastAPI + SQLAlchemy + SQLite + OpenAI Python SDK（调 DeepSeek）。
- 【自查要点】「前后端分离」= 前端一个工程、后端一个工程，各自独立启动、用 HTTP 接口通信。前端默认跑在 5173 端口，后端跑在 8000 端口。记住这两个端口，排错时用得上。
  <!--src:A; ref:讲次《Vibe Coding AI 编程范式》；code:chatgpt-clone frontend/package.json-->

**练习 A2-2｜后端起不来，先查哪一行配置**
你把前端起来了，界面能打开，但一发消息后端就 500。按 A2 课件，九成是什么没配好？具体是哪个文件里的哪两项？

- 【参考答案】九成是 `.env` 里的 **API Key 没配**，或 **base_url 写错**。对应后端服务里读的是 `DEEPSEEK_API_KEY`，客户端 `base_url` 应指向 DeepSeek 的接口地址（host 为 `api.deepseek.com`）。
- 【自查要点】环境变量（`.env`）不进 git（在 `.gitignore` 里），所以别人拉了你的仓库也要自己配一份。密钥缺失时服务初始化会直接抛「DEEPSEEK_API_KEY not found」。
  <!--src:A; ref:code:chatgpt-clone backend .env / app/services/deepseek_service.py-->

**练习 A2-3｜跑通自检清单**
写出「chatgpt-clone 本机跑通」的三条验收标准（对照 A2 验收清单）。

- 【参考答案】① 打开页面能对话；② 回复是**流式**逐字出现的，不是一次性蹦出来；③ 代码块能正常高亮。附一张运行截图作为凭据。
- 【自查要点】「流式」是这个项目最关键的特征。如果回复是等半天然后整段刷出来，说明流式没跑通（多半是前端没按 `data:` 逐块读，或后端没用 StreamingResponse）。
  <!--src:A; ref:讲次《Vibe Coding AI 编程范式》A2 验收清单-->

---

### A3 · 大模型必备 Python 基础
> 来源讲次：《大模型必备 Python 基础》Lesson 22 函数 / Lesson 25-26 类 / 作业《控制流语句与函数练习（附答案）》

**练习 A3-1｜把重复逻辑收成一个函数**
课件 Lesson 22 的引子：要对一列数 `[1,2,5,7]` 求最大值，又要对另一列 `[22,51,19,43]` 求最大值，同样的循环写了两遍。请把它封装成一个函数 `find_max(lst)`，一行调用解决。

- 【参考答案】
  ```python
  def find_max(lst):
      ma = lst[0]
      for i in range(1, len(lst)):
          if ma < lst[i]:
              ma = lst[i]
      return ma

  find_max([1, 2, 5, 7])      # 7
  find_max([22, 51, 19, 43])  # 51
  ```
- 【自查要点】函数的价值就是**复用**。课件正是用「同一段逻辑不想写第二遍」引出函数的。参数是入口、`return` 是出口，缺了 `return` 函数就只在内部算、外面拿不到结果。
  <!--src:A; ref:《大模型必备 Python 基础》Lesson 22 find_max 引子-->

**练习 A3-2｜写一个 Point 类并算两点距离**
照 Lesson 26，定义一个二维坐标点 `Point`，构造时可传 `x, y`（默认 0），再加一个方法 `dis(self, p)` 算它到另一点 `p` 的欧氏距离。算一下 `(1,2)` 到 `(2,1)` 的距离。

- 【参考答案】
  ```python
  class Point:
      def __init__(self, x=0, y=0):
          self.x = x
          self.y = y
      def dis(self, p):
          return ((self.x - p.x) ** 2 + (self.y - p.y) ** 2) ** 0.5

  a = Point(1, 2)
  b = Point(2, 1)
  a.dis(b)   # 1.4142135623730951
  ```
- 【自查要点】`__init__` 是构造方法，创建对象时自动跑；`self` 指「当前这个对象自己」；`self.x = x` 是把传进来的值挂到对象身上（实例属性）。方法的第一个参数永远是 `self`。这三点正是 A4 读后端 `DeepSeekService` 类时要用到的地基。
  <!--src:A; ref:《大模型必备 Python 基础》Lesson 26 Point.dis-->

**练习 A3-3｜手写 5 段代码解析笔记（对照 A3 过关物）**
A3 的过关物是 `notes/python-checkpoint.md`，要用自己的话解释 5 个语法点。请从这 6 个里挑 5 个各写一句解释：变量赋值 / `for` 循环 / `if-else` / 函数定义 / 类与实例 / `import` 导入。

- 【参考答案】示例：
  - 变量赋值 `a = 1`：给内存里的一个对象贴个名字，之后用名字就能取到它；
  - `for i in lst`：把序列里的元素一个个取出来，每次循环体跑一遍；
  - `if-else`：条件成立走 if 分支，否则走 else，用来做判别；
  - `def f(x): ...`：定义一个可复用的功能块，`x` 是入口参数，`return` 是出口；
  - `class C:` + `C()`：class 是模板、`C()` 造出一个具体对象（实例）；
  - `from openai import AsyncOpenAI`：把别的模块里的能力搬进来用。
- 【自查要点】你的解释要能「让没学过的人看懂」，而不是把课件原话抄一遍。能用自己的例子=真懂。
  <!--src:A; ref:A3 过关物 notes/python-checkpoint.md-->

---

### A4 · 读懂 ChatGPT 克隆的后端（深潜专属验收）
> 来源讲次：《Vibe Coding AI 编程范式与快速上手实践》（回头精读 chatgpt-clone 后端）
> 提示：A4 徽章关卡是**闭卷**手绘链路图 + 手答代码追踪题，下面的练习就是照它设计的，做的时候先别看代码。

**练习 A4-1｜手绘请求链路图**
不看代码，画出「你在页面点发送 → 屏幕开始逐字蹦字」这中间数据都经过了谁。至少标出这几站：前端、后端接口、模型服务封装、DeepSeek API。

- 【参考答案】
  ```
  前端页面(React)
    └─ streamChat(): fetch POST 到 后端 /api/chat（body 带 messages）
         └─ 后端路由 chat()：返回 StreamingResponse(generate_sse_stream, media_type="text/event-stream")
              └─ generate_sse_stream()：逐块调用 deepseek_service.stream_chat(...)
                   └─ DeepSeekService：用 AsyncOpenAI(base_url=api.deepseek.com)
                        client.chat.completions.create(stream=True)  ← 真正打到 DeepSeek API
                   ← 每收到一个 chunk（reasoning/content），yield 一条 "data: {json}\n\n"
              ← StreamingResponse 把这些 data 行边生成边发回前端
         └─ 前端 response.body.getReader() 逐块读 → 按 "data: " 拆行 → JSON.parse
              → onContent 把新字拼到界面上 → 于是你看到逐字蹦字
  ```
- 【自查要点】能说出「三层」：**前端发起 → 后端中转（不直接暴露密钥，帮你转发给模型）→ 模型服务封装真正调 API**。理解后端为什么要中转（密钥藏在后端、CORS、统一保存对话）就到位了。
  <!--src:A; ref:code:chatgpt-clone backend app/api/chat.py + app/services/deepseek_service.py-->

**练习 A4-2｜代码追踪题：一次流式回复的字是怎么送出去的**
后端的 `stream_chat` 里，模型每吐一小块字，代码是怎么把它变成前端能收到的一行的？「done」时后端还偷偷干了一件事，是什么？

- 【参考答案】模型每来一个 chunk，取 `chunk.choices[0].delta`，从里面拿 `content`（回答）或 `reasoning_content`（思考），`yield {"type":"content","data": 那一小块}`；上层 `generate_sse_stream` 把它拼成 `data: {json}\n\n` 发出去。收到 `done` 时，后端会把这轮的**用户消息和助手完整回复保存进 SQLite 数据库**（还会在第一条时用前 30 字自动生成对话标题）。
- 【自查要点】「流式」= 边生成边发、不等全部算完；`\n\n` 是 SSE 事件的分隔符；前端靠 `data: ` 前缀识别每一行。能说清「done 时落库」说明你读到了细节，不是只看了主干。
  <!--src:A; ref:code:chatgpt-clone backend app/services/deepseek_service.py stream_chat-->

**练习 A4-3｜代码追踪题：前端怎么把流「读」出来**
前端 `streamChat` 拿到响应后，用什么把流一块块读出来？为什么要留一个 `buffer`？

- 【参考答案】用 `response.body.getReader()` 拿到 reader，循环 `reader.read()` 直到 `done`；每块用 `TextDecoder` 解码，`buffer +=` 累积，按 `\n` 切行，`buffer = lines.pop()` 把最后可能不完整的一行留着，等下一块拼全再处理。以 `data: ` 开头的行 `JSON.parse` 后按 type 分发（reasoning / content / done / error）。
- 【自查要点】`buffer` 的存在是因为网络分块到达时，一条 `data:` 行可能被从中间切断，保留半行、下次拼全，才不会 `JSON.parse` 失败。这是流式前端最容易被忽略的细节，也是面试加分点。
  <!--src:A; ref:code:chatgpt-clone frontend/src/services/api.ts streamChat-->

**练习 A4-4｜闭卷 5 问自测（A4 徽章关卡同款）**
不看代码，口头回答：① 前端请求打到哪个后端接口？② 后端用什么响应类型实现流式？③ 密钥存在前端还是后端？④ 对话记录存在哪？⑤ 前端靠什么标志区分「思考内容」和「正式回答」？

- 【参考答案】① `POST /api/chat`；② `StreamingResponse` + `media_type="text/event-stream"`（SSE）；③ 后端（`.env` 里的 `DEEPSEEK_API_KEY`，前端永远看不到）；④ SQLite 数据库（`conversations` / `messages` 两张表）；⑤ 靠每块的 `type` 字段区分：`reasoning` 是思考、`content` 是回答。
- 【自查要点】这 5 问全答对且不用翻代码，就达到了 A 阶段徽章要求的「不靠 Agent 讲清后端」。
  <!--src:A; ref:code:chatgpt-clone backend + frontend 闭卷5问-->

---

## 二、误区自查（照镜子）

> 读法：如果你心里认同左边那句 X，说明你还没真正掌握 Y，去回看 Z。

1. **如果你以为「AI 写的代码不用看懂，能跑就行」** → 说明你还没掌握「A4 的整个存在意义就是让你亲手读懂后端」。快车道只让你拿到产出，徽章关卡偏偏是闭卷讲链路。回看 A4《Vibe Coding》后端精读，先只跟一条 `/api/chat` 路由走一遍。

2. **如果你以为「Git 就是个代码备份工具，用到再学」** → 说明你还没掌握「Git 是工程协作的底层协议（团队协作 / 审计 / 追踪 / CI-CD 四重职责）」。回看 A1 导学3「Git 不是代码备份，而是工程协作的底层协议」一节。

3. **如果你以为「commit 了代码就到 GitHub 上了」** → 说明你混淆了本地和远程。`commit` 只固化到本地版本历史，必须 `push` 才同步到远程。回看 A1 导学3 四大核心操作对应表。

4. **如果你以为「流式回复就是前端加载动画做得好看」** → 说明你还没掌握流式的本质：后端边生成边发（SSE，一行行 `data:`），前端边收边拼。回看 A4 前端 `getReader` 逐块读 + 后端 `StreamingResponse` 两段。

5. **如果你以为「API Key 可以写在前端代码里方便点」** → 说明你还没理解「后端中转」的意义：密钥必须藏在后端 `.env`，前端只跟自己的后端说话，绝不直接把密钥送到浏览器。回看 A2 项目结构 + A4 链路（前端 → 后端 → 模型三层）。

6. **如果你以为「函数和类是高级内容，写脚本用不上」** → 说明你还没掌握「复用」这个最朴素的动机。课件正是用「同一段循环不想写第二遍」引出函数、用「造一批相似对象」引出类。回看 A3 Lesson 22 函数引子、Lesson 25-26 类。

7. **如果你以为「`.env` 文件应该提交到仓库里，方便同步」** → 说明你还没理解密钥不进 git。`.env` 在 `.gitignore` 里被排除，正是为了别把密钥推上 GitHub。回看 A2 项目的 `.gitignore` 与配置说明。

8. **如果你以为「让 Agent 帮我把 A4 的链路图画了、追踪题答了，也算过关」** → 说明你还没意识到 A4 是闭卷验收（不许 Agent 代做）。让 Agent 代画代答=没拿到徽章。回看 A4 深潜验收说明：闭卷、手绘、手答。

---

## 三、迁移挑战（换个场景用同样方法）

**迁移挑战 1｜用同样的 Vibe Coding 流程做一个待办清单应用**
课件里其实附带了一个 `todo-app` 的纯前端 demo。请你**不照抄**，用 A2 学到的 Vibe Coding 流程，从零指挥编辑器做一个待办清单：能添加、勾选完成、删除任务，刷新后数据还在（用浏览器 localStorage）。要求：先让 Agent 陪你把它跑起来，再自己讲清「点击添加」时数据是怎么进列表、怎么渲染回页面的。

- 【怎么算过关】① 三个功能都能用；② 你能像 A4 那样，口述一次「输入文字 → 点添加 → 出现在列表」的数据流；③ 用 Git 提交至少两次（初版 + 一次改进），说明你把 A1 的提交习惯迁移过来了。
- 【提示】这题故意去掉后端。先体会「纯前端也能是完整应用」，再对比 A2 为什么对话应用一定要后端（因为要藏密钥、要落库）。
  <!--src:A; ref:讲次《Vibe Coding AI 编程范式》附带 todo-app demo；code:todo-app-->

**迁移挑战 2｜把在线 API 的封装迁移到另一个模型**
A2 的后端把 DeepSeek 封装成了一个 `DeepSeekService` 类。假设你现在想换成另一家 OpenAI 兼容的模型服务（比如换个 `base_url` 和模型名），你需要改动哪几个地方？请只描述改动点，不用真跑。

- 【怎么算过关】能指出：改 `.env` 里的 Key、改 `base_url`、改 `self.model` 模型名，其余流式逻辑（`stream=True`、`delta.content`、`yield`）几乎不用动。能说清「为什么改动这么小」= 因为大家都遵循 OpenAI 兼容协议。这正是从「会用课程后端」到「能改课程后端」的关键一步。
  <!--src:A; ref:code:chatgpt-clone backend app/services/deepseek_service.py-->

---

## 四、排错挑战（给你一段带病的代码，修好它）

下面是一位同学写的 chatgpt-clone 后端**流式接口**的简化版，他反馈：**「前端能收到完整回复，但是是等好几秒后一次性蹦出来的，完全没有逐字流式效果。」** 请找出问题并修好。

```python
# app/api/chat.py（有问题的版本）
from fastapi import APIRouter
from ..services.deepseek_service import deepseek_service
from ..schemas import ChatRequest

router = APIRouter(prefix="/api")

@router.post("/chat")
async def chat(request: ChatRequest):
    api_messages = [{"role": m.role, "content": m.content} for m in request.messages]

    result = ""
    async for chunk in deepseek_service.stream_chat(messages=api_messages):
        if chunk["type"] == "content":
            result += chunk["data"]     # 把所有块先攒起来

    return {"reply": result}            # 最后一次性返回
```

- 【症状】前端等待数秒无反应，然后整段回复瞬间出现，没有逐字效果；`Content-Type` 是普通 JSON 而不是事件流。
- 【根因】这个接口把模型吐出的每一块都**攒进 `result` 里，等全部收完才 `return` 一个普通 JSON**。本质上退化成了「非流式」。真正的流式必须用 `StreamingResponse` 把每一块**边生成边发**出去，媒体类型要是 `text/event-stream`（SSE），并且每块要按 `data: {json}\n\n` 的格式 `yield`。
- 【修法】
  ```python
  import json
  from fastapi import APIRouter
  from fastapi.responses import StreamingResponse
  from typing import AsyncGenerator
  from ..services.deepseek_service import deepseek_service
  from ..schemas import ChatRequest

  router = APIRouter(prefix="/api")

  async def generate_sse_stream(request: ChatRequest) -> AsyncGenerator[str, None]:
      api_messages = [{"role": m.role, "content": m.content} for m in request.messages]
      async for chunk in deepseek_service.stream_chat(messages=api_messages):
          # 每收到一块就立刻发出去，不再攒
          yield f"data: {json.dumps(chunk, ensure_ascii=False)}\n\n"

  @router.post("/chat")
  async def chat(request: ChatRequest):
      return StreamingResponse(
          generate_sse_stream(request),
          media_type="text/event-stream",
          headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
      )
  ```
- 【一句话总结】流式的关键动作是 **`yield`（边产边发）+ `StreamingResponse` + `text/event-stream`**；一旦你用了 `return` 攒完再给，就一定不流式。这与 A4 课件后端的写法一致。
  <!--src:A; ref:code:chatgpt-clone backend app/api/chat.py 真实 StreamingResponse 写法-->

---

## 五、面试自测（本阶段技术点常问题）

> 每题给：考察点 · 答题要点 · 一层追问。答案已写全，可自包含背诵。
> 说明：以下题目按 2026 年前端 / AI 应用岗真实高频方向校准，考点集中在 Git 基础、前后端分离、SSE 流式响应、密钥安全、Python 基础。

**面试 1｜Git 里 `add` / `commit` / `push` 分别在做什么？三者顺序能颠倒吗？**
- 考察点：对 Git 三区（工作区 / 暂存区 / 本地仓库 / 远程）的基本理解。
- 答题要点：`add` 把改动放进**暂存区**；`commit` 把暂存区固化成一个本地版本节点；`push` 把本地节点同步到**远程**。顺序不能颠倒：没 add 就没东西可 commit，没 commit 就没东西可 push。commit 只动本地，push 才上远程。
- 追问：「commit 完发现写错了还没 push，怎么改上一条 commit 信息？」→ `git commit --amend`。
  <!--src:B; ref:git 官方三区模型（工作区/暂存区/本地仓库/远程）；zero-to-builder#A1-->

**面试 2｜什么是前后端分离？它带来的好处和必须处理的问题各是什么？**
- 考察点：前后端分离的架构直觉。
- 答题要点：前端（浏览器里的界面）和后端（提供数据 / 接口的服务）是两个独立工程，各自部署、通过 HTTP 接口通信。好处：分工清晰、前端可换、后端可复用给多端。必须处理的问题：跨域（CORS）、接口约定、以及**敏感信息（如 API 密钥）只能放后端**。本课 chatgpt-clone 就是前端 React（5173）+ 后端 FastAPI（8000）。
- 追问：「前端直接 `fetch` 后端为什么会报跨域错误，怎么解决？」→ 浏览器同源策略拦截；后端配 CORS 中间件，允许前端来源（本项目 `allow_origins` 里就列了 `localhost:5173`）。
  <!--src:A; ref:code:chatgpt-clone 前端5173/后端8000/CORS；zero-to-builder#A2-->

**面试 3｜SSE（Server-Sent Events）是什么？为什么大模型对话应用都用它做流式输出？**
- 考察点：流式响应的协议选择（2026 年 AI 岗高频）。
- 答题要点：SSE 是基于普通 HTTP 的**服务端单向推送**。客户端发一次请求，服务端保持连接、持续往回推数据块，格式是一行行 `data: ...\n\n`。大模型逐 token 生成、只需服务端往客户端单向推，正好命中 SSE 的设计；而且它是普通 HTTP，天然兼容代理 / 负载均衡、无需 sticky session，比 WebSocket 好扩展。OpenAI、DeepSeek 等几乎都用 SSE。要注意一个坑：常说的「SSE 自动重连」是浏览器 `EventSource` API 才自带的能力；本课 chatgpt-clone 前端用的是 `fetch` + `response.body.getReader()`（ReadableStream）手动读流，并不是 `EventSource`，所以断线不会自动重连，需要时得自己写重试逻辑。
- 追问：「那和 WebSocket 有什么区别，什么时候该用 WebSocket？」→ WebSocket 是全双工双向、长连接，适合双方都要高频发消息（如在线协作、多人游戏）；只需服务端单向推文本流就用 SSE，更轻更好扩。
  <!--src:B; ref:MDN Server-sent events 与 EventSource 文档；zero-to-builder#A4-->

**面试 4｜后端为什么要「中转」模型请求，而不是前端直接调模型 API？**
- 考察点：密钥安全与后端职责。
- 答题要点：如果前端直接调，API 密钥就会打包进浏览器代码，任何人打开开发者工具就能偷走、拿去盗刷。所以密钥必须藏在**后端** `.env`（且不进 git）；前端只跟自己的后端说话，后端拿密钥去调模型、再把结果流回前端。顺带后端还能统一保存对话、加权限、算用量。
- 追问：「密钥怎么才算安全存放？」→ 放后端环境变量 / `.env` 且写进 `.gitignore`，绝不硬编码进代码、绝不提交到仓库、绝不下发到前端。
  <!--src:A; ref:code:chatgpt-clone backend .env（密钥仅后端）；zero-to-builder#A2-->

**面试 5｜大模型 API 是按什么计费的？「token」是什么？**
- 考察点：token 与计费的基本认知（B1 会深挖，A 阶段要能答概念）。
- 答题要点：大模型按 **token** 计费，token 是模型处理文本的最小单位（约等于一个词 / 几个字），一次调用的费用 ≈（输入 token 数 + 输出 token 数）× 单价，输入和输出通常单价不同。所以 prompt 越长、回复越长，越贵。流式输出不改变计费，只是把同样的 token 边生成边发。
- 追问：「同样一句话，为什么中文和英文消耗的 token 数可能不一样？」→ 不同语言、不同分词器切出来的 token 数不同，中文常常一个字就占一个甚至多个 token。
  <!--src:B; ref:大模型 API token 计费通识（DeepSeek/OpenAI 定价页口径）；zero-to-builder#A2-->

**面试 6｜Python 里函数和类的区别是什么？`self` 是什么？**
- 考察点：Python 面向对象最小地基。
- 答题要点：函数是一段可复用的功能（输入参数 → `return` 输出）；类是「造对象的模板」，能把数据（属性）和操作（方法）打包在一起。`self` 代表「当前这个对象自己」，方法第一个参数固定是它，`self.x = x` 就是把值挂到对象身上。本课后端的 `DeepSeekService` 就是一个类，`__init__` 里初始化了 API 客户端。
- 追问：「`__init__` 什么时候被调用？」→ 创建对象（如 `DeepSeekService()`）时自动执行一次，用来做初始化。
  <!--src:A; ref:《大模型必备 Python 基础》Lesson 22/26；code:chatgpt-clone DeepSeekService；zero-to-builder#A3-->

**面试 7｜什么是环境变量？项目里为什么用 `.env` 文件？**
- 考察点：配置与代码分离。
- 答题要点：环境变量是放在代码外面的配置（如密钥、数据库地址），用 `.env` 文件集中管理、代码用 `os.getenv(...)` 读取。好处：换环境（本地 / 线上）只改 `.env` 不改代码；密钥不写死在代码里、`.env` 不进 git，避免泄露。本课后端用 `python-dotenv` 的 `load_dotenv()` 加载。
- 追问：「别人拉了你的仓库跑不起来，报 `KEY not found`，为什么？」→ 因为 `.env` 没进仓库（在 `.gitignore` 里），对方得自己照 `.env.example` 建一份并填自己的密钥。
  <!--src:B; ref:十二要素应用配置法 / python-dotenv；zero-to-builder#A2-->

**面试 8｜前端拿到 SSE 流后，是怎么一块块读出来并显示的？**
- 考察点：流式前端的实现细节（能答出细节是加分项）。
- 答题要点：`fetch` 拿到响应后用 `response.body.getReader()` 得到 reader，循环 `reader.read()` 直到 `done`；每块用 `TextDecoder` 解码，累加到 buffer，按 `\n` 切行，以 `data: ` 开头的行 `JSON.parse` 出内容，追加到界面上，于是逐字显示。要留一个 buffer 存放被网络切断的半行，下次拼全再解析。
- 追问：「为什么要保留 buffer 里没处理完的最后一行？」→ 网络分块到达时一条 `data:` 行可能被从中间切断，直接解析会报错；留半行等下一块拼全才安全。
  <!--src:A; ref:code:chatgpt-clone frontend/src/services/api.ts；zero-to-builder#A4-->

---

## 六、模拟面试官指令（可直接粘贴给 Agent）

> 把下面整段复制给任意 Agent，让它当考官考你。请**只做考官、不做辅导**。

```
请你扮演一名技术面试官，围绕「零基础用 AI 做出第一个全栈应用」这一阶段考我。规则如下：

【角色纪律】请只做考官、不做辅导。在我明确说「结束，请点评」之前，你不要讲解知识点、不要给正确答案、不要提示思路。我答错也只说「记录了，下一题」，不纠正。

【题库范围】只问以下方向，且尽量贴近真实岗位：
1. Git 基础：add/commit/push/pull 的区别与顺序、本地 vs 远程、PAT/凭证管理、默认分支 main。
2. 前后端分离：前端后端各管什么、端口、CORS 跨域、为什么密钥只能放后端。
3. SSE 流式响应：SSE 是什么、为什么大模型用它、和 WebSocket 的区别、前端如何逐块读取。
4. 大模型 API：token 与计费概念、后端中转的意义、环境变量与 .env。
5. Python 基础：函数 vs 类、self 与 __init__、复用的动机。
6. 读懂后端链路：让我口述「点击发送 → 逐字蹦字」的完整数据流（前端→后端→模型→流式回传）。

【流程】
- 先出 6 道题，一次一题，等我回答完再出下一题。
- 每道题我答完后，你追问一层（更深或更具体的一个小问题），再进入下一题。
- 如果我答「不会」，记录后直接下一题，不要讲解。

【结束点评】当我说「结束，请点评」时，你再一次性给出：每道题我的表现评价（答对/部分/未答）、暴露的薄弱点、以及建议我回看 Stage A 的哪个环节（A1 环境与 Git / A2 跑通克隆 / A3 Python / A4 读懂后端）。

现在，请出第 1 题。
```

---

## 七、能力自评表

> 对每个技能点，诚实勾选你现在处于哪一档。三档标准：**能跟做**（照着课件 / Agent 能做出来）< **能独立做**（关掉教程、自己从零做对）< **能教别人**（能讲清为什么这么做、能纠别人的错）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 配好 Cursor + Git + 网络，完成首次提交推送 | ☐ | ☐ | ☐ |
| 说清 Git add/commit/push/pull 的区别与本地-远程关系 | ☐ | ☐ | ☐ |
| 用 PAT / 凭证管理器解决推送认证失败 | ☐ | ☐ | ☐ |
| 区分 Cursor 四种问答模式并选对场景 | ☐ | ☐ | ☐ |
| 把 chatgpt-clone 在本机跑通（能流式对话、代码高亮） | ☐ | ☐ | ☐ |
| 讲清前后端分离：各管什么、密钥为什么只放后端 | ☐ | ☐ | ☐ |
| 用 `.env` 管理密钥，理解它为什么不进 git | ☐ | ☐ | ☐ |
| 写函数封装重复逻辑 / 写一个带方法的类 | ☐ | ☐ | ☐ |
| 读懂后端主文件里的一个函数和一个类的作用 | ☐ | ☐ | ☐ |
| 闭卷手绘请求链路图（前端→后端→模型→流式回传） | ☐ | ☐ | ☐ |
| 讲清 SSE 流式：yield 边产边发、前端逐块读取 | ☐ | ☐ | ☐ |
| 解释 token 计费的基本口径 | ☐ | ☐ | ☐ |

> 过关线建议：全部至少到「能独立做」，其中「手绘链路图」「讲清 SSE 流式」这两项能到「能教别人」，你才真正拿下 A 阶段徽章，因为徽章关卡就是闭卷讲链路。

---

## 八、落地对照（学完这阶段能接的真实小活）

> 都是这阶段能力就能接的入门级活儿，描述自包含。别小看，它们是你作品集和「超级个体」现金流的第一步。

1. **给自己 / 朋友做一个单页工具站。**
   一个 HTML/CSS/JS 的纯前端小工具（比如单位换算、随机点名、简易记账、倒计时番茄钟），用 A2 的 Vibe Coding 流程指挥编辑器做出来，用 A1 的 Git 技能推到 GitHub Pages 上线。交付物 = 一个能打开的网址。适合练手 + 攒第一个可展示作品。

2. **给个人 / 小团队搭一个「专属对话助手」页面。**
   拿 A2 的 chatgpt-clone 改造：换个欢迎语、换个模型、加一句系统提示词（比如「你是我的英语陪练」），本机或小服务器跑起来给身边人用。你已经会「密钥藏后端、流式回复、保存对话」这套，正好是这类活的核心。

3. **帮不懂技术的人把一个现成开源项目在他电脑上跑起来。**
   很多人有「想用某个开源小工具但装不明白」的需求。你会 Git 拉代码、会看 README、会配 `.env`、会排「后端 500 多半是 Key 没配」这类常见坑。这就是一单「远程装机 / 环境搭建」的小服务。

4. **做一个带流式打字机效果的展示 Demo。**
   很多人想要「像 ChatGPT 那样一个字一个字蹦出来」的效果但不会做。你懂 SSE 流式的前后端两端，可以把这个效果单独封成一个可复用的小组件 / 小页面，作为可以拿去接活的演示。

5. **给小商家 / 自媒体做一个「一句话生成文案」的小页面。**
   前端一个输入框 + 后端中转调模型，输出流式文案。技术上就是 chatgpt-clone 的最小裁剪版：把对话历史去掉、把提示词固定成「按我给的主题写一段小红书文案」即可。是把本阶段能力直接变现的最短路径。

---

<div align="center">

**赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套**

</div>
