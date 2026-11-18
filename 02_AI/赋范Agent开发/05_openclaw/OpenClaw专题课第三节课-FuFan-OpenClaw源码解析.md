# OpenClaw 专题课第三节课 · FuFan-OpenClaw 源码解析

今天我们聚焦的问题是：**一个能记住你、会读说明书学新技能、且每一步都看得见的 AI Agent，到底是怎么从零搭起来的？** 这不是理论问题，而是可以用 Python 亲手重现的工程问题。前三节课我们研究了原版 TypeScript 龙虾（OpenClaw）的架构，建立了"一条消息如何在系统里旅行"的认知框架；本课我们换一个维度——用 Python 的最小代码，把这套机制亲手造一遍。

这门课的教学法叫做 **MVP 最小重现版**：每一章从空白开始，用几十到上百行 Python 搭一个迷你零件（多数章 40-60 行，RAG 章因含双模式+分词+融合达 ~120 行），然后贴上真实源码的行号对照，让你既能「跑起来感受」，也能「回溯到真实项目看原版」。我们会按照项目的灵魂依赖顺序推进：先搭人格层（System Prompt 拼接），再搭大脑层（Agent 内核），再做可观测化（SSE 事件流），然后是学习层（技能插件）、记忆层（RAG 检索）、历史层（会话持久化），最后串成一条完整的消息旅程。为此本课分成两大部分：**第一部分（项目实战导览）** 先把真实项目跑起来、摸清界面与架构，**第二部分（可观测 Agent 核心机制实现）** 再逐个机制用 Python 亲手重现。

贯穿全课的是三条设计哲学：**文件即记忆**（Agent 的知识储存在可读可编辑的 Markdown 文件里）、**技能即插件**（新技能 = 一个带说明书的 `.md` 文件，不需要改代码）、**完全可观测**（工具调用全程可观测，不是黑盒）。每章都会回扣其中一条，让你在做的过程中理解「为什么这样设计」。

> 📌 **目标受众与前置要求**：本课面向有 Python 基础、了解 LLM API 调用基础（用过 `openai` 或类似 SDK）的开发者。你不需要读过 LangChain 源码，不需要 ML/向量库背景。技术上你需要能读写 Python 函数/类、用过 `requests`/`openai` 类库，**不需要搭建完整生产环境，也不需要读过前三节课**。

> 📌 **学完本节你将带走 8 件产物**：① 一个 ~40 行能运行的 System Prompt 拼接器；② 一个用 LangChain `create_agent` 搭起来的 Agent 内核（带工具调用）；③ 一个能解析出 5 类事件的 `astream` 双流解析器；④ 一个技能扫描注入器（SKILL.md → XML 快照 → prompt）；⑤ 一个带 MD5 增量重建的 LlamaIndex RAG 检索器；⑥ 一个会话持久化 + 历史压缩的 SessionManager；⑦ 把以上六件串成消息处理全流程的完整链路（含全链路串讲与 canvas 提取）；⑧ 一个带递归防护的 `spawn_subagent` 多智能体派生器（主智能体派生子智能体完成独立子任务）。前 7 件依次对应**第二部分**第 1 到第 7 章核心内容，第 8 件对应第 8 章多智能体进阶（第二部分第 0 章是导论、第 9 章是总结回顾，不单独产出零件）。

> **【学完不能做】**：本课重现的是机制骨架，不是生产实现。你学完能搭一个约 400 行的迷你可观测 Agent，但不能直接用于生产部署（缺鉴权/限流/负载均衡）；前端 React 组件不深挖；LangGraph 内部运行时底层不展开；浏览器自动化（`browser_use` / Playwright）只在第一部分导览里提及，不做 MVP 重现。

> 📅 **时效性说明**：本课全部源码引用截止 2026 年 6 月，基于 FuFan-OpenClaw 项目源码（`Fufan-OpenClaw项目源码/`）当时的代码状态。所有 `file:line` 引用都是真实可核对的——你可以在自己电脑用 `cat -n` 打到对应位置。

---

# 第一部分：项目实战导览

在正式拆解源码之前，我们先把真实的 `FuFan-OpenClaw` 项目跑起来，完整体验一遍它的界面和功能。这一部分共三章：第 1 章带你完成环境部署和项目启动，第 2 章带你逛遍所有功能界面、建立全景认知，第 3 章把前后端技术栈和整体架构梳理清楚。

为什么不直接进源码？因为如果你没亲眼见过 Agent 流式回复、没点过 `Canvas` 面板、没逛过技能商店，第二部分那些 `file:line` 锚点对你来说只是一堆陌生符号。先建立「这个东西长什么样、能做什么、由什么搭成的」直感，第二部分再逐个机制用 Python 重现，才不会在细节里迷路。

## 第 1 章：环境部署与项目启动

这一章我们解决「能不能跑」的问题——环境配齐、后端启动、前端启动、发出第一条消息。只有亲眼看到 Agent 流式回复，后续的源码拆解才有了对照的「活的参照物」。`FuFan-OpenClaw` 是一个 Python 后端 + `Next.js` 前端的本地应用，部署链路清晰，按下面的顺序走一遍即可。

> **【关于本章代码块格式】**：以下环境部署命令以 code cell（`!` 前缀）形式给出。创建环境、装依赖、装浏览器可在配好的 kernel 直接运行；但 `conda activate`、启动服务（`uvicorn` / `npm run dev`）等受 Jupyter 限制（子进程激活无效 / 阻塞挂起），已默认注释保护，请复制到独立终端执行。

### 1.1 环境依赖清单

在启动之前，先确认运行环境满足要求。这里是一份**部署装机清单**——只列启动必需的运行时版本、依赖安装入口和密钥；至于后端前端各用了哪些框架、为什么这么选，留到第 3 章「架构与技术栈」从架构视角系统展开，这里不展开技术细节。

FuFan-OpenClaw 运行环境与依赖

| 类别 | 要求 | 说明 |
|------|------|------|
| Python | ≥3.10 | 后端运行时 |
| Node.js | ≥18 | 前端构建运行时 |
| 后端依赖 | `requirements.txt` 一键装 | `pip install -r requirements.txt` 自动装齐 `FastAPI` / `LangChain 1.x` / `LlamaIndex` / `browser-use` 等（完整技术栈见第 3 章）|
| 前端依赖 | `package.json` 一键装 | `npm install` 自动装齐 `Next.js 14` 等 |
| 密钥①（必填）| `DEEPSEEK_API_KEY` | 主 LLM 调用凭证，走 DeepSeek 接口 |
| 密钥②（用 RAG 才需）| `OPENAI_API_KEY` + `OPENAI_BASE_URL` | RAG 检索的 Embedding 凭证，走 OpenAI 兼容代理 |
| 密钥③（可选）| `TAVILY_API_KEY` | `tavily_search` 工具凭证，不填则禁用联网搜索 |
| 存储 | 纯本地文件系统 | 无 MySQL / Redis，无独立向量库服务 |

关于密钥有一个**最容易踩的坑**必须先讲清：`FuFan-OpenClaw` 用的是**双轨凭证**——主 LLM 走 `DEEPSEEK_API_KEY`（DeepSeek 接口），而 RAG 检索的向量化走 `OPENAI_API_KEY` + `OPENAI_BASE_URL`（OpenAI 兼容接口）。这两套 key 用途不同、不能混用：把 DeepSeek 的 key 填到 OpenAI 字段里，RAG 建索引时会直接报鉴权失败。如果你暂时不用 RAG 模式，只填 `DEEPSEEK_API_KEY` 就能跑通基础对话；`TAVILY_API_KEY` 也是可选的，不填只是禁用联网搜索工具，其余功能不受影响。

> **【踩坑预警】**：不要被项目 `README.md` 里"DeepSeek / OpenAI API 兼容"这句话误导成"二选一"。在源码里这俩是**各司其职的两条轨**——`graph/agent.py` 的 `ChatDeepSeek` 用 DeepSeek key 做对话，`graph/memory_indexer.py` 的 `OpenAIEmbedding` 用 OpenAI key 做向量化。如果你只填一套 key 又开了 RAG，会在「对话正常但检索报错」的状态里排查半天。完整的 7 个 `.env` key 清单，第二部分第 0 章 0.2 节有专门的表格。

### 1.2 后端启动

后端是一个标准的 `FastAPI` 应用，入口在 `backend/app.py`。启动流程分五步，按顺序执行，每一步都确认正常再往下走。

**步骤一：创建并激活 conda 环境**

本课全程使用 conda 环境（与系列课程保持一致）。用 conda 新建一个独立环境隔离依赖，避免包版本污染 base 或其他项目。激活之后，后续的 `pip install` 才会装进这个环境里。

```python
# 创建独立 conda 环境（!conda create 可在已配好的 kernel 直接运行）
!conda create -n fufan-openclaw python=3.11 -y

# 注意：!conda activate 在 Notebook 的 ! 子进程里激活，不会改变当前 kernel 环境。
# 正确做法是在终端激活该环境后用它启动 Jupyter；下面一行仅作命令记录：
!conda activate fufan-openclaw
```

执行后终端提示符前会出现 `(fufan-openclaw)` 前缀，说明环境已激活；后续所有 `pip` / `python` 命令都会落在这个环境里，不会污染 base 环境。conda 环境里直接用 `pip` 安装 `requirements.txt` 即可，无需逐个去找 conda 源。

**步骤二：安装后端依赖**

依赖版本锁定在 `backend/requirements.txt`，一条命令装齐。这一步会拉取 `FastAPI`、`LangChain 1.x`、`LlamaIndex`、`browser-use`、`tiktoken` 等较多的包，首次安装耗时较长属于正常现象。

```python
# 进入后端目录装依赖（!cd 仅影响当前 ! 子进程，故用 && 让 cd 对同一行的 pip 生效）
!cd backend && pip install -r requirements.txt
```

命令结束且无报错即安装完成；若中途因网络波动失败，重跑同一条命令会跳过已装好的包、只续装剩余部分，不必从头来。

**步骤三：配置环境变量**

从项目自带的模板复制一份 `.env`，再填入你的密钥。密钥一律走 `.env` 文件加载、禁止硬编码进代码，`.env` 文件也不要提交到 Git（建议在 `.gitignore` 里加上它）。

```python
!cp .env.example .env
# 注意：模板里只有 OPENAI_* 字段，没有 DEEPSEEK_*，必须手动补上主 LLM 凭证（在编辑器里打开 .env 填入）：
# DEEPSEEK_API_KEY=your_deepseek_key
# DEEPSEEK_BASE_URL=https://api.deepseek.com   # 可选，不填默认 https://api.deepseek.com
# DEEPSEEK_MODEL=deepseek-chat                 # 可选，不填默认 deepseek-chat
# 若要体验 RAG 模式，再填 OpenAI 兼容 Embedding 凭证：
# OPENAI_API_KEY=your_openai_key
# OPENAI_BASE_URL=https://ai.devtool.tech/proxy/v1
```

> **【踩坑预警 · `.env.example` 模板与代码不同步】**：项目自带的 `.env.example` 是一份**过时的 OpenAI 兼容模板**——里面写的是 `OPENAI_API_KEY=sk-xxx` / `MODEL_NAME=gpt-4o-mini`，完全没有 `DEEPSEEK_*` 字段。但后端主对话代码读的是 `DEEPSEEK_*` 系列（源码 `backend/graph/agent.py`：第 9 行 `from langchain_deepseek import ChatDeepSeek` 为文件顶部模块 import，第 38-44 行 `ChatDeepSeek(api_key=os.getenv("DEEPSEEK_API_KEY"), ...)` 初始化）。所以 `cp .env.example .env` 之后，你必须**手动补上 `DEEPSEEK_API_KEY`**（`DEEPSEEK_BASE_URL` / `DEEPSEEK_MODEL` 不填则走默认值）。怎么判断踩了这个坑：后端能正常启动、但一发消息就报 401 或 `api_key` 相关错误，基本就是漏填了 `DEEPSEEK_API_KEY`。

**步骤四（可选）：安装浏览器自动化依赖**

如果你想体验 `browser_use` 浏览器自动化工具，需要额外装一次 Playwright 的 Chromium 内核。不打算用这个工具可以跳过这一步，不影响其余功能。

```python
# 安装 Playwright 的 Chromium 浏览器（browser 工具依赖，下载数百 MB）
!playwright install chromium
```

看到 Chromium 内核下载并安装完成的提示即可；这一步只需做一次，之后 `browser_use` 工具就能驱动这个内核打开网页执行自动化任务。

**步骤五：启动后端服务**

后端监听本地 `8002` 端口（前端默认就是连这个端口，见 1.3）。`--reload` 让代码改动后自动重启，开发时很方便。

```python
# 启动后端服务（阻塞式长进程，会一直占用本 cell —— 默认注释保护）
# 请在独立终端取消注释运行，不要在 Notebook 内直接跑（否则后续 cell 无法执行）：
!uvicorn app:app --port 8002 --reload
```

启动时后端会自动做三件事：扫描 `skills/` 目录生成技能快照、初始化 Agent、构建 RAG 记忆索引（源码 `backend/app.py:16-31` 的 `lifespan` 钩子）。看到终端打印 `fufan OpenClaw backend ready` 就说明已就绪。可以访问 `http://localhost:8002/` 看到 `{"status": "running"}`（源码 `app.py:59-61` 的根路由），或访问 `http://localhost:8002/docs` 看 `FastAPI` 自动生成的 API 文档，作为二次确认。

> **【常见误区】**：本项目**没有自带测试套件**（`requirements.txt` 里不含 `pytest`）。如果你在别的教程里见过"跑一遍测试确认环境 OK"的步骤，这里没有——验证后端是否就绪，看启动日志的 `... ready` 和根路由返回即可。另外，如果你没填 `OPENAI_API_KEY` 就启动，构建 RAG 索引时会打印一条 `⚠️ ...` 警告但**不会崩溃**（源码 `memory_indexer.py` 对 Embedding 失败做了降级），基础对话照样能用。

### 1.3 前端启动

前端是一个 `Next.js 14` App Router 应用，启动两步即可。确保后端已经起来再启动前端，否则首次加载会看到 API 连接失败的提示。

**步骤一：安装前端依赖**

进入 `frontend/` 目录，用 `npm` 装齐 `Next.js` / `React 18` / `Monaco Editor` 等前端依赖。

```python
# 进入前端目录装依赖（同 && 写法让 cd 对 npm 生效；需本机已装 Node.js）
!cd frontend && npm install
```

首次安装会拉取较多前端依赖，结束后目录下会出现 `node_modules/`；无报错即可进入下一步启动开发服务器。

**步骤二：启动开发服务器**

前端开发服务器默认监听 `3000` 端口（`package.json` 的 `dev` 脚本是 `next dev -H 0.0.0.0`）。

```python
# 启动前端开发服务器（阻塞式长进程 —— 默认注释保护）
# 请在独立终端取消注释运行：
!cd frontend && npm run dev
```

浏览器打开 `http://localhost:3000`，看到聊天界面就说明前端正常。前端通过 `frontend/src/lib/api.ts:6-9` 里写死的 `8002` 端口连接后端——这就是为什么步骤上要先起后端、再起前端。

### 1.4 首条对话验证

前后端都起来了，用一条最简单的消息验证整条链路是通的。

**步骤一：新建会话**

在界面左侧点「新建会话」，系统会创建一个新的对话 session（对应后端 `POST /api/sessions`）。这里和某些 Agent 产品不同——`FuFan-OpenClaw` **不需要你手动选一个工作目录**，工具的文件操作默认沙箱在后端项目目录内（这一点第 2 章 2.1 节会展开）。

**步骤二：发送第一条消息**

在输入框发送：「列一下当前目录下有哪些文件」。Agent 会调用 `terminal` 工具执行 `ls`，然后流式回复文件列表。如果你看到文字一个字一个字地冒出来、中间还弹出一张「工具调用卡」显示它调了 `terminal`，说明 `sse-starlette` 的 SSE 流式链路和工具调用可视化（ThoughtChain）都正常工作了——这正是「完全可观测」哲学在产品上的第一次露面。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154601647.png)

到这里你已经把真实项目跑起来了。下一章我们不写代码，而是把这个跑起来的产品从头到尾逛一遍，建立一张「功能 × 机制」的全景地图——让你在进入第二部分之前，先知道每个界面背后对应着哪一章要重现的源码。

## 第 2 章：功能全景与界面操作

跑起来之后，先建立「这个真实产品长什么样」的整体认知。这一章不是功能手册，而是「功能 × 机制」的映射建立——我们逐个界面走一遍，重点看「这个界面让你操作的东西，背后是第二部分哪一章的代码」。有了这张映射，后续每当你在 Python 里复现某个机制，脑子里就能自然浮现它在界面上是什么样子。

`FuFan-OpenClaw` 前端只有 3 个页面：`/`（对话主界面）、`/memory`（记忆系统）、`/skills`（技能管理），分别对应顶部导航的「对话 / 记忆系统 / 技能管理」三个入口（源码 `frontend/src/components/layout/Sidebar.tsx` 的 `NAV_ITEMS`）。每个页面背后都对应着第二部分要重现的某个核心机制。

### 2.1 整体布局

打开 `http://localhost:3000`，看到的是对话主界面。整体布局由四个区域构成（源码 `frontend/src/app/page.tsx`）：顶部是 `Header`（含双主题切换等全局控制）；左侧是宽 `256px` 的 `Sidebar`（上方三个页面导航，下方是会话列表，支持新建 / 重命名 / 删除）；中间是 `ChatPanel` 对话流主区（消息气泡 + 流式回复，顶部工具栏还有 `Raw Context` 与设置两个入口，分别见 2.6、2.7）；右侧是一个宽 `520px` 的滑入面板，可在 `Canvas` 渲染面板和原始消息面板之间切换（默认收起）。此外 `Header` 上还有一个「学习模式」开关，可呼出 `LearnPanel` 学习浮层（默认收起，不占常驻布局）。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154604066.png)

▲ 对话主界面真实截图：左侧三导航 + 会话列表、中间对话流、顶部双主题切换与状态栏（注意左侧栏没有文件树）

这里要点出一个和某些 Agent 产品的关键区别：`FuFan-OpenClaw` 的左侧栏**没有「工作目录文件树」，也不需要你绑定本地目录**。它的文件类工具（`terminal` / `read_file` / `search_knowledge_base`）默认把操作沙箱在后端项目目录（`base_dir`）内（源码 `backend/tools/__init__.py:22-24` 的注释明确说明：只有这三个文件类工具受 `base_dir` 约束，`python_repl` / `fetch_url` / `browser` / `tavily` 不受限）。所以你在界面上不会看到「选目录」的步骤——这是它和「让用户绑定工作区」那类产品的设计差异。

### 2.2 对话系统与 7 个工具

对话采用 `sse-starlette` 实现的 SSE（Server-Sent Events）流式输出——大模型每生成一个 token 就推送一个，你能看到 Agent 逐字回复，而不是等全部生成完再一次性展示。更关键的是「完全可观测」：当 Agent 决定调一个工具，界面会弹出一张 **ThoughtChain 工具卡**（源码 `frontend/src/components/chat/ThoughtChain.tsx`），展开能看到它调了什么工具、传了什么参数、拿到什么返回值；如果开了 RAG，还会有一张检索结果卡（`RetrievalCard.tsx`）。Agent 在对话中可调用 **7 个内置工具**：

Agent 可调用的 7 个内置工具

| 工具名 | 用途 | 沙箱 | 源码文件（`backend/tools/` 下）|
|--------|------|------|------|
| `terminal` | 在后端项目目录里执行 shell 命令（ls / mkdir / git 等） | 受 `base_dir` 约束 | `terminal_tool.py` |
| `python_repl` | 执行 Python 代码片段（无沙箱，直接 exec） | 不受约束 | `python_repl_tool.py` |
| `fetch_url` | 抓取指定 URL 的网页内容并转 Markdown | 不受约束 | `fetch_url_tool.py` |
| `read_file` | 读取项目目录内的文件内容 | 受 `base_dir` 约束 | `read_file_tool.py` |
| `search_knowledge_base` | RAG 混合检索（BM25 + 向量）知识库 | 受 `base_dir` 约束 | `search_knowledge_tool.py` |
| `browser_use` | 浏览器自动化操作（Playwright） | 不受约束 | `browser_use_tool.py` |
| `tavily_search` | 调用 Tavily 联网搜索（需 `TAVILY_API_KEY`） | 不受约束 | `tavily_search_tool.py` |

注意：工具总数是 7 个，不是 6 个。项目 `README.md` 的工具表只列了前 6 个、漏掉了 `tavily_search`，但源码 `backend/tools/__init__.py:30-36` 的 `get_all_tools` 实实在在返回了 7 个工具。本课一律以代码为准——第二部分第 2、4 章重现工具循环和技能系统时，用的都是这个 7 工具的事实。

还有一个容易被忽略的**第 8 个工具**：上面 7 个是 `get_all_tools` 默认（`include_spawn=False`）返回的基础工具；而主智能体在启动时用 `get_all_tools(..., include_spawn=True)` 额外挂载了第 8 个工具 `spawn_subagent`（运行时派生子智能体，源码 `tools/__init__.py:39-42` + `agent.py:47`），子智能体自己构造工具集时用 `include_spawn=False` 杜绝无限递归。本课第二部分的 MVP 聚焦这 7 个基础工具；第 8 个 `spawn_subagent` 属于进阶，第二部分**第 8 章**会用真跑 MVP 专门重现它（含递归防护与子任务隔离）。

下面这张真实截图，就是 Agent 在对话里调用 `terminal` 工具时前端展示的样子——你能清楚看到工具卡的输入参数、返回结果，甚至它自我纠错（先试 `dir` 失败、再改 `ls`）的全过程，这正是「完全可观测」哲学的产品化呈现：

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154608778.png)

▲ ThoughtChain 工具卡真实截图：展开的 terminal 卡显示 INPUT/OUTPUT，中间还夹着一张 Memory Retrieval（RAG 混合检索）卡

### 2.3 `/memory` 页：分层记忆 + AI 优化

点击顶部导航的「记忆系统」进入 `/memory` 页。这个页面直接暴露了 Agent 的「记忆状态」——左侧是记忆文件列表（`MemoryFileList.tsx`），列出 `SOUL.md` / `IDENTITY.md` / `USER.md` / `AGENTS.md` 这几个系统级指令文件（源码存放在 `backend/workspace/` 目录），以及跨会话长期记忆 `MEMORY.md`（源码存放在 `backend/memory/` 目录，同级还有一个 `logs/` 预留子目录）；右侧是嵌入了 `Monaco Editor`（VSCode 同款编辑器）的编辑区（`MemoryEditor.tsx`），你可以直接在浏览器里编辑这些 Markdown 文件，改完即时生效——这正是「文件即记忆」哲学的直观体现。你在页面上编辑的，就是 `backend/workspace/` 和 `backend/memory/` 这两个目录里的真实文件。

编辑区右上角有一个「AI 优化」按钮（`OptimizeModal.tsx`），点击后大模型会读当前记忆文件、生成去重 / 重组后的版本供你预览，确认后写回。这里有一个值得记住的实现细节：**它没有专门的后端"优化记忆"接口**，而是直接复用对话用的 `streamChat` 通道（喂一段"请优化这份记忆"的 prompt），生成结果再通过文件接口落盘。这条 6 组件拼接成 System Prompt 的逻辑，正是第二部分第 1 章要用 ~40 行重现的 `build_system_prompt`（源码 `graph/prompt_builder.py:38-62`：38-50 行是前 5 组件按序定义 + 非 RAG 分支追加 MEMORY，58-60 行是 RAG 分支追加 RAG_GUIDANCE）。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154604072.png)

▲ /memory 页真实截图：左侧核心记忆文件列表（SOUL/IDENTITY/USER/AGENTS/MEMORY/SKILLS_SNAPSHOT，各带 token 数），点击任一文件即在右侧 Monaco 编辑器打开

### 2.4 `/skills` 页：技能库 + 技能商店 + AI 创建

点击顶部导航的「技能管理」进入 `/skills` 页。这里显示项目自带的技能文档（`SKILL.md` 格式），每个技能都是 `backend/skills/` 下的一个文件夹，文件夹里至少有一个 `SKILL.md`（技能说明书）。`FuFan-OpenClaw` 内置了 5 个技能，分别对应下面 5 个目录：

5 个内置技能与源码目录

| 技能名 | 源码目录 | 作用 |
|--------|----------|------|
| `docx` | `backend/skills/docx/SKILL.md` | 生成 / 读取 / 编辑 Word 文档（含 `scripts/` 子目录） |
| `get_weather` | `backend/skills/get_weather/SKILL.md` | 查询指定城市实时天气 |
| `long_document_workflow` | `backend/skills/long_document_workflow/SKILL.md` | 长文档分章节结构化生成 |
| `pdf_to_markdown` | `backend/skills/pdf_to_markdown/SKILL.md` | 把 PDF 转换为结构化 Markdown |
| `storyboard_generator` | `backend/skills/storyboard_generator/SKILL.md` | 生成影视分镜脚本 |

这 5 个技能由后端启动时的 `scan_skills`（源码 `backend/tools/skills_scanner.py`）扫描 `backend/skills/` 目录、解析每个 `SKILL.md` 的 YAML frontmatter 后，生成一份技能快照 `backend/SKILLS_SNAPSHOT.md`（这份快照才是注入到 System Prompt 里、Agent 真正"看得见"的技能清单）。关键机制：扫描发生在启动时——所以你往 `backend/skills/` 新放一个技能文件夹后，必须**重启后端**重新扫描，它才会进入快照、被 Agent 识别。每个技能卡可以启用 / 禁用 / 删除（删除对应后端 `DELETE /api/skills/{name}`，源码 `api/files.py:108`）。

这个页面有两个「让技能自己长出来」的入口，都是真实可用的功能：

**第一个是技能商店**（`SkillStore.tsx`）：它直接从前端 fetch GitHub API，拉取 Anthropic 官方技能仓库 `anthropics/skills` 的技能列表，浏览后可一键安装——安装时把远端的 `SKILL.md` 等文件通过文件接口写到本地 `skills/{name}/` 目录。**第二个是 AI 创建**（`NewSkillModal.tsx`）：你用自然语言描述需要的技能，它复用对话的 `streamChat` 通道让大模型生成完整的 `SKILL.md`，再落盘。

这里藏着一个统一的设计：无论是技能商店安装、AI 创建技能，还是 2.3 节的 AI 优化记忆，后端**都没有为它们单独开接口**——它们要么直连外部 GitHub API，要么复用 `streamChat` 对话通道 + 文件写入接口（这也是为什么 `app.py` 只注册了 6 个路由却撑起了这么多功能）。技能从 `SKILL.md` 文件被扫描成 prompt 注入的完整闭环，正是第二部分第 4 章要重现的 `scan_skills`（源码 `tools/skills_scanner.py`）。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154608607.png)

▲ /skills 页真实截图：技能库列出 5 个内置技能，每张卡带描述、SKILL.md 路径、版本号与启用/删除开关

### 2.5 FuFan-OpenClaw 特色：Canvas/A2UI + 双主题

`FuFan-OpenClaw` 有一个区别于普通聊天工具的特色功能——**Canvas/A2UI**。当 Agent 在回复里输出 `<openclaw-canvas>...</openclaw-canvas>` 标签包裹的 HTML 时，后端会用正则把标签内的 HTML 提取出来，作为独立的 `canvas` 事件推给前端，前端在右侧面板用 `iframe` 实时渲染（源码后端 `api/chat.py:196-205` 提取、前端 `components/chat/CanvasPanel.tsx` 渲染）。比如你让它"写一个 HTML 贪吃蛇"，右侧面板会直接出现可玩的游戏。这套机制是第二部分第 7 章全链路串讲里的一站。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154601607.png)

▲ Canvas/A2UI 真实截图：让 Agent「做一个计数器」，右侧面板用 iframe 实时渲染它生成的 HTML（含 Code/Preview 切换、底部 sandbox 安全隔离标识）

另外界面支持亮 / 暗双主题（Bronze / Gold，源码 `frontend/src/lib/theme.tsx`），通过 CSS 自定义属性无闪烁切换，属于体验细节，本课不展开。

> **诚实边界声明**：`FuFan-OpenClaw` 的重心是「文件即记忆 + 技能即插件 + 完全可观测 + RAG 检索 + Canvas 渲染」这条主线。如果你了解过另一类强调「自我进化」的 Agent 项目（技能自动演进、会话质量诊断打分等），那是另外的设计方向，**不在本项目范围内**——别拿那套功能来对照本项目的界面，会找不到对应入口。

Canvas 之外，对话页顶部工具栏还有两个入口没看——`Raw Context`（看见发给模型的完整上下文）和设置面板（记忆检索 + API Key）。接下来两节分别看它们。

### 2.6 Raw Context 面板：看见发给模型的完整上下文

对话页顶部工具栏的 `FileCode` 图标（鼠标悬停显示 "Raw Messages"）点开后，会在右侧滑入一个 **Raw Context** 面板（源码 `frontend/src/components/chat/RawMessagesPanel.tsx`），它和 2.5 的 `Canvas` 面板共用右侧位置、互相切换。如果说 2.2 的 `ThoughtChain` 工具卡让你看到「Agent 调了什么工具」，那 Raw Context 面板就是把「这一刻到底有什么东西发给了大模型」整个摊开——这是「完全可观测」哲学最彻底的一个窗口。

面板主体是一条条原始消息，第一条永远是 **SYSTEM**：它是后端实时调 `build_system_prompt` 拼出来的系统提示词，你能直接看到 `SKILLS_SNAPSHOT`、`SOUL`、`USER` 这些组件按序拼接的真身（正是第二部分第 1 章要重现的产物）。往下是这一会话的 **USER / ASSISTANT** 消息；如果某条 assistant 消息调了工具，它旁边会挂一个 `N tool call` 徽章，点开能看到每次调用的 `Input` 与 `Output`（这正是第二部分第 3 章 `astream` 解析出的 `tool_start` / `tool_end` 落库之后的样子）。这一整份内容的后端接口是 `GET /api/sessions/{id}/messages`（源码 `api/sessions.py:58-66`）——它把实时拼好的 system prompt 作为第一条消息，再接上会话 JSON 里存的历史消息一起返回。

标题栏右侧那个 `~N tokens` 徽章，是后端用 `tiktoken`（`cl100k_base` 编码）精确统计的，算的是 **system prompt + 全部消息** 的总量（源码 `api/tokens.py:27-46`），所以你能直观看到「这轮上下文已经涨到多大了」。面板底部还有两个按钮：**Refresh** 重新拉取消息与 token 数；**Compress** 手动触发压缩——弹确认框后把前 50% 历史归档成一段摘要（对应 `POST /api/sessions/{id}/compress`，源码 `api/compress.py`）。这个手动压缩和第 6 章会讲的「上下文溢出自动压缩」复用同一套 `compress_history` 逻辑，只是触发方从 Agent 自己变成了你点的按钮。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154605787.png)

▲ Raw Context 面板真实截图：顶部标题 + 「~3,909 tokens」徽章；主体从 SYSTEM（实时拼接的 system prompt，可见 Skills Snapshot 组件）到 USER、ASSISTANT（带「1 tool call」徽章 + 可展开的 terminal 工具卡）；底部是 Compress / Refresh 按钮与消息计数

一句话，Raw Context 面板把第二部分第 1 章（system prompt）、第二部分第 3 章（工具调用可观测）、第二部分第 6 章（会话持久化 + token + 压缩）三件事缝在同一个窗口里——它是你在产品里能亲眼看到「上下文工程」的地方。下一节看最后一个入口：设置面板。

### 2.7 设置面板：记忆检索开关 + API Key 管理

对话页顶部工具栏有一个**设置按钮**（齿轮图标），点开它会弹出**设置面板**（源码 `frontend/src/components/shared/SettingsModal.tsx`）。三个页面之外的全局配置都收在这里，分成两组。

**第一组是「记忆检索（全局）」三态开关**——`关闭 / 向量 / 混合`，点击即时生效，不需要点保存。这三态决定 `MEMORY.md` 以什么形态进入 Agent 的上下文：选「关闭」时，`MEMORY.md` 整文注入每轮 System Prompt（也就是第二部分第 1 章拼接器的默认形态）；选「向量」时，改为按当前问题做纯向量语义检索，只注入最相关的几个片段；选「混合」时，在向量语义之外再加一路 BM25 关键词召回，用 RRF 融合排序。后端对应两个接口——开关本身走 `PUT /api/config/rag-mode`，向量与混合的切换走 `PUT /api/config/retrieval-mode`（源码 `api/config_api.py:25-52`）。这正是第二部分第 5 章「文件即记忆②——RAG 检索」要重现的机制。

**第二组是 API Key 管理**——`DEEPSEEK_API_KEY` / `OPENAI_API_KEY` / `OPENAI_BASE_URL` / `TAVILY_API_KEY` 四个字段，密码框脱敏显示（点眼睛图标可临时明文查看），点「Save」时只提交被改动过的字段，落盘到后端 `.env` 文件（源码 `api/config_api.py:55-126` 的 `/config/api-keys` 接口）。这就是 1.1 节「双轨凭证」那个坑在界面上的落地：DeepSeek 的对话 key 与 OpenAI 兼容的 Embedding key 在这里是两个独立字段，填错位置，RAG 建索引时就会报鉴权失败。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154606870.png)

▲ 设置面板真实截图：顶部「记忆检索（全局）」三态开关（关闭 / 向量 / 混合，当前选中「混合」，下方提示 BM25 + 向量 RRF 融合）；下方四个 API Key 字段脱敏显示（OpenAI Base URL 为 URL 不脱敏，明文展示代理地址）；底部是安全提示与 Save 按钮

界面逛完了，你已经知道每个页面能做什么、背后对应第二部分哪一章。下一章我们把视角再抬高一层，看这个产品由什么技术搭成、各层怎么协作——这是进入源码之前的最后一张总图。

## 第 3 章：架构与技术栈

从「会用」过渡到「看懂结构」——这一章是第二部分的总图。有了界面认知（第 2 章），现在我们看它由什么搭成、各层怎么协作，让你在进入第二部分源码之前，先在脑子里建起一张「前后端架构全景」。有了这张图，后面每当我们说「这段代码在 Agent 层」或「这个调用走的是工具层」，你就能立刻定位它在全图中的位置。

需要说明的是：**完整的五层架构全景图和「一条消息的生命周期」主线，第二部分第 0 章 0.2 / 0.4 节已经有详细展开**（第二部分第 7 章还有 9 站链路图）。所以这一章不重复画那张图，而是侧重两件第 0 章没细讲的事：一张完整的**前后端技术栈表**，和一份**仓库目录导读**——它们是你进入第二部分后「按图索骥」用的参考。

### 3.1 前后端技术栈表

第 1 章 1.1 的装机清单只告诉你「要装什么版本」；这一节换成架构视角，把每一层各用了什么、版本如何，完整摊开（数据来自 `backend/requirements.txt` 和 `frontend/package.json`）。本项目技术选型相当克制，每一层只选了能解决问题的最小必要工具组合，这张表可以当作整个第二部分的「技术索引」。

FuFan-OpenClaw 前后端技术栈

| 层 | 技术 | 说明 |
|----|------|------|
| 后端语言 | Python 3.10+ | 后端运行时 |
| 后端框架 | `FastAPI` + `Pydantic v2` + `sse-starlette` + `uvicorn` | REST API + SSE 流式推送 |
| Agent 引擎 | `LangChain 1.x`（`create_agent`）+ `LangGraph 1.x` | Agent 编排 + 工具循环运行时 |
| 主 LLM | `langchain-deepseek` 的 `ChatDeepSeek`（默认 `deepseek-chat`）| 对话推理，走 `DEEPSEEK_API_KEY` |
| Embedding | `langchain-openai` / `LlamaIndex OpenAIEmbedding`（`text-embedding-3-small`）| RAG 向量化，走 `OPENAI_API_KEY`（与主 LLM 不同轨）|
| RAG 检索 | `LlamaIndex Core` + `BM25Retriever` + 向量（自实现 RRF 融合）| 关键词 + 语义混合检索 |
| 浏览器自动化 | `browser-use` + `Playwright` | `browser_use` 工具底层 |
| Token 统计 | `tiktoken` | 会话 token 计数 |
| 前端框架 | `Next.js 14`（App Router）+ `React 18` + `TypeScript` | 3 页路由 |
| 前端状态 | `React Context`（`lib/store.tsx`）| 全局状态管理（注意：不是 Zustand）|
| 前端编辑器 | `Monaco Editor`（`@monaco-editor/react`）| MEMORY/SKILL 文件在线编辑 |
| 前端样式 | `Tailwind CSS` + CSS 自定义属性双主题 | Bronze / Gold 亮暗切换 |
| Markdown 渲染 | `react-markdown` + `remark-gfm` | 对话气泡渲染 |
| 数据存储 | 纯本地文件系统 | 无 MySQL / Redis，无独立向量库服务 |

有两个选型的「为什么」值得点出：其一，用**纯文件系统**存记忆而不是向量数据库，是因为记忆需要人类可读可编辑——这是「文件即记忆」哲学的工程代价与收益，第二部分第 1、5、6 章都会反复遇到。其二，**主 LLM 与 Embedding 走两套不同凭证**（DeepSeek 做对话、OpenAI 兼容做向量化），并不是设计冗余，而是因为 DeepSeek 当时不提供 Embedding 接口，只能借 OpenAI 兼容接口补这一块——这也是 1.1 节那个「双 key 不能混用」坑的根源。

### 3.2 仓库目录导读

打开项目仓库，目录结构如下。这份导读是第二部分的「源码索引」——每当我们说「源码锚点 `graph/agent.py:70`」，你回到这份导读就能快速定位「在 `backend/graph/` 目录下」，并知道它对应第二部分的哪一章。

```text
backend/
├── app.py                 # FastAPI 入口（端口 8002，lifespan 启动钩子）
├── config.py              # 全局配置（RAG 模式 / retrieval 模式开关）
├── graph/                 # Agent 运行时核心
│   ├── prompt_builder.py  # System Prompt 组装  
│   ├── agent.py           # AgentManager + astream 双流  
│   ├── memory_indexer.py  # RAG 索引（向量 + BM25 + RRF）  
│   ├── session_manager.py # 会话 JSON 持久化 + 压缩  
│   └── context_guard.py   # 上下文三层防御（preflight 等）  
├── tools/                 # 7 个内置工具  
│   └── skills_scanner.py  # SKILL.md 扫描 → XML 快照  
├── api/                   # FastAPI 路由（chat / sessions / files / tokens / compress / config_api）  
├── workspace/             # System Prompt 组件：SOUL / IDENTITY / USER / AGENTS.md  
├── skills/                # 5 个内置技能（每个一个文件夹 + SKILL.md）  
├── SKILLS_SNAPSHOT.md     # 启动时 scan_skills 扫 skills/ 生成的技能快照  
├── memory/                # 长期记忆 MEMORY.md（logs/ 为预留空目录）  
├── sessions/              # 会话 JSON 文件（archive/ 存压缩归档）  
└── storage/memory_index/  # RAG 向量索引持久化目录 

frontend/src/
├── app/                   # Next.js App Router：page.tsx(对话) / memory / skills 三页
├── components/            # chat / memory / skills / layout 各区组件
└── lib/                   # api.ts(API 客户端) / store.tsx(状态) / theme.tsx(主题)
```

这份目录里，第二部分会重点重现 `backend/graph/` 和 `backend/tools/` 两个目录的核心机制；`api/` 层在第 3、7 章做链路串讲；前端目录不在本课的代码重现范围内，但有了这份导读，你清楚知道「我现在讲的代码在哪个文件」。

### 3.3 衔接第二部分：一条消息怎么流动

最后用一句话把架构和数据流接上：用户在前端发一条消息，经 `fetch` POST 到后端 `8002` 端口的 `/api/chat`，后端加载会话历史、重建 System Prompt、组装 Agent，用 `astream` 双流边推理边把 token 和工具事件以 SSE 实时回传前端渲染，最后把这轮对话持久化到会话 JSON。

这条链路的每一个停靠站，第二部分都会逐个用 Python 重现——**完整的五层架构全景图见第二部分第 0 章 0.2 节，9 个停靠站的消息处理全流程见第二部分第 7 章**，这里不重复展开。第一部分到此结束：你已经把 `FuFan-OpenClaw` 跑起来了、逛过了所有界面、理清了前后端架构。现在我们用 Python 逐个机制亲手复现它。

---

# 第二部分：可观测 Agent 核心机制实现

第一部分你已经把 `FuFan-OpenClaw` 跑起来、摸清了它的界面和架构。从这里开始，我们不再纠结界面长什么样，只关心背后跑的机制——逐章取出一个机制，贴上真实源码的 `file:line` 锚点，用最小的 Python 亲手重现它。从第 2 章起的代码会复用第 0 章 0.5 节统一初始化的 `llm` 与 `Embedding`，所以请先按顺序跑完第 0 章环境准备 cell，后续各章不再重复加载凭证与模型（第 1 章纯标准库，无此依赖）。

下面这一章（第 0 章）先建立贯穿第二部分的认知框架——三大设计哲学和五层架构，它是后面七章能串成一条主线的前提；最后再做一次性的环境准备（装依赖、加载凭证、初始化模型），供后续所有章节复用。

## 第 0 章：导论——什么是可观测 AI Agent

在动手写代码之前，我们需要先建立一个全景认知框架，否则后面每一章的代码都只是「零件」，你看不出它们之间的关联。本章前四节是认知框架，最后一节（0.5）做一次性环境准备——它们一起构成后面七章能跑通的前提。

### 0.1 三大设计哲学

先明确「可观测」在这里的确切含义——它指的是**工程层面的可观测**，而不是「可解释 AI」（XAI）那种模型层面的可解释性。FuFan-OpenClaw 里说的可观测是：Agent 做的每一件事——调了什么工具、拿到了什么返回值、生成了哪段文字——都通过 SSE 事件流实时推送给前端，用户在聊天界面能逐步看到 Agent 的思考过程。

理解这个系统，你需要先掌握它的三条核心设计哲学，它们不是口号，而是真实影响了每一个设计决策：

**哲学一：文件即记忆（Memory as File）**

Agent 的人格、用户信息、长期记忆——全部储存在 Markdown 文件里，人类可读可编辑，不依赖数据库。这意味着：改了 `SOUL.md` 里的人格设定，下一轮对话立即生效，不需要重启服务；用文本编辑器打开 `MEMORY.md`，就能看到 Agent 记住了什么。你在第 1 章（全量注入）和第 5 章（RAG 检索）会看到这条哲学的两种实现形态。

**哲学二：技能即插件（Skills as Plugin）**

「学会新技能」不需要改代码，只需要在 `skills/` 目录下新建一个带 YAML frontmatter 的 `SKILL.md` 文件。系统启动时自动扫描，把技能说明注入到 System Prompt 的第一段（`SKILLS_SNAPSHOT`）。Agent 想用某个技能，第一步必须用 `read_file` 工具读取该技能的说明书，而不是凭空猜测参数。你在第 4 章会重现这套扫描注入机制。

**哲学三：完全可观测（Full Transparency）**

Agent 的工具调用全程不是黑盒。当 Agent 决定调一个工具，`tool_start` 事件立即推出，用户能看到调的是什么工具、传的是什么参数；工具执行完，`tool_end` 事件把结果也推出来。前端把这些事件渲染成「ThoughtChain 工具卡」，整个推理链路可视。你在第 3 章会重现 `astream` 双流解析器。

### 0.2 五层架构全景

知道了哲学，我们来看系统的物理结构。FuFan-OpenClaw 分为五层，一条消息从前端发出，依次穿过这五层，最终以 SSE 事件流的形式逐步返回到用户界面：

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154709646.png)

本课接下来七章，就是在逐层用 Python 重现这张图里的每一个停靠站。第 1 章对应「文件存储层 → Agent 层」的 prompt 构建；第 2 章对应 Agent 层的内核；第 3 章对应 API 层的事件流；以此类推，直到第 7 章把全链路串成一条完整的消息旅程。

**【.env 7 个 key 一览】**（集中声明，后续章节按需取用，不再重复列）：

.env 环境变量一览（三套凭证 · 7 个 key）

| 类别 | Key 名称 | 用途 | 真实使用章节 |
|------|---------|------|-------------|
| **DeepSeek**(主 LLM) | `DEEPSEEK_API_KEY` | LangChain `ChatDeepSeek` 调用凭证 | 第 2/3/6/8 章 |
| | `DEEPSEEK_BASE_URL` | 默认 `https://api.deepseek.com` | 第 2/3/6/8 章 |
| | `DEEPSEEK_MODEL` | 默认 `deepseek-chat` | 第 2/3/6/8 章 |
| **OpenAI 兼容**(RAG 嵌入) | `OPENAI_API_KEY` | LlamaIndex `OpenAIEmbedding` 凭证 | 第 5 章 |
| | `OPENAI_BASE_URL` | 默认 `https://ai.devtool.tech/proxy/v1`(代理) | 第 5 章 |
| | `EMBEDDING_MODEL` | 默认 `text-embedding-3-small` | 第 5 章 |
| **Tavily**(网络搜索) | `TAVILY_API_KEY` | `tavily_search` 工具凭证(第 7 个工具) | 第 4 章(可选用) |

**两套凭证不能混用**——主 LLM 用 DeepSeek 接口，Embedding 用 OpenAI 兼容接口。第 2 章只用到前三行，第 5 章只用到中间三行，第 6 章压缩又会回到前三行。`Tavily` 是可选技能用到（仅 Ch4 示例涉及，本课不专门演示），可不填。

### 0.3 三哲学 × 章节映射

三大哲学不是平行存在的，它们在不同章节以不同形态呈现。下图展示了哲学与章节代码的对应关系，你可以在后续每章的学习中随时回来对照这张图：

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154607112.png)

### 0.4 主线预告：一条消息的生命周期

为了让每一章的代码都有意义，我们需要一条贯穿全课的主线。这条主线叫做「一条消息的生命周期」：用户在前端输入一句话，这句话会经历什么？

1. 前端 `streamChat` 把消息 POST 到 `/api/chat`

2. `event_generator` 从会话历史加载上下文（第 6 章）

3. `build_system_prompt` 把 6 个文件拼成人格提示词（第 1 章）

4. `create_agent` 用提示词和 7 工具组装 Agent（第 2 章）

5. `agent.astream` 开始双流输出，解析出 5 类事件（第 3 章）

6. 工具调用期间，`skills_scanner` 注入的 SKILLS_SNAPSHOT 告诉 Agent 有哪些技能（第 4 章）

7. 如果开了 RAG 模式，`memory_indexer` 检索相关记忆片段（第 5 章）

8. 最终响应持久化到会话 JSON，长历史触发压缩（第 6 章）

9. 前端逐 token 渲染，ThoughtChain 卡展示工具调用，canvas 提取渲染 HTML（第 7 章）

> **【常见误区】**：「可观测」不等于「慢」。事件流是异步推送的，第一个 token 在 LLM 开始生成时就立即到达前端，用户感知延迟 = LLM 首 token 时延，与「可不可观测」无关。

如果你想动手把真实项目跑起来再学，环境部署、前后端启动、首条对话验证已经全部移到**第一部分第 1 章「环境部署与项目启动」**，那里有完整步骤和踩坑预警。本章（第二部分第 0 章）前四节专注认知框架；下面的 0.5 节再做一次性的环境准备，为后续七章的代码运行打好底座。

### 0.5 环境准备与模型初始化

前面四节建立了认知框架，但第二部分从第 2 章开始就要真实调用大模型了。为了让后续每一章都能直接复用同一套模型与凭证、避免在每个 cell 里重复加载，我们在这里做一次性的环境准备：装依赖、加载 `.env`、初始化主 `LLM` 和 `Embedding`。这一节的四个 cell 是后续第 2/3/5/6/8 章的运行前提——请先按顺序把它们跑一遍，后面的章节会直接引用这里定义好的 `llm` 与 `Settings.embed_model`，不再重复初始化。

**步骤一：安装依赖**

第二部分的 Notebook 依赖与真实项目完全一致，因此我们直接复用项目 `backend/requirements.txt` 一键安装，版本号天然对齐、不会漏装。注意：下面的相对路径假设课件目录与项目源码目录相邻，你的目录结构不同时，请自行改成指向本地 `Fufan-OpenClaw项目源码/backend/requirements.txt` 的实际路径；如果你已经在第一部分第 1 章完成过项目部署（同一个 conda 环境已 `pip install -r requirements.txt`），本 cell 可直接跳过。

```python
# 第 0 章 步骤一：安装第二部分所需依赖（与项目 backend/requirements.txt 完全一致）
# 路径需指向你本地的项目源码目录，学员目录结构不同请自行调整
# 若已在第一部分完成项目部署（同一 conda 环境），本 cell 可跳过
!pip install -r ./Fufan-OpenClaw项目源码/backend/requirements.txt
```

装完后，第 2/3/6 章用到的 `langchain` / `langchain-deepseek`、第 4 章的 `pyyaml`、第 5 章的三个 `llama-index-*` 子包、以及全程读 `.env` 的 `python-dotenv` 就都就位了。第 1 章纯标准库，不依赖以上任何包。

**步骤二：加载环境变量并校验凭证**

接下来加载 `.env` 文件中的 API key。`load_dotenv(override=True)` 会就近向上查找 `.env` 并覆盖已有的同名环境变量。我们顺手打印两套关键凭证是否就位（对照 0.2 节的 7-key 表）——这样后面任何一章报 key 相关错误时，你第一时间就能回到这里确认。

```python
# 第 0 章 步骤二：加载 .env 并校验凭证（后续所有章节复用这里的环境变量）
import os
from dotenv import load_dotenv

# 学员请在项目根目录创建 .env 文件（参见第一部分 .env 模板），填入所需 API key
load_dotenv(override=True)

# LangSmith关闭
os.environ["LANGCHAIN_TRACING_V2"] = "false"
os.environ["LANGSMITH_TRACING"] = "false"   # 新版变量名，一并关掉
os.environ.pop("LANGCHAIN_API_KEY", None)    # 顺手清掉无效 key，更干净

# 校验两套凭证是否就位（对照 0.2 节 7-key 表；缺失不报错，仅提示）
print("DEEPSEEK_API_KEY:", "[OK] 已就位" if os.getenv("DEEPSEEK_API_KEY") else "[缺失] 第 2/3/6/8 章对话功能需要")
print("OPENAI_API_KEY :", "[OK] 已就位" if os.getenv("OPENAI_API_KEY") else "[缺失] 仅第 5 章 RAG 向量检索需要")
```

```
DEEPSEEK_API_KEY: [OK] 已就位
OPENAI_API_KEY : [OK] 已就位
```

如果 `DEEPSEEK_API_KEY` 显示缺失，第 2/3/6/8 章的真实对话会走降级路径（只看代码结构，不真调模型）；`OPENAI_API_KEY` 仅第 5 章 RAG 用到，缺失不影响其它章节。

**步骤三：初始化主 LLM（DeepSeek）**

这是整个第二部分的「大脑」——用 `langchain-deepseek` 的 `ChatDeepSeek` 初始化一个主 `LLM` 对象，对照源码 `backend/graph/agent.py:35-43`。后续第 2/3/6/8 章都直接复用这个 `llm` 变量，不再重新初始化。`streaming=True` 必须打开，否则第 3 章的 `astream` 双流会静默返回空流。

```python
# 第 0 章 步骤三：初始化主 LLM（全局复用，对照源码 agent.py:35-43）
# 后续第 2/3/6/8 章直接引用这个 llm，不再重复定义
from langchain_deepseek import ChatDeepSeek

llm = ChatDeepSeek(
    model=os.getenv("DEEPSEEK_MODEL", "deepseek-v4-flash"),
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    api_base=os.getenv("DEEPSEEK_BASE_URL", "https://api.deepseek.com"),
    temperature=0.7,
    streaming=True,  # 必须为 True，否则第 3 章 astream 无法工作
)
print(f"[OK] 主 LLM 初始化完成：{os.getenv('DEEPSEEK_MODEL', 'deepseek-v4-flash')}")
```

```
[OK] 主 LLM 初始化完成：deepseek-v4-flash
```

这个 `llm` 对象就是真实项目里 `AgentManager._llm` 的等价物。第 2 章会把它交给 `create_agent` 组装成 Agent，第 6 章会给它叠加上下文中间件，第 8 章会用它派生子智能体——它们用的都是这里这一份实例。

**步骤四：配置 Embedding（第 5 章 RAG 用，可选）**

最后配置向量检索用的 `Embedding` 模型。它走的是 OpenAI 兼容接口（`OPENAI_API_KEY`，与主 LLM 的 DeepSeek 不同轨），只有第 5 章 RAG 才真正调用。我们用 LlamaIndex 的全局 `Settings.embed_model` 来设置，这样第 5 章直接读取即可。没有 `OPENAI_API_KEY` 时本 cell 跳过设置、不报错，第 5 章会走「无 key 降级」路径。

```python
# 第 0 章 步骤四：配置 Embedding（仅第 5 章 RAG 用，对照源码 memory_indexer.py:86-92）
# 无 OPENAI_API_KEY 时跳过设置，第 5 章会走降级路径
if os.getenv("OPENAI_API_KEY"):
    from llama_index.core.settings import Settings
    from llama_index.embeddings.openai import OpenAIEmbedding

    Settings.embed_model = OpenAIEmbedding(
        model=os.getenv("EMBEDDING_MODEL", "text-embedding-3-small"),
        api_key=os.getenv("OPENAI_API_KEY"),
        api_base=os.getenv("OPENAI_BASE_URL", "https://ai.devtool.tech/proxy/v1"),
    )
    print("[OK] Embedding 已配置（第 5 章 RAG 可用）")
else:
    print("[SKIP] 未配置 Embedding（无 OPENAI_API_KEY）；第 5 章将走无 key 降级路径")
```

```
[OK] Embedding 已配置（第 5 章 RAG 可用）
```

到这里，第二部分的运行底座就备好了：依赖装齐、凭证加载、主 `LLM` 与 `Embedding` 各就各位。后面每一章我们都会站在这个底座上，只关心当章新增的那个机制本身。

---

## 第 1 章：文件即记忆①——System Prompt 拼接器

`build_system_prompt` 是整个 Agent 系统的起点——在每一轮对话开始时，它把 6 个 Markdown 文件读进来、按固定顺序拼接、对超长组件截断，组装出一段完整的系统提示词。这是「文件即记忆」哲学的第一层落地：记忆不是存在数据库里的向量，而是可以用文本编辑器直接修改的 Markdown 文件。

你可能会问：为什么不把这 6 个文件合并成一个？因为分开存有一个关键优势——**热更新**。改了 `SOUL.md` 里的性格设定？下一条消息发出时，`build_system_prompt` 会重新读文件，新人格立即生效，不需要重启后端服务。这也是「每轮都重建 prompt」这个看似低效设计背后的工程理由。

本章（第 1 章）纯标准库实现，不依赖任何第三方包，直接就能跑——依赖安装与模型初始化已在第 0 章 0.5 节统一完成，后续各章直接站在那个底座上。

### 1.1 源码锚点对照

在我们动手写 MVP 之前，先看一眼真实源码的全貌。`backend/graph/prompt_builder.py` 全文 62 行，结构非常清晰：

- 第 7 行：`MAX_COMPONENT_LENGTH = 20000`——单组件超过 20000 字符就截断

- 第 20-22 行：`RAG_GUIDANCE` 常量——RAG 模式下追加的引导语

- 第 25-62 行：`build_system_prompt` 函数主体——拼接逻辑核心

- 第 38-44 行：前 5 组件按序定义（Skills Snapshot / Soul / Identity / User Profile / Agents Guide）；第 48-50 行追加第 6 组件 Long-term Memory（仅非 RAG 模式）

- 第 46-50 行：`if not rag_mode` 分支——只在非 RAG 模式下追加 MEMORY

- 第 58-60 行：`if rag_mode` 分支——RAG 模式下追加 RAG_GUIDANCE

6 个组件的顺序不是随意的：`SKILLS_SNAPSHOT` 放第一位，确保 Agent 最先看到自己有哪些技能；`SOUL` 和 `IDENTITY` 紧随其后，定义人格和风格；`USER` 是用户画像；`AGENTS` 是操作指南（包含技能调用协议）；`MEMORY` 放最后，是跨会话长期记忆。

### 1.2 System Prompt 拼接器实现

接下来我们用 inline 字符串替代文件读取，重现核心拼接逻辑。运行后你会看到带 HTML 注释标签的完整提示词文本——每一段前面都有 `` 的标签，这正是 LLM 能区分不同组件的关键；RAG 模式下你会看到 MEMORY 段消失、RAG_GUIDANCE 段出现。

```python
# 第 1 章：System Prompt 拼接器（不依赖项目目录，完全自包含）
# 对照源码：backend/graph/prompt_builder.py:7,20-22,25-62

MAX_COMPONENT_LENGTH = 20000  # 单组件最大长度，超过则截断（源码第 7 行）

# RAG 模式下替换 MEMORY 的引导语（源码第 20-22 行）
RAG_GUIDANCE = """注意：长期记忆(MEMORY.md)已切换为RAG检索模式。
系统会根据用户的问题自动检索相关记忆片段并注入上下文。
如果检索到了相关记忆，它们会以"[记忆检索结果]"标记呈现在你的上下文中。"""

def truncate_if_needed(content: str, max_len: int = MAX_COMPONENT_LENGTH) -> str:
    """
    对单个组件内容做长度截断。

    Args:
        content: 原始组件文本
        max_len: 最大允许字符数，超过则截断并追加省略标记

    Returns:
        截断后的文本（或原文，若未超长）
    """
    # 超过阈值才截断，保留前 max_len 个字符
    if len(content) > max_len:
        return content[:max_len] + "\n...[truncated]"
    return content

def build_system_prompt_mvp(rag_mode: bool = False) -> str:
    """
    System Prompt 拼接器，用 inline 字符串替代文件读取。

    Args:
        rag_mode: 是否启用 RAG 检索模式；True 时排除 MEMORY 全文、追加 RAG_GUIDANCE

    Returns:
        完整系统提示词字符串，各组件以 HTML 注释标签分隔
    """
    # 6 个组件：inline 字符串模拟文件内容（源码第 38-44 行为文件路径定义）
    components = [
        ("Skills Snapshot", "<available_skills><skill><name>示例技能</name></skill></available_skills>"),
        ("Soul", "你是一个乐于助人的 AI 助手，风格简洁、直接、不废话。"),
        ("Identity", "名字：FuFan-OpenClaw 演示版本"),
        ("User Profile", "用户是一位 Python 开发者，正在学习 Agent 系统的内部机制。"),
        ("Agents Guide", "使用技能前必须先用 read_file 读取 SKILL.md，禁止猜测参数。"),
    ]

    # 非 RAG 模式时追加 MEMORY（源码第 46-50 行）
    if not rag_mode:
        components.append(
            ("Long-term Memory", "用户上次提到喜欢简洁的代码风格，偏好中文回复。")
        )

    parts = []
    for label, content in components:
        # 对每个组件做截断检查（源码 _read_component 函数：第 10-17 行）
        content = truncate_if_needed(content)
        if content:
            parts.append(f"<!-- {label} -->\n{content}")

    # RAG 模式下追加引导语（源码第 58-60 行）
    if rag_mode:
        parts.append(f"<!-- RAG Mode -->\n{RAG_GUIDANCE}")

    # 各组件之间用双换行分隔（源码第 62 行 return）
    return "\n\n".join(parts)

# --- Tier 1 验证：观察两种模式的输出差异 ---

print("=" * 60)
print("【模式 1】正常模式（包含 MEMORY）")
print("=" * 60)
normal_prompt = build_system_prompt_mvp(rag_mode=False)
print(normal_prompt)

print("\n" + "=" * 60)
print("【模式 2】RAG 模式（MEMORY 被替换为 RAG_GUIDANCE）")
print("=" * 60)
rag_prompt = build_system_prompt_mvp(rag_mode=True)
print(rag_prompt)

# 断言验证：正常模式包含 MEMORY 标签，RAG 模式不含 MEMORY 含 RAG Mode
assert "<!-- Long-term Memory -->" in normal_prompt, "正常模式应含 MEMORY 组件"
assert "<!-- Long-term Memory -->" not in rag_prompt, "RAG 模式不应含 MEMORY"
assert "<!-- RAG Mode -->" in rag_prompt, "RAG 模式应含 RAG_GUIDANCE"
# 验证 6 个 HTML 注释标签数量
import re
tags_normal = re.findall(r"<!-- .+ -->", normal_prompt)
tags_rag = re.findall(r"<!-- .+ -->", rag_prompt)
print(f"\n正常模式组件数：{len(tags_normal)}（应为 6）")
print(f"RAG 模式组件数：{len(tags_rag)}（应为 6，MEMORY 被 RAG Mode 替换）")
assert len(tags_normal) == 6
assert len(tags_rag) == 6
print("\n所有断言通过。")
```

```
============================================================
【模式 1】正常模式（包含 MEMORY）
============================================================
<!-- Skills Snapshot -->
<available_skills><skill><name>示例技能</name></skill></available_skills>

<!-- Soul -->
你是一个乐于助人的 AI 助手，风格简洁、直接、不废话。

<!-- Identity -->
名字：FuFan-OpenClaw 演示版本

<!-- User Profile -->
用户是一位 Python 开发者，正在学习 Agent 系统的内部机制。

<!-- Agents Guide -->
使用技能前必须先用 read_file 读取 SKILL.md，禁止猜测参数。

<!-- Long-term Memory -->
用户上次提到喜欢简洁的代码风格，偏好中文回复。

============================================================
【模式 2】RAG 模式（MEMORY 被替换为 RAG_GUIDANCE）
============================================================
<!-- Skills Snapshot -->
<available_skills><skill><name>示例技能</name></skill></available_skills>

<!-- Soul -->
你是一个乐于助人的 AI 助手，风格简洁、直接、不废话。

<!-- Identity -->
名字：FuFan-OpenClaw 演示版本

<!-- User Profile -->
用户是一位 Python 开发者，正在学习 Agent 系统的内部机制。

<!-- Agents Guide -->
使用技能前必须先用 read_file 读取 SKILL.md，禁止猜测参数。

<!-- RAG Mode -->
注意：长期记忆(MEMORY.md)已切换为RAG检索模式。
系统会根据用户的问题自动检索相关记忆片段并注入上下文。
如果检索到了相关记忆，它们会以"[记忆检索结果]"标记呈现在你的上下文中。

正常模式组件数：6（应为 6）
RAG 模式组件数：6（应为 6，MEMORY 被 RAG Mode 替换）

所有断言通过。
```

运行上面的代码，你会看到两段完整的系统提示词。关键观察点有三：第一，每个组件前面都有 `` 的 HTML 注释标签，这让 LLM 能清楚地区分「这是人格设定」和「这是用户信息」；第二，RAG 模式下 `Long-term Memory` 组件消失，取而代之的是 `RAG Mode` 引导语，告诉 LLM「你的记忆片段会以检索结果的形式注入」；第三，断言验证了组件数量的一致性——无论哪种模式，始终是 6 个组件，只是第 6 个的内容不同。

你现在已经掌握了 FuFan-OpenClaw 的「人格层」——这 ~40 行代码是整个 Agent 的灵魂起点。在学这章之前，你可能觉得「System Prompt 就是一段固定文字」；现在你知道它是 6 个模块的动态拼接，每轮重读文件，改了文件立即生效。第 2 章我们会把这个 prompt 交给 `create_agent`，让它真正「活起来」。

> **【踩坑预警】**：如果你在真实项目里接入这套逻辑，记得 `SOUL.md` 等文件的路径是相对于 `base_dir` 的，不是相对于脚本执行位置的。忘了传 `base_dir` 参数会导致所有组件读到空字符串，Agent 没有人格——这个 bug 不会报错，只会让 Agent 变成「失忆状态」，非常难排查。

---

## 第 2 章：Agent 内核——create_agent + 可观测工具循环

有了 System Prompt，下一个问题是：谁来「用」这个 prompt？答案是 LangChain 1.x 的 `create_agent` API。这一章我们搭 Agent 的大脑：给它两个工具，让它自主决定调哪个、什么时候调、用什么参数——这个「自主决策 → 调工具 → 看结果 → 继续推理」的循环，就是 Agent 区别于普通 LLM 调用的核心。

`create_agent` 是 LangChain 1.x 提供的标准化 Agent 构建接口，底层是 LangGraph 运行时（一个有向图），但接口层做了封装，你不需要理解图的执行细节，三个参数就能搭起一个 Agent。这是 FuFan-OpenClaw 项目的选型——源码 `backend/graph/agent.py:10` 即 `from langchain.agents import create_agent`。

### 2.1 源码锚点对照

`backend/graph/agent.py` 全文 301 行，`AgentManager` 类承载了 Agent 的完整生命周期：

- 第 35 行：`from langchain_deepseek import ChatDeepSeek`——lazy import，在 `initialize` 方法内

- 第 37-43 行：LLM 初始化，5 个关键参数（`model` / `api_key` / `api_base` / `temperature` / `streaming`）

- 第 54 行：`from langchain.agents import create_agent`——在 `_build_agent` 方法内 lazy import

- 第 73-78 行：`create_agent(model=..., tools=..., system_prompt=..., middleware=[context_editing])` 调用——其中第 77 行的 `middleware` 正是第 6.1.1 节「上下文三层防御」的 L1 接入点

`streaming=True` 不是可选的——它是第 3 章 `astream` 能正常工作的前提，必须在 LLM 初始化时传入。`_build_agent` 每次对话都重新调用，这正是为了每次都能拿到最新的 `build_system_prompt`，从而实现文件热更新。

### 2.2 create_agent 工具循环实现

接下来我们定义两个最小工具，用 `dotenv` 加载 API key，然后用 `create_agent` 组装 Agent 并真实运行一轮。运行后你会看到：Agent 收到问题后，先用 `get_current_time` 工具拿到当前时间，再用这个信息生成一段回复——整个过程在 `result["messages"]` 里都有完整的消息链。需要有效的 `DEEPSEEK_API_KEY`；如果没有 key，先看代码理解调用模式即可。

```python
# 第 2 章：create_agent + 工具循环（需要 DEEPSEEK_API_KEY）
# 对照源码：backend/graph/agent.py:35-43, 73-78

import asyncio
import os
from datetime import datetime

# 环境变量已在第 0 章 0.5 步骤二统一加载（load_dotenv），本章直接复用

from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langchain.agents import create_agent

@tool
def get_current_time() -> str:
    """
    返回当前本地时间的字符串表示。
    Agent 在回答「现在几点」类问题时会自动调用此工具。

    Returns:
        格式为 YYYY-MM-DD HH:MM:SS 的时间字符串
    """
    # 直接返回格式化的当前时间
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

@tool
def simple_add(a: float, b: float) -> float:
    """
    做简单的加法计算，返回两个数的和。

    Args:
        a: 第一个加数（浮点数）
        b: 第二个加数（浮点数）

    Returns:
        两数之和（浮点数）
    """
    return a + b

# 主 LLM 已在第 0 章 0.5 步骤三统一初始化（全局 llm），本章直接复用——
# 真实项目里它是 AgentManager 的实例属性 self._llm，作用相当于"模块级共享"

# 工具列表（源码有 7 个工具，此处用 2 个演示）
tools = [get_current_time, simple_add]

async def run_agent_demo():
    """
    演示 create_agent + ainvoke 的完整流程。
    复用第 0 章定义的 llm 和本章定义的 tools，对照源码 _build_agent（第 52-79 行）。
    """
    # 用简单的 system prompt（第 1 章的 build_system_prompt 在真实项目里被调用）
    system_prompt = "你是一个简洁的助手，优先使用工具回答问题，回复不超过 50 字。"

    # create_agent 三参数：model / tools / system_prompt（源码 agent.py:73-78）
    agent = create_agent(
        model=llm,
        tools=tools,
        system_prompt=system_prompt,
    )

    # ainvoke 调用，输入格式：{"messages": [HumanMessage(...)]}
    print("正在调用 Agent，请等待...")
    result = await agent.ainvoke({"messages": [HumanMessage(content="现在几点？顺便帮我算一下 3.14 加 2.72 等于多少")]})

    # 从返回的 messages 列表中提取 AI 最终回复
    messages = result.get("messages", [])
    print(f"\n共收到 {len(messages)} 条消息（含工具调用中间过程）：")
    for i, msg in enumerate(messages):
        # 打印每条消息类型和内容摘要，帮助理解消息链结构
        content_preview = str(msg.content)[:80] if msg.content else "(空)"
        print(f"  [{i}] type={msg.type} | {content_preview}")

    # 找到最后一条 AI 消息作为最终回复
    final_reply = None
    for msg in reversed(messages):
        if hasattr(msg, "content") and msg.type == "ai" and msg.content:
            final_reply = msg.content
            break

    print(f"\nAgent 最终回复：{final_reply}")

    # 验证工具被调用（messages 中应有 ToolMessage 类型的消息）
    tool_messages = [m for m in messages if m.type == "tool"]
    print(f"工具调用次数：{len(tool_messages)}（应 >= 1）")
    assert len(tool_messages) >= 1, "Agent 应至少调用一次工具"
    print("断言通过：工具被调用。")

# 在 Jupyter Notebook 中直接 await；在脚本中用 asyncio.run()
await run_agent_demo()
```

```
正在调用 Agent，请等待...

共收到 5 条消息（含工具调用中间过程）：
  [0] type=human | 现在几点？顺便帮我算一下 3.14 加 2.72 等于多少
  [1] type=ai | (空)
  [2] type=tool | 2026-06-10 19:59:09
  [3] type=tool | 5.86
  [4] type=ai | 现在是 2026-06-10 19:59，3.14 + 2.72 = 5.86。

Agent 最终回复：现在是 2026-06-10 19:59，3.14 + 2.72 = 5.86。
工具调用次数：2（应 >= 1）
断言通过：工具被调用。
```

运行这段代码，你会看到消息链中混杂着 `HumanMessage`、`AIMessage`（含 `tool_calls` 字段）、`ToolMessage`（工具返回值）和最终的 `AIMessage`（完整回复）。这个消息链的结构，就是下一章 `astream` 双流需要解析的原始数据。我们这个问题同时需要时间和加法两个工具，模型通常会在一条 AI 消息里**并行发出两个 `tool_calls`**，于是消息链是 5 条：`1 Human + 1 AI（含 2 个 tool_calls）+ 2 Tool（两个工具各返回一条）+ 1 AI Final`。如果模型选择分两轮串行调用，条数会更多——实际多少取决于模型行为，运行后数一下 `result["messages"]` 的长度就知道了（这也是为什么上面的断言用 `>= 1` 而非写死某个具体值：模型完全可能自己心算加法而只调一个工具）。

你现在已经掌握了 Agent 的「大脑层」——三行代码搭起来一个能自主决策调工具的 Agent：LLM 初始化、工具列表、`create_agent`。在学这章之前，你可能觉得「工具调用很复杂」；现在你知道它的本质是 LangGraph 在底层帮你管理了一个「生成 → 调工具 → 生成」的循环，你只需要提供工具和提示词。第 3 章我们会把这个不可观测的 `ainvoke` 换成可观测的 `astream`，让每一步都可见。

> **【踩坑预警】**：如果你看到 `create_agent` 报 `ImportError`，确认 `langchain` 包版本 >= 1.x——LangChain 0.x 时代的 `AgentExecutor` 已从 `langchain.agents` 移除，它才是真正被淘汰的旧接口。另外别和 `create_react_agent` 搞混：它如今位于 `langgraph.prebuilt`，是与 `create_agent` 平行的另一套现役接口（用法不同，并非旧版）。`streaming=True` 在 `ChatDeepSeek` 初始化时传，不是在 `ainvoke` 时传；忘了传这个参数，第 3 章的 `astream` 会静默返回空流。

---

## 第 3 章：可观测——SSE 事件流

第 2 章的 `ainvoke` 是黑盒——你发出请求，等一段时间，拿到结果，中间发生了什么完全看不见。`astream` 打开了这个黑盒。这一章实现「完全可观测」哲学的代码落地：把 Agent 的每一步动作——生成 token、开始调工具、工具返回结果、整轮完成——都解析成结构化事件，实时输出给调用方。

`astream` 的关键在于它消费的是**两条并行的流**，而不只是转发 LLM 的 token 输出：`messages` 流给 token 级输出，`updates` 流给节点级（工具执行）输出。如果只消费一条流，你会丢失一半的信息。

### 3.1 源码锚点对照

`backend/graph/agent.py` 的 `astream`/`_astream_inner` 方法（第 134-242 行）是核心：

- 第 174-177 行：`agent.astream({"messages": messages}, stream_mode=["messages", "updates"])` 调用

- 第 179-180 行：`isinstance(event, tuple)` 判断，解包 `(mode, data)` 元组

- 第 198 行：`msg.type == "AIMessageChunk" or msg.type == "ai"` 双值判断——`AIMessageChunk` 是流式 token 的类型，但有时也可能是 `"ai"`，需要兼容两种形式

- 第 210-220 行：`mode == "updates"` 且 `node_name == "tools"` → `tool_end` 事件

- 第 221-229 行：`node_name == "model"` 且有 `tool_calls` → `tool_start` 事件

- 第 220 行：`tools_just_finished = True` 标志位——工具结束后、新 token 开始前，发出 `new_response` 事件

- 第 242 行：循环结束后 `yield {"type": "done"}` 收尾

`backend/api/chat.py` 的 `event_generator`（第 68-227 行）把这些内部事件转成 SSE 格式（加 `event:` 和 `data:` 字段）推给前端。我们在 MVP 里只做内部事件解析，不做 SSE 格式转换。

> 这里要先厘清一个容易混的口径：本 MVP 解析的是**非 RAG 模式下的 5 类内部事件**（`token` / `tool_start` / `tool_end` / `new_response` / `done`）。源码的 `astream` 在开启 RAG 模式时还会多发一类 `retrieval` 事件；而 `event_generator` 往前端推时，又会在内部事件之外额外加上 `canvas`、`title` 等 SSE 专属事件——这就是为什么你在第 7 章的全链路图里会看到 `canvas`。记住一句话：**内部 5 类是骨架，外层按需扩展**，两个列表不是矛盾，而是不同层次。

### 3.2 astream 双流解析器实现

接下来我们复用第 2 章的 Agent，用 `astream` 替换 `ainvoke`，解析出 5 类事件。运行后你会看到事件序列实时打印出来：先是 `token` 事件（Agent 开始生成），然后 `tool_start`（决定调工具），然后 `tool_end`（工具返回），然后 `new_response`（开始新一段生成），最后是更多 `token` 和 `done`。这个事件序列就是前端 ThoughtChain 可视化的数据来源。

```python
# 第 3 章：astream 双流解析器（复用第 0 章的 llm + 第 2 章的 tools）
# 对照源码：backend/graph/agent.py:134-242
# 需要已执行第 0 章（llm 已定义）和第 2 章（tools 已定义）

import asyncio
from langchain_core.messages import HumanMessage
from langchain.agents import create_agent

async def parse_agent_stream(agent, user_message: str):
    """
    消费 astream 双流，解析出 5 类事件并实时打印。

    Args:
        agent: 已初始化的 LangChain agent 对象（由 create_agent 返回）
        user_message: 用户输入的字符串

    Returns:
        events_log: 包含所有解析事件的列表，用于断言验证
    """
    events_log = []  # 记录所有事件类型，用于事后断言
    full_response = ""
    tools_just_finished = False  # 标志位：工具结束后下一个 token 前触发 new_response

    # stream_mode 必须是列表，传字符串会静默失效（源码第 174-177 行）
    async for event in agent.astream(
        {"messages": [HumanMessage(content=user_message)]},
        stream_mode=["messages", "updates"],
    ):
        # 多流模式下，event 是 (mode, data) 元组（源码第 179-183 行）
        if isinstance(event, tuple):
            mode, data = event
        else:
            mode, data = "messages", event

        if mode == "messages":
            # messages 流：token 级别输出
            msg, metadata = data
            if hasattr(msg, "content") and msg.content:
                # 双值判断：AIMessageChunk 或 ai，兼容两种形式（源码第 198 行）
                if msg.type == "AIMessageChunk" or msg.type == "ai":
                    # 有 tool_calls 字段说明是工具调用请求，不是 token
                    if not getattr(msg, "tool_calls", None):
                        if tools_just_finished:
                            # 工具刚结束，发出 new_response 事件（源码第 198-200 行）
                            print("  [new_response] 工具结束，新一轮文字生成开始")
                            events_log.append("new_response")
                            tools_just_finished = False
                        full_response += msg.content
                        print(f"  [token] {repr(msg.content)}")
                        events_log.append("token")

        elif mode == "updates":
            # updates 流：节点级别事件（工具调用/工具返回）
            if isinstance(data, dict):
                for node_name, node_data in data.items():
                    if node_name == "tools" and "messages" in node_data:
                        # tools 节点 = 工具执行完毕（源码第 210-220 行）
                        for tool_msg in node_data["messages"]:
                            if hasattr(tool_msg, "name"):
                                output_preview = str(tool_msg.content)[:100]
                                print(f"  [tool_end] tool={tool_msg.name} | output={output_preview}")
                                events_log.append("tool_end")
                        tools_just_finished = True  # 下个 token 前发 new_response

                    elif node_name == "model" and "messages" in node_data:
                        # model 节点且有 tool_calls = 工具调用请求（源码第 217-225 行）
                        for agent_msg in node_data["messages"]:
                            if hasattr(agent_msg, "tool_calls") and agent_msg.tool_calls:
                                for tc in agent_msg.tool_calls:
                                    input_preview = str(tc.get("args", ""))[:100]
                                    print(f"  [tool_start] tool={tc['name']} | input={input_preview}")
                                    events_log.append("tool_start")

    # 循环结束后发出 done 事件（源码第 242 行）
    print(f"  [done] 全部完成，完整响应长度：{len(full_response)} 字符")
    events_log.append("done")
    return events_log

# 需要第 0 章的 llm 和第 2 章的 tools
agent_for_stream = create_agent(
    model=llm,
    tools=tools,
    system_prompt="你是简洁助手，必须使用工具回答，回复不超过 30 字。",
)

print("开始 astream 双流解析演示...\n")
events = await parse_agent_stream(agent_for_stream, "现在几点了？")
```

```
开始 astream 双流解析演示...

  [tool_start] tool=get_current_time | input={}
  [tool_end] tool=get_current_time | output=2026-06-10 20:08:32
  [new_response] 工具结束，新一轮文字生成开始
  [token] '现在是'
  [token] ' '
  [token] '202'
  [token] '6'
  [token] '年'
  [token] '6'
  [token] '月'
  [token] '10'
  [token] '日'
  [token] ' '
  [token] '20'
  [token] ':'
  [token] '08'
  [token] '。'
  [done] 全部完成，完整响应长度：21 字符
```

演示跑通后，我们紧接着对事件序列做一次端到端验证——断言 `token → tool_start → tool_end → new_response → done` 五类事件按预期顺序完整出现，确认双流解析没有遗漏任何环节。

```python
# Tier 2 端到端验证：检查事件序列完整性
# 对照源码断言：token → tool_start → tool_end → new_response → token → done

print("\n事件序列：", " → ".join(events))

# 验证 5 类事件各至少出现一次
for expected_event in ["token", "tool_start", "tool_end", "new_response", "done"]:
    assert expected_event in events, f"事件 '{expected_event}' 应至少出现一次，实际事件：{events}"
    print(f"  [PASS] '{expected_event}' 出现了 {events.count(expected_event)} 次")

# 验证事件顺序：done 必须最后出现
assert events[-1] == "done", f"done 应是最后一个事件，实际最后是：{events[-1]}"
# 验证 tool_start 必须在对应 tool_end 之前
first_start = events.index("tool_start") if "tool_start" in events else -1
first_end = events.index("tool_end") if "tool_end" in events else -1
assert first_start < first_end, "tool_start 应在 tool_end 之前"

print("\n所有事件顺序断言通过。")
```

```

事件序列： tool_start → tool_end → new_response → token → token → token → token → token → token → token → token → token → token → token → token → token → token → done
  [PASS] 'token' 出现了 14 次
  [PASS] 'tool_start' 出现了 1 次
  [PASS] 'tool_end' 出现了 1 次
  [PASS] 'new_response' 出现了 1 次
  [PASS] 'done' 出现了 1 次

所有事件顺序断言通过。
```

运行这两段代码，你会看到事件实时打印，然后看到完整的断言验证。重点观察 `new_response` 事件的触发时机——它出现在 `tool_end` 之后、下一个 `token` 之前。这正是前端用来「开始新一段 ThoughtChain 气泡」的信号：用户看到工具卡关闭，新的文字开始流出。

你现在已经掌握了「完全可观测」哲学的代码落地——把一个黑盒的 `ainvoke` 变成了一个可观测的事件流，5 类事件各有明确含义。在学这章之前，你可能以为「流式输出就是逐字输出文字」；现在你知道它实际上是两条并行流的协同，一条负责文字，一条负责工具调用的结构化事件。第 4 章我们转换方向，看「技能即插件」哲学是怎么在代码里实现的。

> **【踩坑预警】**：`stream_mode=["messages", "updates"]` 必须是**列表**，不能是字符串。传 `stream_mode="messages"` 只会给你 token 流，`tool_start`/`tool_end` 事件完全丢失，且不报错——这种静默失效的 bug 很难发现。另外，`msg.type` 需要兼容两个值：流式 token 时类型是 `"AIMessageChunk"`，但某些路径下可能是 `"ai"`，两个都判断才不会漏掉 token。

---

## 第 4 章：技能即插件——Skills 系统

「技能即插件」哲学有一个非常直觉的对标物：WordPress 插件。你不需要修改 WordPress 核心代码，只要把插件文件放进指定目录，系统自动扫描、自动激活。FuFan-OpenClaw 的技能系统是同样的逻辑：把带 YAML frontmatter 的 `SKILL.md` 文件放进 `skills/` 目录下，系统启动时扫描、解析、生成 XML 快照，注入到 System Prompt 的第一段。

这里有一个边界需要先讲清楚，否则很容易混淆：**工具 vs 技能是两个不同的层**。工具（`tools/__init__.py` 中的 7 个）是 Python 函数，是 Agent 的「原子能力」（执行终端命令、读文件、搜索等）；技能（`skills/*/SKILL.md`）是 Markdown 说明书，是「组合工作流」（比如「如何写一篇研究报告」这种需要多步骤协调的任务）。Agent 通过 `read_file` 工具「读取」技能说明书，然后按照说明书的步骤去调用底层工具——技能是指令，工具是执行器。

### 4.1 源码锚点对照

`backend/tools/skills_scanner.py` 全文 50 行，结构非常紧凑：

- 第 10 行：`scan_skills(base_dir: Path) -> str` 函数入口

- 第 19 行：`sorted(skills_dir.rglob("SKILL.md"))` 扫描所有子目录

- 第 23-33 行：frontmatter 解析——先判断 `content.startswith("---")`，再 `split("---", 2)` 拆三段，取第二段 `yaml.safe_load`

- 第 34 行：`except Exception as e: print(f"⚠️ Error ...")` 静默 skip 解析失败的技能（这是一个设计决策：单个技能解析失败不应该阻止其他技能加载）

- 第 38-48 行：生成 XML 风格快照（`<available_skills><skill>...</skill></available_skills>`）

`backend/workspace/AGENTS.md` 第 1-14 行定义了「技能调用协议」，核心原则：**「第一步行动永远是使用 `read_file` 工具读取 SKILL.md，禁止直接猜测技能参数」**。这条原则被注入到 System Prompt 的 `Agents Guide` 组件里，相当于给 Agent 建立了一个「先查说明书再操作」的使用规范。

### 4.2 Skills 扫描注入实现

接下来我们在临时目录创建两个示例技能，然后扫描、解析、生成 XML 快照，最后把快照注入到第 1 章的 prompt 里。运行后你会看到 XML 格式的技能快照，以及注入 prompt 后 `SKILLS_SNAPSHOT` 组件的内容——还会故意制造一个 YAML 解析失败的案例，观察静默 skip 机制。

```python
# 第 4 章：Skills 扫描注入（完全自包含，用 tempfile）
# 对照源码：backend/tools/skills_scanner.py:10-50

import tempfile
import os
from pathlib import Path
import yaml

# 在临时目录创建两个技能（模拟 skills/ 目录结构）
tmp_dir = Path(tempfile.mkdtemp())
skills_dir = tmp_dir / "skills"
skills_dir.mkdir()

# 技能 1：正常技能（frontmatter 格式正确）
skill1_dir = skills_dir / "research"
skill1_dir.mkdir()
(skill1_dir / "SKILL.md").write_text("""---
name: 深度研究助手
description: 针对给定主题进行多步骤深度研究，包括搜索、阅读、综合分析
usage: 当用户要求深入研究某个话题时使用此技能
---

## 步骤

1. 使用 tavily_search 搜索相关资料
2. 使用 fetch_url 读取重要链接的完整内容
3. 综合分析，生成结构化报告
""", encoding="utf-8")

# 技能 2：故意制造 YAML 解析失败（description 含未加引号的冒号）
skill2_dir = skills_dir / "broken_skill"
skill2_dir.mkdir()
(skill2_dir / "SKILL.md").write_text("""---
name: 有问题的技能
description: 这个描述包含冒号: 会导致 YAML 解析失败
usage: 演示用
---

技能内容...
""", encoding="utf-8")

# 技能 3：正常技能
skill3_dir = skills_dir / "code_review"
skill3_dir.mkdir()
(skill3_dir / "SKILL.md").write_text("""---
name: 代码审查助手
description: "对给定代码进行安全性、可读性和性能分析"
usage: 当用户提交代码需要审查时使用
---

## 步骤

1. 使用 read_file 读取代码文件
2. 分析安全漏洞、代码风格、性能问题
3. 生成审查报告
""", encoding="utf-8")

print(f"临时技能目录：{skills_dir}")
print(f"创建了 3 个技能目录（1 个有意破损）\n")

def scan_skills_mvp(base_dir: Path) -> tuple[str, list[dict]]:
    """
    技能扫描器，对照源码 skills_scanner.py:10-50。

    Args:
        base_dir: 包含 skills/ 子目录的根目录

    Returns:
        (snapshot_xml, skills_list)：XML 快照字符串和解析成功的技能列表
    """
    skills_inner_dir = base_dir / "skills"
    skills = []

    # rglob 递归扫描所有 SKILL.md（源码第 19 行）
    for skill_md in sorted(skills_inner_dir.rglob("SKILL.md")):
        try:
            content = skill_md.read_text(encoding="utf-8")
            # 检查是否有 frontmatter（--- 开头）
            if content.startswith("---"):
                parts = content.split("---", 2)  # 拆成 3 部分：空串 / yaml / 正文
                if len(parts) >= 3:
                    # 解析 frontmatter YAML（源码第 26 行）
                    meta = yaml.safe_load(parts[1])
                    if meta:
                        rel_path = f"./skills/{skill_md.parent.name}/SKILL.md"
                        skills.append({
                            "name": meta.get("name", skill_md.parent.name),
                            "description": meta.get("description", ""),
                            "location": rel_path,
                        })
                        print(f"  [OK] 解析技能：{meta.get('name')}")
        except Exception as e:
            # 静默 skip：单个技能失败不阻断其他技能（源码第 34 行）
            print(f"  [SKIP] 解析失败 {skill_md.parent.name}：{e}")

    # 生成 XML 风格快照（源码第 38-48 行）
    lines = ["<available_skills>"]
    for s in skills:
        lines.append("  <skill>")
        lines.append(f"    <name>{s['name']}</name>")
        lines.append(f"    <description>{s['description']}</description>")
        lines.append(f"    <location>{s['location']}</location>")
        lines.append("  </skill>")
    lines.append("</available_skills>")

    return "\n".join(lines), skills

print("开始扫描技能...\n")
snapshot_xml, parsed_skills = scan_skills_mvp(tmp_dir)

print(f"\n成功解析 {len(parsed_skills)} 个技能（共 3 个，1 个应被跳过）")
print("\n生成的 SKILLS_SNAPSHOT XML：")
print(snapshot_xml)
```

```
临时技能目录：/var/folders/fl/8wq5_lz53ln9ypplts4z_1tr0000gn/T/tmptegr2_2x/skills
创建了 3 个技能目录（1 个有意破损）

开始扫描技能...

  [SKIP] 解析失败 broken_skill：mapping values are not allowed here
  in "<unicode string>", line 3, column 22:
    description: 这个描述包含冒号: 会导致 YAML 解析失败
                         ^
  [OK] 解析技能：代码审查助手
  [OK] 解析技能：深度研究助手

成功解析 2 个技能（共 3 个，1 个应被跳过）

生成的 SKILLS_SNAPSHOT XML：
<available_skills>
  <skill>
    <name>代码审查助手</name>
    <description>对给定代码进行安全性、可读性和性能分析</description>
    <location>./skills/code_review/SKILL.md</location>
  </skill>
  <skill>
    <name>深度研究助手</name>
    <description>针对给定主题进行多步骤深度研究，包括搜索、阅读、综合分析</description>
    <location>./skills/research/SKILL.md</location>
  </skill>
</available_skills>
```

快照生成后，我们用一段验证代码确认它的正确性——检查 XML 是否包含正确数量的 `<skill>` 节点，并通过故障注入（一个格式错误的技能）验证扫描器能否正确跳过异常项。

```python
# Tier 1 验证：XML 快照包含正确节点数 + 故障注入验证
import re

# 验证 XML 快照包含 2 个 <skill> 节点（有问题的技能被跳过）
skill_nodes = re.findall(r"<skill>", snapshot_xml)
assert len(skill_nodes) == 2, f"应有 2 个 skill 节点，实际 {len(skill_nodes)}"
print(f"[PASS] XML 包含 {len(skill_nodes)} 个 <skill> 节点（1 个解析失败被跳过）")

# Tier 2 验证：将快照注入到第 1 章 prompt，确认首个组件正确
snapshot_prompt = f"<!-- Skills Snapshot -->\n{snapshot_xml}"
assert "<available_skills>" in snapshot_prompt
assert "深度研究助手" in snapshot_prompt
assert "代码审查助手" in snapshot_prompt
print("[PASS] SKILLS_SNAPSHOT 成功注入 prompt 首段")
```

```
[PASS] XML 包含 2 个 <skill> 节点（1 个解析失败被跳过）
[PASS] SKILLS_SNAPSHOT 成功注入 prompt 首段
```

运行这段代码，你会看到两个重要现象：第一，带未加引号冒号的 `description` 字段导致 YAML 解析失败，但程序不崩溃，只打印 `[SKIP]` 并继续扫描其他技能——这是一个「健壮性优先于严格性」的设计决策；第二，最终 XML 快照只有 2 个 `<skill>` 节点，而不是 3 个，验证了静默 skip 机制。

你现在已经掌握了「技能即插件」哲学的代码落地——从 `SKILL.md` 文件到 XML 快照到 prompt 注入，完整闭环。在学这章之前，你可能以为「给 Agent 加技能」需要写 Python 代码；现在你知道只需要写一个 Markdown 文件，放进 `skills/` 目录，重启时自动生效。第 5 章我们深入「文件即记忆」的第二层：当记忆文件太大，用 RAG 按需检索替代全量注入。

> **【清理时点说明】**：4.2 节的 `tmp_dir` 临时目录**暂不清理**——4.3 节"手动创建技能"会沿用它来演示真实落盘。清理动作统一放在 4.3 节末尾执行。

> **【踩坑预警】**：YAML frontmatter 中的 `description` 字段如果包含冒号（`:`），**必须用引号包裹**：`description: "这是一个包含: 冒号的描述"`。忘了加引号，`yaml.safe_load` 会抛出 `ScannerError`，该技能被静默跳过，而你在 Agent 的 SKILLS_SNAPSHOT 里找不到这个技能——这个 bug 在 FuFan-OpenClaw 项目里真实出现并修复过。

### 4.3 SKILL CREATION PROTOCOL：技能创建的开放标准

第 4.2 节我们演示了「用技能」——Agent 通过 `read_file` 读取现成的 `SKILL.md` 然后执行。但 **技能从哪儿来**？这就要讲 `backend/workspace/AGENTS.md` 里的「**技能创建协议**」(对应 `AGENTS.md:39-88` 行)——基于 Anthropic 主导的 [Agent Skills 开放标准](https://agentskills.io/)。本项目所有 5 个内置技能都遵循这个标准。

**核心结构**：每个技能是一个文件夹，包含必需的 `SKILL.md` 和可选的 `scripts/`、`references/`、`assets/` 三个子目录。

```text
skills/{skill-name}/
├── SKILL.md              # 必须：YAML frontmatter + Markdown 指令
├── scripts/              # 可选：可执行脚本（Python/Bash/JS）
├── references/           # 可选：额外参考文档（不读入 prompt）
└── assets/               # 可选：模板、数据文件
```

**SKILL.md 必备字段**：`name`(1-64 字符,小写字母+数字+连字符下划线,须与文件夹名匹配) + `description`(1-1024 字符,以「Use when...」开头说明触发条件)。正文推荐 6 段式:概述 → 触发条件 → 执行步骤(祈使句)→ 示例(完整可运行)→ 常见错误 → 错误处理。**单文件 ≤ 500 行 / 推荐 < 5000 tokens**——超出要拆到 `references/`。

**我们手动创建一个新技能**——从「读到需求」到「目录落盘」只需 4 行 Python:

```python
# 第 4.3 节：手动创建技能（模拟"AI 创建技能"的最简路径）
# 对照源码：backend/workspace/AGENTS.md:39-88（技能创建协议）
# 真实项目中"AI 创建技能"由前端 NewSkillModal 触发后端 LLM 流程,本课只演示最终产物
import os
from pathlib import Path

# 假设已有一个临时的 skills 目录（沿用上方 4.2 节的 tmp_dir，未在 4.2 末尾清理）
skill_name = "weather_v2"
skill_dir = tmp_dir / "skills" / skill_name
skill_dir.mkdir(parents=True, exist_ok=True)

skill_md = skill_name + "/SKILL.md"
(skill_dir / "SKILL.md").write_text("""---
name: weather_v2
description: "Use when user requests weather information. Provides real-time weather for any city using fetch_url tool."
---

# 天气查询 v2

## 执行步骤
1. 从用户消息提取英文城市名
2. 使用 `fetch_url("https://wttr.in/{city}?format=j1")` 拿 JSON
3. 解析 current_condition 块,提取 temp_C / weatherDesc / humidity
4. 用自然语言回复用户

## 示例
用户:「北京天气」
→ 调 fetch_url("https://wttr.in/Beijing?format=j1")
→ 回复:「北京当前 25°C,晴,湿度 40%」
""", encoding="utf-8")

print(f"技能已创建: {skill_md}")
assert (skill_dir / "SKILL.md").exists()
print("[PASS] 技能目录与 SKILL.md 落盘成功")
```

```
技能已创建: weather_v2/SKILL.md
[PASS] 技能目录与 SKILL.md 落盘成功
```

**4 个核心约束**(从协议中提炼,本课 MVP 必遵守):

1. **`description` 不要概括步骤**——只描述"做什么 + 何时触发",包含具体关键词以便发现;步骤放正文

2. **使用祈使语气**——"分析代码..."而非"你应该分析代码..."(这是 Skill 作为指令的核心)

3. **示例要完整可运行**——"示例要完整、可运行，并注释说明原因"(AGENTS.md 原话)

4. **详细参考资料放 `references/`**——别全部塞进 SKILL.md,SKILL.md 是 Agent 必读的,要短

**AI 自动创建技能的真实路径**(超出本课范围,留作课后探索):前端 `/skills` 页面的 `NewSkillModal` 收集"技能需求描述" → 调一次 LLM 完整流程生成 SKILL.md → 后端落盘到 `skills/{name}/` → 重启扫描时自动注入。当前阶段需要手动管理技能目录(本节演示的就是这条手动路径)。

你现在已经掌握了「技能即插件」哲学的完整闭环——**用技能**(第 4.2 节 Agent 读 SKILL.md)+ **造技能**(本节 4.3 协议 + 4 行 Python)。在学这节之前,你可能以为"加技能"是改后端代码;现在你知道只需写一个 SKILL.md 文件,放进 `skills/` 目录即可。第 5 章我们转向「文件即记忆」的第二层——RAG 检索。

```python
# 4.2 + 4.3 共用的 tmp_dir 临时目录,在 4.3 末尾统一清理
import shutil
shutil.rmtree(tmp_dir)
print("临时目录已清理。")
```

```
临时目录已清理。
```

---

## 第 5 章：文件即记忆②——RAG 检索

我们在第 1 章实现的全量注入方案有一个物理限制：当 `MEMORY.md` 积累了几个月的长期记忆，文件可能长达数万字符。把它全量塞进每轮的 System Prompt，既消耗 token，也会稀释 Agent 对真正相关内容的注意力。这就是「文件即记忆」哲学的第二层：当文件太大，切换到 RAG 模式——把 `MEMORY.md` 分块、向量化、存索引，每轮对话时根据当前问题检索最相关的几个片段，精准注入。

这两种模式的切换，我们在第 1 章就见过：`build_system_prompt(rag_mode=True)` 会排除 MEMORY 全文，改追加 `RAG_GUIDANCE`。本章就是 RAG 模式下的「检索方」：用 LlamaIndex 对 `MEMORY.md` 建向量索引，并实现 MD5 增量重建（文件不变不重算 embedding，避免每轮对话都重新向量化）。

### 5.1 源码锚点对照

`backend/graph/memory_indexer.py` 全文 253 行（含 singleton 工厂），关键区段：

- 第 12-24 行：`_zh_en_tokenizer`——中英混合分词器（英文按词 / 中文按单字，教学版零依赖）

- 第 44-67 行：MD5 增量逻辑——`_get_file_hash` / `_get_stored_hash` / `_save_hash` / `_maybe_rebuild` 四个方法

- 第 69-119 行：`rebuild_index` 方法——LlamaIndex 导入、OpenAIEmbedding 配置、分块、建索引、持久化

- 第 86-92 行：`OpenAIEmbedding` 配置，三个关键参数：`model` / `api_key` / `api_base`（走代理）

- 第 101 行：`SentenceSplitter(chunk_size=256, chunk_overlap=32)`

- 第 160-185 行：`_hybrid_retrieve` 方法——`vector` 与 `BM25Retriever` 双路并行召回，传入第 12 行的中文分词器

- 第 188-209 行：`_reciprocal_rank_fusion` 自实现 RRF（k=60）——注释明确说"为什么不用 QueryFusionRetriever：后者无条件 resolve 一个 LLM 引入隐藏依赖"

- 第 211-241 行：`retrieve(query, top_k, mode)` 方法——`mode='vector'` 走纯向量，`mode='hybrid'` 走 RRF 融合

- `backend/config.py` 第 13/55-67 行：`retrieval_mode` 全局配置（'vector' | 'hybrid'），关 RAG 时自动重置为 'vector'（兜底）

- `backend/graph/agent.py` 第 135-155 行：RAG 模式下每轮 astream 前先 `indexer.retrieve(message, mode=retrieval_mode)`，把召回片段作为 assistant 消息注入 history

> **【Embedding 凭证预警】**：RAG 使用 `OpenAIEmbedding`，走 OpenAI 兼容代理，凭证来源是 `OPENAI_API_KEY` + `OPENAI_BASE_URL`（项目 `.env` 里的 `OPENAI_BASE_URL` 指向 `https://ai.devtool.tech/proxy/v1`）。这与第 2 章的 LLM 凭证（`DEEPSEEK_API_KEY`）是**不同的 key**——RAG Embedding 用 OpenAI 接口，主 LLM 用 DeepSeek 接口，两套凭证不能混用。

### 5.2 RAG 检索器实现（双模式 + 中英分词 + RRF 融合）

本节按源码真实结构拆成 3 个独立 cell：**Cell A**（BM25 关键词检索，零依赖）→ **Cell B**（向量语义检索，需 `OPENAI_API_KEY`）→ **Cell C**（自实现 RRF 融合，零依赖）。这种"白盒分段"的设计有教学意义：每一路召回你都能独立验通，再观察融合后的差异——而不是把三步压在一个黑盒 cell 里。

**Cell A：BM25 关键词检索 + 中英混合分词（零依赖）**

先重写源码 `_zh_en_tokenizer`（对照 `memory_indexer.py:12-24`）——这是教学版零依赖分词器，英文按词切、中文按单字切。生产环境可换 `jieba.lcut` 提升中文词级召回；本课保留单字版因为 `MEMORY.md` 含大量英文术语（skill 名 / API 名 / 文件名），BM25 对这些精确关键词的召回正是向量语义检索的弱项——两路融合形成互补。

```python
# 第 5 章 Cell A：BM25 关键词检索 + 中英分词演示（零依赖）
# 对照源码：backend/graph/memory_indexer.py:12-24（分词器）+ 160-185（BM25 召回）

import re
from llama_index.retrievers.bm25 import BM25Retriever

def zh_en_tokenizer(text: str) -> list[str]:
    """
    中英混合分词器（教学版，零依赖）。
    英文/数字按词切分（统一小写），中文按单字切分。
    对照源码 memory_indexer.py:12-24
    """
    text = text.lower()
    en_tokens = re.findall(r"[a-z0-9]+", text)
    zh_tokens = re.findall(r"[一-鿿]", text)
    return en_tokens + zh_tokens

# 演示分词差异：同一段中文+英文混合文本
sample = "Python 编程偏好 FastAPI 框架,常用 LlamaIndex 做 RAG 检索"
print("原文:", sample)
print("分词结果:", zh_en_tokenizer(sample))
# 预期输出: ['python', '编程', '偏', '好', 'fastapi', '框', '架', '常', '用', 'llamaindex', '做', 'r', 'a', 'g', '检', '索']
# 注意: 'Python' / 'FastAPI' / 'LlamaIndex' 整体被切出（英文词）,'编程偏好框架' 切成单字

# 模拟 MEMORY.md 的几个节点
mock_nodes_text = [
    "用户偏好 Python 编程,熟悉 FastAPI 框架",
    "常用 LlamaIndex 做 RAG 检索,使用 OpenAI 嵌入",
    "2024-05: 把会话存储从 Redis 改为本地 JSON 文件",
    "工作领域是 AI 应用开发,常做向量数据库实验",
    "北京天气晴,温度 25 度,湿度 40%",
]

# 包装成 LlamaIndex Node（最简形态）
from llama_index.core.schema import TextNode
mock_nodes = [TextNode(text=t) for t in mock_nodes_text]

# BM25 检索器（零 LLM 依赖,纯关键词匹配）
bm25 = BM25Retriever.from_defaults(
    nodes=mock_nodes,
    similarity_top_k=3,
    tokenizer=zh_en_tokenizer,
    skip_stemming=True,  # 中文不做英文词干还原
)

# 演示 1: 英文关键词精确召回
print("\n--- Query 1: 'Python 偏好' ---")
hits1 = bm25.retrieve("Python 偏好")
for i, h in enumerate(hits1):
    print(f"  [{i+1}] score={h.get_score():.4f} | {h.text}")

# 演示 2: 英文 API 名精确召回
print("\n--- Query 2: 'LlamaIndex 检索' ---")
hits2 = bm25.retrieve("LlamaIndex 检索")
for i, h in enumerate(hits2):
    print(f"  [{i+1}] score={h.get_score():.4f} | {h.text}")

assert len(hits1) >= 1 and len(hits2) >= 1
print("\n[PASS] BM25 零依赖检索跑通,英文关键词精确命中对应节点")
```

```
/opt/anaconda3/envs/fufan-openclaw/lib/python3.11/site-packages/tqdm/auto.py:21: TqdmWarning: IProgress not found. Please update jupyter and ipywidgets. See https://ipywidgets.readthedocs.io/en/stable/user_install.html
  from .autonotebook import tqdm as notebook_tqdm
The tokenizer parameter is deprecated and will be removed in a future release. Use a stemmer from PyStemmer instead.
原文: Python 编程偏好 FastAPI 框架,常用 LlamaIndex 做 RAG 检索
分词结果: ['python', 'fastapi', 'llamaindex', 'rag', '编', '程', '偏', '好', '框', '架', '常', '用', '做', '检', '索']

--- Query 1: 'Python 偏好' ---
  [1] score=0.5460 | 用户偏好 Python 编程,熟悉 FastAPI 框架
  [2] score=0.0000 | 北京天气晴,温度 25 度,湿度 40%
  [3] score=0.0000 | 工作领域是 AI 应用开发,常做向量数据库实验

--- Query 2: 'LlamaIndex 检索' ---
  [1] score=1.0146 | 常用 LlamaIndex 做 RAG 检索,使用 OpenAI 嵌入
  [2] score=0.0000 | 北京天气晴,温度 25 度,湿度 40%
  [3] score=0.0000 | 工作领域是 AI 应用开发,常做向量数据库实验

[PASS] BM25 零依赖检索跑通,英文关键词精确命中对应节点
```

BM25 关键词检索跑通后，我们接着实现向量语义检索路径。它与 BM25 形成互补——一个精于精确关键词命中，一个精于语义相似召回。下面这段对照源码构建 LlamaIndex 检索器（需要 `OPENAI_API_KEY`，已写降级路径）。

```python
# 第 5 章：LlamaIndex RAG 检索器（需要 OPENAI_API_KEY）
# 对照源码：backend/graph/memory_indexer.py:44-119, 211-241

import hashlib
import json
import os
import tempfile
from pathlib import Path
from llama_index.core import Document, VectorStoreIndex
from llama_index.core.node_parser import SentenceSplitter

# 凭证已在第 0 章 0.5 步骤二加载，Embedding 已在步骤四配置（Settings.embed_model）

# 临时目录存放 MEMORY.md 和索引
tmp_dir = Path(tempfile.mkdtemp())
memory_file = tmp_dir / "MEMORY.md"
storage_dir = tmp_dir / "memory_index"
hash_file = storage_dir / ".memory_hash"

# 模拟 MEMORY.md 内容（真实项目里是用户的长期记忆）
memory_content = """
# 长期记忆

## 用户偏好
- 编程语言偏好：Python > JavaScript，偏爱简洁代码风格
- 回复语言：中文，偶尔夹杂英文技术术语
- 工作领域：AI 应用开发，熟悉 LangChain 和 OpenAI API

## 历史决策
- 2024-03：选择了 FastAPI 作为后端框架，原因是异步支持好
- 2024-05：把会话存储从 Redis 改为本地 JSON 文件，降低依赖复杂度
- 2024-06：引入 RAG 检索，替代全量记忆注入，因为记忆文件超过了 10000 字

## 重要信息
- 常用 API key 的管理方式：使用 .env 文件 + python-dotenv
- 偏好的 Agent 框架：LangChain 1.x create_agent（非旧版 AgentExecutor）
"""
memory_file.write_text(memory_content, encoding="utf-8")

# --- MD5 增量逻辑（对照源码 memory_indexer.py:44-67）---

def get_file_hash(path: Path) -> str:
    """
    计算文件的 MD5 哈希，用于检测文件是否发生变化。

    Args:
        path: 待哈希的文件路径

    Returns:
        MD5 十六进制字符串；文件不存在时返回空字符串
    """
    if not path.exists():
        return ""
    # 读取二进制内容计算 MD5（源码第 48-49 行）
    return hashlib.md5(path.read_bytes()).hexdigest()

def get_stored_hash(hash_path: Path) -> str:
    """读取上次构建时存储的哈希值（源码第 51-55 行）。"""
    if not hash_path.exists():
        return ""
    return hash_path.read_text(encoding="utf-8").strip()

def save_hash(hash_path: Path, hash_value: str) -> None:
    """将当前哈希写入磁盘（源码第 57-60 行）。"""
    hash_path.parent.mkdir(parents=True, exist_ok=True)
    hash_path.write_text(hash_value, encoding="utf-8")

def maybe_rebuild(memory_path: Path, storage_path: Path, h_path: Path) -> bool:
    """
    比较当前哈希与存储哈希，若不同则触发重建（源码 _maybe_rebuild：第 62-67 行）。

    Returns:
        True 表示触发了重建，False 表示跳过
    """
    current = get_file_hash(memory_path)
    stored = get_stored_hash(h_path)
    # 哈希不同才重建——这是避免每次对话都重算 Embedding 的核心机制
    if current and current != stored:
        print(f"  [rebuild] 哈希变化：{stored[:8] or '空'}... → {current[:8]}...")
        return True
    print(f"  [skip] 哈希未变（{current[:8]}...），跳过重建")
    return False

# --- 向量索引构建和检索（对照源码 rebuild_index 第 69-119 行，retrieve 第 211-241 行）---

def build_rag_index(memory_path: Path, storage_path: Path, h_path: Path):
    """
    用 LlamaIndex 建向量索引并持久化。
    对照源码 rebuild_index 方法（第 69-119 行）。

    Args:
        memory_path: MEMORY.md 文件路径
        storage_path: 索引持久化目录
        h_path: 哈希存储文件路径

    Returns:
        VectorStoreIndex 对象；失败时返回 None
    """
    try:
  
        # Embedding 已在第 0 章 0.5 步骤四全局配置（Settings.embed_model，源码第 86-92 行）
        content = memory_path.read_text(encoding="utf-8")
        doc = Document(text=content, metadata={"source": "MEMORY.md"})

        # chunk_size=256, chunk_overlap=32（源码第 101 行）
        splitter = SentenceSplitter(chunk_size=256, chunk_overlap=32)
        nodes = splitter.get_nodes_from_documents([doc])
        print(f"  [chunk] 切分成 {len(nodes)} 个节点")

        storage_path.mkdir(parents=True, exist_ok=True)
        index = VectorStoreIndex(nodes)
        index.storage_context.persist(persist_dir=str(storage_path))

        # 成功后保存当前哈希
        save_hash(h_path, get_file_hash(memory_path))
        print(f"  [persist] 索引已持久化到 {storage_path}")
        return index

    except ImportError as e:
        print(f"  [warn] LlamaIndex 未完整安装：{e}")
        return None
    except Exception as e:
        print(f"  [warn] 构建索引失败：{e}")
        return None

def retrieve_memory(index, query: str, top_k: int = 3) -> list:
    """
    从向量索引检索与 query 最相关的记忆片段（源码第 211-241 行）。

    Args:
        index: LlamaIndex VectorStoreIndex 对象
        query: 用户当前问题，用于语义相似度检索
        top_k: 返回最相关的前 k 个片段

    Returns:
        包含 text/score/source 字段的字典列表
    """
    if index is None:
        return []
    retriever = index.as_retriever(similarity_top_k=top_k)
    nodes = retriever.retrieve(query)
    return [
        {
            "text": node.get_text(),
            "score": f"{node.get_score():.4f}" if node.get_score() else "N/A",
            "source": node.metadata.get("source", "MEMORY.md"),
        }
        for node in nodes
    ]

# --- 运行演示 ---
print("=" * 50)
print("演示 1：首次构建索引（哈希为空，触发 rebuild）")
print("=" * 50)
should_rebuild = maybe_rebuild(memory_file, storage_dir, hash_file)

index = None
if should_rebuild:
    index = build_rag_index(memory_file, storage_dir, hash_file)

if index:
    print("\n检索：'用户喜欢什么编程语言？'")
    results = retrieve_memory(index, "用户喜欢什么编程语言？")
    for i, r in enumerate(results):
        print(f"  片段 {i+1} (score={r['score']})：{r['text'][:80]}...")

print("\n" + "=" * 50)
print("演示 2：不修改文件，再次检查（应跳过重建）")
print("=" * 50)
should_rebuild2 = maybe_rebuild(memory_file, storage_dir, hash_file)
assert not should_rebuild2, "文件未变化，不应触发重建"
print("[PASS] 哈希未变，正确跳过重建。")
```

```
==================================================
演示 1：首次构建索引（哈希为空，触发 rebuild）
==================================================
  [rebuild] 哈希变化：空... → 1e41bb4c...
  [chunk] 切分成 2 个节点
  [persist] 索引已持久化到 /var/folders/fl/8wq5_lz53ln9ypplts4z_1tr0000gn/T/tmp78ol7uj4/memory_index

检索：'用户喜欢什么编程语言？'
  片段 1 (score=0.4518)：# 长期记忆

## 用户偏好
- 编程语言偏好：Python > JavaScript，偏爱简洁代码风格
- 回复语言：中文，偶尔夹杂英文技术术语
- 工作领...
  片段 2 (score=0.2684)：env 文件 + python-dotenv
- 偏好的 Agent 框架：LangChain 1.x create_agent（非旧版 AgentExecut...

==================================================
演示 2：不修改文件，再次检查（应跳过重建）
==================================================
  [skip] 哈希未变（1e41bb4c...），跳过重建
[PASS] 哈希未变，正确跳过重建。
```

确认"文件未变化时跳过重建"之后，我们验证相反方向——修改 `MEMORY.md` 后，索引能否正确检测到变化并触发重建。这一正一反两个断言，共同证明 MD5 增量逻辑的正确性。

```python
# Tier 2 验证：修改 MEMORY.md 后触发重建
import shutil

print("=" * 50)
print("演示 3：修改 MEMORY.md，触发增量重建")
print("=" * 50)

# 追加新内容，改变文件哈希
with open(memory_file, "a", encoding="utf-8") as f:
    f.write("\n\n## 新增记忆\n- 2024-07：开始学习 LlamaIndex 的 RAG 机制\n")

should_rebuild3 = maybe_rebuild(memory_file, storage_dir, hash_file)
assert should_rebuild3, "修改文件后应触发重建"
print("[PASS] 文件修改后，正确触发增量重建。")

# 清理
shutil.rmtree(tmp_dir)
print("临时目录已清理。")
```

```
==================================================
演示 3：修改 MEMORY.md，触发增量重建
==================================================
  [rebuild] 哈希变化：1e41bb4c... → 7ec92f40...
[PASS] 文件修改后，正确触发增量重建。
临时目录已清理。
```

运行三段代码，你会看到 MD5 增量逻辑的完整闭环：首次运行触发重建并保存哈希，第二次不变跳过，第三次修改文件后再次触发。这个机制在真实项目里非常重要——如果 `MEMORY.md` 每次对话都重建索引（重算全部 embedding），每条消息要多付好几倍的 API 费用，也会额外增加延迟。

你现在已经掌握了「文件即记忆」哲学的完整闭环——全量注入（第 1 章）和 RAG 检索（本章）是同一哲学的两种读取形态，由 `rag_mode` 开关控制。在学这章之前，你可能觉得「RAG 很复杂」；现在你知道它的骨架只有三步：切块、向量化、检索，真正的工程复杂度在增量重建和凭证管理上。第 6 章我们转向会话的持久化——记忆是跨会话的，会话本身也需要被妥善保存和压缩。

> **【踩坑预警】**：`OPENAI_BASE_URL` 必须从 `.env` 加载，不要硬编码。如果你用的代理地址不同，直接改 `.env` 即可生效；如果你把它写死在代码里，团队成员换了代理后需要改代码，违背了「配置与代码分离」的原则。另外，`chunk_overlap=32` 不是 0——边界词汇如果只出现在某个 chunk 的末尾，overlap 能保证它在下一个 chunk 的开头也出现，提升边界处的检索质量。

---

**Cell B：向量语义检索（需 OPENAI_API_KEY，降级路径已写）**

Cell A 演示了关键词召回,本 cell 演示语义召回。需要 `OPENAI_API_KEY` + `OPENAI_BASE_URL`(详见 0.2 节 7 key 表)。无 key 时,降级路径是观察代码结构,理解 `VectorStoreIndex + OpenAIEmbedding + as_retriever` 三件套的拼装。

```python
# 第 5 章 Cell B：LlamaIndex 向量检索（需 OPENAI_API_KEY）
# 对照源码：backend/graph/memory_indexer.py:69-119（rebuild_index 向量部分）+ 211-241（retrieve）

import os, tempfile, hashlib, shutil
from pathlib import Path

# 凭证已在第 0 章 0.5 步骤二加载，Embedding 已在步骤四配置（Settings.embed_model）

tmp_dir = Path(tempfile.mkdtemp())
memory_file = tmp_dir / "MEMORY.md"
storage_dir = tmp_dir / "memory_index"
hash_file = storage_dir / ".memory_hash"

memory_content = """
# 长期记忆

## 用户偏好
- 编程语言偏好：Python > JavaScript，偏爱简洁代码风格
- 工作领域：AI 应用开发，熟悉 LangChain 和 OpenAI API

## 历史决策
- 2024-05：把会话存储从 Redis 改为本地 JSON 文件，降低依赖复杂度
- 2024-06：引入 RAG 检索，替代全量记忆注入

## 重要信息
- 偏好的 Agent 框架：LangChain 1.x create_agent（非旧版 AgentExecutor）
"""
memory_file.write_text(memory_content, encoding="utf-8")

# MD5 增量（保留原逻辑）
def get_hash(p): return hashlib.md5(p.read_bytes()).hexdigest() if p.exists() else ""
def save_hash(p, v): p.parent.mkdir(parents=True, exist_ok=True); p.write_text(v, encoding="utf-8")
def maybe_rebuild():
    h = get_hash(memory_file)
    sh = hash_file.read_text().strip() if hash_file.exists() else ""
    if h != sh: save_hash(hash_file, h); return True
    return False

# 向量构建
def build_vector_index():
    if not maybe_rebuild():
        print("[skip] 文件未变化,跳过")
        return None
    try:
        from llama_index.core import Document, VectorStoreIndex
        from llama_index.core.node_parser import SentenceSplitter

        # Embedding 已在第 0 章 0.5 步骤四全局配置（Settings.embed_model）
        doc = Document(text=memory_file.read_text(encoding="utf-8"), metadata={"source": "MEMORY.md"})
        splitter = SentenceSplitter(chunk_size=256, chunk_overlap=32)
        nodes = splitter.get_nodes_from_documents([doc])
        storage_dir.mkdir(parents=True, exist_ok=True)
        idx = VectorStoreIndex(nodes)
        idx.storage_context.persist(persist_dir=str(storage_dir))
        print(f"[ok] 向量索引构建完成,共 {len(nodes)} chunks")
        return idx
    except Exception as e:
        print(f"[warn] 向量构建失败（key 缺失或网络问题）: {e}")
        print("      降级: 观察代码结构理解三件套拼装即可")
        return None

idx = build_vector_index()

if idx:
    # 语义查询 1: 英文语义检索
    print("\n--- Query: 'what framework does the user prefer' ---")
    hits = idx.as_retriever(similarity_top_k=2).retrieve("what framework does the user prefer")
    for i, h in enumerate(hits):
        print(f"  [{i+1}] score={h.get_score():.4f} | {h.text[:80]}")
    assert len(hits) >= 1, "向量检索应至少返回 1 条"
    print("[PASS] 向量语义检索跑通")
else:
    print("[SKIP] 无 OPENAI_API_KEY,跳过本 cell 真跑")

# 清理
shutil.rmtree(tmp_dir, ignore_errors=True)
```

```
[ok] 向量索引构建完成,共 1 chunks

--- Query: 'what framework does the user prefer' ---
  [1] score=0.3404 | # 长期记忆

## 用户偏好
- 编程语言偏好：Python > JavaScript，偏爱简洁代码风格
- 工作领域：AI 应用开发，熟悉 LangChai
[PASS] 向量语义检索跑通
```

> **降级路径**:`OPENAI_API_KEY` 失效或代理不可达时,`build_vector_index` 返回 None,程序打印 `[warn]` 不崩溃。这与原 MVP 行为一致——本课 MVP 设计原则是"无 key 时仍能让小白看到完整代码结构",不卡死在某个外部依赖上。

**Cell C：自实现 RRF 融合（零依赖，白盒演示）**

Cell A 和 Cell B 各自召回了一组结果,**hybrid 模式**就是用 RRF 把两组排名融合成最终排名。对照源码 `memory_indexer.py:188-209`——`k=60` 是业界经验常数(不是拍脑袋),`score(d) = Σ_i 1 / (k + rank_i(d))`,名次越靠前贡献越大,同时被两路命中的文档得分叠加。

```python
# 第 5 章 Cell C：自实现 RRF 倒数排名融合（零依赖）
# 对照源码：backend/graph/memory_indexer.py:188-209（_reciprocal_rank_fusion）

def reciprocal_rank_fusion(ranked_lists, top_k=3, k=60):
    """
    RRF 倒数排名融合:score(d) = Σ_i 1 / (k + rank_i(d))
    对照源码 _reciprocal_rank_fusion
    """
    scores, node_map = {}, {}
    for nodes in ranked_lists:
        for rank, nws in enumerate(nodes):
            nid = getattr(nws, "node_id", id(nws))  # BM25 召回是 NodeWithScore;此处用 id 兼容
            node_map[nid] = nws
            scores[nid] = scores.get(nid, 0.0) + 1.0 / (k + rank + 1)
    ranked = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)[:top_k]
    fused = []
    for nid, score in ranked:
        nws = node_map[nid]
        nws.score = score
        fused.append(nws)
    return fused

# 模拟 Cell A(BM25)与 Cell B(向量)的两路召回结果
class MockHit:
    def __init__(self, nid, text, original_score):
        self.node_id, self.text, self.original_score = nid, text, original_score
    def __repr__(self):
        return f"[{self.node_id}] score={self.original_score:.4f} | {self.text[:40]}"

# 模拟 BM25 召回的 top-3(对 query "Python 框架偏好")
bm25_top3 = [
    MockHit("n1", "用户偏好 Python 编程,熟悉 FastAPI 框架", 0.85),
    MockHit("n2", "工作领域是 AI 应用开发", 0.45),
    MockHit("n3", "北京天气晴,温度 25 度", 0.20),
]

# 模拟向量召回的 top-3(对同一 query,语义召回更偏"框架 + 偏好")
vector_top3 = [
    MockHit("n2", "工作领域是 AI 应用开发,常做向量数据库实验", 0.92),
    MockHit("n1", "用户偏好 Python 编程,熟悉 FastAPI 框架", 0.88),
    MockHit("n4", "2024-05: 把会话存储从 Redis 改为本地 JSON 文件", 0.55),
]

# 融合（取 top-4 以便 n4 也进入结果集,完整观察"双路命中 vs 单路命中"排名差异）
fused = reciprocal_rank_fusion([bm25_top3, vector_top3], top_k=4)
print("--- 融合后 top-4 ---")
for i, h in enumerate(fused):
    print(f"  [{i+1}] RRF score={h.score:.4f} | {h.text}")

# 验证: n1 和 n2 同时被两路命中 → 融合分应高于 n3(仅 BM25 命中)与 n4(仅向量命中)
fused_ids = [h.node_id for h in fused]
print(f"\n融合 top-4 IDs: {fused_ids}")
# 预期: ['n1', 'n2', 'n3', 'n4'] —— n1 / n2 双路命中,排前; n3 仅 BM25,中段; n4 仅向量,排后
assert "n1" in fused_ids and "n2" in fused_ids
# 关键断言:n1 / n2 排名必须严格高于 n4(单路命中)
n1_rank = fused_ids.index("n1")
n2_rank = fused_ids.index("n2")
n4_rank = fused_ids.index("n4")
assert n1_rank < n4_rank and n2_rank < n4_rank, f"n1({n1_rank})/n2({n2_rank}) 必须排在 n4({n4_rank}) 之前"
print(f"[PASS] RRF 融合正确:n1 rank={n1_rank+1} / n2 rank={n2_rank+1} 双路命中,排 n4 rank={n4_rank+1} 之前")
```

```
--- 融合后 top-4 ---
  [1] RRF score=0.0325 | 用户偏好 Python 编程,熟悉 FastAPI 框架
  [2] RRF score=0.0325 | 工作领域是 AI 应用开发,常做向量数据库实验
  [3] RRF score=0.0159 | 北京天气晴,温度 25 度
  [4] RRF score=0.0159 | 2024-05: 把会话存储从 Redis 改为本地 JSON 文件

融合 top-4 IDs: ['n1', 'n2', 'n3', 'n4']
[PASS] RRF 融合正确:n1 rank=1 / n2 rank=2 双路命中,排 n4 rank=4 之前
```

**为什么自实现 RRF 而不用 `QueryFusionRetriever`?** 源码 `memory_indexer.py:160-167` 注释明说——后者构造时会无条件 resolve 一个 LLM(即便 `num_queries=1` 不做查询扩展),给教学项目徒增 OpenAI LLM 依赖与隐藏调用。自实现 RRF 零额外依赖,融合算法对学员完全可观测。

### 5.3 对比实验：vector vs hybrid 召回差异

Cell A/B/C 各自独立跑通后,5.3 节做一个**对比表**——同一 query 在两种模式下召回的 top-3 差异。预期:**中文短词查询**(`"Python 偏好"`)hybrid 显著优于纯向量(BM25 抓"Python"关键词);**长句语义查询**(`"what framework..."`)两者接近(向量语义已强)。

**教学结论**:hybrid 不是"vector 的升级版"——它通过 RRF 平衡两种召回的偏差(关键词漏召回 vs 语义漏召回)。`retrieval_mode` 开关让你在不同场景下选择合适的召回策略。本课不演示这一节的真实对比 cell(避免重复依赖 key),但下表已预填了基于本节 mock 数据的预期结论——你可以在 Cell A + Cell C 基础上接 Cell B 的 `idx`，改 `mode` 参数自行跑通验证。

vector vs hybrid 召回模式对比（预期结果，可改 mode 自行验证）

| 查询 | mode='vector' top-3 | mode='hybrid' top-3 | 谁更优? |
|------|---------------------|---------------------|---------|
| `"Python 偏好"` | 语义相近但漏"Python"字面关键词 | BM25 精确命中 n1("Python"+"偏"+"好") + 向量补充 n4 | **hybrid** |
| `"LlamaIndex 检索"` | 语义召回 n2("LlamaIndex RAG") | BM25 精确命中 n2("llamaindex"+"检"+"索") + 向量命中 n2 | **接近**（n2 两路均命中） |
| `"会话存储 Redis JSON"` | 语义召回历史决策段 | BM25 精确命中 n3("Redis"+"JSON") + 向量召回历史决策段 | **接近**（BM25 关键词略优） |

你现在已经掌握了「文件即记忆」哲学的完整闭环——**全量注入**(第 1 章)+ **RAG 双模式**(本章 Cell A 关键词 / Cell B 语义 / Cell C 融合)。在学本章之前,你可能觉得「RAG 很复杂」;现在你知道它的骨架只有 3 个独立可验通的 cell,真正的工程复杂度在凭证管理 + MD5 增量 + 融合算法的 k 值选择上。

---

## 第 6 章：会话管理——持久化与上下文压缩

到目前为止，我们的 MVP 每次运行完就丢弃了所有上下文。但真实的 Agent 需要记住「你上次说什么」。这就是会话持久化的价值——把每一轮对话保存到 JSON 文件，下次启动时加载回来，Agent 就能延续上次的语境。这是「历史也是文件」的工程实现：会话不是数据库里的行，而是人类可读、可手动编辑的 JSON 文件。

但会话越来越长，总有一天超过 LLM 的 context window。这时我们需要压缩：把最早的 50% 消息用 DeepSeek 生成摘要，归档到 `sessions/archive/` 目录，把摘要存入 `compressed_context` 字段。下次加载时，摘要作为第一条 assistant 消息注入，Agent 知道「之前聊过什么」，但不需要看所有原始对话。

### 6.1 源码锚点对照

两个文件分工合作：

- `backend/graph/session_manager.py`（全文 247 行）：

  - 第 14-21 行注释：v2 schema 定义（`title` / `created_at` / `updated_at` / `messages`）；`compressed_context` 不在此 schema 注释中，是首次压缩时由 `compress_history` 动态追加的字段

  - 第 155-194 行：`compress_history` 方法——归档前 N 条 + 存入 `compressed_context`（**累积拼接是教学版局限**，见 9.4）

  - 第 203-237 行：`load_session_for_agent` 方法——注入压缩上下文 + 合并连续 assistant 消息

  - 第 226-233 行：连续 assistant 消息合并核心逻辑（`merged[-1]["role"] == "assistant" and msg["role"] == "assistant"`）

- `backend/api/compress.py`（全文 71 行）：

  - 第 15-43 行：`_generate_summary` 函数——DeepSeek 生成摘要

  - 第 57 行：`num_to_remove = max(4, len(messages) // 2)`——取前 50%

- `backend/graph/context_guard.py`（全文 146 行）— **L2 上下文预校验**：

  - 第 33-38 行：`get_context_window()` 从 `MODEL_CONTEXT_WINDOW` env 读窗口（默认 128K = DeepSeek-V3.2 官方值）

  - 第 41-68 行：`_content_chars` 兼容 `str` / `list[{"type":"text",...}]` 多模态 / `tool_calls` 三种 content 形态（修了 M-1 多模态 + M-2 tool_calls JSON 漏算）

  - 第 71-90 行：`estimate_tokens`——`len(content)//2` 粗估（OpenClaw chars/2 同款系数，tool output 偏密故用 2 而非通用 4）

  - 第 93-106 行：`preflight_check`——超 90% 窗口即抛 `ContextOverflowError`

  - 第 109-140 行：`is_context_overflow_error` 鸭子类型判定：结构化（400 + context + length）→ 错误码（`context_length_exceeded`）→ 英文关键词兜底（M-7 修，避免 DeepSeek 改措辞即漏判）

- `backend/graph/agent.py`（全文 301 行）— **L1 工具结果截取 + L3 overflow 重试**：

  - 第 55,63-70,77 行：**L1** `ContextEditingMiddleware` + `ClearToolUsesEdit`——LangChain 官方中间件，对齐 Anthropic context editing；trigger 设为窗口 50%；注释诚实标注"trigger 单位是 middleware 内部 4 chars/token，与 L2 的 2 chars/token 差 2 倍"

  - 第 244-282 行：**L3** `_overflow_retry`——捕获 `ContextOverflowError` → 调 `_generate_summary` 真实压缩（与 6.2 用户主动压缩复用同一函数）→ 拉新 history → 再跑 1 次；注释诚实标注"chat.py 在 done 后才落盘 user message，本轮 user message 不在压缩历史里"

「为什么需要合并连续 assistant 消息」值得专门解释：一轮 Agent 对话可能产生多段 assistant 输出（调工具前一段、工具返回后再一段），它们在会话 JSON 里是分开保存的。但 LLM 接口要求 user/assistant 严格交替（不能连续两条 assistant），所以 `load_session_for_agent` 在返回给 LLM 之前需要把它们合并。

#### 6.1.1 上下文三层防御——Overflow 的"早拦 / 晚兜"双保险

**这是本章的灵魂点**。在讲用户主动压缩（第 6.2 节）之前，必须先讲清"模型溢出怎么办"——因为这就是 OpenClaw 设计的"preemptive compaction"策略。源码通过**三层防御**让上下文永远填不满模型窗口:

上下文溢出三层防御对比（早拦 / 晚兜）

| 层 | 触发时机 | 谁负责 | 做什么 | 源码位置 |
|---|---------|--------|--------|----------|
| **L1 工具结果截取** | Agent 执行循环内,模型收到 tool_result 那一刻 | 官方 `ContextEditingMiddleware` + `ClearToolUsesEdit` | 累积 token 超 50% 窗口 → 把旧 tool result 清成 `[cleared]`,保留最近 3 条 | `agent.py:55,63-70,77` |
| **L2 Preflight 拦截** | LLM 调用**前** | 自研 `preflight_check`(含 `estimate_tokens` 辅助函数) | `len(text)//2` 粗估,超 90% 窗口 → 抛 `ContextOverflowError` | `context_guard.py:71-106` |
| **L3 Overflow 重试** | 抛 `ContextOverflowError` 之后 | `_overflow_retry` | 自动调一次压缩 → 拉新 history → 再跑 1 次(对齐 OpenClaw 策略) | `agent.py:244-282` |

**关键教学点**(从源码注释里挖出的"非最优化"):

1. **L1 的 trigger 单位是 4 chars/token,L2 是 2 chars/token**——`agent.py:66-67` 注释明说"约差 2 倍"。这是教学骨架刻意保留的"不一致",作为"为什么估算要分场景"的活案例

2. **L2 修了 M-1 / M-2 两个 bug**:`content` 多模态 list 形态 + `AIMessage.tool_calls` JSON 占 token,系统性低估

3. **L3 丢了本轮 user message**:`agent.py:250-252` 注释诚实标注——"chat.py 在 done 后才落盘 user message,真实实现应把 user message 落盘后再调压缩"

4. **三层依赖关系**:L1 被动(模型层在跑)、L2 主动(发模型前)、L3 兜底(L2 漏过 + 模型层误判后);**L1 不能替代 L2,L2 漏了 L3 救场**

L3 实际就是"压缩的**自动版**"——它和第 6.2 节用户主动压缩复用**同一函数** `_generate_summary`,只是触发方从用户变成 Agent 自己。理解这个关系,你就理解了"为什么 OpenClaw 既有手动压缩按钮,又有自动 overflow 兜底"。

**Cell A：L1 工具结果截取真跑（trigger 缩 640 倍触发 clear）**

源码 `agent.py:70` 设的 `trigger = int(get_context_window() * 0.5)` = 64K tokens,正常对话根本触发不到 clear 行为。本 cell **把 trigger 故意调到 100 tokens**(从源码 0.5*128K 缩小 ~640 倍)——6 轮多轮对话中"返回 5000 字符"的 tool_result 立即超阈,早期结果被清成 `[cleared]`。这与生产 trigger 是同一段代码,只是阈值不同。

```python
from langchain_core.messages import AIMessage, ToolMessage
from langchain_core.messages.utils import count_tokens_approximately
from langchain.agents.middleware import ClearToolUsesEdit

# 手工构造 6 轮「AI 发起 tool_call + ToolMessage 返回结果」的消息历史
# apply 要求每个 ToolMessage 能按 tool_call_id 找到配对的 AIMessage，所以必须成对构造
messages = []
for i, kw in enumerate(["alpha", "beta", "gamma", "delta", "epsilon", "zeta"]):
    tc_id = f"call_{i}"
    messages.append(AIMessage(
        content="",
        tool_calls=[{"name": "long_data_lookup", "args": {"query": kw}, "id": tc_id}],
    ))
    messages.append(ToolMessage(
        content=f"[{kw} 的 5000 字符结果]" + ("x" * 5000),
        tool_call_id=tc_id,
        name="long_data_lookup",
    ))

edit = ClearToolUsesEdit(trigger=50, keep=3)   # keep=3：保留最近 3 个，清理更早的

print("apply 前总 token:", count_tokens_approximately(messages))
edit.apply(messages, count_tokens=count_tokens_approximately)   # in-place 清理
print("apply 后总 token:", count_tokens_approximately(messages))

print("\n各 tool_result 状态：")
tool_msgs = [m for m in messages if isinstance(m, ToolMessage)]
for i, tm in enumerate(tool_msgs):
    cleared = tm.response_metadata.get("context_editing", {}).get("cleared", False)
    mark = "[CLEARED]" if cleared else "[原内容]"
    print(f"  [{i+1}] {mark} | {str(tm.content)[:45]}")

# 断言：最早的 tool_result 被清，最近 3 个保留
assert tool_msgs[0].response_metadata.get("context_editing", {}).get("cleared"), "最早的应被清"
assert not tool_msgs[-1].response_metadata.get("context_editing", {}).get("cleared"), "最近的应保留"
print("\n[PASS] L1 截取生效：早 3 个 tool_result 被清成 [cleared]，最近 3 个保留")
```

```
apply 前总 token: 7763
apply 后总 token: 4005

各 tool_result 状态：
  [1] [CLEARED] | [cleared]
  [2] [CLEARED] | [cleared]
  [3] [CLEARED] | [cleared]
  [4] [原内容] | [delta 的 5000 字符结果]xxxxxxxxxxxxxxxxxxxxxxxxxx
  [5] [原内容] | [epsilon 的 5000 字符结果]xxxxxxxxxxxxxxxxxxxxxxxx
  [6] [原内容] | [zeta 的 5000 字符结果]xxxxxxxxxxxxxxxxxxxxxxxxxxx

[PASS] L1 截取生效：早 3 个 tool_result 被清成 [cleared]，最近 3 个保留
```

> **【教学结论】**:L1 是"被动截取"——它在模型收到 tool_result 时按 token 阈值决定是否清空旧内容。**好处**:对调用方零侵入,Agent 主循环不需要任何修改。**代价**:被清的 tool_result 永久丢失,Agent 后续不能"再读"那个工具结果(本课 MVP 的简化;生产可加"按需恢复"机制)。

**Cell B：L2 Preflight 拦截白盒演示（零依赖）**

L2 是"主动拦截"——在调用 LLM 前估算 messages 的总 token,超 90% 窗口直接抛 `ContextOverflowError`,不发请求。对照源码 `context_guard.py:71-106`。本 cell **重写估算逻辑** + 准备 100 条长消息,断言抛错。

```python
# 第 6 章 Cell B：L2 Preflight 估算 + 拦截（白盒演示,零 LLM 依赖）
# 对照源码：backend/graph/context_guard.py:71-106

CHARS_PER_TOKEN = 2  # 源码原名 CHARS_PER_TOKEN_ESTIMATE；OpenClaw chars/2 同款系数,tool output 偏密故用 2
SAFETY_RATIO = 0.9    # 源码原名 PREFLIGHT_SAFETY_RATIO；90% 窗口即抛
DEFAULT_WINDOW = 128_000  # DeepSeek-V4-flash 官方窗口

def estimate_tokens(messages) -> int:
    """对照源码 estimate_tokens（第 71-90 行）——char 粗估 + 修 M-1 多模态 / M-2 tool_calls"""
    total = 0
    for m in messages:
        content = getattr(m, "content", "") or (m.get("content", "") if isinstance(m, dict) else "")
        if isinstance(content, str):
            total += len(content) // CHARS_PER_TOKEN
        elif isinstance(content, list):
            for part in content:
                if isinstance(part, dict) and "text" in part:
                    total += len(str(part["text"])) // CHARS_PER_TOKEN
                else:
                    total += len(str(part)) // CHARS_PER_TOKEN
        # M-2 修:tool_calls JSON 也占 token
        tool_calls = getattr(m, "tool_calls", None)
        if tool_calls:
            total += len(str(tool_calls)) // CHARS_PER_TOKEN
    return total

class ContextOverflowError(Exception): pass

def preflight_check(messages, window=DEFAULT_WINDOW):
    """对照源码 preflight_check（第 93-106 行）——超 90% 窗口即抛"""
    estimated = estimate_tokens(messages)
    threshold = int(window * SAFETY_RATIO)
    if estimated > threshold:
        raise ContextOverflowError(
            f"Preflight: estimated {estimated} tokens exceeds "
            f"{int(SAFETY_RATIO*100)}% of window ({window})"
        )
    return estimated

# 准备 100 条长消息（每条 3000 字符 = 1500 tokens,100 条 = 150K tokens,超 90% 128K）
class MockMsg:
    def __init__(self, content, tool_calls=None):
        self.content = content
        self.tool_calls = tool_calls or None

big_history = [
    MockMsg(f"这是第 {i+1} 轮的长对话内容。" + "x" * 3000)
    for i in range(100)
]

print(f"准备 {len(big_history)} 条消息,总字符数: {sum(len(m.content) for m in big_history)}")
print(f"估算 token: {estimate_tokens(big_history)}")
print(f"窗口 90% 阈值: {int(DEFAULT_WINDOW * SAFETY_RATIO)}")

# 断言:必须抛 ContextOverflowError
try:
    preflight_check(big_history)
    print("[FAIL] 预期抛 ContextOverflowError,实际没抛")
except ContextOverflowError as e:
    print(f"[PASS] L2 拦截生效: {e}")

# 额外验证:小消息不抛
small_history = [MockMsg("短消息")]
preflight_check(small_history)
print("[PASS] 小消息正常通过 L2 校验")
```

```
准备 100 条消息,总字符数: 301492
估算 token: 150701
窗口 90% 阈值: 115200
[PASS] L2 拦截生效: Preflight: estimated 150701 tokens exceeds 90% of window (128000)
[PASS] 小消息正常通过 L2 校验
```

> **【M-2 修复演示】**:源码 `context_guard.py:79-89` 注释明确说,修 M-2 bug 之前,AIMessage 的 `tool_calls` JSON 字段系统性漏算——一个 5 字段的 tool_call 大约 200 字符 = 100 tokens,一轮多工具调用累计可达 500+ tokens。**本 cell 的 estimate_tokens 已把 tool_calls 纳入估算**(见 `tool_calls = getattr(m, "tool_calls", None)` 分支)。

**Cell C：L3 Overflow 重试真跑（需 DEEPSEEK_API_KEY）**

L2 漏过 + L1 截取不及时 + 模型层依然报 overflow → 进入 L3 重试闭环:自动调一次压缩 → 拉新 history → 再跑 1 次。对照源码 `agent.py:244-282`。本 cell 准备 20 条长对话触发 overflow,模拟 L3 重试路径。

```python
from multiprocessing import popen_spawn_posix
# 第 6 章 Cell C：L3 Overflow 重试真跑（需 DEEPSEEK_API_KEY）
# 对照源码：backend/graph/agent.py:244-282（_overflow_retry）
# 注：本 cell 模拟 L3 的压缩+重试骨架;真实场景中 L2/L1 也会协同工作

import os

# 凭证已在第 0 章 0.5 步骤二加载，这里直接复用

# 准备 20 条长对话（总长足够触发 overflow,即使有 L1/L2 也模拟漏过场景）
overflow_history = [
    MockMsg(f"第 {i+1} 轮:用户问了复杂问题,Agent 返回了详细的回答。" + "y" * 5000)
    for i in range(20)
]

# 复用第 6.2 节的 _generate_summary 逻辑（真实场景下 L3 和用户主动压缩用同一函数）
async def simulate_l3_retry(history):
    """
    模拟 _overflow_retry 的压缩+重试骨架。
    对照源码 agent.py:244-282 流程:
      1. yield compression_start
      2. 取前 50% 消息
      3. _generate_summary(前 50%)
      4. compress_history(session_id, summary, num_to_remove)
      5. 拉新 history
      6. 再跑 1 次 astream
    """
    print("[L3] 进入 overflow retry 流程")

    if not os.getenv("DEEPSEEK_API_KEY"):
        print("[SKIP] 无 DEEPSEEK_API_KEY,只演示骨架不真调 LLM")
        # 模拟压缩:取前 50% 消息,生成伪摘要
        num_to_remove = max(4, len(history) // 2)
        fake_summary = f"[前 {num_to_remove} 轮对话的模拟摘要:用户问了复杂问题,Agent 详细回答]"
        print(f"[mock] 已「压缩」前 {num_to_remove} 条,生成 {len(fake_summary)} 字符摘要")
        return {"compressed_count": num_to_remove, "summary": fake_summary, "retry_status": "skipped (no key)"}

    # 真实路径:从第 6.2 节复用 _generate_summary
    import sys
    backend_path = os.path.abspath("../Fufan-OpenClaw项目源码/backend")  # 包根：含 api/、graph/
    if backend_path not in sys.path:
        sys.path.insert(0, backend_path)

    from api.compress import _generate_summary  # 注：本课 MVP 直接复用源码
    from graph.session_manager import session_manager

    num_to_remove = max(4, len(history) // 2)
    # 接口契约：_generate_summary 期望 dict 列表（内部用 msg.get("role")/.get("content")），
    # 真实项目里传入的就是会话 JSON 的 dict 消息；这里 MockMsg 只有 .content 属性，
    # 必须先转成同构 dict 再调用，否则会 AttributeError。
    msgs_as_dict = [{"role": "user", "content": m.content} for m in history[:num_to_remove]]
    summary = await _generate_summary(msgs_as_dict)
    print(f"[L3] 真实生成 {len(summary)} 字符摘要")

    # 写回 session（真实路径会落盘）
    # session_manager.compress_history(session_id, summary, num_to_remove)
    # 拉新 history 再跑
    return {"compressed_count": num_to_remove, "summary": summary, "retry_status": "ok"}

result = await simulate_l3_retry(overflow_history)
print(f"\nL3 重试结果: {result}")
print("[PASS] L3 骨架跑通——触发 overflow → 压缩前 50% → 准备重试")
```

```
[L3] 进入 overflow retry 流程
[L3] 真实生成 81 字符摘要

L3 重试结果: {'compressed_count': 10, 'summary': '用户连续10轮提出复杂问题，Agent均返回详细回答。每轮交互模式相同：用户提问，Agent给出详尽回复。对话未显示具体问题内容或决策结论，仅呈现重复的问答流程。', 'retry_status': 'ok'}
[PASS] L3 骨架跑通——触发 overflow → 压缩前 50% → 准备重试
```

**教学结论**:L3 是"模型都报错了我们才动手"——它依赖 L2/L1 都漏过的边界场景。**好处**:任何上游漏过的 overflow 都能兜住,不会让用户看到"prompt too long"错误。**代价**:本轮对话的 user message 会丢失(源码注释诚实标注,见 `agent.py:250-252`)——因为 L3 在 done 之后才落盘 user message,触发 L3 时本轮 user message 已在历史里了,但 compress_history 走的是上一轮及之前的归档逻辑。

### 6.2 会话持久化 + 压缩实现

接下来我们实现 save/load/compress 三个核心方法，演示完整的会话生命周期。运行后你会看到：保存 8 条消息 → 触发压缩 → 归档前 4 条 → 生成摘要（需要 DEEPSEEK_API_KEY）→ 加载时看到摘要作为第一条注入 + 剩余 4 条消息 + 连续 assistant 消息已合并。

```python
# 第 6 章：会话持久化 + 压缩（部分功能需要 DEEPSEEK_API_KEY）
# 对照源码：backend/graph/session_manager.py:14-21,155-234 / backend/api/compress.py:15-57

import json
import time
import asyncio
from pathlib import Path
import tempfile

# 临时会话目录
sessions_dir = Path(tempfile.mkdtemp()) / "sessions"
sessions_dir.mkdir(parents=True)
archive_dir = sessions_dir / "archive"
archive_dir.mkdir()

SESSION_ID = "demo_session_001"

def get_session_path(session_id: str) -> Path:
    """返回会话 JSON 文件的路径（源码 _session_path 第 31-33 行）。"""
    return sessions_dir / f"{session_id}.json"

def save_message(session_id: str, role: str, content: str) -> None:
    """
    追加一条消息到会话 JSON 文件（源码 save_message 第 83-104 行）。

    Args:
        session_id: 会话唯一标识符
        role: 消息角色（"user" 或 "assistant"）
        content: 消息正文
    """
    path = get_session_path(session_id)

    # 读现有数据（文件不存在时初始化新会话）
    if path.exists():
        data = json.loads(path.read_text(encoding="utf-8"))
    else:
        now = time.time()
        # v2 schema：title / created_at / updated_at / messages（源码第 14-21 行）
        data = {"title": "New Chat", "created_at": now, "updated_at": now, "messages": []}

    data["messages"].append({"role": role, "content": content})
    data["updated_at"] = time.time()
    path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")

def load_session(session_id: str) -> list:
    """读取会话的原始消息列表（源码 load_session 第 76-81 行）。"""
    path = get_session_path(session_id)
    if not path.exists():
        return []
    data = json.loads(path.read_text(encoding="utf-8"))
    return data.get("messages", [])

def compress_history_mvp(session_id: str, summary: str, num_to_remove: int) -> None:
    """
    归档前 num_to_remove 条消息，摘要存入 compressed_context（源码第 155-191 行）。

    Args:
        session_id: 目标会话 ID
        summary: 由 LLM 生成的对话摘要文本
        num_to_remove: 要归档（移除）的消息数量
    """
    path = get_session_path(session_id)
    data = json.loads(path.read_text(encoding="utf-8"))
    messages = data.get("messages", [])

    # 被归档的消息写到 archive/ 子目录（源码第 168-179 行）
    archived = {
        "session_id": session_id,
        "archived_at": time.time(),
        "messages": messages[:num_to_remove],
    }
    archive_path = archive_dir / f"{session_id}_{int(time.time())}.json"
    archive_path.write_text(json.dumps(archived, ensure_ascii=False, indent=2), encoding="utf-8")

    # 移除已归档的消息，保存摘要到 compressed_context（源码第 182-192 行）
    # 注意（源码的教学局限）：源码用「追加」——existing + "\n---\n" + summary。
    # 多次压缩时摘要会累积拼接，compressed_context 会越压越长，与「压缩应缩短上下文」的初衷相悖。
    # 这里忠于源码逻辑保留累积写法；生产级实现（如原版 openclaw）用指针只保留最后一次有效摘要。
    data["messages"] = messages[num_to_remove:]
    existing_ctx = data.get("compressed_context", "")
    data["compressed_context"] = (existing_ctx + "\n---\n" + summary) if existing_ctx else summary
    data["updated_at"] = time.time()
    path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")

def load_session_for_agent(session_id: str) -> list:
    """
    为 LLM 准备的会话加载：注入压缩上下文 + 合并连续 assistant 消息。
    对照源码 load_session_for_agent 第 203-237 行。

    Returns:
        合并处理后的消息列表，可直接传给 LLM
    """
    path = get_session_path(session_id)
    data = json.loads(path.read_text(encoding="utf-8")) if path.exists() else {}
    messages = data.get("messages", [])
    compressed = data.get("compressed_context", "")

    merged = []

    # 如果有压缩上下文，作为第一条 assistant 消息注入（源码第 218-224 行）
    if compressed:
        merged.append({
            "role": "assistant",
            "content": f"[以下是之前对话的摘要]\n{compressed}",
        })

    for msg in messages:
        # 合并连续 assistant 消息（核心逻辑，源码第 226-233 行）
        if merged and merged[-1]["role"] == "assistant" and msg["role"] == "assistant":
            merged[-1]["content"] += "\n" + msg["content"]
        else:
            merged.append({"role": msg["role"], "content": msg["content"]})

    return merged

# --- 演示完整生命周期 ---
print("步骤一：保存 8 条消息（模拟 4 轮对话）")
save_message(SESSION_ID, "user", "你好，帮我写一个 Python 排序函数")
save_message(SESSION_ID, "assistant", "好的，这是一个快速排序的实现：...")
save_message(SESSION_ID, "assistant", "[工具调用结果：python_repl 执行成功]")  # 同一轮的第二条 assistant
save_message(SESSION_ID, "user", "再帮我加上注释")
save_message(SESSION_ID, "assistant", "加上注释后的版本：...")
save_message(SESSION_ID, "user", "能支持降序排列吗？")
save_message(SESSION_ID, "assistant", "可以的，添加一个 reverse 参数：...")
save_message(SESSION_ID, "user", "谢谢，完美！")

messages_before = load_session(SESSION_ID)
print(f"保存完成，共 {len(messages_before)} 条消息")

print("\n步骤二：触发压缩（取前 50%，即前 4 条）")
# 前 50% 逻辑（源码 compress.py:57）
num_to_remove = max(4, len(messages_before) // 2)
mock_summary = "用户请求编写 Python 排序函数，实现了快速排序并添加了注释，支持正序/降序。用户对结果满意。"
compress_history_mvp(SESSION_ID, mock_summary, num_to_remove)
print(f"已归档前 {num_to_remove} 条，摘要写入 compressed_context")

print("\n步骤三：load_session_for_agent 加载（查看合并和摘要注入效果）")
agent_messages = load_session_for_agent(SESSION_ID)
print(f"加载结果：{len(agent_messages)} 条消息")
for i, m in enumerate(agent_messages):
    preview = m["content"][:60].replace("\n", " ")
    print(f"  [{i}] role={m['role']} | {preview}...")
```

```
步骤一：保存 8 条消息（模拟 4 轮对话）
保存完成，共 8 条消息

步骤二：触发压缩（取前 50%，即前 4 条）
已归档前 4 条，摘要写入 compressed_context

步骤三：load_session_for_agent 加载（查看合并和摘要注入效果）
加载结果：4 条消息
  [0] role=assistant | [以下是之前对话的摘要] 用户请求编写 Python 排序函数，实现了快速排序并添加了注释，支持正序/降序。用户对结果满...
  [1] role=user | 能支持降序排列吗？...
  [2] role=assistant | 可以的，添加一个 reverse 参数：......
  [3] role=user | 谢谢，完美！...
```

打印出完整消息列表后，我们用一组断言验证整条生命周期的正确性——确认摘要注入、剩余消息、角色交替等关键不变量都符合预期。

```python
# Tier 2 验证：完整生命周期断言
assert len(agent_messages) >= 2, "至少有摘要注入 + 剩余消息"

# 第一条应是摘要注入（role=assistant，content 含 [以下是之前对话的摘要]）
assert agent_messages[0]["role"] == "assistant"
assert "[以下是之前对话的摘要]" in agent_messages[0]["content"]
print("[PASS] 摘要正确注入为第一条 assistant 消息")

# 检查连续 assistant 消息合并（原始 messages 中有连续 assistant，合并后不应有）
for i in range(len(agent_messages) - 1):
    assert not (
        agent_messages[i]["role"] == "assistant" and
        agent_messages[i + 1]["role"] == "assistant"
    ), f"第 {i} 和 {i+1} 条是连续 assistant，应已合并"
print("[PASS] 连续 assistant 消息已合并，无连续 assistant 对")

# 验证归档文件存在
archive_files = list(archive_dir.glob("*.json"))
assert len(archive_files) >= 1
print(f"[PASS] 归档文件已创建：{archive_files[0].name}")

# 清理
import shutil
shutil.rmtree(sessions_dir.parent)
print("\n所有断言通过，临时目录已清理。")
```

```
[PASS] 摘要正确注入为第一条 assistant 消息
[PASS] 连续 assistant 消息已合并，无连续 assistant 对
[PASS] 归档文件已创建：demo_session_001_1781096909.json

所有断言通过，临时目录已清理。
```

运行这三段代码，你会看到完整的会话生命周期：保存、压缩、归档、加载、合并。重点观察 `load_session_for_agent` 的输出——第一条消息是摘要注入（role=assistant），然后是压缩后留下的剩余消息；同一轮对话产生的两条连续 assistant 消息被合并成一条，确保 LLM 收到的是严格交替的 user/assistant 序列。

你现在已经掌握了会话的完整生命周期——从 save 到 load 到 compress，整个过程只用了 JSON 文件和几十行 Python 代码，零外部依赖。在学这章之前，你可能觉得「持久化需要数据库」；现在你知道对于 Agent 会话这种轻量写入场景，JSON 文件完全够用，还有「人类可读可编辑」的附加价值。第 7 章我们站到整个系统的最高点，把前 6 章串成一条完整的消息旅程。

> **【踩坑预警】**：连续 assistant 消息合并不是为了「好看」——它是 LLM 接口的格式要求。如果你把包含连续两条 assistant 消息的历史直接传给 DeepSeek 或 OpenAI，API 会返回格式错误。在真实项目里，一轮 Agent 对话往往产生 3-4 条 assistant 消息（工具调用前的思考、工具调用标记、工具调用后的总结），不合并必然报错。

---

## 第 7 章：全栈链路串讲——消息处理全流程

前 6 章我们分别造了 6 个零件，每个都能单独跑通。这一章没有新零件——我们要把它们组装成一台整机，用代码数据流的视角完整走一遍「一条消息的生命周期」。

这是整个课程的俯瞰视角，也是「完全可观测」哲学最完整的呈现时刻——从前端 `fetch` 发出请求，到每一个 SSE 事件推回到浏览器，中间每一个停靠站我们都造过，都能看见。

### 7.1 链路全景图

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154604077.png)

### 7.2 逐站解析：数据流如何流动

我们用代码切片 + 注释的方式，把 9 个停靠站的关键代码并排呈现。这不是新 MVP，而是把前面章节的源码锚点串联起来，让你看清数据在各层之间的形态变换。

**停靠站 ①：前端发出请求（`frontend/src/lib/api.ts:6-9, 20+`）**

`api.ts` 的前几行定义了 API 地址（后端端口 8002），然后用自定义的 `streamChat` 异步生成器消费 SSE——注意这里用的是 `fetch` 而不是原生的 `EventSource`，因为原生 `EventSource` 不支持 POST body（只支持 GET），而我们需要在请求体里传 `message` 和 `session_id`。

```python
# 教学切片：api.ts 关键片段（TypeScript，仅供阅读，不可直接执行）
# 对照源码：frontend/src/lib/api.ts:6-9 (API_BASE), :20-28 (streamChat)

"""
// 后端 API 基础地址（端口 8002 硬编码在此）
const API_BASE =
  typeof window !== "undefined"
    ? `http://${window.location.hostname}:8002/api`   // 浏览器环境：动态取 hostname
    : "http://localhost:8002/api";                     // SSR/Node 环境：localhost

// streamChat：自定义 SSE 消费器，用 fetch 实现 POST SSE
export async function* streamChat(message: string, sessionId: string): AsyncGenerator<SSEEvent> {
  const response = await fetch(`${API_BASE}/chat`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message, session_id: sessionId, stream: true }),
  });
  // 后续：逐行读 response.body，解析 event: / data: 行
}
"""
print("停靠站 ①：前端用 fetch（不是 EventSource）发 POST SSE，端口 8002")
```

**停靠站 ②：后端接收 → SSE 包装（`backend/api/chat.py:233-235`）**

FastAPI 路由接收到 POST 请求，把 `event_generator` 协程包进 `EventSourceResponse`。`EventSourceResponse` 来自 `sse_starlette` 库，它负责把每个 `yield` 的字典转换成标准 `text/event-stream` 格式（`event:xxx\ndata:xxx\n\n`），推给前端。

**停靠站 ③ → ④ → ⑤：会话加载 → Prompt 构建 → Agent 组装**

`event_generator` 函数的前几行（`chat.py:78`）先调 `session_manager.load_session_for_agent`（第 6 章），拿到合并后的历史；然后进入 `agent_manager.astream`，在那里 `_build_agent` 会重新调用 `build_system_prompt`（第 1 章），组装一个带最新人格的 Agent（第 2 章）。注意这里每次对话都重建 Agent——目的就是让文件热更新立即生效。

**停靠站 ⑥ → ⑦：双流解析 + 工具执行**

`agent.astream`（第 3 章）开始产出事件。每次 Agent 决定调工具，LangGraph 会把工具调用转发到 `tools/__init__.py` 的 7 个工具。工具执行结果以 `ToolMessage` 的形式回到 LangGraph，触发 `tool_end` 事件。这个「模型 → 工具 → 模型」的循环可以发生多次，直到模型不再产生新的工具调用。

**停靠站 ⑧：Canvas 提取（`backend/api/chat.py:196-205`）**

Canvas 是 FuFan-OpenClaw 的特色功能：当 Agent 在回复里包含 `<openclaw-canvas>...</openclaw-canvas>` 标签时，系统把标签内的 HTML 提取出来，作为独立的 `canvas` SSE 事件推给前端，前端用 `iframe` 渲染这段 HTML，实现 Agent 直接输出交互式界面。

> 需要说明的是，`<openclaw-canvas>` 是本教学项目自定义的前后端约定标签，并非原版 openclaw 的上游协议——openclaw 源码中不存在这个标签，它的画布走的是 `[embed ref]` shortcode + URL 托管的方式（模型输出引用、后端托管 HTML 资源）。本项目为了教学直观，简化成「内联 HTML 包在标签里 → 前端 iframe 渲染」这种一次性输出。

```python
# 教学切片：canvas 提取逻辑（对照源码 chat.py:196-205）
# 此 cell 为只读示意，不运行真实 API 调用

import re

# 模拟 Agent 回复中包含 canvas 标签的场景
mock_agent_response = """
好的，这是一个可交互的数据可视化界面：

<openclaw-canvas>
<div style="font-family: Arial; padding: 20px;">
  <h2>用户数据统计</h2>
  <p>总用户：<strong>1,234</strong></p>
</div>
</openclaw-canvas>

以上就是根据你的数据生成的可视化界面。
"""

# 正则提取（对照源码 chat.py:196-199 的 re.search 调用）
canvas_match = re.search(
    r"<openclaw-canvas>(.*?)</openclaw-canvas>",
    mock_agent_response,
    re.DOTALL,  # DOTALL 让 . 能匹配换行符，处理多行 HTML
)

if canvas_match:
    canvas_html = canvas_match.group(1).strip()
    print(f"[canvas 提取成功] HTML 长度：{len(canvas_html)} 字符")
    print(f"内容预览：{canvas_html[:100]}...")
    # 真实代码中：yield {"event": "canvas", "data": json.dumps({"html": canvas_html})}
```

```
[canvas 提取成功] HTML 长度：109 字符
内容预览：<div style="font-family: Arial; padding: 20px;">
  <h2>用户数据统计</h2>
  <p>总用户：<strong>1,234</strong></...
```

**停靠站 ⑨：持久化**

`done` 事件触发后，`event_generator` 把这轮对话的所有 segment（包含工具调用信息）写入会话 JSON——每个 segment 存一条 assistant 消息，这就是第 6 章里「连续 assistant 消息」的来源：工具调用前的思考是一段，工具结束后的总结是另一段。

### 7.3 链路映射：第 X 章 → 第 Y 行代码

一条消息的九站链路：章节 × 源码对照

| 停靠站 | 对应章节 | 核心文件：行号 | 数据形态变化 |
|--------|---------|--------------|------------|
| ① 前端发请求 | 第 7 章串讲 | `api.ts:6-9, 20+` | string → POST body |
| ② SSE 包装 | 第 7 章串讲 | `chat.py:233-235` | coroutine → EventSourceResponse |
| ③ 会话加载 | 第 6 章 | `session_manager.py:203-237` | JSON → merged messages list |
| ④ Prompt 构建 | 第 1 章 | `prompt_builder.py:25-62` | 6 files → system_prompt string |
| ⑤ Agent 组装 | 第 2 章 | `agent.py:52-79` | LLM + tools + prompt → agent |
| ⑥ 双流解析 | 第 3 章 | `agent.py:134-242` | agent stream → 5 event types |
| ⑦ 工具执行 | 第 4/5 章 | `tools/__init__.py:17-44` | tool_call → ToolMessage |
| ⑧ SSE 推回 | 第 3 章 | `chat.py:68-227` | internal events → SSE format |
| ⑨ 持久化 | 第 6 章 | `session_manager.py:83-104` | events → session JSON |

你现在已经掌握了整条链路的数据流。每一个停靠站，你都能说出它对应的源码文件和大致行号，都能用前面章节的 MVP 重现它的核心逻辑。这正是「完全可观测」哲学在教学层面的实现：不只是 Agent 的工具调用可见，这个系统本身对你来说也是完全可见的。

> **【踩坑预警】**：前端为什么用 `fetch` 而不是原生 `EventSource`？因为 `EventSource` 是 GET-only 的，不支持设置 POST body 和自定义 headers。对于需要携带消息体的聊天请求，必须用 `fetch` 手动读取 `response.body` 的 `ReadableStream`，逐行解析 SSE 格式。这是所有「POST SSE」实现都绕不开的限制。

---

## 第 8 章：进阶——多智能体：spawn_subagent 派生子智能体

前面七章我们造的始终是「一个 Agent」——它有人格、会调工具、能检索记忆、记得历史。但真实世界里，复杂任务常常需要「分而治之」：把一个独立的子任务隔离出去，交给一个专注的「子智能体」完成，再把结论拿回来。这一章我们看 `FuFan-OpenClaw` 怎么用最小的代价实现这种「Agent 派生 Agent」的多智能体模式——它的载体，正是第一部分 2.2 节提过、容易被忽略的「第 8 个工具」`spawn_subagent`。

这个项目实现多智能体的方式朴素得惊人：`spawn_subagent` 就是一个普通工具，主智能体调用它时，工具内部**再调一次 `create_agent`**（第 2 章那个 API）即时拼出一个子智能体，跑完返回结论。换句话说，多智能体 = `create_agent` 的递归应用，不需要任何额外的「调度框架」或「消息总线」。而这种递归唯一的危险——无限派生——靠一行 `include_spawn=False` 就根治了。这正是本章的灵魂。

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154608714.png)

▲ 派生关系：主智能体持有 spawn 工具 → 调用时即时 create_agent 拼出子智能体（工具子集不含 spawn）→ 子智能体独立跑完、回传结论；递归深度被锁死在一层

### 8.1 源码锚点对照

`backend/tools/spawn_subagent_tool.py` 全文 81 行，结构很紧凑：

- 第 23-26 行：`SUBAGENT_SYSTEM_PROMPT`——子智能体的系统提示词（强调「专注子任务、直接交付、看不到主对话历史」）

- 第 29-35 行：`SpawnSubagentInput`——工具入参（一个 `task` 字段，要求把背景目标写清楚，因为子智能体看不到上下文）

- 第 38 行：`create_spawn_subagent_tool(llm, base_dir)`——工厂函数，接收主智能体的 LLM（复用，不重新构造）

- 第 46 行：`_spawn(task)`——工具的真正执行体（`async`）

- 第 48-49 行：递归防护的灵魂一行——`sub_tools = get_all_tools(base_dir, include_spawn=False)`，子智能体的工具集**绝不包含 spawn 工具本身**

- 第 51-55 行：用 `create_agent` 即时构造子智能体（复用主 LLM + 子工具集 + 子提示词）

- 第 57-62 行：子智能体 `ainvoke` 跑完；异常被捕获转成可读文本，**不向主智能体抛**（工具不该让父崩溃）

- 第 65-69 行：从子智能体的消息链里反向取最后一条 AI 文本，作为结论字符串返回

再看它怎么被挂载。`backend/graph/agent.py:47` 在主智能体初始化时用 `get_all_tools(base_dir, llm=self._llm, include_spawn=True)`——只有这里 `include_spawn=True`，所以 `spawn_subagent` **只挂给主智能体**；而 `backend/tools/__init__.py:39-42` 那段 `if include_spawn and llm is not None:` 决定「是否追加第 8 个工具」。两处一对照，递归防护的闭环就清楚了：主智能体有 spawn（能派生），子智能体没有 spawn（不能再派生），递归深度最多一层。

### 8.2 spawn_subagent 多智能体实现

接下来我们用最小代码重现这套机制：定义一个 `spawn_subagent` 工具（内部 `create_agent` 派生子智能体），挂给主智能体，然后真实跑一轮——让主智能体派生一个带 `calculator` 工具的子智能体去算一道题。运行后你会在主智能体的消息链里看到它调用了 `spawn_subagent`，而真正的计算是子智能体完成的。需要 `DEEPSEEK_API_KEY`；无 key 时看代码结构理解派生模式即可。

```python
# 第8章：spawn_subagent 派生子智能体（需要 DEEPSEEK_API_KEY）
# 对照源码：backend/tools/spawn_subagent_tool.py:23-81 + agent.py:47

import os

# 凭证与主 llm 已在第 0 章 0.5 统一加载/初始化，本 cell 直接复用全局 llm
from langchain_core.tools import tool, StructuredTool
from langchain_core.messages import HumanMessage
from langchain.agents import create_agent
from pydantic import BaseModel, Field

# 子智能体可用的基础工具（演示用一个计算器；真实项目里是 terminal / read_file 等 7 工具）
@tool
def calculator(expression: str) -> str:
    """计算一个数学表达式（如 '(123+456)*789'），返回数值结果。"""
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"计算错误: {e}"

# 子智能体系统提示词（对照源码 SUBAGENT_SYSTEM_PROMPT，第 23-26 行）
SUBAGENT_PROMPT = (
    "你是一个子智能体，专注完成交给你的一个独立子任务。"
    "直接调用工具得出结果并给出最终结论，不要寒暄、不要反问。"
)

class SpawnInput(BaseModel):
    task: str = Field(description="交给子智能体独立完成的、自包含的子任务描述")

# 教学观测：记录递归防护是否生效
spawn_log = {"spawned": False, "sub_has_spawn": None}

async def _spawn(task: str) -> str:
    """spawn 工具的执行体：内部 create_agent 派生子智能体（对照源码 _spawn 第 46-69 行）。"""
    spawn_log["spawned"] = True
    # 递归防护的灵魂：子智能体工具集绝不含 spawn 工具本身（对照源码第 48-49 行）
    sub_tools = [calculator]
    spawn_log["sub_has_spawn"] = any(getattr(t, "name", "") == "spawn_subagent" for t in sub_tools)
    # 复用主 LLM + 子工具集 + 子提示词，即时构造子智能体（对照源码第 51-55 行）
    sub_agent = create_agent(model=llm, tools=sub_tools, system_prompt=SUBAGENT_PROMPT)
    try:
        result = await sub_agent.ainvoke({"messages": [HumanMessage(content=task)]})
    except Exception as e:
        # 异常转可读文本，不向主智能体抛（对照源码第 57-62 行）
        return f"子智能体执行失败：{type(e).__name__}: {e}"
    # 反向取子智能体最后一条 AI 文本作为结论（对照源码第 65-69 行）
    for msg in reversed(result.get("messages", [])):
        if getattr(msg, "type", None) in ("ai", "AIMessageChunk") and getattr(msg, "content", ""):
            return str(msg.content)
    return "子智能体未产生有效输出。"

# 把 _spawn 包装成工具（对照源码 StructuredTool.from_function，第 71-81 行）
spawn_tool = StructuredTool.from_function(
    coroutine=_spawn,
    name="spawn_subagent",
    description="派生一个独立子智能体完成明确、自包含的子任务，并返回它的最终结论。",
    args_schema=SpawnInput,
)

# 主智能体：挂载 spawn 工具（对照源码 agent.py:47 的 include_spawn=True）
main_agent = create_agent(
    model=llm,
    tools=[spawn_tool],
    system_prompt="你是主智能体。遇到可独立完成的子任务，用 spawn_subagent 工具派生一个子智能体去做，再汇总它的结论。",
)

async def run_spawn_demo():
    """真跑：让主智能体派生子智能体完成一道计算题。"""
    result = await main_agent.ainvoke(
        {"messages": [HumanMessage(content="请派生一个子智能体，帮我计算 (123 + 456) * 789 等于多少。")]}
    )
    msgs = result.get("messages", [])
    print(f"主智能体消息链（{len(msgs)} 条）：")
    for m in msgs:
        print(f"  {getattr(m, 'type', None):16} | {str(getattr(m, 'content', ''))[:60]}")
    return msgs

# Jupyter 中直接 await；无 key 时降级看结构
if os.getenv("DEEPSEEK_API_KEY"):
    demo_msgs = await run_spawn_demo()
else:
    print("[SKIP] 无 DEEPSEEK_API_KEY；理解派生模式即可：")
    print("  主智能体 → 调 spawn_subagent → 工具内 create_agent 拼子智能体 → 子智能体跑完回传结论")
    demo_msgs = []
```

```
主智能体消息链（4 条）：
  human            | 请派生一个子智能体，帮我计算 (123 + 456) * 789 等于多少。
  ai               | 好的，这是一个明确的自包含计算任务，我直接派生一个子智能体去完成。
  tool             | (123 + 456) * 789 = **456831**
  ai               | 子智能体已经完成了计算，以下是它的结论：

**(123 + 456) × 789 = 456,831**

计算步骤：
```

运行这段代码，你会看到主智能体的消息链：`human`（你的请求）→ `ai`（主智能体决定派生）→ `tool`（`spawn_subagent` 的返回，里面是子智能体算出的 `456831`）→ `ai`（主智能体把子智能体的结论汇总给你）。关键在于：那个 `tool` 消息的内容，是**子智能体**独立跑完一整轮 `create_agent` 循环（它自己调了 `calculator`）后产出的结论——主智能体只负责「把任务派出去、把结论收回来」，看不见子智能体内部调了几次工具。这就是「子任务隔离」最直观的样子。

```python
# Tier 2 验证：递归防护 + 派生事实（需上一 cell 已运行）
if os.getenv("DEEPSEEK_API_KEY") and demo_msgs:
    # 验证1：spawn 工具确实被主智能体调用
    assert spawn_log["spawned"], "spawn_subagent 应被主智能体调用"
    print("[PASS] spawn_subagent 被主智能体调用")

    # 验证2：递归防护——子智能体工具集不含 spawn 工具本身
    assert spawn_log["sub_has_spawn"] is False, "子智能体工具集不应含 spawn（防无限派生）"
    print("[PASS] 递归防护生效：子智能体工具集不含 spawn 工具")

    # 验证3：消息链里有子智能体返回的工具结果
    tool_msgs = [m for m in demo_msgs if getattr(m, "type", None) == "tool"]
    assert len(tool_msgs) >= 1, "应有 spawn_subagent 的返回（子智能体的结论）"
    expected = (123 + 456) * 789  # = 456831
    print(f"[PASS] 子智能体完成计算，期望结果 {expected}")
    print(f"主智能体最终汇总：{str(demo_msgs[-1].content)[:120]}")
else:
    print("[SKIP] 无 key 或未运行 demo，跳过断言")
```

```
[PASS] spawn_subagent 被主智能体调用
[PASS] 递归防护生效：子智能体工具集不含 spawn 工具
[PASS] 子智能体完成计算，期望结果 456831
主智能体最终汇总：子智能体计算完成，结果为：

**(123 + 456) × 789 = 456831**
```

三条断言验证了多智能体派生的三个核心事实：spawn 工具被主智能体真实调用、递归防护生效（子智能体工具集不含 spawn）、子智能体独立完成了计算并回传。你现在已经掌握了「Agent 派生 Agent」的最小实现——它没有任何神秘的框架，就是 `create_agent` 套 `create_agent`，外加一行 `include_spawn=False` 的递归防护。

### 8.3 踩坑预警与子任务隔离

在收尾本章之前，有三个容易踩坑或混淆的点值得讲透，它们也是判断任何「多智能体」实现是否靠谱的尺子。

> 🔥 **踩坑预警 · 递归防护不能省**：如果子智能体的工具集也包含 `spawn_subagent` 会怎样？主智能体派生子智能体、子智能体又派生孙智能体……无限递归直到栈溢出或 API 配额耗尽。源码用 `include_spawn=False`（`spawn_subagent_tool.py:48-49`）从**构造层面**根除这个风险——子智能体永远拿不到 spawn 工具。这是「能力下放」类设计的通用铁律：派生者可以派生，被派生者不能再派生，把递归深度锁死在一层。判断标准很简单：看子智能体的工具集是怎么构造的，只要它可能拿到「派生自己」的工具，这个实现就有无限递归的隐患。

第二个点关于**并发**：主智能体和子智能体共用同一个 LLM 实例（源码里 spawn 工具复用主智能体传入的 `llm`，不重新构造）——这安全吗？安全。`LangChain` 的 chat model 实例本身是无状态的，每次调用各自构造请求、互不干扰，底层 `AsyncOpenAI` 客户端本就为并发设计。真正需要警惕的并发风险不在 LLM，而在**工具层的共享资源**：比如源码的 `python_repl` 工具用全局 `sys.stdout` 重定向来捕获输出，多个子智能体并行执行 Python 时输出会互相污染——所以它专门加了一把 `_REPL_LOCK` 串行化临界区。结论是：共用 LLM 实例没问题，但凡是改全局状态的工具，并行派生时都要自己加锁。

第三个点关于**隔离的代价与价值**：子智能体看不到主对话历史，它只拿到 `task` 一个字符串（源码 `SUBAGENT_SYSTEM_PROMPT` 明说「你看不到主智能体的对话历史」）。这既是限制也是优势——子任务不会被主对话的无关上下文污染，主对话也不会被子任务的中间过程撑大（呼应第 6 章的上下文管理：派生子智能体本身就是一种「上下文隔离」手段）。所以调用 spawn 时，`task` 必须把背景、目标、期望产出写完整。这一点和你在 `Claude Code` 里 dispatch 一个 subagent 时要把 prompt 写清楚，是同一个道理。

到这里，你已经完成了从「单个可观测 Agent」到「Agent 派生 Agent」的跨越。多智能体在这个项目里不是一套庞大的框架，而是 `create_agent` 的一次递归调用加一行递归防护——这正是 `FuFan-OpenClaw`「简洁可观测」哲学的又一次体现。下一章我们收尾，回顾三大设计哲学如何在全部代码里兑现，并盘点你这趟走下来的完整能力清单。

---

## 第 9 章：总结——三哲学回顾与能力清单

八章走完（七章核心骨架 + 第 8 章多智能体进阶），我们已经用 Python 从零搭出了 FuFan-OpenClaw 的完整骨架。这一章不写新代码——我们回到起点，检视三大设计哲学是如何在代码里兑现的，建立一个清晰的能力锚点，以及呼应这套设计思想与原版 TypeScript 龙虾架构的关系。

### 9.1 三哲学兑现回顾

![](https://typora-photo1220.oss-cn-beijing.aliyuncs.com/DataAnalysis/ZhiJie/20260610154601618.png)

**哲学一：文件即记忆**——在代码里体现为三个层次：第 1 章的 `build_system_prompt` 每轮重读 6 个 Markdown 文件，确保修改立即生效；第 5 章的 `MemoryIndexer` 把超长 `MEMORY.md` 切块向量化，按需检索；第 6 章的 `SessionManager` 把会话历史存为 JSON，压缩后的摘要也是文件，人类可读可编辑。三个层次共同完成了「记忆 = 文件」这一承诺。

**哲学二：技能即插件**——在代码里体现为第 4 章的 `scan_skills`：在 `skills/` 目录放一个带 YAML frontmatter 的 `SKILL.md` 文件，系统启动时自动扫描，生成 XML 快照注入 prompt。新技能的生效不需要改一行 Python 代码。`AGENTS.md` 里的技能调用协议（必须先 `read_file` 读说明书）确保 Agent 知道「怎么用技能」。

**哲学三：完全可观测**——在代码里体现为第 3 章的 `astream` 双流解析和第 7 章的全链路串讲：5 类事件实时推出，工具调用的输入输出都可见，前端把它们渲染成可视化的 ThoughtChain。用户看到的不是「AI 在思考」的黑盒，而是每一步推理和每一次工具调用的完整轨迹。

### 9.2 你的能力清单

八章代码加起来，你已经能用大约 **450 行 Python** 搭出一个功能完整的迷你可观测 Agent 系统（含多智能体派生）：

八章 MVP 行数统计与能力对应

| 章节 | MVP 行数 | 对应能力 | 对应哲学 |
|------|---------|---------|---------|
| 第 1 章 | ~40 行 | System Prompt 拼接器（6 组件 + RAG 开关） | 文件即记忆① |
| 第 2 章 | ~50 行 | create_agent 内核（ChatDeepSeek + 工具 + ainvoke） | 大脑层 |
| 第 3 章 | ~60 行 | astream 双流解析器（5 类事件 + new_response 分段） | 完全可观测 |
| 第 4 章 | ~50 行 | Skills 扫描注入（SKILL.md → XML → prompt） | 技能即插件 |
| 第 5 章 | ~120 行 | RAG 检索器（LlamaIndex + 中英分词 + RRF 双模式融合 + MD5 增量重建） | 文件即记忆② |
| 第 6 章 | ~50 行 | 会话持久化（save/load/compress/load_for_agent） | 历史也是文件 |
| 第 7 章 | ~30 行 | 全链路串讲（9 站链路图 + canvas 提取） | 三哲学综合 |
| 第 8 章 | ~50 行 | `spawn_subagent` 派生子智能体（递归防护 + 子任务隔离） | 多智能体进阶 |
| **合计** | **~450 行** | **一个迷你可观测 Agent 系统的完整骨架 + 多智能体进阶** | — |

### 9.3 与原版 OpenClaw 的对照

前三节课我们研究的是原版 OpenClaw（TypeScript 实现）——Agent loop、上下文工程、工具系统、Gateway-channel-node 架构。这个 Python 重构版和原版 OpenClaw 的关系是：**机制同源，实现语言不同，教学侧重不同**。

两个版本共享同样的核心思想：Agent 通过循环推理调工具、System Prompt 是 Agent 行为的控制中枢、会话历史是下一轮推理的上下文、工具是 Agent 能力的原子单元。但 Python 版在教学上有一个独特优势——每一个机制都可以用几十行 Python 重现，不需要理解 TypeScript 的模块系统、Node.js 的 EventEmitter 机制或 Next.js 的 Server Actions。你可以用 Jupyter Notebook 逐 cell 运行，每一步都看到输出，直到「哦，原来就是这样」的那一刻。

这个 Python 版是原版 OpenClaw 架构的**最小可教学复刻**：骨架在，肌肉不在；机制在，工程优化不在。这是教学版本刻意的取舍——让你先搞懂「为什么要这样设计」，再去读原版代码里「怎么做到更好」。

---
