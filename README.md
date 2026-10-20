# DCPA — AI 认知探测评估平台

DCPA（Dynamic Cognitive Probing Assessment）是一个面向教育场景的 AI 认知测评平台。学生与 AI 进行多轮对话式评估，AI 不给标准答案，而是像研究员 / 面试官一样**动态追问**，从对话中捕捉学生的思维质量，并自动生成结构化的认知评估报告。

线上环境：**newdcpa.cssat.cn**

---

## 它能做什么

### 1. 对话式认知测评

学生选择一个场次（Batch）开始评估，AI 基于预设的**探针（Probe）**发起提问，并根据学生的回答实时决定下一步：温和澄清、深挖追问，还是切换话题。全程 SSE 流式输出，打字机效果。

每轮学生回答都会被实时观测，产出五维认知信号：

| 维度 | 含义 |
|------|------|
| 抽象水平 | 从具体现象上升到抽象规律的能力 |
| 反思深度 | 是否审视自己的推理过程（元认知） |
| 不确定性处理 | 直面、命名还是回避未知 |
| 新颖性 | 给出常规答案还是生成式想法 |
| 投入度 | 对话中的倾向性变化 |

评估结束后自动生成认知报告：分维度评分、逐会话快照、趋势 sparkline、质性视角解读。

### 2. 两种评估形态

- **认知访谈（classic）**：研究员风格的一对一深度对话，用于日常认知探测。
- **强基面试自测（interview）**：参考清华强基计划结构化面试设计的模拟面试。AI 面试官给出材料 + 主问题，学生可先阅读思考（可配置阅读时长上限），随后开始作答；面试官层层深挖追问，结束后生成面试评估报告（回答轨迹 / 结构化思维亮点 / 应变与深挖表现 / 提升建议）。

### 3. 场次与主题管理

- **批次制**：一次评估多个学生，统一探针配置、时长与轮次限制。
- **主题制**：一个场次挂多个主题（Topic），学生开测时随机抽取，或由管理员预分配；每个主题拥有独立的探针序列。
- **自助报名**：开放配置后，学生可在个人首页直接加入场次，无需管理员逐一分配。
- **探针配置器**：可视化编辑每个主题的五段式探针序列（主问题 → 概念澄清 → 反例挑战 → 视角切换 → 综合收尾），支持导出 / 复制。

### 4. 学生端体验

- 我的评估：场次卡片展示进行中会话的主题、进度、完成数。
- 答题页：材料阅读 / 作答双倒计时、剩余轮数提示、实时对话流。
- 评估记录与认知报告：中英文双语。

---

## 技术架构

```
┌─────────────────┐      SSE 流式       ┌──────────────────┐
│  前端 React SPA  │ ◄─────────────────► │  后端 FastAPI     │
│  Vite + TS      │   REST /api/*       │  SQLAlchemy 2.0  │
│  TanStack Query │                     │  async PG/JSONB  │
└─────────────────┘                     └────────┬─────────┘
                                                 │
                                    ┌────────────┴────────────┐
                                    │  PostgreSQL             │
                                    │  users / batches /      │
                                    │  topics / probes /      │
                                    │  sessions / events      │
                                    └─────────────────────────┘
```

- **后端** `apps/api/`：Python + FastAPI + SQLAlchemy（async）+ PostgreSQL。所有学生回答、AI 消息、认知信号都以**事件流**（session_events）形式落库，报告聚合用 SQL 完成。
- **前端** `artifacts/dcpa/`：React 18 + Vite + TypeScript + Tailwind + TanStack Query，pnpm 管理。
- **Android 客户端** `apps/android/`：可选的移动端封装。

### 核心目录

```
dcpa/
├── apps/
│   ├── api/dcpa/
│   │   ├── routers/        # sessions（测评主流程）、me（学生端）、admin（管理端）
│   │   ├── agents/         # runtime_agent（对话编排）、_persona（人设/指令模板）、summarization（报告）
│   │   ├── models/         # ORM：batch / topic / probe / session / user
│   │   └── schemas/        # Pydantic 输入输出
│   └── android/            # Android 客户端
├── artifacts/dcpa/
│   └── src/
│       ├── pages/          # dashboard / session（答题页）/ report / admin/*
│       ├── components/     # UI 组件
│       └── lib/            # API 客户端、认证、i18n
├── seed_*.py               # 场次/主题/探针种子脚本（paramiko 远程执行，幂等）
└── deploy_new.py           # 部署编排：git → 服务器 pull → 重启 API → 前端构建
```

### 关键机制

- **persona_mode**：会话级人设开关（`classic` 认知研究员 / `interview` 强基面试官），决定系统提示词、追问策略与报告模板，同一套对话引擎复用。
- **探针序列（linear 模式）**：每主题 5 个节点依序推进，`rounds_per_node` 控制每节点追问轮数；答满字数或触发追问即切换。
- **批次 extra 配置**：`max_turns` / `max_minutes` / `reading_minutes` / `idle_timeout_minutes` / `self_enroll_enabled` / `retake_enabled` / `persona_mode` 全部热配置，无需改代码。
- **计时体系**：作答计时从学生首次发言起算（阅读材料不占作答时长），阅读 / 作答 / 空闲三重上限由后端强制执行，前端倒计时仅作展示。

---

## 快速开始

### 后端

```bash
cd apps/api
pip install -r requirements.txt
# 配置 .env：DATABASE_URL、LLM API key 等
uvicorn dcpa.main:app --reload --port 6881
```

### 前端

```bash
cd artifacts/dcpa
pnpm install
pnpm dev
```

### 生产部署

标准流程（deploy_new.py 已编排）：

1. 本地 `git push`
2. 服务器 `git pull`
3. `systemctl restart dcpa-api`
4. 前端目录 `pnpm build`（产物由 nginx 托管）

> 注意：仅 git pull 不会更新线上页面，前端修改后必须重新构建。

### 数据初始化

参考仓库内种子脚本（幂等，可重复执行）：

```bash
python seed_qiangji_interview_selftest.py   # 强基面试自测演示场次（7 主题 + 演示账号）
```

---

## 适用场景

- 强基计划 / 综合评价等高校选拔的**模拟面试自测**
- 研究场景下的**认知能力探测**与纵向追踪
- 教学场景中的**思维质量反馈**工具

## License

仅限项目授权方内部使用。
