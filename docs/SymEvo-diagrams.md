# SymEvo 图规格说明书（给生成 AI 的提示词集）

> 用途：为开源发布准备 SymEvo 方案配图。**核心 3 张**（详细描述，可直接粘贴），其余为**备选图**（简要描述，需要时再展开）。主方案见 `docs/SymEvo.md`。
> 提示词只描述"画什么内容、包含哪些节点与流程"，不指定生成工具。通用要求：英文标签 + 关键环节附中文小注；不同系统用不同颜色区分（框架/CLI=蓝、内环=绿、外环=橙、质量门/回滚=红、记忆/数据=紫）；关键路径加粗并标注方向箭头；避免交叉线。

---

# 核心图 1：SymEvo 整体架构图（最重要，README 首屏）

## 这张图回答的问题
SymEvo 是什么？外环（进化用户项目）、内环（进化自己）、传播闭环（改进如何惠及所有用户）三条路径如何组成一个自加速系统。

## 布局与层级（自上而下 4 层 + 2 条环 + 1 条传播闭环）

### 第 1 层 · 用户
- 唯一的"人"节点，放顶部居中。
- 标注：人工负责初始 vision/spec、审阅、npm publish 发布。

### 第 2 层 · 控制面：symevo CLI（蓝）
- 一个大的 CLI 框，框内列出 11 个子命令：init / setup / start / stop / run / status / eject / migrate / vision / spec / proof。
- 框上标注："控制面 —— 安装、配置、调度、质检，不直接写业务代码"。
- 与第 3 层引擎的关系用虚线调用箭头（run / run --self / proof 等）。

### 第 3 层 · 共享引擎层（最宽的核心框，蓝）
框内 6 个子模块，每个子模块一句话说明：
1. **会话编排**：evolve.sh（外环）/ evolve-self.sh（内环）共同 source evolve_common.sh；多阶段 A1 评估 → A2 规划 → B 实现 → B-eval 评估。
2. **验证门**：栈检测（detect_stack.sh）得出 build/test/lint 命令；每步改动后验证，失败让 agent 修，最多 3 轮。
3. **PROOF9 质量门**：REQ 台账、scope 匹配、gate 证据文件、回归阻断提交。
4. **议题管道**：GitHub 三类标签（agent-input / agent-self / agent-help-wanted）、净票排序、边界消毒、自动回复与关闭。
5. **记忆系统**：learnings.jsonl（append-only 真源）→ 每日时间加权合成 → active_learnings.md 注入每个 session prompt。
6. **技能系统**：多份 SKILL.md 行为规范，含 skill_evolve（技能自我进化）。

框下标注："共享层 —— 内环与外环复用同一套引擎"。

### 第 4 层 · Agent 层（底部）
- Agent 适配器节点（4 个）：claude / codex / opencode / ollama → 指向"外部 AI Agent CLI"节点 → 指向"LLM API"节点。
- 标注：run_agent 统一接口，语言无关，可插拔。

## 两条进化环

### 外环（橙色、细实线、顺时针）——"进化任意目标项目"
节点链：symevo CLI → 用户目标项目（`.evolve/` 框架目录：IDENTITY / vision / spec / scripts / skills / state）→ 项目源码与构建门（栈检测）→ git 提交 / PR → 回到 CLI（下个 session）。
环上标注："项目持续进化 —— 现状能力，完全保留"。可加 📦 图标表示用户项目。

### 内环（绿色、粗实线、顺时针）——"框架进化自己"
节点链：CLI `--self` → symevo 自身仓库（src/ + templates/ + docs/ + skills/）→ 质量门（npm build / npm test / npm lint + proof --full）→ 提交成功，或 失败自动回滚 + 捕获 REQ → 回到 CLI（下个 session）。
环上标注："框架自己优化自己 —— SymEvo 新增"。在"质量门"节点旁用红色标注回滚分支。

## 传播闭环（紫色、粗虚线、带方向箭头）
节点链：内环改进 → 写入 templates/ 模板 → npm publish（人工）→ npm registry → 用户 npx symevo init 获取进化后的新引擎 → 用户项目变好 → 项目沉淀的 learnings 反哺引擎（供内环 A1 评估参考）→ 闭环。
环上标注："正反馈 —— 框架强 → 项目好"。

## 视觉要求
- 右下角放图例：蓝=框架/控制面、绿=内环、橙=外环、紫=传播闭环、红=门/回滚。
- 三环用线型区分（实线/粗实线/粗虚线），避免交叉线，关键箭头标注方向。
- 生成后要能一眼看出：**中间一台引擎、左边外环跑用户项目、右边内环跑自己、一条紫色虚线把两边连成闭环**。

---

# 核心图 2：进化会话时序图（方法核心，多阶段 A1/A2/B/B-eval）

## 这张图回答的问题
一次进化 session 从触发到收尾，内部到底怎么跑？为什么拆成多阶段而不是一个大 prompt？

## 参与方（横向生命线，8 个）
定时触发（cron / CI）· 编排器（evolve.sh / evolve-self.sh）· run_agent 调用层 · 外部 AI 编码代理（LLM）· git · PROOF9 质量门 · Journal / 记忆 · audit-log 分支。

## 详细流程（步骤 1–10，含消息内容与产物）

1. **触发**：定时触发 → 编排器："开始 Day N 会话"。
2. **准备**：编排器内部做三件事——栈检测、gh 拉取三类 issues、生成轨迹块（近期 outcome / 任务成功率 / 子系统集中度）。
3. **A1 评估**：编排器 → Agent："Phase A1 评估"（prompt 注入轨迹 + active_learnings + 上次 CI 失败日志 + issues）；Agent 读自身源码（抽样）、自测 build/test、研究竞品 → 产出 `session_plan/assessment.md` → git 提交 assessment。
4. **A2 规划**：编排器 → Agent："Phase A2 规划"（只喂 assessment + issues，不让它读源码）；Agent 产出 `session_plan/tasks.md` 任务文件（唯一交付物）。
5. **B 实现**：循环遍历每个任务——编排器 → Agent："Phase B 实现 Task N"；Agent 小步实现 + 写测试 → git 提交（提交信息带 "Task N"）。
6. **B-eval 评估**：编排器 → Agent："Phase B-eval 审阅"；评估器检查改动质量，有问题则进入修复循环（loop）。
7. **机械验证**：编排器 → PROOF9 门：跑 build / test / lint + `proof run --full`。
8. **结果分支**（alt）：
   - 通过：写 JOURNAL（agent 写，缺失则模板 fallback）→ git tag dayN → push；
   - 失败（3 轮仍失败）：git 回滚到会话起点 → 自动在 REQUIREMENTS.md 追加 REQ（Source: "Day N session revert"）→ journal 记录失败原因。
9. **收尾**：编排器 → audit-log 分支：写 `outcome.json`（通过/回滚/任务数/耗时）并推送。
10. 会话结束，等待下次调度。

## 视觉要求
- 用分支框（alt）和循环框（loop）明确标注第 5、6、8 步。
- 失败/回滚路径用红色，通过路径用绿色；产物文件名用等宽字体样式标注。
- 时间方向自上而下，能看出"一个 session = 一次编排器驱动的多阶段流水线"。

---

# 核心图 3：PROOF9 质量门图（独特资产：数据流 + REQ 状态机）

## 这张图回答的问题
自主改代码如何被约束为"可验证、可追踪、坏了会回归阻断"？REQ 的生命周期是怎样的？

## 上半部分：数据流（横向，7 个节点）

1. **改动文件列表**：`git diff HEAD` + untracked 文件。
2. **scope 匹配**：filterReqsByScope——glob 模式（如 `src/auth/*`）或路由模式（如 `POST /auth/login` → 路径组件比对）。
3. **命中 REQ 集合**：从 REQUIREMENTS.md 台账解析出相关 REQ（含 Source / Severity / Obligations / Status）。
4. **逐 gate 执行**：按 REQ 的 Obligations 跑 gate（BUILD / UNIT / LINT / SEC / CONTRACT / E2E / PERF / DEMO / MANUAL）；自动探测工具，无工具则 SKIP（不判成败）。
5. **证据归档**：每个 gate 结果写入 `.evolve/evidence/REQ-XXXX/dayN-commit-gate-时间戳.txt`（截断 500 行）。
6. **汇总判定**：全部通过 → satisfied（记录 "Day N, commit sha"）；曾 satisfied 现失败 → regressed；waived 且未到期 → 跳过。
7. **结果门**：regressed → 阻断提交（必须修复）；全部通过 → 放行提交。

## 下半部分：REQ 状态机

- `open` →（证据满足）→ `satisfied`
- `satisfied` →（回归）→ `regressed` →（修复并再验证）→ 回到 `satisfied`
- `open` →（临时豁免）→ `waived` →（到期自动）→ 回到 `open`
- 新建（capture / 会话内自动捕获）→ `open`

## 视觉要求
- 数据流用蓝色、证据路径用紫色、状态机里 regressed 用红色强调。
- 上下两部分用一条分隔线或分区框隔开；状态机画成环形流转，能看出"豁免有期限、回归会阻断"。

---

# 备选图（仅简要描述，需要时再展开）

| 图 | 内容与用途 | 何时用 |
|---|---|---|
| **模块组件图** | symevo 仓库结构：src/ CLI 11 命令 + utils ↔ templates/（scripts/skills/state/workflows）；绿色虚线高亮 SymEvo 新增文件（evolve_common.sh、evolve-self.sh、extract_trajectory.py、memory_synth.py、skill_evolve.sh、SELF_IDENTITY.md、ROADMAP.md、memory/、session_plan/、3 个新 workflow） | CONTRIBUTING / 开发指南，帮协作者认路 |
| **会话生命周期状态图** | 现状（单 prompt PHASE 1–7 一次调用）vs 目标（A1→A2→B→B-eval 多阶段）的状态机对比，失败回滚分支标红 | 方案文档 §2/§3 对照，解释"改了什么" |
| **记忆系统图** | 会话反思 → learnings.jsonl（append-only 真源）→ memory_synth.py 时间加权压缩 → active_learnings.md → 注入下个会话 prompt，形成闭环；含 LEARNINGS.md 迁移箭头 | 记忆设计章节 |
| **技能自进化图** | skill_evolve 三重门（计数器≥5 / 24h 冷却 / build-test 通过）→ git worktree 隔离 → agent refine/create/retire SKILL.md → 验证 → 提交或回滚 | 技能设计章节 |
| **部署拓扑图** | 本地 cron（crontab 标记、.env 0600、evolve.log；Windows 不支持→WSL/CI）vs GitHub Actions（3 次重试、按栈装工具链、bot 身份、secrets 按 step 注入）双通道触发 evolve.sh；人工 npm publish → 用户 init | docs/INSTALL.md |
| **仓库目录树** | 文本树：SymEvo 新增文件标 🆕；[self-evolvable] 标内环可改范围（src/、templates/、docs/、skills/），[protected] 标不可改（package.json 版本、dist/、.github/workflows/ 自身） | README 结构章节 |
| **路线图 Gantt** | M0 基线 → M1 内环 MVP → M2 记忆与轨迹 → M3 技能自进化 → M4 收敛与传播，各阶段带验收 milestone | README 底部 / Roadmap 页 |
