# 产品/业务岗 AI 全栈 · 人读版

> 不以写码为业，也能把一个想法带到「生产级 PRD → 自己主导的 MVP → 部署跑通的工业项目」。
> 在平台正式上线前，这份文档就是你的完整地图：阶段序列表=地铁图、到站验收勾选清单=进度条、阶段里程碑达成声明=徽章。

---

## 这条路适合谁

- **适合**：不以写代码为业、但要对产品结果负责的人——产品经理、业务/运营负责人、创业者、想独立做出 MVP 的探索者。会用电脑、能读懂需求，但不打算靠手写代码谋生。
- **不适合（都给了出口，不关门）**：
  - 已以写代码为业、想深入 Agent 工程框架/RAG/微调的工程师 → 改走 **engineer-transform**（起点即 Agent 开发进阶）
  - 连终端与 AI 编辑器都没碰过、想先建立信心的零基础新手 → 改走 **zero-to-builder**（节奏更缓，多一站 Python 基础再回到产品线）
  - 只想旁听不想每站交产出的人 → 本路径每站要交产出物，建议先单点试听 Stage 3 再决定

## 入口自测（用节点号定位从哪站进）

1. 用过 Cursor 或 Claude Code 这类 AI 编辑器写过东西吗？没用过 → 从 **A1** 开始
2. 独立用 AI 编辑器跑通过一个前后端小应用吗？跑通过 → **A2** 可快速过，重点放在 A3
3. 写过 Cursor Rules 或接过一次 MCP 吗？会 → **A3** 可快速过，直接进阶到 B1
4. 亲手写过一份结构完整、可交付的 PRD 吗？没写过 → **C1** 是本路径重心，务必不要跳
5. 日常已在用 Claude Code 做工业级开发、想学 Agent 框架而非产品交付？是 → 建议改走 **engineer-transform**

## 节奏与承诺（留余量口径）

- **推荐节奏**：约每周 8 小时、共 12 周。
- **可产出（附工时区间，非「N 周必得」）**：一份生产级 PRD（含 6 篇调研笔记，约 15h）、一个由你主导「需求→设计→测试」全程的 MVP（AlphaProject 智能投研平台，约 35h）、两个部署跑通并逐项自检的工业项目（约 13h）。实际用时随基础与投入浮动。

---

## 阶段序列表（地铁图降级）

```
A 会用 AI 编辑器做出能跑的东西
   A1 导学·环境配置 ──▶ A2 Vibe Coding·ChatGPT克隆 ★ ──▶ A3 Cursor进阶+MCP·NL2SQL
        │                                                        │
        ▼（课程包全家桶安装：A3 提前准备预告，B 阶段起快车道由课程包驱动）
B 教会 Agent 按你的方法干活（Skills）
   B1 Agent Skills 详解 ★ ──▶ B2 Spec-Kit 与 Superpowers
        │
        ▼
C AlphaProject 三部曲（本路径核心）
   C1 上篇·调研+生产级PRD ★ ──▶ C2 中篇·架构选型+UI+环境 ──▶ C3 下篇·全栈测试体系
        │
        ▼
D 项目落地：两个工业项目部署跑通并自检
   D1 FF-Coworker AI 产品经理 ★ ──▶ D2 私域运营autopilot / AI HR小慧（二选一）
        │
        ▼
   并轨 → engineer-transform · 阶段 B（Agent 开发进阶）
```

`★` = 该阶段的**深潜专属到站验收**（关掉 Agent 代工才能完成），阶段徽章闸门就在它上面：快车道拿产出、深潜道拿徽章，原理不可跳过。

| 阶段 | 节点 | 标题 | 主/支线 | 难度 | 视频h | 实操h | 快车道 |
|---|---|---|---|---|---|---|---|
| A | A1 | 导学：环境配置与工具准备 | 主线 | 入门 | 2 | 2 | 基座 Agent 轻快车道 |
| A | A2 ★ | Vibe Coding 范式（ChatGPT 克隆） | 主线 | 入门 | 4 | 6 | 基座 Agent 轻快车道 |
| A | A3 | Cursor 进阶 + MCP（NL2SQL 助理） | 主线 | 进阶 | 5 | 6 | cursor-vibe-coding-pack |
| B | B1 ★ | Agent Skills 详解：原理到从零开发 | 主线 | 进阶 | 5 | 5 | agent-skills-authoring-pack |
| B | B2 | 顶尖 Skills：Spec-Kit 与 Superpowers | 主线 | 进阶 | 6 | 5 | spec-driven-dev-pack / superpowers-workflow-pack |
| C | C1 ★ | AlphaProject 上篇：调研 + 生产级 PRD | 主线 | 进阶 | 5 | 6 | ai-product-dev-pack |
| C | C2 | AlphaProject 中篇：架构选型·UI·环境 | 主线 | 进阶 | 5 | 7 | spec-driven-dev-pack / ai-product-dev-pack |
| C | C3 | AlphaProject 下篇：全栈测试体系 | 主线 | 硬核 | 5 | 7 | ai-product-dev-pack / superpowers-workflow-pack |
| D | D1 ★ | 部署 #1：FF-Coworker AI 产品经理 | 主线 | 进阶 | 2 | 4 | ai-product-dev-pack |
| D | D2 | 部署 #2：autopilot / 小慧（二选一） | 主线 | 进阶 | 2 | 5 | ai-product-dev-pack |

> 时长为经验估算、因人而异。路径中提到的课件、能力包与项目都在本套课程资料内，照着走不需要任何课程外资源。

---

## 阶段达成声明（徽章依据）

- **阶段 A 徽章**：你已经能用 AI 编辑器从零做出前后端应用，并接 MCP 做出自然语言查数据库的助理。*闸门：A2 深潜专属到站验收。*
- **阶段 B 徽章**：你已经能把一个工作方法写成会触发的 Skill，并用 Spec-Kit 把需求拆成 spec/plan/tasks。*闸门：B1 深潜专属到站验收。*
- **阶段 C 徽章**：你已经能主导一个 AI 全栈产品从调研、PRD、架构与 UI、到全栈测试的完整交付。*闸门：C1 深潜专属到站验收。*
- **阶段 D 徽章**：你已经能拿到任何带 AGENTS.md 的工业项目，指挥 Agent 部署跑通并逐项完成验收。*闸门：D1 深潜专属到站验收。*

---

## 到站验收勾选清单（进度条）

### 阶段 A
- [ ] **A1** GitHub 上有你的第一个提交（凭据：commit 链接）
- [ ] **A2 ★深潜专属** chatgpt-clone 本机跑通；关掉 Agent 代写，自己给它加「清空对话」按钮并跑通，用一句话说清数据从前端→后端→模型→前端的流转（凭据：git diff 显示你手写的改动 + 页面实测清空生效）
  - 子验收①前端能打开页面 · ②后端流式返回一条回复 · ③手加按钮并解释数据流
- [ ] **A3** NL2SQL 助理跑通一次自然语言查库（凭据：查询结果截图 + 生成的 SQL）
  - 子验收①一条 Cursor Rule 生效 · ②接通一个 MCP Server · ③中文问题→SQL 与图表

### 阶段 B
- [ ] **B1 ★深潜专属** 关掉 agent-skills-authoring-pack，手写一个 SKILL.md（description+边界+一个引用），并做触发测试：该进的输入触发、负向边界的输入不触发（凭据：SKILL.md 文件 + 触发/不触发各一条记录）
  - 子验收①写出 description 与正/负边界 · ②触发测试进/不进各一例
- [ ] **B2** 用 Spec-Kit 把一个小需求走完 spec/plan/tasks，三件套齐且一致（凭据：三个文件 + 一致性自查勾选清单）
  - 子验收①Spec-Kit 装好并生成 spec.md · ②生成一致的 plan.md 与 tasks.md

### 阶段 C（核心）
- [ ] **C1 ★深潜专属** 产出生产级 PRD + 6 篇调研笔记；关掉 pack 代笔，亲笔主导「问题定义 / 目标用户 / 核心功能范围与非目标」三节，并为每个功能标注调研依据（凭据：你主笔的 PRD 章节文件 + 与课件参照 specs/prd.md 结构 diff）
  - 子验收①6 篇调研笔记 · ②PRD 骨架 · ③亲笔三节核心章节
- [ ] **C2** 架构基线锁定 + Spec-Kit 装好 + 6 大功能规范化 + UI 设计参考导出（凭据：specs 目录树 + 架构选型取舍说明）
  - 子验收①技术栈选型取舍 · ②环境初始化+Spec-Kit · ③6 功能拆 spec/plan/tasks + UI 导出
- [ ] **C3** 后端 pytest 通过 + 前端交互测试 + 一条全链路旅程 + run-feature 收口（凭据：pytest 通过输出 + 全链路测试记录）
  - 子验收①后端 pytest 绿 · ②前端交互测试 · ③全链路 + run-feature 收口

### 阶段 D
- [ ] **D1 ★深潜专属** FF-Coworker 本机跑通；不交给 Agent 全代跑，自己逐条执行 6 条 AGENTS.md 验证标准并打勾（3210 监听、首页、派发任务见 DONE、curl SSE 首帧 meta 末帧 end、未配 Key 🟡 DEMO、错误 agent 返回 404），并手写三层降级机制说明（凭据：6 条逐条打勾 + curl 输出产物文件 + 机制说明文件，对照 docs/PRD.md 第 5 章）
  - 子验收①服务起、页面可交互 · ②逐条 curl 自检 + 手写机制说明
- [ ] **D2** 所选第二个项目（autopilot 3333 / 小慧 8788）逐项自检全勾，**无 D1 提示独立完成**（累积微验收）
  - 子验收①安装启动+健康检查 · ②逐条验证标准打勾（含权限拦截/模式判定业务点）

---

## 复习锚点（对抗遗忘曲线）

- A1 的 Git 提交流程 → 在 **A2** 复用
- A2 的前后端分离/流式 → 在 **C2** 搭前端时复检
- A3 的 Rules/MCP → 在 **C2** 环境配置时复检
- B1 的 Skill 用法 → 在 **C1**、**D1** 复检
- B2 的 spec/plan/tasks → 在 **C2**、**C3** 复检
- C1 的 PRD 判读 → 在 **D1** 读项目 docs/PRD.md 时复检
- D1 的部署自检法 → 在 **D2** 无提示独立复用（累积微验收）

## 毕业后可解锁

- **engineer-transform · 阶段 B**：把「会交付产品」升级为「会造 Agent 工程系统」
- **Stage 5. Agent 部署上线&运维优化**：把 MVP 从本机推进到上线与运维
- **Stage 7 · Part 1 更多敏捷项目**：SRE、AI 产品经理等，继续扩充部署作品集
- **RAG / 多模态方向线**：产品若涉及知识库问答或文档理解，沿 rag-basics-pack→rag-advanced-pack 继续深挖

---

出品：**赋范空间（FuFan Space）** · 《大模型 Agent 开发实战》
