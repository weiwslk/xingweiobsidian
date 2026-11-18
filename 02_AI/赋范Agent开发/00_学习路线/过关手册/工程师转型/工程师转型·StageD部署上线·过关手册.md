# 工程师转型 Agent 开发 · Stage D 部署上线 · 过关手册

> 这一册对着 Stage D 的四个必修节点出题：D1 智能体部署上线基础理论、D2 主流智能体接口设计与部署上线、D3 智能体容器化交付与集群编排入门、D5 Agent 评估与优化（D4 追踪与运维课件更新中，本册不出题）。题目全部从课件与 ai-todo / text-analyzer-api / 文搜图等真实源码里来，数字、字段名、命令都逐字核对过。
>
> 用法：先自己动手做，再看参考答案；面试自测部分每题都带一层追问，练的是被面试官往深里问时还能接得住。

---

## 一、随堂练习

每题配【参考答案】和【自查要点】。做的时候先合上答案自己写，再对。

### D1 智能体部署上线基础理论

**练习 1.1** 有人把一个能在自己电脑跑通的 `demo.py` 直接发给朋友，声称"这就算把服务给别人用了"。按课件"脚本与服务的本质差异"表，指出这种做法漏掉了服务形态的哪几个维度。

【参考答案】课件那张对照表列了六个维度：谁触发执行（脚本是自己在终端敲命令，服务是任何人发 HTTP 请求都能触发）、生命周期（脚本跑完就退出，服务 7×24 小时常驻）、多端复用（脚本需要对方装 Python 环境，服务是浏览器/curl/任何语言都能调）、协议规范（脚本的函数签名只有写代码的人看，服务走 HTTP+JSON 跨平台标准）、输入校验（脚本自己写 if/else，服务由框架统一处理）、错误返回（脚本 raise Exception，服务返 HTTP 状态码加错误体）。发 `demo.py` 本质还是"一次性运行"，不是"能被任意调用方持续调用"。
【自查要点】能说出"最大的变化不是代码量，而是被任意调用方使用这个新的存在方式"，而不是只背维度名。
<!--src:A; ref:讲次《智能体项目开发必备基础 FastAPI》1.3 脚本到服务的本质区别-->

**练习 1.2** FastAPI 常被说成"不是从零实现的框架"。说出它内部的三个核心组成部分，各自负责什么。

【参考答案】三个组成：Starlette 提供路由和中间件，把 HTTP 请求翻译成调用哪个 Python 函数；Pydantic 提供数据校验，把请求里的 JSON 翻译成 Python 对象、反过来也行；OpenAPI 生成器把代码自动转成 OpenAPI 规范，再渲染出 Swagger UI 交互文档。FastAPI 的价值不在把这些能力重写一遍，而在把成熟能力组装好、给一个统一顺手的开发接口。
【自查要点】能把 Pydantic 和 OpenAPI/Swagger 的因果链串起来：协作需要规范，规范标准是 OpenAPI，OpenAPI 的 Python 写法是 Pydantic，所以写 6 行代码就白得一份可交互文档。
<!--src:A; ref:讲次《智能体项目开发必备基础 FastAPI》3.1 向内：FastAPI 由什么组成-->

**练习 1.3** `llm-stream-api` 的 `stream.py` 里有一个 `/chat/benchmark` 接口，用 `_simulate_async_job`（`await asyncio.sleep(0.5)` 循环十次）和 `_simulate_sync_job`（`time.sleep(0.5)` 循环十次）做并发对照。`concurrency` 参数默认多少、取值范围是什么？课程实测同任务 sync 比 async 慢多少倍？

【参考答案】`BenchmarkRequest` 里 `concurrency: int = Field(default=100, ge=1, le=300)`，默认 100、范围 1 到 300。单个模拟任务是 sleep 0.5 秒重复十次、约 5 秒。async 版用 `await asyncio.sleep` 会把控制权交还事件循环，同一 worker 能同时挂起大量等待型请求；sync 版 `time.sleep` 阻塞线程。课程能力包实测同任务 sync 比 async 慢 3.02 倍。课件还给了直觉：同步 worker 模式下单次请求 15 秒，一分钟大约只能处理 4 个。
【自查要点】关键是理解"LLM 调用天然是等待型 IO，async 在等上游返回时让出事件循环"，不是背 3.02 这个数。
<!--src:A; ref:code:llm-stream-api/stream.py 的 /chat/benchmark；pack:agent-fastapi-service-pack 使用原则-->

### D2 主流智能体接口设计与部署上线

**练习 2.1** ai-todo 部署课里前端和后端各监听哪个端口？为什么第二章要在腾讯云控制台临时加两条入站规则，第四章 nginx 上线后又要把它们收回？

【参考答案】后端 uvicorn/gunicorn 监听 12234，前端临时用 `python -m http.server` 监听 12233，与本机 dev 模式端口分配一致。云控制台默认只放行 22/80/443，第二章要让公网直接打开 `公网IP:12233` 看前端、前端 JS 跨域调 `公网IP:12234`，所以手动加 TCP 12233、12234 两条来源 0.0.0.0/0 的入站规则。第四章 nginx 接管入口后，后端 host 从 `0.0.0.0` 收回 `127.0.0.1`（端口 12234 不变）、kill 临时前端、nginx 在 80/443 统一接管，这两个公网端口就退役、控制台规则删掉，公网入口只剩 22/80/443。
【自查要点】能讲清"loopback = 127.0.0.1 对应的虚拟网卡，只监听它等于物理隔离，公网摸不到，只能由本机的 nginx 进来"。
<!--src:A; ref:讲次《Agent 智能体部署上线》第二章 uvicorn 公网裸跑、第四章 nginx 入口-->

**练习 2.2** 写 ai-todo 的 systemd unit 时，`[Service]` 段里 `WorkingDirectory`、`EnvironmentFile`、`ExecStart` 三个字段分别对应你本地手敲的哪条命令？为什么本课全程用 `Type=exec` 而不是默认的 `Type=simple`？

【参考答案】三字段一一映射手敲命令：`cd /home/ubuntu/ai-todo` 对应 `WorkingDirectory`，读取 `.env` 对应 `EnvironmentFile`，启动命令对应 `ExecStart`。`Type=exec` 让 systemd 等到新进程映像真正加载完成才把服务标 active；默认的 `Type=simple` 只要 fork 成功就立刻标 active，启动命令路径写错、venv 不存在时也会给假绿灯。用 exec 拿到的状态信号更准。
【自查要点】unit 文件不是网上抄的模板，是把手敲命令翻译成 systemd 能读懂的格式；`ExecStart` 里 gunicorn 走绝对路径 `/home/ubuntu/ai-todo/.venv/bin/gunicorn`，`--workers 5` 来自公式 (2×CPU 核心数)+1、腾讯云 2 核算出 5。
<!--src:A; ref:讲次《Agent 智能体部署上线》第三章 systemd 守护进程-->

**练习 2.3** 这门部署课为什么锁定腾讯云香港 VPS 加 `.online` 国际域名，而不是大陆机房加中国域名？HTTPS 证书为什么必须找 CA 签、Let's Encrypt 签的是什么等级、有效期多久？

【参考答案】`.online` 是国际域不需要 ICP 备案、香港机房也不在大陆工信部管辖范围，走港区免备案直连能立刻对外解析；大陆机房加中国域名拿到域名后必须先做 ICP 备案（通常 2 到 3 周）才能解析，节奏太慢。HTTPS 的"服务器身份证明"需要外部信任锚：CA 是浏览器内置信任的机构，给域名签证书证明"这把公钥的持有者确实控制该域名"，签发前用 ACME 流程要求申请人证明控制该域名，中间人碰不到 DNS 和 VPS 就拿不到证书。Let's Encrypt 是免费 CA，只签 DV（域名验证）证书，有效期 90 天、到期前 30 天可续期，certbot 一条命令搞定签发、改配置、自动续期、reload。
【自查要点】能补一句"将来要承接大陆 to-C 业务，备案是绕不过去的合规要求"，说明选港区是节奏取舍不是安全捷径。
<!--src:A; ref:讲次《Agent 智能体部署上线》第一章 ICP 备案说明、第四章 4.1 为什么需要 HTTPS-->

### D3 智能体容器化交付与集群编排入门

**练习 3.1** 看 ai-todo 的 `Dockerfile`：为什么先单独 `COPY requirements.txt` 再 `pip install`、之后才 `COPY . /app`？它为什么建一个 uid/gid 1000 的普通用户跑服务、`CMD` 为什么写成 `["uvicorn", ...]` 这种数组形式？`EXPOSE` 的是哪个端口？

【参考答案】先拷 `requirements.txt` 再装依赖，是为了利用镜像分层缓存：依赖没变时这一层缓存可复用，改源码重 build 时不必重装依赖，快很多。建 `--system --uid 1000 --gid 1000` 的普通用户 app 并 `USER app`，是不用 root 跑服务、缩小攻击面。`CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "12234"]` 是 exec 形式，uvicorn 直接作为容器 PID 1，`docker stop` 的信号能直达进程、优雅退出；shell 形式会多包一层 sh、信号传不进去。`EXPOSE 12234`。
【自查要点】能点出 `.dockerignore` 排除 `.env`/`.venv`/`data`/`__pycache__`，密钥不进镜像；基础镜像是 `FROM python:3.12-slim`。
<!--src:A; ref:code:ai-todo/Dockerfile；讲次《容器化部署：Docker + Compose》第三章-->

**练习 3.2** 本机是 Apple Silicon（arm64）、腾讯云是 amd64，`docker run hello-world` 输出里带 `(arm64v8)`。这说明镜像有什么属性、会带来什么坑？国内为什么装完 Docker Desktop 要立刻配 mirror，Docker Hub 匿名拉取的限额是多少？

【参考答案】镜像 build 时会被编译成特定架构的二进制，带"架构"这个属性：arm64 镜像只能在 arm64 机器上跑，本机 arm64 镜像不能直接拿去 x86 服务器起容器，跨架构要用 buildx 构建多架构镜像。Docker Hub（registry-1.docker.io）架在美国，国内链路又慢又抖，且对匿名拉取限额很严：每 6 小时 100 次。mirror 是 Hub 的国内透明代理，把镜像缓存在国内 CDN、拉取快还绕开匿名限额，常用 `docker.m.daocloud.io` 等；透明代理只对公开仓库生效，真私有镜像仍要 `docker login`。
【自查要点】能说清 mirror 是"透明代理"——`docker pull` 不用改镜像名，daemon 自动走 mirror。
<!--src:A; ref:讲次《容器化部署：Docker + Compose》第二章 2.2 配 mirror、2.4 hello-world-->

**练习 3.3** K8s 跟 Docker Compose 最大的差别是什么？用一行 `kubectl apply -f deployment.yaml` 声明"要 3 个 frontend Pod"后，控制面和工作节点各组件是怎么把它变成运行中的 Pod 的？为什么集群内访问从来不用 Pod IP？

【参考答案】核心差别是从命令式变声明式：不说"去启动一个 Pod"，而说"我期望集群里始终有 3 个 frontend Pod"，K8s 用 reconcile loop 不停比较期望与实际、差了就补多了就杀。apply 链路：yaml 发给 api-server，走完认证/授权/admission 准入写入 etcd（期望状态）；controller-manager 里 Deployment Controller watch 到新对象建一个 ReplicaSet，ReplicaSet Controller 再 watch 到建 3 个 Pending Pod；scheduler watch 到 Pending Pod，跑 filter 加 score 两阶段决策把 Pod 的 nodeName 绑到某节点；该节点的 kubelet watch 到"这个 Pod 是我的"，调 containerd 拉镜像起容器、汇报状态回 api-server，Pod Running。Pod IP 不稳定：delete 后重建是全新 Pod、IP 重分配，所以集群内永远用 Service DNS 名访问。
【自查要点】能补一句 kube-proxy 名字带 proxy 但不转发数据包，它只把 Service/EndpointSlice 变化翻译成 iptables 规则，真正转发的是 Linux 内核 netfilter。
<!--src:A; ref:讲次《Kubernetes 集群编排入门》第一章 1.1 全景、1.3.4 kubectl apply 7 步链路-->

### D5 Agent 评估与优化

**练习 5.1** "凭感觉觉得它还行"为什么靠不住？说出课件里 Agent 通用七类评估角度，并解释为什么单靠"任务结果评估"定位不了问题。

【参考答案】Agent 输出是自然语言或一串动作，同一问题常有多种都算对的答法，没法拿 `==` 比；它分多步走，中间调错工具、漏步、绕路，哪怕最后蒙对过程也是坏的；模型带随机性，今天对明天可能错；它还会真删数据改状态。通用七类：任务结果评估、工具与动作评估、过程轨迹评估、依据与状态一致性评估、多轮交互与状态保持评估、规则安全与权限评估、抗扰稳定性与回归稳定性评估。任务结果评估只看终点不看路径，没法告诉你错在哪个环节，所以要配过程轨迹评估等一起看。
【自查要点】能说出任务结果和抗扰稳定性两类是"全部 Agent 都适用、没有弱适用场景"，工具/轨迹/依据类对纯对话助手弱适用。
<!--src:A; ref:讲次《Agent 评估与优化》第一章 1.1 到 1.7 通用七类-->

**练习 5.2** LangSmith 分哪四大功能块、各管什么？把一个纯 Agent 接上 LangSmith 只做哪三件事、要往 `.env` 里补哪四个变量？

【参考答案】四大功能块：Tracing（看一次跑了啥，记录单次运行完整轨迹，是过程轨迹评估的数据来源）、Datasets & Experiments（看一批对不对，用题库批量打分出成绩单）、Comparison（看哪版更好，多个版本成绩单并排比，承接抗扰稳定性与回归稳定性评估）、Annotation Queues（人工逐条打分校准机器裁判）。接入三件事：去 LangSmith 生成一把 API Key、往 `.env` 补四个变量、在业务入口函数上加一行 `@traceable` 装饰器，业务逻辑几乎一行没改。四个变量：`LANGSMITH_TRACING=true`（总开关）、`LANGSMITH_API_KEY`、`LANGSMITH_PROJECT`（给项目取名，trace 归到它名下）、`LANGSMITH_ENDPOINT`（服务地址）。
【自查要点】能说清 Dataset 是题库、Example 是一道题（含输入和参考答案）、Experiment 是跑一遍打一遍分的成绩单；trace 只记录不打分。
<!--src:A; ref:讲次《Agent 评估与优化》第二章 2.1 到 2.3 LangSmith 四大块与接入-->

**练习 5.3** 做回归对比的评估，为什么必须先固定可复现？说出可复现三件套。计算器项目里 `create_agent` 的模型为什么设 `temperature=0`？

【参考答案】没有可复现的评估，优化就是玄学：分数上蹿下跳时你分不清是改动生效了还是随机波动。可复现三件套：固定时间、每例重置种子、串行 `max_concurrency=1`（三件套缺一，成绩单就不能拿来比版本）。计算器 Agent 设 `temperature=0` 是固定为 0、让结果可复现，这样同一道题多次跑答案稳定，才能拿去比。课件里的计算器还刻意用大数相乘（12345678 × 87654321 = 1082152022374638）逼模型必须调 `multiply` 工具而不是心算。
【自查要点】能补一句第三章给记账 Agent 跑评估时，`evaluate` 会加 `max_concurrency`（串行保证可复现）和 `metadata`（标注模型与 prompt 版本）两个参数。
<!--src:A; ref:讲次《Agent 评估与优化》第二章 2.2 最小 Agent、2.6 Experiment；pack:agent-evaluation-pack 可复现三件套-->

---

## 二、误区自查

一条条对，命中就是你还没吃透的点。

1. **"部署就是把 uvicorn 在服务器上跑起来。"** 错。手动 SSH 上去前台起 uvicorn，SSH 一断服务就挂、reboot 起不来、进程崩了没人重启，这只算连通验证不算上线。工业上线的基线是 systemd 守护：崩了自动重启、reboot 自启、SSH 断开互不影响。<!--src:A; ref:讲次《Agent 智能体部署上线》第三章-->

2. **"生产环境后端应该监听 0.0.0.0 才能对外服务。"** 错。nginx 上线后后端要收回 `127.0.0.1`（loopback），公网完全摸不到后端端口，只能经 nginx 从 80/443 进来。直接把应用监听公网是暴露面太大、缺 TLS、没限流。<!--src:A; ref:讲次《Agent 智能体部署上线》第四章-->

3. **"把 API Key 直接写进代码或打进镜像最省事。"** 错。`.env` 里的 `OPENROUTER_API_KEY` 泄露后别人能刷你的 token 配额。纪律是绝不 git push、scp 单独传、服务器 `chmod 600`、`.dockerignore` 排除 `.env` 不进镜像，容器里靠环境变量注入。<!--src:A; ref:讲次《Agent 智能体部署上线》2.3 .env 安全；code:ai-todo/Dockerfile-->

4. **"涉及 LLM 调用的路由用 def 还是 async def 无所谓。"** 错。LLM 调用是等待型 IO，选 `def` 在高并发时会塌方。课程实测同任务 sync 比 async 慢 3.02 倍。写涉及 LLM/网络/DB 的路由前先判异步姿势。<!--src:A; ref:pack:agent-fastapi-service-pack 使用原则；code:llm-stream-api/stream.py-->

5. **"Docker 就是更省资源的虚拟机。"** 错。VM 模拟整台电脑、每台跑独立 OS 内核、启动几十秒；容器共用宿主机 Linux 内核，靠 namespace 和 cgroup 隔离文件系统/网络/进程空间，镜像几十 MB、启动不到一秒。一句话：VM 模拟整台电脑，容器只隔离进程。<!--src:A; ref:讲次《容器化部署：Docker + Compose》1.4 容器 vs 虚拟机-->

6. **"K8s 就是 yaml 写得更复杂的 Compose。"** 错。K8s 补齐了 Compose 给不了的三件能力：跨节点调度、自动扩缩、更强的自愈（livenessProbe 失败重启、readinessProbe 失败摘流量、副本偏离自动补齐、节点宕机跨机迁移），Compose 的 restart 只管容器进程退出。<!--src:A; ref:讲次《Kubernetes 集群编排入门》1.1 全景-->

7. **"最后答案对了，这个 Agent 就是好的。"** 错。任务结果只看终点：中间调错工具、漏确认、绕路死循环，哪怕蒙对答案过程也是坏的。要配过程轨迹评估，而轨迹评估依赖完整 trace，这正是 LangSmith 的 Tracing 要提供的数据。<!--src:A; ref:讲次《Agent 评估与优化》1.1、1.3-->

8. **"成绩单低分就该换个更强更贵的模型。"** 错。看到低分先归因：指导不足（prompt 或工具说明能补，便宜）还是能力不足（换模型，贵）。三杠杆要按先便宜后贵的顺序：改 prompt、改工具说明、再换更强模型，别一上来换最贵的。<!--src:A; ref:pack:agent-evaluation-pack 使用原则；讲次《Agent 评估与优化》第四章-->

---

## 三、迁移挑战

把 Stage D 学的东西挪到没讲过的新场景，考的是迁移不是复述。

**迁移 1** 你 B 阶段手搓的那个不依赖框架的 DeepResearch 内核现在只能在本机命令行跑。现在要把它做成一个能被前端调用、能流式返回、能上线被别人访问的服务。按 Stage D 的顺序，写出你从"本机脚本"到"可被调用的线上服务"要补齐的四件事，每件各给一个具体动作。

【参考方向】① 接口：用 FastAPI 把内核包成 HTTP 接口，Pydantic 定请求/响应模型，多轮检索的实时输出走 `StreamingResponse` + `async def` + `async for yield`，跨域配 CORSMiddleware。② 容器：写 `Dockerfile`（先 COPY requirements.txt 装依赖再拷源码、非 root、exec 形式 CMD、EXPOSE），`.dockerignore` 拦 `.env`，需要向量库等依赖就用 docker-compose 编排。③ 追踪：给关键调用接可观测性看每次请求的耗时与 token（对应 D4，本课件更新中，按其上线后标准补）。④ 评估：设计评估集与指标，固定可复现三件套跑一份评估报告，据数据定一处优化。上线部署走 systemd + nginx 反代 + certbot HTTPS。
<!--src:A; ref:engineer-transform#D1 累进检查；讲次《Agent 智能体部署上线》全篇-->

**迁移 2** 文搜图项目的 `docker-compose.prod.yml` 里 backend 服务用 `deploy.resources.reservations.devices` 声明了 1 张 nvidia GPU，5 个服务靠 healthcheck 和 depends_on 编排。现在要把这套 compose 搬上 K8s 做多副本 GPU 推理。说清楚：compose 的 `restart` 到 K8s 换成什么、GPU 在 K8s 里怎么申请、为什么本课这套 3090 + k3s 实验按"整卡调度"理解最稳。

【参考方向】compose 的 `restart: unless-stopped` 只管进程退出，K8s 换成更强的自愈：livenessProbe 失败重启容器、readinessProbe 失败从 Service 摘流量、副本数偏离自动补齐、节点宕机跨机迁移。GPU 申请：2017 年 NVIDIA 的 Device Plugin 把宿主机 GPU 注册成 `nvidia.com/gpu` 扩展资源，pod 用 `nvidia.com/gpu: 1` 整卡申请，调度粒度只到整数卡。2025 年 8 月 K8s 1.34 把 DRA（Dynamic Resource Allocation）核心升级到 GA，能按显存/算力精细分配，但要集群版本、DRA driver、云厂商都配齐才能用，本课这套 3090 + k3s + Device Plugin 仍按整卡调度理解最稳。k3s 默认用嵌入式 SQLite 替代 etcd、把 kube-proxy 合进 agent 进程，`kubectl get pods -n kube-system` 看不到独立 kube-proxy Pod 是正常现象。
<!--src:A; ref:code:text2image_search/docker-compose.prod.yml；讲次《Kubernetes 集群编排入门》1.1、第四章-->

---

## 四、排错挑战

一段真实的坏代码，你来当医生。

**背景**：某同学参考 `llm-stream-api` 写了个 Agent 对话服务，本机点着都正常，一压测就崩。他的路由长这样（故障已埋进代码）：

```python
from fastapi import FastAPI
from openai import OpenAI   # 注意：同步客户端
import time

app = FastAPI()
client = OpenAI(base_url="...", api_key="...")

@app.post("/chat")
def chat(req: ChatRequest):          # 注意：普通 def
    resp = client.chat.completions.create(
        model="qwen/qwen-turbo",
        messages=[{"role": "user", "content": req.message}],
    )
    return {"reply": resp.choices[0].message.content}
```

【症状】单个请求 curl 一切正常，响应也对；但一上 100 并发，绝大多数请求排长队、超时，服务器 CPU 没打满但吞吐塌到几乎不动。压测数据里同样的负载，改造后比现在快约 3 倍。

【根因】两处叠加。第一，路由写成普通 `def`，FastAPI 会把它丢进有限的线程池跑；第二，里面用的是同步 `OpenAI` 客户端，`create()` 是阻塞调用，一次 LLM 调用以秒计，线程被死死占住不让出。等待型 IO 全程阻塞线程，线程池很快被占满，后续请求只能排队。这正是"LLM 调用天然适合异步等待"却用了同步姿势的塌方，对应课程实测 sync 比 async 慢 3.02 倍。

【修法】把路由改成 `async def`，客户端换成 `AsyncOpenAI`，调用处 `await`，等上游返回时把控制权交还事件循环，同一 worker 就能同时挂起大量等待型请求。像 `llm-stream-api/stream.py` 那样：`llm = AsyncOpenAI(...)`、`async def chat_stream(...)`、`stream = await llm.chat.completions.create(..., stream=True)`、`async for chunk in stream`。若确实要用同步阻塞库，退一步的做法是把阻塞调用丢线程执行（如 `anyio.to_thread.run_sync`），别在事件循环里直接阻塞。
<!--src:A; ref:code:llm-stream-api/stream.py；pack:agent-fastapi-service-pack 使用原则-->

---

## 五、面试自测

十道题，每题带【考察点】【答题要点】【追问】。追问是面试官往深里问的那一层，先把主问答顺，再顶住追问。

**Q1** 把一个 Python 脚本升级成能被别人调用的后端服务，到底要补齐什么？为什么 2026 年的 AI 工程里 FastAPI 是高频选择？

【考察点】脚本与服务的本质区别、FastAPI 在 AI 时代的定位。
【答题要点】补的不是代码量而是"被任意调用方持续使用"这个新形态：谁触发（任何人发 HTTP）、生命周期（常驻）、多端复用（任何语言可调）、协议规范（HTTP+JSON/OpenAPI）、输入校验（框架统一）、错误返回（HTTP 状态码）。FastAPI 高频三条：LLM 调用天然适合异步等待（async def + ASGI）、结构化输出离不开 Pydantic（接口和 LLM 工具调用共用同一套模型）、Python AI 生态大量服务层就是 FastAPI/ASGI 这条路线。
【追问】FastAPI 内部三大组成各管什么？答：Starlette 路由与中间件、Pydantic 校验、OpenAPI 生成器出文档，FastAPI 只是把成熟能力组装好给统一接口。
<!--src:A; ref:讲次《智能体项目开发必备基础 FastAPI》1.3、3.1、3.4-->

**Q2** 一个涉及 LLM 调用的 FastAPI 路由，你怎么决定写 `def` 还是 `async def`？有没有数据支撑？

【考察点】异步姿势决策、并发原理。
【答题要点】LLM/网络/DB 这类等待型 IO 优先 `async def`，让出事件循环、同一 worker 挂起更多等待请求；纯 CPU 密集或只能用同步阻塞库时另作处理。数据：`stream.py` 的 benchmark 用 async/sync 两种 sleep 模拟，课程能力包实测同任务 sync 比 async 慢 3.02 倍。
【追问】为什么同步 worker 模式一分钟大约只能处理 4 个请求？答：假设单次请求 15 秒，同步 worker 同一时刻只能处理一个，一分钟 60 秒除以 15 秒约 4 个；async 在等待时交还事件循环所以能并发挂起更多。
<!--src:A; ref:讲次《智能体项目开发必备基础 FastAPI》3.4；code:llm-stream-api/stream.py-->

**Q3** 对话助手 Agent 对外要暴露哪几条业务出口？工具调用 Agent 比它多了什么？

【考察点】Agent 类型与接口设计，业务出口先于协议。
【答题要点】对话助手四条：聊天接口、流式回复接口（同一聊天能力的打字机返回方式）、会话历史接口、会话管理接口（新建/删除/改标题，管的是 session 状态）。聊天和流式本质是一件事的两种返回方式。工具调用 Agent 在对话四条基础上多了工具调用过程接口（按事件推送"思考/工具调用/工具返回/最终回复"）和工具结果接口，本质是把"它在干什么"产品化，事件流是它和对话助手最大的差别。
【追问】任务执行 Agent 又多了什么对象？答：任务和会话分成两个对象，任务有自己的 ID 和状态生命周期（queued→running→done/failed/cancelled），要单开任务创建/状态/结果/取消/步骤日志一组出口。
<!--src:A; ref:讲次《主流 Agent 类型及接口设计》第一章 1 到 3-->

**Q4** 把一个 Agent 做成 API 服务化，最核心的设计原则是什么？"本地能通、并发一上就崩"你怎么排？

【考察点】无状态化与水平扩展、部署故障诊断。
【答题要点】核心是无状态化与幂等：把会话状态外置，接口才能水平扩展。会话状态如果放进程内全局变量（像 text-analyzer 里 `HISTORY = {}` 那样），服务重启即丢、多实例之间也不共享，真实项目要接数据库持久化。并发崩多半是请求间共享了可变状态或缺超时控制，按课件把 Agent 调用改成无状态加超时控制，别在请求间共享可变状态。
【追问】上线后为什么还要收回 loopback、加鉴权？答：nginx 上线后后端收回 127.0.0.1 缩小暴露面；业务路由默认公开会被扫到随意调用（LLM 接口会烧 token 配额），生产必须加鉴权，nginx Basic Auth / IP 白名单 / JWT 选一种。
<!--src:A; ref:讲次《Agent 智能体部署上线》第二章、第四章；code:text-analyzer-api/analyzer.py-->

**Q5** 为什么不能手动 uvicorn 就算上线，要交给 systemd？unit 文件里 `Restart`/`StartLimit` 这组字段防的是什么？

【考察点】进程守护、systemd 三件套。
【答题要点】手动前台 uvicorn 有三个隐患：SSH 断进程死、reboot 不自启、崩了没人拉起。systemd 是 PID 1、所有进程的祖先，用 unit 文件 + systemctl + journal 三件套守护。`Restart=on-failure` 非零退出码自动重启，`StartLimitBurst=5` + `StartLimitIntervalSec=60s` 限定 60 秒内最多重启 5 次、超过就放弃，防应用反复崩溃把 CPU 烧光；`RestartSec=5` 是重启间隔。
【追问】怎么验证自动重启真的生效？答：`sudo pkill -9 -f gunicorn` 杀掉主进程，`sleep 6` 后 `systemctl status` 仍显示 active 但主 PID 已变，说明 systemd 在 RestartSec 后拉起了新实例；reboot 自启不用单测，`enable --now` 里的 enable 已配好。
<!--src:A; ref:讲次《Agent 智能体部署上线》第三章 3.1 到 3.4-->

**Q6** HTTP 到底不安全在哪？HTTPS 解决了哪三件事？为什么把 TLS 卸载交给 nginx 做、中间人搞不到同名证书？

【考察点】HTTPS 原理、TLS 卸载、证书信任链。
【答题要点】HTTP 是明文：请求响应像明信片，中间任何能抓包的节点都能读（对话内容/会话 ID 裸奔）、还能改（运营商塞广告、注入脚本），协议本身没有内容是否被改的校验。HTTPS = HTTP + TLS，TLS 三件事：加密（中间节点只看到密文）、防篡改（改动密文解密时 MAC 校验失败）、服务器身份证明（CA 签的证书）。nginx 做 TLS 卸载：监听 443、把 HTTPS 解开拿明文 HTTP 转给后端 `127.0.0.1:12234`，好处是职责分离、certbot 深度集成、工业事实标准。中间人拿不到同名证书是因为 CA 签发前用 ACME 要求证明控制该域名，中间人控制不了域名 DNS 也碰不到 VPS。
【追问】HTTPS 的加密密钥是浏览器提前知道的吗？答：不是。握手时浏览器用内置 CA 公钥加证书公钥验真假，双方各出随机数和临时公钥临时协商出一把会话密钥，这把钥匙从不在网络上传输，之后用它对称加密通信。
<!--src:A; ref:讲次《Agent 智能体部署上线》4.1 为什么需要 HTTPS-->

**Q7** 讲讲 Docker 镜像分层，以及你会怎么写 Dockerfile 让重 build 更快、更安全。

【考察点】镜像分层与 Dockerfile 工程实践。
【答题要点】镜像按 layer 组织、每层是一次文件系统变更的差分，是不可变制品，通常基于 base image 叠自己的层（`FROM python:3.12-slim`），命中缓存的层直接跳过、不同镜像共享底层 base layer。工程实践：先 COPY requirements.txt 再 pip install、之后才 COPY 源码（依赖没变时装依赖那层缓存复用）；建非 root 用户（uid/gid 1000）跑服务；`.dockerignore` 排除 `.env`/`.venv`/`__pycache__` 让密钥和本地 venv 不进镜像；CMD 用 exec 数组形式让进程作为 PID 1、docker stop 信号直达。
【追问】本机 arm64 build 的镜像能直接上 amd64 服务器吗？答：不能，镜像带架构属性、arm64 只能跑 arm64，跨架构要用 buildx 构建多架构镜像。
<!--src:A; ref:code:ai-todo/Dockerfile；讲次《容器化部署：Docker + Compose》1.2、第二章 2.4-->

**Q8** compose 已经能单机编排多服务，为什么还要上 K8s？K8s 的 reconcile loop 是什么？

【考察点】编排升级、声明式与控制器机制。
【答题要点】compose 是单机编排、restart 只管进程退出；K8s 补齐跨节点调度、自动扩缩、更强自愈三件。reconcile loop 是"声明期望 + 持续对齐"：你写"我要 3 个副本"，controller-manager 里几十个 controller 各自 watch 关心的对象、周期性比较期望与实际、不一致就修正。api-server 是唯一枢纽，所有组件不直接通信全走它、它本身无状态、状态全在 etcd（k3s 默认用嵌入 SQLite 替代）。
【追问】集群内为什么用 Service 名而不是 Pod IP？kube-proxy 转发数据包吗？答：Pod IP 不稳定（重建就换），永远用 Service DNS 名；kube-proxy 不转发数据包，它只把 Service/EndpointSlice 变化翻译成 iptables 规则写进内核，真正转发的是 Linux 内核 netfilter。
<!--src:A; ref:讲次《Kubernetes 集群编排入门》1.1、1.2.4、1.3-->

**Q9** 为什么 Agent 比传统程序难评估？你会从哪些角度评一个 Agent？

【考察点】评估认知地图、通用七类。
【答题要点】难在：输出是自然语言/动作没法拿 `==` 比、多步走要看轨迹、模型带随机性、有真实副作用（删数据改状态）、多轮要记上文、每改 prompt 换模型可能修好 A 弄坏 B。评估分两层：通用七类（任务结果、工具与动作、过程轨迹、依据与状态一致性、多轮交互与状态保持、规则安全与权限、抗扰稳定性与回归稳定性）回答"作为智能体是否可靠"；按 Agent 类型追加的特殊角度回答"作为某类业务 Agent 是否专业"，查表挑几个叠加通用七类拼出完整清单。
【追问】任务结果评估的局限是什么，为什么要配轨迹评估？答：任务结果只看终点不看路径、定位不了错在哪个环节；过程轨迹评估看漏步/绕路/重复/死循环，但依赖完整 trace，这正是 LangSmith 的 Tracing 提供的数据来源。
<!--src:A; ref:讲次《Agent 评估与优化》第一章-->

**Q10** 一份评估成绩单出来了，某项指标偏低。你怎么从这份成绩单走到一次真正的优化，且不把别的指标改坏？

【考察点】评估驱动优化、可复现与回归。
【答题要点】先保证可复现（固定时间、每例重置种子、串行 max_concurrency=1），否则分数波动分不清是改动还是随机。再对低分归因：指导不足（prompt 或工具说明能补，便宜）还是能力不足（换模型，贵）。按三杠杆先便宜后贵的顺序动手：改 prompt、改工具说明、换更强模型，每改一次重测并在 Comparison 界面并排比，确认目标指标涨了且原本满分的没被改坏才算通过。裁判按判什么选：有唯一答案用规则裁判（快、准、零成本），主观开放输出才用 LLM-as-judge，机器裁判都要人工抽检校准。
【追问】为什么别一上来就换最贵的模型？答：换模型是最贵的杠杆且有代价，回归测试常逮到"换模型后好几条原来能答对的用例集体掉分"的退化；很多低分其实是 prompt 或工具说明写不清、补一句就好，先便宜后贵更划算。
<!--src:A; ref:讲次《Agent 评估与优化》第四章；pack:agent-evaluation-pack 使用原则-->

---

## 六、模拟面试官指令

把下面这段整段发给你的 Agent，让它扮演面试官，闭卷抽考你 Stage D，不许你翻笔记、也不许它替你答。

> 你现在是一位大模型 Agent 工程岗的技术面试官，我是候选人，正在面部署上线方向。请按下面规则主持一场 20 分钟的模拟面试：
> 1. 从这五个主题各抽至少一题，顺序打乱：脚本到服务的本质区别与 FastAPI 组成、async/sync 并发决策、Agent 接口设计与业务出口、systemd + nginx + certbot 上线链路、Docker 镜像分层与跨架构、K8s reconcile loop 与 Service/Pod、Agent 评估七类角度与可复现三件套、评估驱动优化三杠杆。
> 2. 每题我答完，你必须追问一层往深挖（例如我讲 async 更好，你就追问同步 worker 一分钟大概能处理几个 15 秒的请求、为什么）。
> 3. 我答错或含糊时先别公布答案，用一个反问把我逼到正确路径上；两次还不对再给标准答案并指出我卡在哪。
> 4. 全程别替我写代码、别替我组织答案，只提问、只追问、只点评。
> 5. 结束时给我一张三行小结：答得稳的、一追问就露怯的、完全没接住的，各列两条，并指出我该回看 Stage D 哪个节点。

---

## 七、能力自评表

对着技能点自评三档：能跟做（照着课件/Agent 带能做出来）、能独立做（合上课件自己能做）、能教别人（能给别人讲清为什么这么做）。诚实打，露馅的那栏就是你的回看清单。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 讲清脚本与服务的六维本质差异 | ☐ | ☐ | ☐ |
| 用 FastAPI + Pydantic 写带 Body/Path/Query 三入口和 HTTPException 的规范接口 | ☐ | ☐ | ☐ |
| 判断路由该写 async def 还是 def 并说清并发原理 | ☐ | ☐ | ☐ |
| 用 StreamingResponse + AsyncOpenAI 写流式 LLM 接口、配 CORS | ☐ | ☐ | ☐ |
| 按 Agent 类型设计业务出口（对话/工具/任务/工作流） | ☐ | ☐ | ☐ |
| 写 systemd unit 守护服务，配 Restart/StartLimit 并验证自动重启 | ☐ | ☐ | ☐ |
| 用 nginx 反代做 TLS 卸载、certbot 签 HTTPS 并把后端收回 loopback | ☐ | ☐ | ☐ |
| 写利用分层缓存、非 root、exec CMD 的 Dockerfile，`.dockerignore` 拦密钥 | ☐ | ☐ | ☐ |
| 写 docker-compose 编排多服务（healthcheck/depends_on/卷/GPU） | ☐ | ☐ | ☐ |
| 讲清 K8s reconcile loop、控制面/工作节点组件、Service 与 Pod IP | ☐ | ☐ | ☐ |
| 说全 Agent 评估通用七类角度并给每类举例 | ☐ | ☐ | ☐ |
| 用 LangSmith 建题库、跑 Experiment、做 Comparison 版本对比 | ☐ | ☐ | ☐ |
| 固定可复现三件套、按三杠杆先便宜后贵做评估驱动优化 | ☐ | ☐ | ☐ |

<!--src:A; ref:engineer-transform#D1 engineer-transform#D2 engineer-transform#D3 engineer-transform#D5-->

---

## 八、落地对照

Stage D 的验收要拿真东西对，不是打卡。逐条对，缺哪件补哪件。

1. **手写的 Agent API 服务过 curl 冒烟**：一个对话/任务接口，用 curl 发真实请求拿到正确响应，异常入参返回规范错误码；核心接口逻辑自己写、脚手架可让 Agent 生成。回看锚 engineer-transform#D2。<!--src:A; ref:engineer-transform#D2 验收-->

2. **无鉴权被拒、异常入参规范报错**：接口加上基本鉴权与错误处理，无鉴权请求被拒、异常入参返回规范错误码，curl 记录提交仓库。回看锚 engineer-transform#D2。<!--src:A; ref:engineer-transform#D2 子验收-->

3. **服务被打成镜像、容器里健康检查通过**：D2 的服务用你写的 Dockerfile 构建成功，容器内服务健康检查通过、curl 可访问，Dockerfile 与运行日志提交仓库。回看锚 engineer-transform#D3。<!--src:A; ref:engineer-transform#D3 验收-->

4. **一页部署方案草图**：标出你的 Agent 上线要解决的无状态化/并发/鉴权/成本四点，逐点写出处理方式。回看锚 engineer-transform#D1。<!--src:A; ref:engineer-transform#D1 验收-->

5. **一份可复现的 Agent 评估报告**：评估集 + 指标 + 优化前后对比，评估脚本可复跑得到相同指标，报告给出至少一处基于数据的优化动作，评估集/脚本/报告提交仓库。回看锚 engineer-transform#D5。<!--src:A; ref:engineer-transform#D5 验收-->

6. **进入 E 阶段前的累进自检**：不看笔记说出一个 Agent 从"本机脚本"到"可被调用的线上服务"要补齐的四件事（接口/容器/追踪/评估），各举一个你实际做过的动作。回看锚 engineer-transform#D2。<!--src:A; ref:engineer-transform#D 累进检查-->

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
