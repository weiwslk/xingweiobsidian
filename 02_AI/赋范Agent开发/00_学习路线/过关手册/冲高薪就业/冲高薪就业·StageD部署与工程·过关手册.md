# 冲高薪就业 · Stage D 部署与工程 · 过关手册（面试向）

> 路径：冲高薪就业（career-sprint） ｜ 阶段：D 部署与工程（D1–D6） ｜ 风味：求职 · 面试向加重
>
> 这一阶段决定了面试官对你的一句判断：你是"只在 notebook 里跑过 demo"，还是"能把 Agent 服务化、容器化、评估优化、工程化驾驭"的人。
> 很多候选人 RAG、微调都能聊，一到"你怎么把它上线、怎么扛并发、怎么证明它没退化、怎么控成本"就露怯，这一册就是替你把这段补上。
>
> 用法：先做「随堂练习」确认动手过关；再用「误区自查 / 迁移挑战 / 排错挑战」查漏；
> 最后把「面试自测」当口试反复刷，用「模拟面试官指令」找人（或 Agent）给你上一场压力面。
> 答案全在手册里，不依赖任何外部资料。

**阶段 gate 提醒**：D6 驾驭工程是本阶段的深潜专属徽章 gate。关掉课程能力包，自己手搓一个 Mini Harness 跑通一轮 Agent Loop（读取信息 → 判定退出 → 调用工具 → 回填迭代），能与课件手搓版结构对上才算过。上线技能真不真，看的是你能不能独立说清 async 决策、四条上线红线、GPU 整数调度、可复现三件套这几件"面试官最爱挖"的事。

**本阶段来源讲次索引**（题目均锚课件与真实源码，标注见每题）：
- D1《智能体部署上线基础理论 FastAPI》：async vs sync 并发实测 / 三入口与响应模型 / 流式接口 / 422·404·CORS 报错排查
- D2《主流 Agent 类型及接口设计》＋《Agent 智能体部署上线》：接口形态选型 / SSE vs WebSocket / 生产四红线 / 上线六件产物验收（配套代码 `代码/ai-todo`）
- D3《容器化部署 Docker + Compose》＋《Kubernetes 集群编排入门》：Dockerfile 与 compose / 工作负载选型 / GPU 整数调度 / 部署故障诊断
- D4《Agent 评估与优化：从评估体系到 LangSmith 实战》：可复现三件套 / 七类评估角度 / 三杠杆优化 / 三种裁判
- D5《上下文工程管理基础入门》＋《上下文工程进阶——组合编排实战》：压缩五件套 / prefix cache 红线 / 叠加顺序
- D6《Harness Engineering 驾驭工程》第一至六节：Agent Loop 四相 / 三铁律 / Context Rot / 外部验证 / HQS 五类失败模式

---

## 一、随堂练习（每节点动手小题）

> 每题先自己做，再对参考答案。做不动就回看对应讲次，不要跳过 gate。

### D1 · FastAPI 服务化基础

**练习 D1-1**：你要写一个 FastAPI 路由，函数体里用 `AsyncOpenAI` 调大模型。该写 `async def` 还是 `def`？如果里面误用了 `time.sleep(3)` 会怎样？

- 参考答案：写 `async def` + `await`，这是 LLM 服务路由的默认选择。函数体里绝不能出现 `time.sleep` / `requests.get` 这类同步阻塞调用，它们会卡住整个事件循环，让并发请求全部排队。必须用同步库时，放到 `def` 路由或用线程池封装。自查：能对上课件的决策树分支（调 LLM/网络/DB 且库支持 await → async def）才算真懂，而不是给所有路由无脑加 async。
<!--src:A; ref:D1《FastAPI》async 使用决策 7.7; look:career-sprint#D1-->

**练习 D1-2**：接口返回 `422 Unprocessable Entity`，你第一步看哪里？会不会去改业务函数？

- 参考答案：先看响应体 `detail[].loc` 和 `type` 定位是哪个字段哪种错，比如 `loc:["body","text"]` + `type:string_too_short` 就是 body 里的 text 太短。422 意味着请求根本没进函数体、卡在 Pydantic 校验，改的是"请求怎么发"，不是业务代码。一报 422 就翻业务逻辑是典型误诊。
<!--src:A; ref:D1《FastAPI》7.1 报错排查; look:career-sprint#D1-->

**练习 D1-3**：浏览器报 CORS 红屏，同一个接口用 curl 却能通。这说明什么？怎么修？

- 参考答案：curl 能通说明后端业务逻辑没坏，问题只在浏览器跨域：响应缺 `Access-Control-Allow-Origin` 头。修法是后端加 `CORSMiddleware` 明确允许前端的源，别去改业务函数。红线：开发和生产两套 CORS 配置，禁止 `allow_origins=["*"]` 与 `allow_credentials=True` 同时开，API Key 一律进 `.env`。
<!--src:A; ref:D1《FastAPI》5.5–5.9 跨域与约束; look:career-sprint#D1-->

### D2 · 接口设计与部署上线

**练习 D2-1**：nginx 接管 443 之后，后端 uvicorn 的 `--bind` 该保持 `0.0.0.0:12234` 吗？

- 参考答案：不该。必须从 `0.0.0.0:12234` 改成 `127.0.0.1:12234`，把后端收回 loopback，公网只经 nginx 443，后端不直接对外。否则公网能直连后端绕过 nginx，且业务路由若没鉴权就是任人调用、LLM 接口直接烧 token 配额。这是课件配套项目 `代码/ai-todo` 的真实端口（`config.py` 里 `port: int = 12234`，前端跑 12233）。
<!--src:A; ref:D2《Agent部署上线》红线二 loopback；code:代码/ai-todo/config.py; look:career-sprint#D2-->

**练习 D2-2**：聊天要打字机流式效果，接口用 SSE 还是 WebSocket？`Response` 类型选哪个？

- 参考答案：用 SSE，不需要 WebSocket。判据只看一件事：客户端要不要在不发新 HTTP 请求的前提下主动推数据给服务端？打字机流式只是服务端推客户端，SSE 就够，还能复用 HTTP 基建、自动重连。Response 类型选 `StreamingResponse(gen(), media_type="text/event-stream")`；流式 token 用 `JSONResponse` 就不流式了。只有实时语音这类双向推送才上 WebSocket。
<!--src:A; ref:D2《主流Agent类型及接口设计》2.5/2.6 协议选型; look:career-sprint#D2-->

**练习 D2-3**：部署跑完了，你怎么判断"真的上线成功"而不是"浏览器能打开就行"？至少说 4 项验收。

- 参考答案（六件产物任取四）：① `systemctl status` 是 active(running) 且 `kill -9` 后能自动拉起；② `nginx -t` 通过、site 已软链；③ 浏览器 HTTPS 地址栏有锁标；④ curl 域名下的 `/health` 探针返回 `{"status":"ok"}`；⑤ `certbot renew --dry-run` 通过、有续期定时器；⑥ 日志三板斧（journal / nginx access / 应用层）都有内容。外加职责分离：公网用 IP 直连后端 12234 端口应被拒。验证证书续期只能 `--dry-run`，真跑会消耗 Let's Encrypt 配额（同域名 7 天仅 5 次成功签发）。
<!--src:A; ref:D2《Agent部署上线》第5章六件产物 + 4.2.5 dry-run; look:career-sprint#D2-->

### D3 · 容器化与集群编排

**练习 D3-1**：把 compose 项目搬 K8s，milvus / etcd / minio 该用 StatefulSet 还是 Deployment？全用 Deployment 会怎样？

- 参考答案：有持久状态、需要稳定身份的（milvus / etcd / minio）必须用 StatefulSet，靠稳定 Pod 名（如 `milvus-0`）+ `volumeClaimTemplates` 给每个 Pod 独立 PVC + headless Service（`clusterIP: None`）拿稳定 DNS。无状态的 backend / frontend 用 Deployment。5 个服务全用 Deployment 看着省事，但 milvus 重启后向量索引全丢、minio 图片全丢、etcd 元数据全丢，还会多副本抢同一个 PVC 写冲突崩溃。
<!--src:A; ref:D3《K8s集群编排》3.2 工作负载选型; look:career-sprint#D3-->

**练习 D3-2**：4 张 RTX 3090（共 96 GiB），每个 backend pod 只占 12 GiB，理论能装 8 个 pod，为什么第 5 个一直 Pending？

- 参考答案：因为 `nvidia.com/gpu` 是整数调度资源，单位是 integer，不支持 `0.5` 小数申请，scheduler 只看"卡占用/未占用"二元状态、不看显存利用率。4 张卡就是 4 个调度单位，1 卡 1 pod 独占后只能装 4 个，第 5 个因 `Insufficient nvidia.com/gpu` 挂 Pending。跨卡显存也不能聚合（单 pod = 单 CUDA context 只能跑单卡）。要多装得走 time-slicing / MPS 共享，但在线推理不建议（切片切换 p95 抖）。
<!--src:A; ref:D3《K8s集群编排》4.2-4.3 GPU 调度; look:career-sprint#D3-->

**练习 D3-3**：镜像 push 到服务器上，容器进程一起来就 `exec format error` 秒退。根因和修法？

- 参考答案：根因是镜像架构与主机不符，多半是在 arm64 机（如 Apple Silicon）打的镜像丢到 amd64 服务器上跑。用 `docker inspect <镜像> | grep Architecture` 确认，再用 buildx 带 `--platform linux/amd64` 重新 build。别去重装 docker，那和架构错配无关。
<!--src:A; ref:D3《容器化 Docker+Compose》故障诊断表; look:career-sprint#D3-->

### D4 · 评估与优化（LangSmith）

**练习 D4-1**：你要拿评估成绩单做版本回归对比，动手前必须先落地哪三件套？

- 参考答案：可复现三件套，缺一这份成绩单就不能做回归对比。① 固定时间：把"当前时间"设成常量（课件用 `EVAL_NOW=2026-06-04T12:00:00`），别用真实系统时间，否则"上周三"每天指向不同日期、参考答案对不上；② 每例重置 + 独立评估库：每条用例跑前 reset 成同一份种子数据，且用独立的 `eval_*.db` 而非生产库 `accounting.db`；③ 串行 `max_concurrency=1`：共用一个评估库、每条都要重置，并发会让几条同时重置写入把状态搞乱。
<!--src:A; ref:D4《Agent评估与优化》第三章 3.2 可复现性; look:career-sprint#D4-->

**练习 D4-2**：成绩单里 `date_correct` 只有 0.20，`context_resolved` 只有 0.17。这两个该用同一种手段修吗？

- 参考答案：不该。`date_correct` 是"指导不足"（模型有能力只是没被告知先调 resolve_date），写细 system prompt 就能从 0.20 拉到 1.00，零模型成本，优先做；`context_resolved` 是多轮跨轮记忆的"能力不足"，再怎么写 prompt 都补不动（会找错记录甚至打转），只有换更强模型（课件里 coder-flash 换成 deepseek-v4-pro）才从 0.17 到 1.00。顺序永远先便宜（prompt / 工具说明）后贵（换模型），把钱花在刀刃上。
<!--src:A; ref:D4《Agent评估与优化》第四章 4.3-4.5 三杠杆; look:career-sprint#D4-->

### D5 · 上下文工程

**练习 D5-1**：一个工具密集型的代码 Agent 上下文超窗了，你在"观察遮蔽"和"摘要"里怎么选？`keep_recent` 设几？

- 参考答案：工具密集型优先观察遮蔽（ObservationMask）或工具结果清除（ToolResultClear），因为推理质量取决于"做了什么决策、为什么"，不是工具返回的原始数据；对话型才优先 trim_messages + LLM 摘要（观察遮蔽会把用户的话也丢掉，对话型禁用）。`keep_recent=1` 是黄金参数：保住当轮查到的原文，绝不设 0，否则当轮代码片段还没被用上就被压成一句话，Agent 答不出"第 42 行的逻辑"。
<!--src:A; ref:D5《上下文工程进阶》压缩五件套选型; look:career-sprint#D5-->

**练习 D5-2**：有人喊"太贵了"，把 SummarizationMiddleware 的触发阈值调到很激进。为什么可能账单不降反升？

- 参考答案：因为降本的主力是 prefix cache 命中，不是省 token。DeepSeek prefix cache 按字节级前缀匹配命中，命中 token 按标准价 10% 计费（命中 ¥0.2/1M 对未命中 ¥2.0/1M，省 90% 输入成本）。激进压缩会改动消息中段，让从修改点起的所有后续 token 全部 cache miss，省下几百 token 却让整段缓存归零，账单反升。正确姿势：system prompt 放"你是谁"（静态利于缓存），messages 放"聊了什么"（动态摘要），摘要绝不塞进 system prompt。
<!--src:A; ref:D5《上下文工程基础入门》3.1.5 cache 红线; look:career-sprint#D5-->

### D6 · 驾驭工程 Harness（gate）

**练习 D6-1（gate 核心）**：不借课程包，手搓一个 Mini Harness 主循环，说清 Agent Loop 的四相分别做什么、循环从哪里退出、怎么防失控。

- 参考答案：四相是 Gather Context → Verify → Take Action → Iterate。相一 Gather：把完整对话历史 `messages` 和工具的 JSON 描述清单一起发给大模型拿决策；相二 Verify（唯一出口）：`if not message.tool_calls` 为真（返回的是纯文本没有工具调用）就判定任务完成、终止循环返回最终答案；相三 Take Action：有工具调用就执行，注意执行顺序别弄错；相四 Iterate：回填结果进 messages 再进下一轮。防失控靠硬刹车 `max_steps=50`（`while steps < config.max_steps`）。课件的 `mini_harness/` 是 11 个文件 = 1 个配置中心 `config.py` + 10 个机制模块，`core.py` 一个文件承载机制①② Agent Loop + Tool Use。
<!--src:A; ref:D6《手搓 Mini Harness》第二章 2.1 四相 + config.py; look:career-sprint#D6-->

**练习 D6-2**：一个正在运行的会话，用户中途要你加个新工具。你直接加吗？

- 参考答案：不直接加。运行中途改 toolset / memory / system prompt 违反 Prompt Cache 三铁律：三者共同构成缓存前缀 key，前缀任何改变就让已缓存 token 全部作废，实测同样请求响应时间约 200ms 变约 1200ms、输入 token 重新计费。正确做法是拦截并说明代价，把修改放到当前会话结束后或新会话开始时执行；用户坚持就明确告知代价再执行，并建议同时做一次上下文压缩。
<!--src:A; ref:D6《手搓 Mini Harness》4.2 三铁律; look:career-sprint#D6-->

---

## 二、误区自查

> 逐条对照。命中任一条，说明对应知识点还没掌握，按提示回看。

1. **如果你以为"给所有 FastAPI 路由无脑加 async def 最保险"** → 你还没掌握 async 决策。async 函数体里调同步库（`requests` / `time.sleep`）反而卡死事件循环，比 def 更慢；简单 dict/字符串逻辑硬写 async 只是徒增复杂度。回看 D1 async 使用决策。
<!--src:A; ref:D1《FastAPI》7.7 async 决策; look:career-sprint#D1-->
2. **如果你以为"接口报 CORS 红屏就是后端逻辑坏了"** → 你还没分清 CORS 只是缺响应头、且只拦浏览器。curl / Postman 打同一接口能通就证明后端没问题，去配 `CORSMiddleware` 即可，别改业务代码。回看 D1 跨域章节。
<!--src:A; ref:D1《FastAPI》5.5-5.9 跨域; look:career-sprint#D1-->
3. **如果你以为"浏览器能打开就算上线成功"** → 你还没掌握上线验收。没验 `/health`、没验 `kill` 后自动重启、没验证书续期，隐患全留到线上爆。必须逐项过六件产物。回看 D2 第 5 章。
<!--src:A; ref:D2《Agent部署上线》第5章验收; look:career-sprint#D2-->
4. **如果你以为"后端一直 `0.0.0.0` 裸跑没关系，反正有 nginx"** → 你还没守住 loopback 红线。nginx 接管后后端必须收回 `127.0.0.1`，否则公网可直连后端绕过 nginx，无鉴权任人调用烧 token。回看 D2 生产红线二。
<!--src:A; ref:D2《Agent部署上线》红线二; look:career-sprint#D2-->
5. **如果你以为"有状态服务用 Deployment、多副本共享一个 PVC 也能跑"** → 你还没掌握工作负载选型。milvus/etcd/minio 用 Deployment 会重启丢数据，多副本抢同一 PVC 写冲突崩溃，必须 StatefulSet + 独立 PVC。回看 D3 工作负载选型。
<!--src:A; ref:D3《K8s集群编排》3.2; look:career-sprint#D3-->
6. **如果你以为"K8s 多加副本能让单个模型 pod 变快"** → 你记错了扩展方向。多副本是横向扩到 N 个独立 pod / N 张卡，单 pod 内 GIL + 单 CUDA stream 永远串行；单 pod 吞吐靠 vLLM/Triton 的 continuous batching（利用率 30%→80-95%）。GPU 推理还用不上 HPA（整数资源）。回看 D3 GPU 调度。
<!--src:A; ref:D3《K8s集群编排》4.2-4.3; look:career-sprint#D3-->
7. **如果你以为"评估直接连生产库、并发跑快点无所谓"** → 你还没掌握可复现三件套。评估要用独立 `eval_*.db`、每例重置种子、`max_concurrency=1` 串行，缺一成绩单就没法做版本回归。回看 D4 第三章 3.2。
<!--src:A; ref:D4《Agent评估与优化》3.2; look:career-sprint#D4-->
8. **如果你以为"给算数、精确字段这类题也用大模型当裁判更聪明"** → 你还没掌握裁判选型。有唯一正确答案的题用规则裁判 `exact_match`（快、准、零成本），大模型判对错得自己再算一遍、可能算错。只有主观开放输出才上 LLM-as-judge，且机器裁判都要人工抽检校准。回看 D4 第二章 2.8。
<!--src:A; ref:D4《Agent评估与优化》2.8 三种裁判; look:career-sprint#D4-->

---

## 三、迁移挑战

> 换个场景，用同一套方法。能迁移才算真掌握，这是"懂了没"的核心判据。

**迁移 1（换服务：客服 Agent → 数据分析 Agent 上线）**：你在 D2 把一个客服 Agent 用 systemd + nginx + certbot 部署上了香港 VPS。现在换成把 E4 的数据分析 Agent（要连数据库、返回图表、可能长任务）上线，**接口协议、鉴权、验收**分别怎么调整？
- 参考答案：接口协议上，出图这类一次性结果用 `JSONResponse`，若要展示 SQL 生成/执行的过程日志则叠一条 SSE；文件下载类结果用 `FileResponse`。鉴权照旧走 `APIRouter(dependencies=[Depends(verify_api_key)])`，但要注意 EventSource/下载链不能带 header，`verify_api_key` 要兼容从 `?api_key=` 取 key，`/health` 放鉴权之外。验收仍是六件产物 + 职责分离，额外加一条：数据库连接凭证同样走 `.env` + `chmod 600`，绝不进 git、不打进 tarball。方法链不变，换的只是协议形态和多一个数据库凭证纪律。
<!--src:A; ref:D2《Agent部署上线》红线+验收 + 接口形态选型; look:career-sprint#D2-->

**迁移 2（换关注点：质量退化 → 成本失控）**：你已经会用 D4 的评估体系抓"Agent 答得对不对"。现在老板只关心"Agent 越跑越贵"，你要把同一套"先定位再动手"的方法迁到成本问题上，怎么排查？
- 参考答案：思路一样是"先归因再动手"，但归因维度换成缓存和上下文。先查是不是运行中途改过 toolset / memory / system prompt（违反三铁律 → cache miss，是"越跑越贵越慢"的最常见来源）；排除后查上下文长度，看 `cache_hit_tokens` 是否被打回 0，是则说明有压缩改了消息中段打碎了 prefix。修法对应 D5：cache_boundary 放最外层锁前缀，tail_trim 作日常主力只删中段已完成的 tool/ai 对，摘要只当保险丝。课件把这条列为全课最大的单点降幅，且降幅来源不是"省 token"，而是"让 token 按 cache_hit 单价计费"（命中价为未命中的 10%）——把 `cache_hit_tokens` 从 0 拉起来，比一味压缩省下的那几百 token 值钱得多。
<!--src:A; ref:D5《上下文工程进阶》叠加顺序 + D6 三铁律; look:career-sprint#D5-->

---

## 四、排错挑战

> 下面是一份真实感的"把 compose 项目搬上线"配置（新手常踩的写法），一连撞了多个问题。
> 请先自己找出所有毛病，再看答案。

```yaml
# docker-compose.prod.yml —— 目标：把 数据分析 Agent（backend）+ milvus 向量库上线到 GPU 服务器
services:
  milvus:
    image: milvusdb/milvus:v2.4.0
    # 无 healthcheck、无 volumes
  backend:
    build: .
    environment:
      - MILVUS_HOST=localhost          # 连 milvus
      - OPENROUTER_API_KEY=sk-or-v1-abc123    # 直接写死在 compose 里
    ports:
      - "12234:12234"                  # host 端口写死
    # 无 depends_on
```

```dockerfile
# Dockerfile —— backend 镜像
FROM python:3.11
COPY . /app                           # 把整个目录（含 .env）拷进镜像
WORKDIR /app
RUN pip install -r requirements.txt
CMD python main.py
```

```yaml
# k8s：把 milvus 搬上集群（片段）
kind: Deployment                       # milvus 用 Deployment
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: milvus
          resources:
            limits:
              nvidia.com/gpu: 0.5      # 想让 backend 半张卡
          # 无 readinessProbe
```

**参考答案**（每条按【症状】→【根因】→【修法】三段拆解）：

1. **【症状】**backend 容器里连不上 milvus，日志 `host not found` 或连接被拒。**【根因】**：`MILVUS_HOST=localhost` 把 localhost 硬编码进容器，容器里的 localhost 是它自己，不是 milvus 服务。**【修法】**：改用 compose 内部网络的服务名 `MILVUS_HOST=milvus`，靠内部 DNS 解析，别把宿主机视角的 localhost 写进容器。
2. **【症状】**凭证泄露、配额被刷爆。**【根因】**：`OPENROUTER_API_KEY` 明文写进 compose，且 Dockerfile `COPY . /app` 把 `.env` 一起打进镜像，镜像一分发 key 就跟着走。**【修法】**：key 移进 `.env`（`.gitignore` 拦住、`chmod 600`），compose 用 `env_file` 或变量引用；Dockerfile 配 `.dockerignore` 拦掉 `.env` 和密钥，代码 tarball 与 `.env` 分开走独立通道。
3. **【症状】**backend 先于 milvus 起来，启动即 `CrashLoopBackOff` / 连不上向量库。**【根因】**：compose 没写 `depends_on` + milvus 没 `healthcheck`，依赖顺序失控；milvus 也没挂 `volumes`，重启数据全丢。**【修法】**：给 milvus 加 healthcheck，backend 加 `depends_on: {milvus: {condition: service_healthy}}`，并给 milvus 挂持久卷。
4. **【症状】**换台机器部署撞 `address already in use`，host 12234 被占。**【根因】**：host 端口写死 `"12234:12234"`。**【修法】**：host 端口变量化（如 `"${BACKEND_PORT:-12234}:12234"`），容器内端口不变，冲突时只改 `.env`；先 `ss -tlnp | grep 12234` 看是谁占着，别乱 kill。
5. **【症状】**milvus 上 K8s 后重启数据全丢、两副本写冲突崩溃。**【根因】**：milvus 是有状态服务却用了 `Deployment` + `replicas: 2`。**【修法】**：改 `StatefulSet` + `volumeClaimTemplates` 给每个 Pod 独立 PVC + headless Service，副本按需缩到 1 或做 milvus 集群模式。
6. **【症状】**backend pod 一直 `Pending`，事件里 `Insufficient nvidia.com/gpu`。**【根因】**：`nvidia.com/gpu: 0.5` 不成立，GPU 是整数调度资源、不支持小数申请。**【修法】**：改 `nvidia.com/gpu: 1` 整卡独占；真要共享得配 time-slicing / MPS，且在线推理慎用。
7. **【症状】**滚动升级时前端间歇 `502 Bad Gateway`。**【根因】**：没配 `readinessProbe`，新 pod 进程一起就被标 Ready，流量打到模型还没 load 完的 pod。**【修法】**：加 `/api/ready` + readinessProbe，覆盖 80–180 秒冷启动窗口。

（综合 D2 生产红线 + D3 容器与 K8s 故障诊断表：localhost 硬编码 / 密钥进镜像 / 依赖顺序 / host 端口 / 工作负载选型 / GPU 整数 / 探针缺失 七类高频坑）
<!--src:A; ref:D2 红线 + D3 故障诊断表；code:代码/ai-todo/config.py; look:career-sprint#D3-->

---

## 五、面试自测（15 题 · 面试向加重）

> 每题给「考察点 / 答题要点 / 两层追问链」。追问链是面试官会怎么一层层往深挖，
> 你要能扛住第二层追问，才算真掌握而不是背了个结论。刷题时先捂住答案自己讲一遍。

**Q1 · FastAPI 路由什么时候写 async def、什么时候写 def？有没有量化依据？**
- 考察点：异步姿势的工程判断，不是背"async 更快"。
- 答题要点：调 LLM / 网络 IO / DB 且库支持 await（httpx / asyncpg / AsyncOpenAI）→ `async def` + await；调同步 IO 库（requests / psycopg2）→ `def` 或线程池；CPU 密集短任务用 `def`、重任务走进程池/独立服务；简单 dict/字符串逻辑 `def` 即可。量化依据：课程 100 并发实测，`async` 用 `await asyncio.sleep` 让出事件循环耗时 5.01 秒，`sync` 用 `time.sleep` 阻塞线程 15.12 秒，慢 3.02 倍。（D1《FastAPI》async 决策）
- 追问链：① 为什么 sync 在高并发下排队变慢？→ sync 会占用 Starlette 底层 AnyIO 线程池，默认容量仅 40 tokens，超过即排队；② `async def` 里不小心调了 `requests.get` 会怎样？→ 同步阻塞卡住整个事件循环，所有并发请求全堵，反而比纯 def 更糟。
<!--src:A; ref:D1《FastAPI》3.4/6.6/7.7 并发实测; look:career-sprint#D1-->

**Q2 · FastAPI 接口返回 422、404、CORS 红屏，你分别怎么定位？**
- 考察点：报错先查表不猜的诊断素养。
- 答题要点：422 是请求进了 Pydantic 校验没过，看响应体 `detail[].loc` + `type` 定位哪个字段、改请求，不翻业务代码；404 是路径或资源不存在，资源查不到要 `raise HTTPException(status_code=404, ...)` 标准抛错，不能用 200 假装成功；CORS 红屏是跨域缺 `Access-Control-Allow-Origin` 头、只拦浏览器，先用 curl 验后端能不能通，能通就去配 `CORSMiddleware`。（D1《FastAPI》7.1）
- 追问链：① 为什么"没找到"不能用 200 + 空返回搪塞？→ 前端无法判断成败、走不了错误分支，必须给 404；② CORS 里为什么禁止 `allow_origins=["*"]` 和 `allow_credentials=True` 同时开？→ 带凭证的跨域请求通配源是安全红线，浏览器也会拒绝，生产要显式列白名单。
<!--src:A; ref:D1《FastAPI》5.5-5.9/7.1; look:career-sprint#D1-->

**Q3 · LLM 接口调用要几秒到几十秒，前端一直白屏等，怎么解决？为什么流式接口要 async？**
- 考察点：流式接口的必要性与异步配合。
- 答题要点：上 `StreamingResponse` + `stream=True` 做流式，媒体类型 `text/event-stream`，让 token 边生成边推，用户看到打字机效果而不是空白等几秒再整段刷出。流式接口默认 `async def` + `AsyncOpenAI` + `async for ... yield`，因为要在等上游 token 的间隙让出事件循环、支撑并发，同步版会把线程占死。（D1《FastAPI》流式能力）
- 追问链：① LLM 接口卡住就无脑加超时/重试对不对？→ 不对，根因是没做流式，加超时治标不治本；② 流式 token 用 `JSONResponse` 会怎样？→ 前端拿不到 event 边界、根本不流式，必须 `StreamingResponse`。
<!--src:A; ref:D1《FastAPI》流式端点; look:career-sprint#D1-->

**Q4 · 把一个带 API Key 的 Agent 部署上公网，你会守哪几条红线？**
- 考察点：生产上线的安全纪律，这是"会不会真上线"的分水线之一。
- 答题要点：四条红线。① 凭证四纪律：`.gitignore` 拦 `.env`、绝不 push，`.env` 与代码 tarball 分开走独立 scp（`tar --exclude='.env'`），落地 `chmod 600`，泄露立即重置 key；② nginx 接管后后端 `--bind` 从 `0.0.0.0` 收回 `127.0.0.1`，公网只经 443；③ 生产业务路由必鉴权，`APIRouter(dependencies=[Depends(verify_api_key)])` 批量加，`/health` 放鉴权之外；④ certbot 只走 snap 安装（Ubuntu 24.04 先卸 apt 版再 `snap install --classic certbot`）。（D2《Agent部署上线》）
- 追问链：① `.env` 泄露的真实后果是什么？→ 数小时内被爬虫扫走刷爆配额，是真金白银事故；② 为什么 `verify_api_key` 要兼容从 Query 取 key，而不只认 header？→ EventSource（SSE）和下载链不能加自定义 header，只能走 `?api_key=`，否则前端功能用不了。
<!--src:A; ref:D2《Agent部署上线》2.3/4.2 + 鉴权 14.1; look:career-sprint#D2-->

**Q5 · SSE 和 WebSocket 怎么选？聊天流式、日志流、实时语音分别用哪个？**
- 考察点：实时协议选型，一个判据讲清而不是罗列区别。
- 答题要点：只看一件事：客户端要不要在不发新 HTTP 请求的前提下主动推数据给服务端。要（实时语音双向）→ WebSocket；不要（只服务端推客户端）→ SSE。打字机流式、工具调用过程、日志流全用 SSE，简单、复用 HTTP 基建、自动重连。Response 类型：一次性 JSON 用 `JSONResponse`，流式 token 用 `StreamingResponse(media_type="text/event-stream")`，文件下载用 `FileResponse`。（D2《主流Agent类型及接口设计》2.5/2.6）
- 追问链：① 只需服务端推流却上 WebSocket 有什么代价？→ 徒增复杂度、丢掉 HTTP 基建复用和自动重连；② 上线走国内还是香港 VPS？→ 要域名做正式 HTTPS 选香港付费（免 ICP 备案、1-2 天走通），国内免费 VPS 绑域名做 HTTPS 要 ICP 备案等 2-3 周，课堂等不起。
<!--src:A; ref:D2《主流Agent类型及接口设计》2.5/2.6/2.12; look:career-sprint#D2-->

**Q6 · 部署完怎么验收才算"真上线"？为什么验证书续期只能 dry-run？**
- 考察点：上线验收的完整性意识。
- 答题要点：逐项过六件产物：① systemd active(running) 且 `kill -9` 后 6 秒内自动拉起；② `nginx -t` 通过、site 软链就位；③ HTTPS 端到端有锁标；④ curl 域名下的 `/health` 探针返回 `{"status":"ok"}`；⑤ `certbot renew --dry-run` 通过、有 `snap.certbot.renew.timer`；⑥ 日志三板斧（journal / nginx access / 应用层）都有内容。外加职责分离：公网直连后端端口 12234 应被拒。验证书续期必须 `--dry-run`，因为真跑 `certbot renew` 会消耗 Let's Encrypt 配额（同域名 7 天仅 5 次成功签发），几次就把配额烧完。（D2《Agent部署上线》第 5 章）
- 追问链：① `/health` 返回 degraded/503 一般是什么问题？→ 常是缺环境变量或数据库连不上，看返回的 issues 列表定位；② 为什么只看 nginx access log 不够？→ 看不到请求到没到后端、业务有没有异常栈，必须同时看 journal 和应用层 log。
<!--src:A; ref:D2《Agent部署上线》第5章 + 4.2.5; look:career-sprint#D2-->

**Q7 · K8s 上 StatefulSet 和 Deployment 怎么选？有状态服务用错会怎样？**
- 考察点：工作负载选型，Compose 玩家上 K8s 最容易翻车的一步。
- 答题要点：判据一句话：有持久状态、需要稳定身份 → StatefulSet；无状态、可随意替换 → Deployment。milvus/etcd/minio 用 StatefulSet，靠稳定 Pod 名（`milvus-0`）+ `volumeClaimTemplates` 独立 PVC + headless Service（`clusterIP: None`）拿稳定 DNS，重启后名字和磁盘都不变；backend/frontend 用 Deployment，副本可互换、滚动升级自愈标配。有状态服务误用 Deployment 或多副本共享 PVC，会重启丢数据、多 Pod 抢同一 PVC 写冲突崩溃。（D3《K8s集群编排》3.2）
- 追问链：① headless Service 的 `clusterIP: None` 起什么作用？→ 不分配集群 IP，让 Pod 展开成 `etcd-0.etcd` 这类稳定 DNS，有状态副本才能互相寻址；② 一个 3 个 StatefulSet 同模式的场景，怎么让 AI 帮你写 yaml 又不出错？→ 让 AI 出一份模板再按字段填，逐个写别一次写 8 个（字段互串），生成后先 `kubectl apply --dry-run=server` 让 api-server 校验。
<!--src:A; ref:D3《K8s集群编排》3.2/5.4; look:career-sprint#D3-->

**Q8 · K8s 上 GPU 是怎么调度的？为什么想给一个 pod 分半张卡不行？**
- 考察点：GPU 整数调度这个高频实战考点。
- 答题要点：`nvidia.com/gpu` 是整数调度资源，单位是 integer，不支持 `0.5` 小数申请，scheduler 只看卡"占用/未占用"二元状态、不看显存利用率。4 张卡就是 4 个调度单位，哪怕每 pod 只占 12 GiB、物理上一张卡能塞几个，scheduler 也只让 1 卡 1 pod，第 5 个挂 Pending（`Insufficient nvidia.com/gpu`）。跨卡显存不能聚合（单 pod = 单 CUDA context 跑单卡）。要共享得配 time-slicing（开发/离线）、MPS（多小模型）、MIG（A100/H100 硬隔离，3090 不支持）。（D3《K8s集群编排》4.2-4.3）
- 追问链：① 在线推理服务能用 time-slicing 共享 GPU 吗？→ 不建议，切片切换延迟抖动、p95 会爆；② 加副本能让单个模型推理变快吗？→ 不能，多副本是横向扩到 N 张卡各跑一份，单 pod 内 GIL + 单 CUDA stream 串行，单 pod 吞吐靠 vLLM 的 continuous batching（利用率 30%→80-95%），GPU 场景 HPA 也用不上。
<!--src:A; ref:D3《K8s集群编排》4.2-4.3; look:career-sprint#D3-->

**Q9 · 容器部署常见故障：exec format error、CrashLoopBackOff、滚动升级 502、pod Running 但 nvidia-smi 找不到卡，各是什么根因？**
- 考察点：容器/K8s 故障诊断按症状定位的能力。
- 答题要点：`exec format error` = 镜像架构与主机不符（arm64 镜像上 amd64 机），buildx `--platform linux/amd64` 重 build；`CrashLoopBackOff` = 依赖没就绪就启动（milvus 先于 etcd/minio），看 Events 补 initContainer 等依赖，别反复 apply；滚动升级 `502` = 没配 readinessProbe，流量打到模型没 load 完的 pod，配 `/api/ready` 覆盖 80-180 秒冷启动；pod Running 但容器里 `nvidia-smi` 找不到卡 = 漏了 `runtimeClassName: nvidia`（调度成功但没用 nvidia runtime）。诊断总则：先 `kubectl describe pod` 看 Events 最后几行拿真实错误，别猜、别盲目重试。（D3 故障诊断表）
- 追问链：① 为什么遇 CrashLoopBackOff 不能只"重启试试"？→ K8s 会自己指数退避重启，重复 apply 是"声明期望"不是重试执行，问题原地打转；② k3s 上 Ingress 一直 ECONNREFUSED 怎么回事？→ k3s 内置 traefik 不是 nginx-ingress，照搬 `ingressClassName: nginx` 不生效，要改 `traefik`。
<!--src:A; ref:D3 容器与K8s故障诊断表; look:career-sprint#D3-->

**Q10 · 给 Agent 做评估，怎么保证成绩单能拿来做版本回归？**
- 考察点：评估可复现性，做 A/B 对比的前提。
- 答题要点：可复现三件套缺一不可。① 固定时间：当前时间设成常量（`EVAL_NOW`），否则"上周三"每天指向不同日期、参考答案对不上（前端/生产仍用真实时间，只评估时固定）；② 每例重置 + 独立评估库：每条用例跑前 reset 成同一份种子数据，用独立 `eval_*.db` 不碰生产 `accounting.db`；③ 串行 `max_concurrency=1`：共用评估库、每条要重置，并发会让状态错乱。可复现不是消灭大模型自身随机，是把环境层干扰摁住，让波动只剩模型自身抖动。（D4《Agent评估与优化》3.2）
- 追问链：① 大模型同一条题这次调工具、下次心算，分数本就会抖，三件套还有意义吗？→ 有，三件套锁的是环境噪声，模型自身抖动是可接受基线，环境不锁就分不清是改动生效还是噪声；② 为什么评估必须用独立库？→ 评估会产生脏数据，直连生产库既污染生产又污染后续评估，四套评估之间也各用一个库互不串味。
<!--src:A; ref:D4《Agent评估与优化》3.2; look:career-sprint#D4-->

**Q11 · 成绩单某项指标低，你怎么决定是改 prompt、改工具说明还是换模型？裁判怎么选？**
- 考察点：评估驱动优化的选型判断，把钱花在刀刃上。
- 答题要点：三杠杆按先便宜后贵选，对应三个环境变量。先分低分属"指导不足"还是"能力不足"：指导不足（工具编排、如实复述、安全边界）→ 写细 system prompt（PROMPT_VARIANT）+ 写清工具 docstring（TOOLS_VARIANT），零模型成本优先做；能力不足（多轮跨轮记忆等提示词补不动的）→ 换更强模型（MODEL_NAME），贵、最后做。裁判按判什么选：有唯一答案用规则裁判 `exact_match`（快准零成本），主观开放输出才用 LLM-as-judge，机器裁判都要人工抽检校准。（D4《Agent评估与优化》4.3-4.5 / 2.8）
- 追问链：① 举个"prompt 补不动只能换模型"的实例？→ 多轮指代 `context_resolved` 用弱模型（coder-flash）再写 prompt 都补不动、会找错记录，换 deepseek-v4-pro 才从 0.17 到 1.00；而 `date_correct` 写细 prompt 就从 0.20 到 1.00；② 为什么有唯一答案的题千万别用大模型判对错？→ 大模型判"算得对不对"得自己再算一遍、本身可能算错，不如规则比对参考值又快又准。
<!--src:A; ref:D4《Agent评估与优化》4.3-4.5/2.8; look:career-sprint#D4-->

**Q12 · 上下文超窗要压缩，有哪几种技术？工具密集型和对话型 Agent 怎么选？**
- 考察点：压缩五件套的选型，按数据类型而不是一刀切。
- 答题要点：五件套。工具结果清除 ToolResultClear（LLM 把每条工具输出压成一行结论，`keep_recent=1` 保当轮原文，节省 80-90%）；观察遮蔽 ObservationMask（完全丢弃 Observation 只留占位符，工具密集型用、对话型禁用）；摘要 Summarization（超阈值把早期消息换成摘要，摘要放 messages 不放 system）；硬截断 MessageTrim（零 LLM 调用只留最近 N 条）；Compaction（全量替换历史开新窗口，150K→约 79K 质量不降）。口诀：工具密集型优先观察遮蔽/工具结果清除，对话型优先 trim + LLM 摘要。（D5《上下文工程进阶》）
- 追问链：① 为什么对话型 Agent 不能用观察遮蔽？→ 客服/教学场景的"观察"就是用户的话，遮蔽用户的话是灾难性信息丢失；② `keep_recent=0` 会出什么问题？→ 当轮查到的代码片段还没被 LLM 用上就压成一句话，Agent 答不出"第 42 行的逻辑"，所以 `keep_recent=1` 是黄金参数。
<!--src:A; ref:D5《上下文工程进阶》压缩五件套; look:career-sprint#D5-->

**Q13 · 有人为了省钱把压缩调得很激进，结果账单反而涨了，为什么？怎么设计才 cache 友好？**
- 考察点：prefix cache 与降本的真实关系，这是成本工程的关键认知。
- 答题要点：降本主力是 prefix cache 命中，不是省 token。DeepSeek prefix cache 按字节级前缀匹配，命中 token 按标准价 10% 计费（命中 ¥0.2/1M 对未命中 ¥2.0/1M，省 90% 输入成本），以 64 tokens 为存储单元，通常第 4-6 次请求开始命中。激进压缩改动消息中段，让从修改点起的后续 token 全部 cache miss，省几百 token 却让整段缓存归零。cache 友好设计：system prompt 放"你是谁"（静态利于缓存），messages 放"聊了什么"（动态摘要），叠加顺序从外到内 cache_boundary → tail_trim → summary → call_limit。课件把这条列为全课最大的单点降幅，且降幅来源不是"省 token"、而是"让 token 按 cache_hit 单价计费"——把 `cache_hit_tokens` 从 0 拉起来。（D5《上下文工程基础入门》3.1.5 + 进阶第 5 章）
- 追问链：① 为什么摘要绝不能塞进 system prompt？→ system prompt 每轮都变 cache 永远命不中，动态内容（PR 编号等）也不能放 system 前缀，要挪到 user message 尾部；② 判断是不是打碎了 cache 看什么指标？→ 看 `cache_hit_tokens` 是否被压缩打回 0，回 0 就是有中段删改破坏了前缀字节稳定。
<!--src:B; ref:D5《上下文工程基础入门》3.1.5；DeepSeek 官方 context caching 定价（命中为未命中 10%，一手核实）; look:career-sprint#D5-->

**Q14 · 不借框架手搓一个 Agent，主循环长什么样？从哪里退出、怎么防失控？**
- 考察点：Agent Loop 的本质，驾驭工程 gate 的核心。
- 答题要点：Agent 不是智能实体，是一个反复调用 LLM 的代码循环。四相：Gather Context（把完整 `messages` + 工具 JSON 描述发给大模型拿决策）→ Verify（`if not message.tool_calls` 为真即无工具调用、判定完成、终止返回最终答案，这是唯一出口）→ Take Action（有工具调用就执行，注意执行顺序）→ Iterate（回填结果进 messages 再下一轮）。防失控靠硬刹车 `max_steps=50`（`while steps < config.max_steps`）。课件 `mini_harness/` 是 11 文件 = 1 个 `config.py` 配置中心 + 10 个机制模块，`core.py` 承载 Agent Loop + Tool Use。（D6《手搓 Mini Harness》第二章）
- 追问链：① 这个循环本身有哪些脆弱点？→ 循环失控、吞掉工具报错、长会话把自己搞傻、无声烧钱，被归纳成八大故障模式，Harness 的价值就是系统性应对它们；② 八大核心机制里哪个管"结果对不对"？→ Verification Loop，LLM 的自我确认没有验证效力，一段代码只有被真实执行（如 pytest）过才算验证过。
<!--src:A; ref:D6《手搓 Mini Harness》第二章 2.1 + config.py; look:career-sprint#D6-->

**Q15 · 一个 Agent 在长会话里"越跑越傻/忘记决定"，和"越跑越贵越慢"，是同一个问题吗？怎么系统地给整段会话做体检？**
- 考察点：区分质量问题与成本问题，加会话诊断方法论。
- 答题要点：不是同一个。"越跑越贵越慢"多半是缓存问题：运行中改过 toolset / memory / system prompt 违反三铁律，前缀变了 cache 全作废（实测 200ms→1200ms）；"越跑越傻"是 Context Rot（质量问题）：旧信息、重复日志、过期决策把有效信号淹没，反制用 head+tail+middle summary 分段压缩、用 todo 清单替代靠历史记任务、子任务委派只传必要上下文。给整段会话系统体检用 HQS（Harness Quality Score，0-10 分），喂完整 turn-by-turn 轨迹，从五类失败模式归类：goal_loss（跟丢诉求）/ boundary_break（越权）/ context_loss（忘了前面说的）/ capability_gap（缺对的工具）/ hybrid（交织）。（D6《手搓 Mini Harness》4.2-4.3 + FF-OpenHermes 第 12 章 HQS）
- 追问链：① 为什么给会话打分必须喂整段轨迹而不是最后一轮？→ goal_loss / context_loss 是跨轮病症，只看最后一轮会漏掉"跟丢目标""前面说过又忘"；② HQS 里 LLM 自己生成的 `diagnosed_at` 时间戳能信吗？→ 不能，模型会编时间，落库前必须用真实服务端时间覆盖。
<!--src:A; ref:D6《手搓 Mini Harness》4.3 Context Rot + FF-OpenHermes 第12章 HQS 五类失败模式（源码 diagnostics.py:37 项目预定义）; look:career-sprint#D6-->

---

## 六、模拟面试官指令（压力面模式）

> 把下面整段复制给一个 Agent（或找同学扮演），让它给你上一场 Agent 部署与工程岗的压力面。
> 这是"考官"不是"教学助手"，它只考不教，追问到你答不出为止，最后逐题评价。

```
你现在是一位大模型应用 / Agent 工程岗的资深技术面试官，正在面试我。请严格进入「压力面」模式，全程遵守：

1. 角色纪律：你只做考官，不做辅导。面试进行中绝不解释知识点、绝不给正确答案、绝不提示，
   只提问、只追问、只记录。我答错或答不全，你不纠正，继续往下挖或换题。
2. 出题范围（从这几类里出，覆盖到位）：FastAPI async/sync 决策与并发、接口报错排查（422/404/CORS）、
   流式接口与 SSE/WebSocket 选型、生产上线四红线与上线验收、Docker/Compose 与 Dockerfile 常见坑、
   K8s 工作负载选型（StatefulSet/Deployment）、GPU 整数调度、容器与 K8s 故障诊断、
   评估可复现三件套与三杠杆优化、三种裁判、上下文压缩五件套与 prefix cache 降本、
   Agent Loop 四相与 max_steps、Prompt Cache 三铁律、Context Rot 与 HQS 会话诊断。
3. 压力面规则：每道主问题至少追问两层，往原理和量化深处挖（例如问完 async/sync，追问 100 并发实测数字、
   再追问 AnyIO 线程池默认容量）。我答得上就继续加深，直到我答不出或明显卡壳为止，再转下一题。
   语气专业、简洁、不寒暄、不鼓励。一次只发一个问题，等我回答后再发下一个。
4. 时长：共出 8~10 道主问题（含追问）。中途不要总结、不要给答案。
5. 收尾（全部问完后一次性输出）：
   - 逐题打分（0–5 分）并指出我每题在第几层追问卡住、暴露了哪个知识盲区；
   - 给一份「按薄弱点排序的复习清单」，只指出该回看哪个主题（如"GPU 整数调度""prefix cache 叠加顺序"），
     不展开讲解知识点；
   - 给一个总体判断：以当前水平能不能过 Agent 部署与工程岗技术面，还差哪几块。

现在开始，直接发第一个问题，不要复述这些规则。
```

---

## 七、能力自评表

> 每个技能点，诚实勾选你在哪一档。求职面试要求你至少到"能独立做"，核心项要能"教别人"（讲清原理与量化）。

| 技能点 | 能跟做（照着课件/Agent 走） | 能独立做（关掉 Agent 自己完成） | 能教别人（讲清原理与取舍） |
|---|---|---|---|
| FastAPI async/def 决策 + 并发量化 | ☐ | ☐ | ☐ |
| 接口报错排查（422/404/CORS/卡住） | ☐ | ☐ | ☐ |
| 流式接口（StreamingResponse + AsyncOpenAI） | ☐ | ☐ | ☐ |
| SSE/WebSocket 与 Response 类型选型 | ☐ | ☐ | ☐ |
| 生产上线四红线（凭证/loopback/鉴权/certbot） | ☐ | ☐ | ☐ |
| 上线验收六件产物 + 职责分离 | ☐ | ☐ | ☐ |
| Dockerfile + compose 编排（含常见坑） | ☐ | ☐ | ☐ |
| K8s 工作负载选型（StatefulSet/Deployment） | ☐ | ☐ | ☐ |
| GPU 整数调度与共享方案（独占/切片/MPS/MIG） | ☐ | ☐ | ☐ |
| 容器与 K8s 故障诊断（按症状定位） | ☐ | ☐ | ☐ |
| 评估可复现三件套 | ☐ | ☐ | ☐ |
| 三杠杆优化 + 三种裁判选型 | ☐ | ☐ | ☐ |
| 上下文压缩五件套 + prefix cache 降本 | ☐ | ☐ | ☐ |
| 手搓 Mini Harness（Agent Loop 四相 + max_steps） | ☐ | ☐ | ☐ |
| 三铁律 / Context Rot / HQS 会话诊断 | ☐ | ☐ | ☐ |

自评红线：只要"生产上线四红线""GPU 整数调度""评估可复现三件套""Agent Loop 四相"这四项还停在"能跟做"，就别急着投部署/工程岗，这几块是面试官最爱挖、也最容易露怯的地方。

---

## 八、落地对照（这批技能对应 JD 里的哪些要求）

> 把你 Stage D 练下来的本事，对齐到当下大模型应用 / Agent 工程 / LLMOps 岗招聘里真实出现的要求条目。
> 面试和简历里，用左边的技能去正面回应右边的 JD 语言。

| 你在 Stage D 掌握的 | 对应的典型 JD 要求条目 |
|---|---|
| FastAPI 服务化 + async 并发 + 流式接口（D1） | "熟悉 FastAPI，能把模型/Agent 能力封成高并发 HTTP/流式服务，理解异步编程" |
| 接口设计 + systemd/nginx/HTTPS 上线 + 鉴权（D2） | "负责 AI 应用后端接口设计与部署上线，掌握反向代理、HTTPS、接口鉴权与限流" |
| Docker/Compose + Kubernetes GPU 编排（D3） | "熟悉 Docker/K8s 容器化与集群编排，具备 GPU 资源调度与线上服务运维经验" |
| 容器与 K8s 故障诊断（D3） | "具备线上问题定位与故障排查能力，能独立处理部署/编排类线上故障" |
| Agent 评估体系 + LangSmith 可观测（D4） | "搭建 Agent/LLM 评估体系，用可量化指标做回归验证与效果优化（评估驱动开发）" |
| 上下文工程 + prefix cache 成本优化（D5） | "熟悉上下文工程与 Prompt/KV Cache 优化，有大模型推理成本控制经验" |
| 驾驭工程 / Agent Loop / 会话诊断（D6） | "理解 Agent 运行时机制（工具调用循环、上下文管理、验证），能工程化保障长会话稳定性"（2025–2026 新增高频要求） |
| 一个已服务化 + 容器化的 Agent（D6 gate 产物） | 简历项目栏可写"独立完成 Agent 从本机到公网 HTTPS、容器化交付、评估验收、成本优化的全链路"，证明你不是只跑过 demo |

**给简历/面试的一句话打法**：多数候选人只会写"用 LangChain 做了个 Agent demo"，你要能写"把 Agent 用 FastAPI 服务化（async 抗并发）、systemd+nginx+HTTPS 上线并守住四条安全红线、Docker/K8s 容器化并做 GPU 整数调度、用可复现评估体系做版本回归、靠 prefix cache 把月账单从 1200 压到 400"。这套"能上线、能扛并发、能证明没退化、能控成本"的工程完整度，就是这一波岗位里别人聊不深你能聊透的差异化。

---

*出品：赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套*
