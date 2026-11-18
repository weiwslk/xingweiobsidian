# 企业落地 / 一人公司 · Stage C 部署上线 · 过关手册

> 本册对应路径 business-landing 的 Stage C（部署上线）：把 B 阶段打样好的 MVP 接口化、容器化，变成别人能调用、随时能重启、换台机器也能起的服务。
> 两个节点：business-landing#C1（接口设计与部署上线）、business-landing#C2（容器化交付与集群编排入门）。
> 用法：先合上课件自己做，做完再对答案。练习向，重点在面试自测那一节，每题都带一层追问，把你逼到能讲清楚为止。

---

## 一、随堂练习

每题先自己动手或口述，再对【参考答案】，最后用【自查要点】给自己打分。

### 练习 1（C1）：把 ai-todo 的配置字段背下来并解释为什么配这三样

不看代码，写出 `config.py` 里 `Settings` 的服务端口、鉴权 key、模型 ID 三个字段的默认值，并说清为什么这些值要从 `.env` 加载而不是写死在代码里。

【参考答案】默认端口 `port = 12234`；鉴权 `api_key = "dev-local-key-change-me"`；模型 `llm_model = "z-ai/glm-5.1"`，走 OpenRouter 网关（`llm_base_url` 指向 OpenRouter 的 OpenAI 兼容 endpoint）调 GLM 5.1。配置从 `.env` 加载的三个理由课件写得很直白：代码里不出现裸 API key；不同环境（dev / prod）切换只改 `.env`；配置错了启动时 Pydantic 校验就报错，不会等到调用时才炸。`get_settings()` 用 `@lru_cache` 保证整个进程只加载一次 `.env`。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》1_主流Agent类型及接口设计 + code:ai-todo/config.py-->

【自查要点】端口 12234 记成 8000 算错；api_key 默认值忘了；说不清「启动即校验」这条 Pydantic 好处的，回看 business-landing#C1 的深潜道。

### 练习 2（C1）：手写一份最小可跑的 systemd unit，并解释三个关键字段

为 ai-todo 写 `/etc/systemd/system/ai-todo.service` 的 `[Service]` 段，要求进程崩了自动重启、开机自启，并说明 `Type=exec`、`--workers` 数字、`WantedBy` 分别管什么。

【参考答案】核心字段：`WorkingDirectory=/home/ubuntu/ai-todo`、`EnvironmentFile=/home/ubuntu/ai-todo/.env`、`ExecStart=/home/ubuntu/ai-todo/.venv/bin/gunicorn main:app -k uvicorn_worker.UvicornWorker --workers 5 --bind 127.0.0.1:12234`，配 `Restart=on-failure` + `RestartSec=5` + `StartLimitBurst=5` + `StartLimitIntervalSec=60s`，`[Install]` 段 `WantedBy=multi-user.target`。三个字段：`Type=exec` 等新进程映像真正加载完成才标 active，`Type=simple` 只要 fork 成功就标 active，会给启动命令写错、venv 不存在时假绿灯。`--workers 5` 来自 gunicorn 官方公式 `(2 × CPU 核心数) + 1`，腾讯云 2 核算出来是 5。`WantedBy=multi-user.target` 表示开机进入多用户模式时自动拉起。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》2_Agent智能体部署上线 第3章-->

【自查要点】worker 数写成随手一个 4 或 8、说不出公式的扣分；`--bind` 写成 `0.0.0.0` 的要注意：那是第二章的中间态，nginx 上线后收回 `127.0.0.1`；忘了 `gunicorn` 和 `uvicorn-worker` 不在 `requirements.txt` 里、要单独 `pip install` 的，回看第 3.3 节。

### 练习 3（C1）：把后端从公网裸跑收回 loopback，验证三个 curl 结果

nginx 反代配好后，你要把后端 host 从 `0.0.0.0` 改回 `127.0.0.1`（端口 12234 不变），并 kill 掉临时前端。写出改动命令，并预测三条验证 curl 的结果。

【参考答案】改 unit：`sudo sed -i 's|--bind 0.0.0.0:12234|--bind 127.0.0.1:12234|' /etc/systemd/system/ai-todo.service` → `daemon-reload` → `restart ai-todo`；`sudo pkill -f "http.server 12233"`；`sudo nginx -t && sudo systemctl reload nginx`。三条验证 curl 与预期：

```bash
curl -i  http://myagent-lab.online/health   # 期望 200，经 nginx 反代
curl -m 3 http://43.129.193.131:12233/       # 期望 connection refused，临时前端已停
curl -m 3 http://43.129.193.131:12234/health # 期望 timeout，后端收回 loopback 公网到不了
```

第三条 timeout 的原因：`127.0.0.1` 是 loopback 接口，数据包在内核里就路由回本机、不出物理网卡，公网到不了 12234 监听套接字。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.2.3节-->

【自查要点】把 12234 收回后还预期公网能直连的，说明没理解 loopback；忘了改端口不动、只改 host 的算对。

### 练习 4（C2）：为 ai-todo 写第一版 Dockerfile，讲清 COPY 顺序和非 root

写出 ai-todo 的 Dockerfile 关键行，并解释为什么先 `COPY requirements.txt` 再 `COPY . /app`，以及为什么要建 `app` 用户。

【参考答案】关键行：`FROM python:3.12-slim`；`WORKDIR /app`；`COPY requirements.txt .` + `RUN pip install -r requirements.txt`（课件里走清华 pip 源加速）；`COPY . /app`；`RUN groupadd ... && useradd ... app`（uid/gid 1000）+ `USER app`；`EXPOSE 12234`；`CMD ["uvicorn","main:app","--host","0.0.0.0","--port","12234"]`。先拷 `requirements.txt` 单独成层：依赖没变时这层缓存可复用，改源码重 build 时 pip install 那层直接命中 `CACHED`，build 从分钟级降到几秒。建非 root `app` 用户跑服务：万一被攻破，拿到的也只是 app 权限。CMD 用 exec 形式（JSON 数组）让 uvicorn 直接作为 PID 1，`docker stop` 的信号能直达。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第3章 + code:ai-todo/Dockerfile-->

【自查要点】把 `COPY . /app` 放在 pip install 前面的，缓存全废；用 root 跑、CMD 写成 shell 字符串形式的扣分。

### 练习 5（C2）：把 `.env` 挡在镜像外，用 `--env-file` 注入

说明 `.dockerignore` 为什么必须排除 `.env`，以及容器启动时配置从哪来。写出带端口映射、配置注入、数据卷的 `docker run` 命令。

【参考答案】`.dockerignore` 排除 `.env`（同时 `!.env.example` 放行模板）：镜像 push 到 Docker Hub 后任何人能 pull，`.env` 一旦进镜像，`OPENROUTER_API_KEY` 就泄露、数小时内可能被爬虫扫走刷爆 token 配额。所以镜像只装代码和依赖，配置在运行时注入：
```bash
docker run -d --name ai-todo \
    -p 12234:12234 \
    --env-file .env \
    -v ~/docker-data/ai-todo:/app/data \
    ai-todo:0.1
```
`-p 12234:12234` 把容器内端口映射到宿主机（前置课 nginx 的 `proxy_pass` 指向 `127.0.0.1:12234` 一行不用改）；`--env-file` 把 `.env` 里每行 `KEY=value` 注入容器环境变量；`-v` 把 `todos.db` 落到宿主机磁盘。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第3章 + code:ai-todo/.dockerignore-->

【自查要点】没提「镜像公开、key 会泄露」这条根因的答不到位；把配置写死进镜像、换环境重 build 的思路要纠正。

### 练习 6（C2）：用 compose 起 Milvus 三件套，解释启动顺序怎么保证

文搜图项目的 Milvus 由 etcd + minio + milvus-standalone 三件套组成。写出让 standalone 等前两者就绪再起的 compose 字段，并说清「服务名当 hostname」是怎么回事。

【参考答案】用 `depends_on` + `condition: service_healthy`：standalone 的 `depends_on` 写 `etcd: {condition: service_healthy}` 和 `minio: {condition: service_healthy}`，等两个底层组件都 healthy 再启。standalone 启动慢，`healthcheck` 里 `start_period: 90s` 给 90 秒预热窗口。standalone 通过 `ETCD_ENDPOINTS: etcd:2379`、`MINIO_ADDRESS: minio:9000` 找依赖：直接用服务名当 hostname，不配 IP，compose 内部网络的 DNS 自动解析。`docker compose -f docker-compose.deps.yml up -d` 后 `docker compose ps` 三行全 `(healthy)` 才算就绪，真正 healthy 还要等约 60-90 秒。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第5章 + code:text2image_search/docker-compose.deps.yml-->

【自查要点】`depends_on` 只写服务名不加 `condition` 的，只保证「先起进程」不保证「起好」；把服务间用 IP 互连、不用服务名的要改。

---

## 二、误区自查

逐条判断对错，把「以为对其实错」的地方揪出来。

1. **「后端 `uvicorn --host 0.0.0.0:12234` 在公网能打开，就算上线了。」** 错。这只是「连通验证」不是上线：SSH 一断进程就挂、reboot 不自启、崩了没人拉起、流量全明文、暴露面还大。正解是 systemd 守护 + 后端收回 `127.0.0.1` + nginx 统一在 80/443 接管入口。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第2.4/3章-->

2. **「`.env` 跟代码一起 scp、或者打进 Docker 镜像，反正传输是加密的没事。」** 错。传输加密（SSH 隧道）跟静态存储泄露是两回事：`.env` 要走独立 scp 通道 + 服务器上 `chmod 600`；容器化时 `.env` 必须 `.dockerignore` 排除、`--env-file` 注入，绝不打进镜像，因为镜像 push 后任何人能 pull。
<!--src:A; ref:Stage5/02 第2.3节 + Stage5/03 第3章-->

3. **「nginx 里给 SSE 端点配了 `proxy_pass` 就能流式输出。」** 错。反代默认会 buffer，token 会被攒住变成干等几十秒。SSE 端点要配三件套：`proxy_buffering off` + `gzip off` + `add_header X-Accel-Buffering no`，并把 `proxy_read_timeout` 加到 86400s 防长连接被踢。用 `location = /chat/events`、`location = /chat/stream` 精确匹配，优先级最高、绝不被业务正则抢走。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.3节-->

4. **「本机 `docker build` 出来的镜像本机能跑，推上云服务器就能跑。」** 错。Apple Silicon 是 arm64、腾讯云 CVM 是 amd64，arm64 镜像在 amd64 裸 Linux 上 `docker run` 会直接 `exec format error`。本机看起来能跑 amd64 镜像是 Docker Desktop 的 QEMU 仿真兜底（慢 5-10 倍），这层不上云。要用 `docker buildx build --platform linux/amd64` 主动 build 对架构。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第4.1/4.2节-->

5. **「`docker restart` 后数据还在，说明持久化没问题。」** 错。`docker restart` 只是把同一个容器停了再启、容器没删，数据当然在。真正要证明的是「容器删掉重建数据还在」：只有用 `-v` 把数据写到宿主机磁盘，`docker rm -f` 删容器后重建才不会丢。这就是 stateless 容器 + stateful 数据。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第3章卷挂载-->

6. **「nginx `-t` 通过、reload 也没报错，浏览器却 500，那是 nginx 的 bug。」** 错。这是每个按课程流程做的人都会撞的坑：`/home/ubuntu` 默认权限 750，nginx worker 以 `www-data` 身份跑，既不是 owner 也不在 group，读不到 `frontend/index.html`，`error.log` 里全是 `Permission denied`。修法是 `chmod o+x /home/ubuntu` 给穿越权 + `chmod -R o+rX .../frontend` 开读权，不是改 nginx。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.3.1节-->

---

## 三、迁移挑战

把 Stage C 学到的骨架搬到新场景，检验你是真会还是只会照抄。

### 迁移 1：把这套上线流程复制到你选定的第二个业务场景

你在 B 阶段选的是知识库 / 流程 / 运维三选一。现在要把「接口化 → systemd → nginx + HTTPS → 日志监控」这条路径复制到第二个场景。问：哪些件是可原样复用的通用件、哪些必须按场景重写？

【参考思路】可原样复用（沉淀成部署清单模板）：systemd unit 的骨架（只改 `WorkingDirectory` / `ExecStart` 路径与 worker 数）、nginx site config 的 TLS 卸载 + 80→443 重定向 + 限流防爬那几段、certbot 一键签证命令、logrotate 规则、`/health` 探针接法。必须按场景重写：业务路由前缀（ai-todo 是 `/chat/stream` `/chat/events` `/sessions` `/todos` `/health`，没有 `/api/` 统一前缀，换项目要按真实前缀重写 `location`）、哪些端点是 SSE 要走透传三件套、后端监听端口、`.env` 里的外部服务凭证。判定标准：如果第二个场景明显比第一个快，说明通用件真沉淀下来了。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.3节 + business-landing#C1-->

### 迁移 2：从 `docker run` 单容器迁到 compose 多服务

ai-todo 单进程用 `docker run -p -v --env-file --restart` 一条就够；文搜图有 5 个服务（etcd / minio / milvus-standalone / backend / frontend）。问：`docker run` 的哪些开关对应到 compose 的哪些字段？为什么多服务非用 compose 不可？

【参考思路】映射关系：`-p 宿主:容器` → `ports:`；`-v` → `volumes:`；`--env-file` → `environment:` 或 `env_file:`；`--restart unless-stopped` → `restart: unless-stopped`；`--name` → 服务名（服务名同时是内部网络 hostname）。多服务非用 compose 的原因：5 个服务的启动顺序、健康依赖（standalone 必须等 etcd/minio healthy）、内部网络互相寻址（服务名当 hostname、内部 DNS 解析）用 `docker run` 一条条起要 5 套命令、谁先谁后全靠人脑记、关机重启全手动。compose 把这些写进一份 yaml，`docker compose up` 一条命令拉起全部；`depends_on` + `healthcheck` 解决启动顺序，顶层 `networks` 解决服务发现。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第5章 + code:text2image_search/docker-compose.deps.yml-->

---

## 四、排错挑战

故障已经埋进下面这份真实配置。按【症状】→【根因】→【修法】三段走。

场景：你在 VPS 上按课程写好了 nginx 生产 site config（片段如下），`sudo nginx -t` 通过、`sudo systemctl reload nginx` 也没报错。

```nginx
server {
    listen 443 ssl http2;
    server_name myagent-lab.online;
    # ... certbot 注入的 ssl_* 若干行 ...

    location ~ ^/(sessions|todos|health) {
        proxy_pass http://127.0.0.1:12234;
        proxy_set_header Host $host;
    }

    location / {
        alias /home/ubuntu/ai-todo/frontend/;
        try_files $uri $uri/ /index.html;
    }
}
```
你只跑了 `sudo nginx -t && sudo systemctl reload nginx`，没跑别的。浏览器以 HTTPS 打开 `myagent-lab.online`。

【症状】`/health`、`/todos` 这类业务接口 curl 能拿 200，但浏览器打开根路径不是 ai-todo 界面，而是 **500 Internal Server Error**。`/var/log/nginx/myagent-error.log` 里刷满 `stat() "/home/ubuntu/ai-todo/frontend/index.html" failed (13: Permission denied)`。

【根因】业务接口走 `proxy_pass` 反代给后端进程，跟文件权限无关，所以 200 正常。但前端是 nginx 直接读磁盘文件 serve 的：nginx worker 以系统用户 `www-data` 跑，而 Ubuntu 上 `/home/ubuntu` 默认权限是 `drwxr-x---`（750），owner 是 `ubuntu`、group 是 `ubuntu`，其他用户（other）完全没权限。`www-data` 既不是 owner 也不在 group，被目录权限拦在外面，读不到 `frontend/index.html`，nginx 返 500。reload 不报错是因为语法没问题，坑在运行期权限。

【修法】给 nginx 开前端目录访问权限（两行，不是改 nginx 配置）：
```bash
sudo chmod o+x /home/ubuntu                       # 只加 x：能穿过去但列不出目录里别人的东西
sudo chmod -R o+rX /home/ubuntu/ai-todo/frontend  # 大写 X：目录和已有执行位的文件加 x，普通文件只加 r
```
再刷新浏览器，500 变成 ai-todo 三栏界面。注意 `o+x` 只加穿越权不加读权、大写 `X` 不会把 `.css` 误标成可执行，这两个细节是课件专门点破的。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.3.1节-->

---

## 五、面试自测

10 题，每题带【考察点】【答题要点】【追问】。先自己讲一遍，再看要点补漏，最后必须把追问也答上。

### 面试题 1：智能体服务化的本质是什么？

【考察点】是否理解「从本机脚本到生产端点」的观念转变，而不是只会敲命令。

【答题要点】服务化的本质是把「一次函数调用」变成「一个长期可用、有契约、能被重试的端点」。要搞清请求生命周期与错误边界：谁来调、并发多少、崩了怎么恢复、配置怎么外置、失败返什么。ai-todo 用 FastAPI 的 lifespan 做应用启停钩子（启动时建 agent / store 单例挂到 `app.state`）、`/health` 暴露探活、`.env` 外置配置，都是把「能跑」升级成「能上线」的动作。

【追问】lifespan 相比在模块顶层直接 new 一个 agent，好处是什么？
【追问参考】lifespan 是 FastAPI 推荐的启停钩子，资源初始化（模型加载、连接池、会话存储）放这里、关闭时统一清理，且拿到的是单例（`app.state.agent`），请求间复用而不是每次重建；模块顶层 new 会在 import 时就执行、时机不可控，也拿不到优雅关闭。
<!--src:A; ref:Stage5/01《智能体部署上线基础理论》+ Stage5/02 第1.2/2.3节 code:ai-todo/main.py-->

### 面试题 2：接口鉴权为什么要同时支持 `Authorization: Bearer` 和 `?api_key=`？

【考察点】是否理解浏览器原生 API 的限制，而不是随手写个鉴权。

【答题要点】`auth.py` 的 `verify_api_key` 默认从 `Authorization: Bearer <key>` 头读，这是 OpenAI / Claude / OpenRouter 等平台的事实标准，把 key 传递方式跟生态对齐；同时兼容 `?api_key=xxx` query 参数，因为浏览器两个原生场景加不了自定义 header：EventSource（SSE）不能加 header、`<a href>` 下载也不能加 header，这两种必须把 key 放 URL 里。Header 优先、Query 兜底，两者都缺返 401。`/health` 在鉴权之外，给部署层探活用。

【追问】把 key 放 query 参数有什么风险，生产上怎么补？
【追问参考】query 里的 key 会进 nginx access log、浏览器历史、Referer，比 header 更容易泄露；课程明说本课业务路由全公开、直接 curl，真实生产一定要鉴权，常见做法是 nginx Basic Auth、IP 白名单、Cloudflare Access、JWT 选一种，字符串相等比较的简单 key 只够教学。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第1章 code:ai-todo/auth.py-->

### 面试题 3：systemd unit 里 `Type=exec` 和 `Type=simple` 有什么区别？worker 数怎么定？

【考察点】是否理解「假绿灯」问题和 gunicorn worker 公式。

【答题要点】`Type=exec` 等新进程映像真正加载完成才把服务标 `active`；默认的 `Type=simple` 只要 fork 成功就立刻标 active，启动命令路径写错、venv 不存在时也会给假绿灯，课程全程用 `Type=exec` 拿更准的状态。`--workers 5` 来自 gunicorn 官方公式 `(2 × CPU 核心数) + 1`，腾讯云 2 核轻量服务器算出来正好 5。生产用 `gunicorn -k uvicorn_worker.UvicornWorker` 而不是直接 `uvicorn`：worker 生命周期管理、跟 systemd 干净协作、「开发跑 uvicorn、生产跑 gunicorn 套 uvicorn worker」是 FastAPI 生态事实标准。

【追问】`Restart=on-failure` 配了 `StartLimitBurst=5` + `StartLimitIntervalSec=60s` 是防什么？怎么验证自动重启真的生效？
【追问参考】防应用反复崩溃把 CPU 烧光：非零退出码自动重启，但 60 秒内最多重启 5 次、超过就放弃。验证：`sudo pkill -9 -f gunicorn` 后 `sleep 6`，再 `systemctl status ai-todo` 应仍 `active (running)` 但主 PID 已变，说明 systemd 在 `RestartSec=5` 后拉起了新实例。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第3.3节-->

### 面试题 4：为什么要用 nginx 做反向代理、把后端收回 `127.0.0.1`，而不是让 uvicorn 自己挂证书对外？

【考察点】TLS 卸载 + 职责分离 + loopback 隔离。

【答题要点】nginx 监听 443 承担 TLS 卸载（解开 HTTPS 拿到明文 HTTP，再转给后端 `127.0.0.1:12234`）。选 nginx 三个理由：职责分离（nginx 专管 TLS，应用重启不影响 HTTPS 握手）、certbot 深度集成（一条命令签发 + 改配置 + 自动续期 + reload）、工业事实标准（限流 / SSE 透传 / 静态资源 / 多应用共用 443 都是它的开箱标配）。后端收回 `127.0.0.1` 后是 loopback 接口，数据包在内核里就路由回本机、不出物理网卡，公网 curl 12234 会 timeout，只能经 nginx 进来。uvicorn 虽有 `--ssl-keyfile` 能自挂证书，但应用跟 TLS 绑死、证书续期要重启应用、性能不如 nginx 的 C 实现，工程上几乎不这么走。

【追问】`proxy_set_header X-Forwarded-Proto $scheme` 少配会出什么问题？
【追问参考】HTTPS 在 nginx 终止后转给后端的是明文 HTTP，后端若不看这个 header 会以为客户端本来就走 HTTP，导致 redirect 链路搞错（比如生成 http:// 的跳转），所以要把真实协议透给后端。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.1/4.2节-->

### 面试题 5：HTTPS 比 HTTP 多做了哪三件事？证书为什么必须由 CA 签？

【考察点】TLS 三性质 + 信任锚。

【答题要点】HTTPS = HTTP + TLS，TLS 做三件事：加密（中间节点看到的是密文）、防篡改（改动密文浏览器解密时 MAC 校验会失败）、服务器身份证明（CA 签发的证书证明「这台服务器确实是 myagent-lab.online」）。加密和防篡改由协议本身自动完成，身份证明必须有外部信任锚：浏览器只信内置 CA 公钥列表里的 CA 签的证书。握手时浏览器用内置 CA 公钥验证证书签名 + 验服务器持有匹配私钥，双方再各出随机数临时协商出一把从未上网传输的会话密钥做对称加密。Let's Encrypt 是免费 CA，只签 DV（域名验证级）证书，有效期 90 天、到期前 30 天可续。

【追问】中间人想伪装这个域名，直接也去 CA 申一张同名证书行不行？
【追问参考】不行。CA 签发前要求申请人证明控制该域名（certbot 跑的 ACME HTTP-01 挑战就是在做这件事：nginx 80 端口能通 + `.well-known/acme-challenge` 路径可达）。中间人控制不了该域名的 DNS、碰不到这台 VPS，拿不到证书；浏览器解密握手时发现签名对不上，直接 `ERR_CERT_AUTHORITY_INVALID` 拒连。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.1节-->

### 面试题 6：nginx 里 SSE 端点为什么要单独配、怎么配？location 匹配优先级是怎样的？

【考察点】SSE 透传三件套 + location 精确匹配。

【答题要点】SSE（`/chat/events`、`/chat/stream`）用 `location =` 精确匹配，优先级最高，绝不被业务正则抢走。透传三件套：`proxy_buffering off` + `gzip off` + `add_header X-Accel-Buffering no always`，防御性关掉所有缓冲层，否则 token 被 buffer 攒住变干等；再把 `proxy_read_timeout` / `proxy_send_timeout` 加到 86400s（默认 60s 长连接会被踢）；还要 `proxy_set_header Connection ""` 维持长连接。location 优先级：`=` 精确匹配最高、`~` 正则次之、`location /` 普通前缀兜底。ai-todo 业务路由用 `location ~ ^/(sessions|todos|health)` 正则反代，没有 `/api/` 统一前缀，得按真实前缀写。

【追问】本机测 SSE 明明顺畅，为什么课程还坚持配三件套？
【追问参考】Ubuntu 默认 nginx 没开 gzip、FastAPI `StreamingResponse` 用 chunked 主动 flush，所以默认就顺；但生产容易撞坑：同事改全局 `gzip on`、上了 CDN 边缘 buffer、换发行版默认 buffer on，三件套是把本 location 锁死、不受全局影响的防御性配置。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.3节-->

### 面试题 7：入口限流和防爬怎么配？为什么对敏感路径返 404 而不是 403？

【考察点】limit_req / limit_conn + 信息不泄露。

【答题要点】限流先在 `http` 层定义共享内存 zone（放 `/etc/nginx/conf.d/`）：`limit_req_zone $binary_remote_addr zone=api_rate:10m rate=5r/s`（10MB 约装 16 万个 IP 状态、每 IP 每秒 5 请求）和 `limit_conn_zone ... zone=conn_per_ip:10m`。业务路由和普通流式端点叠 `limit_req zone=api_rate burst=10 nodelay`（允许短暂突发到 10、再超返 429）；多事件 SSE 端点叠 `limit_conn conn_per_ip 2`（单 IP 最多 2 条并发，防恶意开 100 条 SSE 拖死后端）。防爬用 `location ~ /\.(env|git|venv) { deny all; return 404; }`。返 404 而不是 403：403 等于告诉对方「路径存在但被禁了」，404 连「文件存不存在」的信息都不给，公网每分钟都有扫描器试 `/.git/config`、`/.env`。

【追问】限流的 key 为什么用 `$binary_remote_addr` 而不是 `$remote_addr`？
【追问参考】两者都是按客户端 IP 限流，`$binary_remote_addr` 是 IP 的二进制紧凑形式，占用共享内存更小，同样的 zone 内存能装下更多 IP 状态，是 nginx 官方推荐的限流 key 写法。
<!--src:B; ref:Stage5/02 第4.3.2节；nginx 官方 limit_req 文档（$binary_remote_addr 省内存为官方推荐）-->

### 面试题 8：Docker 的镜像、容器、仓库分别是什么？容器和虚拟机差在哪？

【考察点】三对象关系 + 容器隔离机制。

【答题要点】镜像是 build 出来的不可变只读模板（代码 + 解释器 + 依赖 + 配置，按层组织）；容器是镜像跑起来的运行实例（有自己的文件系统 / 网络 / 进程空间，但共用宿主机内核）；仓库是镜像的远程存放与分发处（Docker Hub 域名 `docker.io`，云厂商也有自家仓库）。容器 vs 虚拟机：VM 靠 Hypervisor 虚拟出完整硬件、每台跑一个独立 OS 内核，隔离最彻底但每台几个 GB、启动几十秒；容器不虚拟硬件、共用宿主 Linux 内核，靠 namespace（隔离视图）和 cgroup（资源控制）两个内核机制隔离进程，镜像几十 MB、启动不到一秒。一句话：VM 模拟整台电脑，容器只隔离进程。

【追问】既然容器共用宿主内核，为什么 Docker Desktop 在 macOS 上还要跑一个 Linux 虚拟机？
【追问参考】容器依赖 Linux 的 namespace / cgroup 内核机制，macOS 和 Windows 本身没有，所以 Docker Desktop 底层偷偷跑一个轻量 Linux VM 来承载容器，我们敲的命令只是控制这个 VM 里的 docker daemon。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第1章-->

### 面试题 9：本机是 Apple Silicon，要把镜像部署到腾讯云 amd64，会撞什么坑？怎么解？

【考察点】跨架构 build + buildx + Hub push 规则。

【答题要点】本机 arm64 build 的镜像直接 push 到 Hub、再在腾讯云 amd64 上 `docker run`，容器进程起不来、报 `exec format error`（裸 Linux 无仿真兜底，内核看到不认识架构的二进制直接拒绝）。本机 Apple Silicon 上跑 amd64 镜像看似能跑，是 Docker Desktop 的 QEMU 透明仿真兜底、慢 5-10 倍，这层不上云。解法：`docker buildx build --platform linux/amd64 -t <你的Hub用户名>/ai-todo:latest --load`（`--load` 加载到本机镜像列表，`--push` 直接推 Registry）。push 前用 `docker inspect ... | grep Architecture` 验一眼。Docker Hub 硬性规则：镜像名前缀必须等于你的 Hub 用户名，不一致直接 401/403 拒绝；push 前 `docker login` 现在用 Personal Access Token，密码登录已淘汰。

【追问】多平台一次 build（`--platform linux/arm64,linux/amd64`）为什么 `--load` 会报错？
【追问参考】本机 docker daemon 存储不支持一个 `name:tag` 同时挂多份架构（每个 tag 只能挂一个具体 manifest），所以多平台只能 `--push`：Registry 用 manifest list（OCI Image Index）实现「一个 tag 挂多架构」，`docker pull` 时按客户端 CPU 自动挑对应那份返回。本课只 build amd64 一份给云端，单平台不撞这个限制。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第4章-->

### 面试题 10：文搜图为什么在 Mac 上用 hybrid（Milvus 进 docker、backend 留 host）？全 docker 上生产服务器时又怎么改？

【考察点】GPU 加速的宿主约束 + compose 服务发现。

【答题要点】Apple Silicon 的 Metal / MPS 加速被苹果锁死在 macOS 原生进程，Docker Desktop 底层的 Linux VM 拿不到，所以 Mac 上走 hybrid：Milvus 三件套进 docker（省去手装 etcd / minio / 编译 Milvus），backend 留在 host 用 venv 跑才能吃到 MPS（单次 embedding 1-3 秒），backend 通过 `MILVUS_URI` 指向 `localhost:19530` 连 docker 里的 Milvus。上生产服务器（Linux + NVIDIA GPU）时全 docker：backend 进容器、连接改成指向 `milvus-standalone:19530`（服务名当 hostname、compose 内部网络解析），GPU 通过 `deploy.resources.reservations.devices` 声明 `driver: nvidia / count: 1 / capabilities: [gpu]`，靠 Docker 29+ 内置 CDI 或 nvidia-container-toolkit 透传。

【追问】backend 的 GPU 镜像为什么用多阶段构建？healthcheck 的 `start_period` 为什么设到 1800s？
【追问参考】多阶段构建：builder 阶段用完整 `python:3.11` 装依赖、runtime 阶段切 `python:3.11-slim` 只 COPY venv，把 5 GB 镜像（cu128 wheel 自带 cuDNN/cuBLAS 等 NVIDIA 用户态库）做小。`start_period=1800s`（30 分钟）是给首次从 ModelScope 下两个 2B 模型（约 8 GiB）预留的窗口，期间 `docker compose ps` 显示 `unhealthy` 是模型还在下、不是出错。另外注意：清华源没有 cu128 wheel，误用会静默拉到 CPU 版 torch、embedding 从 1-2 秒退化到 30-60 秒，torch 要走 `download.pytorch.org/whl/cu128`。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第5章 + code:text2image_search/backend/Dockerfile-->

---

## 六、模拟面试官指令

把下面这段贴给一个 Agent，让它扮演面试官对你连环追问，你只能凭记忆和课件作答，不许让它替你回答。

```
你是一位资深的 Agent 应用上线面试官。我刚学完「企业落地 / 一人公司」路径的 Stage C：把一个 FastAPI + LangGraph 的 Agent（ai-todo）接口化、部署到腾讯云香港 VPS（systemd + nginx + certbot HTTPS + 日志监控），并容器化（Dockerfile + compose 多服务，了解 K8s 编排入门）。

请按下面规则面我：
1. 从「服务化本质 → 接口鉴权 → systemd 守护 → nginx 反代与 HTTPS → SSE 透传与限流 → 容器化 → 跨架构 → compose/K8s」的顺序，一次只问一题，等我答完再追问。
2. 每题至少追问一层「为什么」或「不这么做会怎样」，我含糊时继续逼问，不要放水。
3. 我答错时不要直接给答案，先反问一个能让我自己发现漏洞的问题（例：我说「uvicorn --host 0.0.0.0 就上线了」，你反问「SSH 断开后这个进程还在吗？reboot 之后呢？」）。
4. 重点盯这几个高频翻车点：后端有没有收回 loopback、.env 有没有独立走 scp 且 chmod 600、SSE 有没有关 buffer、镜像架构对不对、.env 有没有被 .dockerignore 挡在镜像外、compose 依赖有没有等 healthy。
5. 全部问完给我一份分项评价：哪些能独立讲清、哪些只是背下来、哪些还没通，并指出我该回看 business-landing#C1 还是 #C2 的哪一节。
```

---

## 七、能力自评表

对每个技能点，诚实圈出你现在的档位：能跟做（照课件抄能跑通）／能独立做（合上课件自己能从头做）／能教别人（能讲清为什么、能诊断别人的坑）。

| 技能点 | 能跟做 | 能独立做 | 能教别人 |
|---|---|---|---|
| 讲清智能体服务化本质（端点 / 契约 / 生命周期）+ lifespan 单例 | ☐ | ☐ | ☐ |
| 设计并解释接口鉴权双通道（Bearer header + query 兜底）| ☐ | ☐ | ☐ |
| 手写 systemd unit 守护后端（Type=exec / worker 公式 / 自动重启）| ☐ | ☐ | ☐ |
| 把后端收回 loopback + nginx 反代，理解 TLS 卸载与 loopback 隔离 | ☐ | ☐ | ☐ |
| certbot 一键签 HTTPS + dry-run 续期，讲清 CA / DV / 90 天 | ☐ | ☐ | ☐ |
| 配 SSE 透传三件套 + location 优先级 + 限流防爬 | ☐ | ☐ | ☐ |
| 排 nginx reload 通过但 500（www-data 权限）这类运行期坑 | ☐ | ☐ | ☐ |
| 写第一版 Dockerfile（COPY 顺序 / 非 root / exec CMD）+ .dockerignore 挡 .env | ☐ | ☐ | ☐ |
| 用 buildx 跨架构 build amd64 并 push Docker Hub，避开 exec format error | ☐ | ☐ | ☐ |
| 用 compose 编排多服务（depends_on + healthcheck + 服务名 DNS + 卷持久化）| ☐ | ☐ | ☐ |
| 讲清 compose→K8s 的映射（Deployment/StatefulSet、runtimeClassName + nvidia.com/gpu）| ☐ | ☐ | ☐ |

自评规则：整张表如果有一行还停在「能跟做」，说明那一节你只会照抄，回看对应节点的深潜道再练一遍；目标是至少把前 10 行推到「能独立做」，把上线相关的前 7 行推到「能教别人」。

---

## 八、落地对照

对照 Stage C 该交付的产物，逐条核你手上是不是真有、真能跑。

1. **接口化服务**：选定场景 MVP 的接口层，本地可启动、你亲手 `curl` 通拿到正确业务结果（不靠 Agent 代跑），并能手绘一张请求从入口到模型再到响应的数据流图。对应 business-landing#C1 深潜验收。
<!--src:A; ref:business-landing#C1 checkpoint-->

2. **公网可访问的 HTTPS 服务**：ai-todo（或你的场景服务）以 HTTPS 跑在你的域名上，地址栏有锁标，`curl -i` 打域名的 `/health` 返 200，后端已收回 `127.0.0.1`、nginx 是唯一入口。对应部署上线课的四件齐活：入口 / 加密 / 隔离 / 续期。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第4.2节-->

3. **进程守护 + 可观测**：systemd 守着后端（崩了自动重启、reboot 自启、日志归 journal），logrotate 给 nginx 和应用日志都配了轮转（daily / rotate 14 / compress），`/health` 探针可接外部监控。对应部署上线课第 3、5 章产物。
<!--src:A; ref:Stage5/02《主流智能体接口设计与部署上线》第3/5章-->

4. **容器化交付**：一条 `docker build` 出镜像、`docker run` 起容器，`.env` 走 `--env-file` 注入不进镜像、数据走卷持久化（删容器重建数据还在），跨架构镜像已 `--platform linux/amd64` build 并能在 amd64 服务器跑起。对应 business-landing#C2 checkpoint。
<!--src:A; ref:business-landing#C2 checkpoint + Stage5/03 第3/4章-->

5. **多服务一键起**：多服务场景有一份 `docker-compose.yml`，`docker compose up` 一条命令拉起全部，`depends_on` + `healthcheck` 保证启动顺序，服务间用服务名互连，改配置无需改镜像。对应 business-landing#C2 的验收标准。
<!--src:A; ref:Stage5/03《智能体容器化交付与集群编排入门》第5章-->

6. **可复制的部署清单**：把 systemd unit 骨架、nginx site config（TLS + 限流防爬）、certbot 命令、Dockerfile / compose 模板抽成清单，能明显更快地复制到第二个场景。对应路径承诺的可复制方法包。
<!--src:A; ref:business-landing promise + Stage5/02 第6章-->

---

赋范空间（FuFan Space）·《大模型 Agent 开发实战》学习路径配套
