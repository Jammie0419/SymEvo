# SymEvo 方案 —— 让 code-evolve 学会自我进化（同时继续进化项目）

> 状态：方案草案（方法名 SymEvo 暂定）
> 目标仓库：`code-evolve`（本仓库，TypeScript + bash 模板的 npm CLI，当前 v0.2.0）
> 参考来源：`yologdev/yoyo-evolve`（Rust 自进化 agent，MIT，~115k 行源码 + 2118 行 `scripts/evolve.sh`）
> 一句话：**SymEvo = 双循环共生进化** —— 内环让 code-evolve 在自身仓库上像 yoyo 一样自主优化自己；外环保留"进化任意目标项目"的既有能力；两环共享同一套引擎（会话编排、验证门、记忆、技能、PROOF9 质量门），内环对引擎的改进自动通过 `templates/` 传播到所有外环用户。

---

## 1. 现状与目标

| 维度 | yoyo-evolve（参考对象） | code-evolve（现状） | SymEvo（目标） |
|---|---|---|---|
| 语言/技术栈 | Rust agent（yoagent） | TypeScript CLI + bash/python 模板 | **统一 TypeScript/Node（保留）** |
| 进化对象 | **只有自己**（自己的 src/） | **只有目标项目**（用户仓库的 .evolve/） | **自己 + 目标项目**（双环） |
| 会话编排 | 多阶段、多次调 agent（A1→A2→B→B-eval） | 单次 prompt 内嵌 PHASE 1–7 | 多阶段编排（内环先行，外环跟进） |
| 记忆 | learnings.jsonl 存档 + active_learnings.md 每日合成 | LEARNINGS.md 单文件、无合成 | 追加式存档 + 时间加权合成 |
| 技能 | 14 个技能，含 skill-evolve（自我进化技能） | 5 个技能，静态 | 技能自进化（skill_evolve） |
| 轨迹感知 | outcome.json 审计分支 + extract_trajectory.py | 无 | 移植 |
| 失败处置 | 回滚 + journal + REQ/学习反思 | 回滚 + journal + REQ 自动捕获 | 保留并强化（加反思步骤） |
| 议题闭环 | 净票 + 已解决反馈 + LLM 承诺扫描 | 净票 + 三类标签 + 回复 | 补"已解决反馈"闭环 |

**目标（What）**：把 yoyo 的七类自进化机制移植进 code-evolve 框架，使：
1. code-evolve 自身仓库（本仓库）进入自我进化循环 —— agent 自主评估自身源码、规划、实现、过门（`npm run build` / `npm test` / `npm run lint` / `proof run --full`）、失败回滚、记账；
2. 既有"进化任意目标项目"功能完全保留、且因引擎本身被进化而**间接受益**（内环改的是 `templates/` 里的引擎，用户每次 `init` 拿到的是进化后的版本）；
3. 最终形成闭环：**框架进化自己 → 框架变强 → 框架进化的项目变好 → 项目里沉淀的 learnings 反哺框架**。

**非目标（Not What）**：不把 yoyo 的 Rust agent REPL 搬进来；不重写 code-evolve 为 Rust；不做 yoyo 的社交会话（social.sh）/赞助体系/多 provider 客户端。

---

## 2. 当前 code-evolve 进化方法拆解（现状基线，要保留什么）

> 本节拆解 **code-evolve 现有的"进化一个目标项目"方法**，即 `templates/scripts/evolve.sh` 为核心、CLI（`src/`）为控制面的完整机制。它本身源自 yoyo-evolve 早期版本的 evolve.sh（注释写明 "Adapted from yoyo-evolve's evolve.sh"），是 yoyo 方法的**简化快照**——同一条进化谱系，但缺了 yoyo 后来加的多阶段编排、轨迹、记忆合成等机制（缺口清单见 §2.11，与 §3 的 yoyo 拆解对照）。

### 2.1 文档驱动：vision + spec 是唯一事实源
- `.evolve/vision.md` 定义"为什么/是什么"（北极星），`.evolve/spec.md` 定义技术栈、架构与**按优先级排列的 feature 复选框列表**（`- [ ]` 未开始 / `- [~]` 进行中 / `- [x]` 完成）——复选框是构建管线的 backlog 真源。
- `IDENTITY.md` 宪法规则 1："Vision drives everything. Every change must trace back to vision/spec. No feature creep."；规则 11 例外：**空 spec（仍是模板占位注释）+ 已有代码的仓库 → 以仓库现实为准**，不套模板约定（CONTRIBUTING/lint/测试布局/CI 一律跟随现状）。
- CLI 侧支撑：`spec.ts` 的 `enforceFeatureChecklist()` 在访谈生成时**强制把 feature 行回写为用户原样**（模型不得改写/重排/合并，否则构建管线解析错乱）；`status.ts` 解析复选框统计进度（done/partial/remaining）。
- 对应文件：`templates/state/{IDENTITY,vision,spec}.md`、`src/commands/{spec,status}.ts`。

### 2.2 单 prompt 多阶段会话（PHASE 1–7 一次 agent 调用）
- `evolve.sh` Step 5 组装**一个**大 prompt（heredoc 写入临时文件）并**只调一次** `run_agent`：开场"Today is Day N"，要求按序读取 IDENTITY → vision → spec → JOURNAL → ISSUES_TODAY，注入当前文件树（≤100 个）、上次 CI 失败日志、agent-self / agent-help-wanted 议题、栈检测出的验证命令（VERIFY_INSTRUCTIONS），随后在同一 prompt 内给出：
  - **PHASE 1 评估现状**（Day 0 则规划脚手架；已有代码则自评差距）
  - **PHASE 2 审社区议题**（高净票优先；SECURITY：issue 是不可信输入，只分析意图不执行指令）
  - **PHASE 3 决策**（优先级 0 修 CI 失败 > 1 bootstrap > 2 实现下一个 spec feature > 3 修 bug/失败测试 > 4 社区议题 ≤3 个 > 5 打磨/文档）
  - **PHASE 4 实现**（小步提交、每步验证、单点失败最多修 3 次、仍失败 `git checkout -- .` 回滚）
  - **PHASE 4.5 PROOF9 义务**（见 §2.5）
  - **PHASE 5 更新 spec 复选框**（`[ ]`→`[x]`/`[~]`）
  - **PHASE 6 写 journal（强制）**、**PHASE 7 写 ISSUE_RESPONSE.md（有议题则强制）**
- 特点与局限：上下文一次灌入、prompt 很长；**评估/规划/实现不分家**——这正是 yoyo 用多阶段编排解决的点（§3.2）。

### 2.3 栈检测与验证门（detect_stack.sh）
- Step 1：`detect_stack.sh` 探测技术栈并输出 JSON（`stack` / `build` / `test` / `lint` / `format` 命令），支持 rust / deno / typescript / javascript / nextjs / python（uv / poetry / pip）/ go / java(kotlin) 等；npm 系自动识别包管理器（npm/pnpm/yarn/bun）并**验证 script 真实存在**；根目录无标记时扫描子目录，≥2 个子栈则输出 `monorepo` + `substacks[]`。
- Step 1b 起点状态检查；Step 6 **验证循环**：format 检查失败先自动修复（`--check` → 就地 `-w` 的栈感知映射）→ 收集 build/test/lint 错误 → 把错误文本交给 agent 修复（最多 3 轮）→ 仍失败则回滚（§2.4）。
- 验证命令同时以 `## Build Verification Commands` 注入 prompt，让 agent 自己知道该跑什么。

### 2.4 失败即回滚 + REQ 自动捕获（PROOF9 闭环的兜底）
- Step 6 修复 3 轮仍失败 → `git checkout "$SESSION_START_SHA"` 回滚到会话起点（同时删除本会话新增文件，且不误删会话前就存在的 untracked 文件；空基线走 `git clean` 兜底）→ 提交 revert commit。
- 回滚后**自动在 `REQUIREMENTS.md` 追加一条 REQ**：Source = "Day N session revert"、Severity = high、Obligations 按错误文本启发式（含 test/spec/jest/pytest → `[UNIT, BUILD]`，否则 `[BUILD]`）、Scope = `src/**,**/*.ts,**/*.py,**/*.js`，并单独提交。坏改动以"待办 REQ"形式进入下个会话，而不是消失。
- 这就是"自主改代码但坏了不烂尾"的机制。

### 2.5 PROOF9 质量门（REQ 台账 + gate 证据，CLI + 会话内双轨）
- `REQUIREMENTS.md` 是 **Markdown 台账**：`## REQ-XXXX: title` + 字段（Source / Severity / Scope / Obligations / Evidence / Status，可选 Satisfied by / Waiver reason / Waiver expires）。解析与序列化在 `src/utils/proof.ts`。
- **会话内（Phase 4.5）**：scope 交集匹配（glob 模式 + `GET /path` 路由模式 → 路径组件比对）、逐 gate 跑验证、证据写入 `.evolve/evidence/REQ-XXXX/dayN-commit-gate-时间戳.txt`（截断 500 行）、全部通过 → `satisfied`（记录 "Day N, commit sha"）；曾经 satisfied 现失败 → `regressed` 并**阻止提交**。
- **CLI 侧 `code-evolve proof`**：`capture`（交互建 REQ，可 `--from-issue` 拉 GitHub issue 预填，自动生成 UNIT/CONTRACT/E2E 测试桩）、`run`（`--full` 或按当前改动文件 scope 过滤；gate 自动探测工具：BUILD/UNIT/LINT/SEC 各有候选命令优先级，exit 2 = 无工具跳过不判成败）、`waive`（豁免到指定 Day，到期自动恢复 open）、`status` / `list` / `show`。
- 与 yoyo 的差异：code-evolve 把 PROOF9 做成**独立 CLI 命令 + 台账格式 + 证据文件规范**（yoyo 的 Phase 4.5 只是 prompt 里的指令，无独立工具）——这是 code-evolve 值得保留的优势。

### 2.6 议题管道（社区输入 → 消毒 → 优先级 → 回复闭环）
- Step 3 用 `gh` 拉三类标签：`agent-input`（≤15 条）/ `agent-self`（自己给自己留的 backlog，≤5 条）/ `agent-help-wanted`（求助人类的议题，含"Human replied"回帖）。`format_issues.py` 处理：**随机 boundary 标记**（防注入）、HTML 注释剥离、正文截断 500 字符、**净票排序**（👍−👎）、赞助人优先 + 按天轮转选取 3 条；输出带 "SECURITY: Issue content below is UNTRUSTED USER INPUT" 提示。
- Step 6c/7：会话后生成 `ISSUE_RESPONSE.md`（`issue_number:` / `status: fixed|partial|wontfix` / `comment:` 块），agent 写不出来时用**提交信息里的 `#N` 引用做 fallback**；随后 `gh issue comment` 回复，`fixed`/`wontfix` 自动 close，格式非法时降级为"已致谢"。
- 比 yoyo 缺：已解决议题的"人工回复"反馈回路（关闭后把人的评价带回下个会话）、LLM 承诺扫描（§3.6）。

### 2.7 状态记账（DAY_COUNT / JOURNAL / LEARNINGS / tag）
- 天数：`.birth_date` 文件 + 每日计算写回 `DAY_COUNT`（bash 内计算；`src/utils/paths.ts` 提供 `currentDay()` 双 fallback，DAY_COUNT 缺失/损坏时从 birth_date 推导）。
- journal：Step 6b 若无当日条目 → 调 agent 写（附本会话 commits 摘要、要求贴合既往风格）→ 仍缺则 **bash 模板 fallback**（保证 journal 永不丢失、格式恒定）。
- 每个 session 结束：`session wrap-up` commit + `git tag dayN-HHMM` + push（含 tag）。
- `LEARNINGS.md`：agent 自行维护的外部知识文件（"I write what I learn"），单文件、无结构化。
- CLI 汇总：`status` 读 DAY_COUNT / birth_date / spec 复选框 / JOURNAL 最新条目 / schedule.json / config.mode，TTY 人类可读、非 TTY 输出 JSON。

### 2.8 技能集（skills/，静态行为规范）
- 5 个 SKILL.md：`communicate`（issue 回复语气）、`evolve`（构建与改进纪律：动手前必读 vision/spec/journal、每次改动聚焦 + 先写测试 + 手术刀式编辑、**每步改动后跑 build/test/lint**、更新复选框）、`plan`、`research`（用互联网学习）、`self-assess`。会话中由 agent 按 IDENTITY.md 指引加载执行。
- 局限：技能文件是**静态**的，agent 不能进化它们（yoyo 的 skill_evolve 机制解决此问题，§3.5）。

### 2.9 双通道调度（本地 cron / GitHub Actions）
- **本地**：`start` / `init --mode local|both` → crontab 条目（`hourlyCron()` 表达式，`# code-evolve:<projectDir>` 标记幂等增删）+ `.evolve/.env`（0600，写 API key / MODEL / AGENT / PATH，供 cron 无环境时 source）+ `schedule.json`（仅展示用，不参与门控）；原生 Windows 不支持（提示 WSL/CI）。
- **CI**：`init --with-ci` / `--mode ci|both` → `.github/workflows/evolve.yml`（`init` 按所选 agent 模板化：AGENT/MODEL 环境、CLI 安装命令、`# code-evolve:secrets` 标记内的密钥块——密钥只在需要的 step 上注入）+ `evolve-ci.yml` 安全网（改名避免覆盖用户自己的 ci.yml）。工作流含 **3 次重试**（等 15/45 分钟）、按栈装工具链（rust/go/java）、bot 身份提交。
- **agent 适配器**：`scripts/agents/{claude,codex,opencode,ollama}.sh` 统一 `run_agent <prompt_file> <model> <timeout_cmd> <timeout>` 接口 + `agent_detect_error`（claude 的流式 JSON error 标记）+ `check_agent`；`timeout`/`gtimeout` 杀进程树，spawnSync 超时兜底。

### 2.10 运维与安全细节
- `BOUNDARY_NONCE` 内容边界标记（与 format_issues.py 的 sanitize 配套防注入）；prompt/证据临时文件管理；greenfield 仓库（无 HEAD）先播种初始 commit 使回滚逻辑有基线；`.gitignore` 自动追加（ISSUES_TODAY / ISSUE_RESPONSE / .env / evolve.log / schedule.json / evidence/）并在 `eject` 时清理；可选 sponsor 门（`ENABLE_SPONSORS=true` 时按档位跳过时段，`--force` 绕过）；`src/utils/agent.ts` 的 `draftWithAgent`（访谈起草）：临时目录 + 0600 权限 + 330s 超时，失败静默回退静态模板。

### 2.11 小结：现状基线清单（已有能力 vs yoyo 缺口）

| 类别 | code-evolve 已有（保留） | yoyo 有而 code-evolve 缺（§3 逐条拆解，§6 落地） |
|---|---|---|
| 编排 | 单 prompt 多 PHASE | **多阶段 A1/A2/B/B-eval + session_plan 任务文件** |
| 验证 | 栈检测 + 3 轮修复 + 回滚 + REQ 捕获 | 同左（保留）＋ 评估器 agent 审阅 |
| 质量门 | PROOF9 CLI + 台账 + 证据（**优于 yoyo**） | 保留为主，补 self 预设 |
| 议题 | 净票/消毒/三类标签/回复 | 补"已解决反馈"闭环 + 承诺扫描 |
| 记忆 | LEARNINGS.md 单文件 | **JSONL 存档 + 时间加权合成** |
| 轨迹 | 无 | **审计分支 outcome.json + 任务成功率 + 子系统集中度硬门** |
| 技能 | 5 个静态技能 | **skill_evolve（计数器+冷却+worktree）** |
| 运维 | 3 次重试/栈工具链/bot 身份/适配器 | 补 bot token 工作流、model 降级、每阶段预算 |

---

## 3. yoyo-evolve 自进化方法拆解（要移植什么）

以下机制按"从 yoyo 源码中提取的规律 → 在 code-evolve 中的落点"组织。yoyo 源码已抓取核对（`scripts/evolve.sh` 2118 行、`scripts/skill_evolve.sh`、`scripts/extract_trajectory.py`、`scripts/format_issues.py`、`.github/workflows/evolve.yml`、`IDENTITY.md`）。

### 3.1 宪法（IDENTITY.md）：自我即目标
- yoyo 规则（IDENTITY.md）：只能改自己的源码；**每次改动必须过 build+test，坏了就回滚并在 journal 里记录失败**；每 session 必须写 journal；**先写测试再写功能**；社区 issue 比自己的猜测更有价值；可以用互联网学习。
- 落点：code-evolve 已有 IDENTITY.md（面向目标项目），新增 **self 模式专属宪法**（`SELF_IDENTITY.md`，见 §6.3），把"你是这个框架本身，你要让框架变强"写进去。

### 3.2 多阶段编排（A1 评估 → A2 规划 → B 实现 → B-eval 评估器修复）
- yoyo 关键做法：**一个 session 拆成多次独立 agent 调用**，各自有 time budget：
  - **A1 Assessment agent**：读自身源码（抽样）、近 10 条 journal、memory、跑 `cargo build && cargo test`、查近 5 次 CI 结果（失败看日志找规律）、研究竞品、读 agent-self backlog，产出结构化 `session_plan/assessment.md`（Build Status / Recent Changes / Source Architecture / Self-Test Results / Evolution History / Capability Gaps / Bugs / Open Issues / Research Findings），**不写任务文件、不实现**。
  - **A2 Planning agent**：只读 assessment + issues，产出 `session_plan/` 下的 **task 文件**（这是唯一交付物）；实现阶段按任务文件执行。
  - **B Implementation**：按 task 文件逐个实现、测试、小步提交。
  - **B-eval Evaluator agent**：机械检查（build/test）通过后，由一个评估 agent 审阅改动，带修复循环。
- 好处：评估与规划分离 → 规划质量高；每次调用 prompt 短、上下文干净；失败可定位到阶段。
- code-evolve 现状：单 prompt（§2.2）→ 升级为多阶段，**外环同样受益**。

### 3.3 轨迹感知（Trajectory Awareness）
- `extract_trajectory.py`（61KB）合成三块注入 A1/A2 prompt（3KB 上限 ≈750 token）：
  1. 近期 session 结果（审计分支的 outcome.json）；
  2. **任务成功率**（从 git log 解析 "Task N" 提交）；未选择任务的 session 单独列出；
  3. 窗口内回滚数 + **子系统集中度门**（近 N 次自进化 diff 命中同一子系统 ≥50% 时，把"就地再改"的冲动转为进 tracker）—— 这是防止 agent 反复改同一个模块的**算术级硬门**，yoyo 注释明确写了"用判断性措辞的规则会被它想约束的冲动重新谈判，所以用纯算术"。
- 落点：移植为 `extract_trajectory.py`（python3 已是依赖）+ 审计分支（§6.2）。

### 3.4 记忆：追加存档 + 时间加权合成
- 原始记忆：`memory/learnings.jsonl`（append-only，永不压缩，真源）；
- 合成记忆：每日任务把 JSONL 读入，**时间加权压缩**（近期=全文，久远=主题摘要），写入 `memory/active_learnings.md`，加载进每个 prompt。
- code-evolve 现状：`LEARNINGS.md` 单文件（agent 自己维护，§2.7）→ 升级为双文件制（§6.2/6.3）。

### 3.5 技能自进化（skill_evolve.sh）
- 独立 cron 工作流，**三重门**才跑：session 计数器 ≥5（`.skill_evolve_counter`）+ 24h 冷却（`.skill_evolve_last_run`）+ 当前 main 通过 build/test；
- 用 **git worktree** 隔离，避免与主进化 session 冲突；agent 对自己的 SKILL.md 做 refine/create/retire；改动自动提交，build 坏了自动回滚；完成或跳过都重置计数器（冷却控频率、不控结果）；`SKILL_EVOLVE_DRY_RUN=true` 可只出 prompt 不花钱。

### 3.6 议题闭环
- 三类标签 agent-input / agent-self / agent-help-wanted（code-evolve 已有，§2.6）；
- 净票评分 + 轮转选取 + 边界消毒（code-evolve 已有，直接来自 yoyo 移植）；
- **补缺口**：已解决 issue 的反馈回路（关闭后把"Human's comment"带回下个 session）、pending replies 检测、**LLM 判断的承诺扫描**（扫自己 issue/discussion 里答应过但没做的事）。

### 3.7 运维与验证纪律
- workflow：GitHub App bot token（contents+issues 写权限）、**失败 3 次重试（等 15min/45min）**、concurrency 排队不取消、每阶段 timeout 预算、model 降级（FALLBACK_PROVIDER）、RTK 输出压缩；
- 验证：测试先行、build+test 门、回滚+journal、mutation testing（Rust 侧 cargo-mutants；TS 侧对应工具见 §9 风险表）。

---

## 4. 技术栈决策：统一选 TypeScript/Node（code-evolve 为基底）

用户要求"两个源码使用源码不一致，考虑修改融合的便利性，选定一个一致的"。**结论：以 code-evolve 的 TypeScript + bash 模板为唯一基底，yoyo 的方法以"移植方法论"方式融入，不引入任何 Rust 代码。**

| 对比项 | 选 code-evolve（TS/Node） | 选 yoyo（Rust） |
|---|---|---|
| 进化引擎本质 | bash 编排脚本 + 外部 agent CLI（claude/codex/…），语言无关 | 同左（yoyo 的 evolve.sh 也是 bash） |
| 需迁移的 Rust 面 | 无（yoyo 的 Rust 是它的 agent REPL，我们不需要） | 需重写整个 CLI + 模板 → 全部推倒 |
| 现有资产 | CLI 11 子命令 + 全套模板 + 测试 + docs | 只有 agent REPL 对我们是多余的 |
| 融合成本 | 低：只移植"方法"（脚本/prompt 逻辑），接口 `run_agent` 天然一致 | 高：npm 包生态、ts-jest CI、发布管线全丢 |
| 与外部 agent 的适配 | 已有 4 个适配器（claude/codex/opencode/ollama） | 只有自带的 Rust agent |

关键洞察：**两家的进化引擎本来就是同源**（code-evolve 的 evolve.sh 注释写明 "Adapted from yoyo-evolve's evolve.sh"），所以"融合"本质是把 yoyo 后来加的机制（多阶段、轨迹、记忆合成、技能进化）以同样语言移植回共享祖先进化出的分支，不存在跨语言改写问题。

---

## 5. SymEvo 总体架构

```
                    ┌─────────────────────────────────────────────┐
                    │            code-evolve 引擎（共享层）          │
                    │  evolve_common.sh：阶段函数 / run_agent / 门  │
                    │  PROOF9 质量门 · 议题管道 · 记忆 · 技能        │
                    └──────┬──────────────────────────┬──────────┘
                           │                          │
              ┌────────────▼───────────┐  ┌───────────▼────────────┐
              │  内环：自我进化（Sym）    │  │  外环：项目进化（Evo）    │
              │  code-evolve 仓库本身     │  │  任意目标项目            │
              │  evolve-self.sh         │  │  既有 evolve.sh（升级后） │
              │  目标 = src/ templates/  │  │  目标 = 用户项目源码      │
              │  门 = npm build/test +   │  │  门 = 栈检测 build/test  │
              │       lint + proof --full│  │       + proof            │
              │  宪 法 = SELF_IDENTITY   │  │  宪法 = IDENTITY.md      │
              └────────────┬───────────┘  └───────────┬────────────┘
                           │                          │
                    改进写入 templates/（引擎本身）      init 时拿到新引擎
                           └──────────────┬───────────┘
                                 闭环：框架强 → 项目好
```

- **内环（Sym）**：在 code-evolve 仓库的 `.github/workflows/evolve-self.yml` 定时（如每 8h）触发 `evolve-self.sh`，A1 评估自身 → A2 规划 → B 实现 → B-eval，门 = `npm run build` + `npm test` + `npm run lint` + `code-evolve proof run --full`（对自身的 REQUIREMENTS.md），失败回滚并写 journal + 自动捕获 REQ；session 结果推送到审计分支。
- **外环（Evo）**：现有 `code-evolve init/run/start/…` 全保留（§2 拆解的能力即基线）；`evolve.sh` 升级为多阶段（可选，默认先保持兼容，内环验证后再放开）。
- **模式选择**：`.evolve/config.json` 增加 `self: true|false`（或子命令 `run --self`）。内环模式只在 code-evolve 自身仓库启用；外环模式行为不变。

---

## 6. 分模块实现方案（文件级）

### 6.1 CLI 侧（`src/`，TypeScript）

| 文件 | 改动 |
|---|---|
| `src/commands/init.ts` | 新增 `--self` 标志：初始化 self 模式专属文件（SELF_IDENTITY.md、ROADMAP.md、memory/、计数器、evolve-self.yml 工作流模板、审计分支说明）。复用现有目录复制/权限/迁移逻辑。 |
| `src/commands/run.ts` | 新增 `--self` 路由到 `evolve-self.sh`；环境变量 `EVOLVE_TARGET=self|project`、`PHASE_TIMEOUTS`。 |
| `src/commands/status.ts` | 展示内环状态（self session 计数、最近 outcome、轨迹摘要、技能计数器）。 |
| `src/commands/proof.ts` | 增加 `--self` 预设门（build/test/lint 全量，不依赖栈检测，因为目标就是本仓库）；REQ scope 缺省 `src/**,templates/**,*.json`。 |
| `src/commands/skill-evolve.ts`（新） | `code-evolve skill-evolve`：包装 skill_evolve.sh（门检查、dry-run、force）。 |
| `src/commands/memory-synthesize.ts`（新） | `code-evolve memory-synthesize`：触发 learnings.jsonl → active_learnings.md 合成。 |
| `src/utils/config.ts` | `EvolveConfig` 增加 `self?: boolean`、`selfEveryHours?`；CI 模板函数支持 self 工作流。 |
| `src/utils/trajectory.ts`（新） | 轨迹块的 TS 侧工具（读取审计分支 outcome.json，供 status 展示）；主体逻辑在 python 脚本。 |

### 6.2 模板脚本（`templates/scripts/`，bash/python —— 内环改这里 = 进化引擎本身）

| 文件 | 改动 |
|---|---|
| `evolve_common.sh`（新） | 抽取共享函数：`run_agent` 封装（含错误检测/降级）、阶段计时、门函数（build/test/lint 抽象）、journal 追加、回滚工具、REQ 自动捕获。被 evolve.sh 与 evolve-self.sh 共同 source。 |
| `evolve.sh` | **重构为多阶段**（分阶段调用 `run_agent`）：A1 assessment → A2 plan（写 `session_plan/tasks.md`）→ B implement（按任务实现+小步提交）→ B-eval 评估器修复循环 → 验证/回滚 → journal → issue 回复。**默认保持现有单 prompt 行为可回退**（配置开关 `PHASES=multi|single`，先在 self 模式用 multi，稳定后外环切 multi）。 |
| `evolve-self.sh`（新） | 内环编排：source `evolve_common.sh`；目标固定为仓库自身；门 = npm build/test/lint + `proof run --full`；session 结束写 `session_plan/outcome.json` 并 push 到 `audit-log` 分支；调 trajectory 生成。 |
| `extract_trajectory.py`（新） | 移植 yoyo 三块逻辑：outcome 渲染、任务成功率（解析 git log "Task N"）、回滚窗口 + 子系统集中度硬门。输出注入 A1/A2 prompt。 |
| `memory_synth.py`（新） | 时间加权压缩合成：读 `memory/learnings.jsonl` → 写 `memory/active_learnings.md`（近期全文、久远主题摘要）。 |
| `skill_evolve.sh`（新） | 移植三重门（计数器≥5 + 24h 冷却 + build/test 过）+ git worktree 隔离 + 自动提交/回滚 + dry-run。 |
| `format_issues.py` | 保持（已有净票/消毒/轮转，§2.6）；按需补"已解决 issue 反馈"格式。 |

### 6.3 状态文件（`templates/state/`）

| 文件 | 说明 |
|---|---|
| `IDENTITY.md` | 保持（外环宪法，§2.1）；self 模式提示链接到 SELF_IDENTITY.md。 |
| `SELF_IDENTITY.md`（新） | 内环宪法：只能改 src/、templates/、docs/、skills/、workflows/；每次改动必须过 `npm run build` + `npm test` + `npm run lint` + `proof run --full`；坏了回滚并 journal；先写测试再写功能；禁止触碰 package.json 版本号与 dist/（发布管线保留给人）；社区 issue 优先。 |
| `ROADMAP.md`（新） | 内环的"spec"：以 checkbox 列表定义 code-evolve 自身的功能路线（如"多阶段编排""轨迹感知""技能进化"…），status 解析复用现有 spec.md 复选框逻辑（§2.1）。 |
| `memory/learnings.jsonl`（新） | append-only 学习存档（真源，永不压缩）。 |
| `memory/active_learnings.md`（新） | 每日合成，注入每个 session prompt。 |
| `session_plan/`（新，需入库） | A1 输出 assessment.md、A2 输出 tasks、session 末 outcome.json。 |
| `.skill_evolve_counter` / `.skill_evolve_last_run`（新） | 技能进化门状态。 |

### 6.4 工作流（`.github/workflows/` —— 本仓库与安装模板各一份）

| 文件 | 说明 |
|---|---|
| `evolve-self.yml`（新） | 内环主循环：cron（每 8h）+ workflow_dispatch；GitHub App bot token；npm ci + build/test/lint；失败 3 次重试（等 15/45min）；concurrency 排队。模板同时放入 `templates/workflows/` 供 `init --self` 安装。 |
| `skill-evolve.yml`（新） | 独立 cron（错开主循环），跑 skill_evolve.sh。 |
| `memory-synth.yml`（新） | 每日合成任务，跑 memory_synth.py 并提交 active_learnings.md。 |
| `evolve.yml` / `ci.yml` | 外环模板保持（§2.9）；本仓库自身的 CI 增加"proof 全量门"步骤（防止内环改动破坏质量门自身）。 |

### 6.5 技能（`templates/skills/`）

| 技能 | 改动 |
|---|---|
| `self-assess/` | 强化为 A1 评估 agent 的 prompt 依据：结构化评估模板（Build Status / Capability Gaps / Bugs / Research Findings…）。 |
| `evolve/` | 更新为多阶段工作流说明（A1/A2/B/B-eval 各阶段职责）。 |
| `skill-evolve/`（新） | 教 agent 如何评估/新建/退役 SKILL.md（移植 yoyo 的 skill-evolve 技能要点）。 |
| `trajectory/`（新） | 教 agent 读取并运用轨迹块（含子系统集中度门的含义）。 |
| `plan/`、`research/`、`communicate/` | 保持/微调。 |

---

## 7. 内环与外环的隔离与安全

1. **保护文件扩展**（内环 agent 不可改）：`package.json`（版本/依赖）、`dist/`、`package-lock.json`、`.github/workflows/` 自身；外环 agent 仍不可改 `.evolve/IDENTITY.md`、`evolve.sh`、`format_issues.py` 等（现有清单保留，§2.10）。
2. **双保险门**：内环改动先过机械门（npm build/test/lint + proof --full），再过 B-eval 评估器；任一失败 → 回滚到 `SESSION_START_SHA` + journal 记录 + 自动捕获 REQ（复用现有回滚与 REQ 捕获代码，§2.4）。
3. **模式隔离**：`self` 与 `project` 模式由 config 显式区分，`run --self` 绝不落到外环逻辑，反之亦然；防止"目标项目是 code-evolve 副本"时行为混淆。
4. **人保留最后一道闸**：内环自动提交到**独立分支或 PR**（而非直推 main）为可选项（yoyo 直推 main + 回滚）；建议初期直推 main 复刻 yoyo 以完整闭环，同时 `docs/` 每周人工 review 一次。
5. **审计分支**：`audit-log` 分支只追加 outcome.json（每 session 一个），不参与 main 构建，防轨迹数据污染主分支历史。

---

## 8. 里程碑路线图

| 阶段 | 内容 | 验收标准 |
|---|---|---|
| **M0 基线** | 现状 code-evolve 0.2.0（§2 拆解的能力即基线）；本仓库已有 CI（jest + tsc） | `npm test` / `npm run build` 绿 |
| **M1 内环 MVP** | `evolve_common.sh` 抽取 + `evolve-self.sh` + 多阶段 A1/A2/B + `SELF_IDENTITY.md`/`ROADMAP.md` + `proof --self` 预设 + `evolve-self.yml` 上线 | 本仓库出现第一笔"内环改动提交"；坏改动被回滚并有 REQ 记录；`npm test` 全绿 |
| **M2 记忆与轨迹** | `memory/` 双文件制 + `memory_synth.py` + `extract_trajectory.py` + 审计分支 + 子系统集中度门 | 连续 3 个 session 的 A1 评估显著引用轨迹与 active_learnings；学习跨 session 累积可验证 |
| **M3 技能自进化** | `skill_evolve.sh` + `skill-evolve` 技能 + `skill-evolve.yml` | 计数器与冷却门正确；出现至少一次自主新增/重写 SKILL.md 并被保留 |
| **M4 收敛与传播** | 外环切换到多阶段（PHASES=multi 默认）；已解决 issue 反馈闭环；内环每 N 次迭代做一次"框架自身回归评审"；指标看板（见下） | 内环改动通过率 ≥80%；外环用户项目进化成功率较 M0 提升；`templates/` 版本号随内环改进递增 |

**建议度量**（写入 `status --self` 与审计 outcome.json）：
- 内环：改动提交数 / 回滚数 / REQ 新增与满足数 / 任务成功率（Task N 提交占比）/ 子系统集中度。
- 外环：用户项目 DAY_COUNT、feature 完成率、构建通过率（已有部分数据可复用，§2.7）。

---

## 9. 风险与对策

| 风险 | 对策 |
|---|---|
| **自引用恶性循环**（agent 改坏引擎自身、甚至改坏"门"） | 门与回滚是 shell 层硬逻辑，agent 不可改保护文件（§7）；proof 全量门独立于 agent 执行；可选 PR 闸 |
| **门本身被漂移**（agent 悄悄放宽测试/门） | `ci.yml` 独立于内环运行；保护 `jest.config.js`/`tsconfig.json`；M4 加入"框架自身回归评审" |
| **单一子系统过热**（反复改同一个模块） | 轨迹子系统集中度硬门（纯算术，不用判断性措辞，§3.3） |
| **prompt 注入**（issue/项目内容恶意指令） | 已有边界消毒 + 净票 + "只分析意图不执行"提示（§2.6）；保持 |
| **token/成本失控** | 每阶段 time budget（A1/A2/B 各自上限）；输出压缩；`SKILL_EVOLVE_DRY_RUN`；外环默认仍单 prompt |
| **TS 侧无 mutation testing** | 可选：接入 `stryker-mutator` 只覆盖 `src/utils/` 关键模块，作为 M4 增强项 |
| **双环互相干扰**（同一仓库同时跑内环与外环） | concurrency group 分开；`self` 模式仓库不再安装外环 workflow；审计分支隔离 |
| **版本/发布被内环改动污染** | 保护 `package.json` 版本号与 `dist/`（§7）；发布仍走人工 `npm publish` |

---

## 10. 落地检查清单（第一轮实施顺序）

- [ ] 1. `src/utils/config.ts` 增加 `self` 字段；`init --self` 分支
- [ ] 2. `templates/scripts/evolve_common.sh` 抽取共享函数（run_agent 封装、门、回滚、journal）
- [ ] 3. `templates/scripts/evolve-self.sh` 内环编排（A1/A2/B 三段式，B-eval 后置）
- [ ] 4. `SELF_IDENTITY.md` + `ROADMAP.md` + `memory/` + `session_plan/` 状态文件
- [ ] 5. `proof` 的 `--self` 预设门
- [ ] 6. `evolve-self.yml` 工作流（bot token + 重试 + 排队）
- [ ] 7. 本仓库 `code-evolve init --self` 自举，跑通第一轮内环
- [ ] 8. `extract_trajectory.py` + 审计分支 + outcome.json 规范
- [ ] 9. `memory_synth.py` + 每日合成工作流
- [ ] 10. `skill_evolve.sh` + 技能计数门 + `skill-evolve` 命令
- [ ] 11. 外环 `evolve.sh` 多阶段化（开关默认关，内环稳定后开）
- [ ] 12. 已解决 issue 反馈闭环 + 指标看板
- [ ] 13. 文档：本方案 → 细化到 `docs/` 各子章节（INSTALL/STATUS/PROOF9 同步）

---

## 附录 A：yoyo → code-evolve 移植对照表

| yoyo 源码 | 移植目标（code-evolve） | 移植方式 |
|---|---|---|
| `scripts/evolve.sh`（A1/A2/B/B-eval） | `evolve_common.sh` + `evolve-self.sh` + `evolve.sh` | 逻辑重写为共享祖先进化分支（同源 bash） |
| `scripts/extract_trajectory.py` | `templates/scripts/extract_trajectory.py` | 几乎直移（python3 已是依赖） |
| `scripts/skill_evolve.sh` | `templates/scripts/skill_evolve.sh` | 逻辑直移，门命令换 npm |
| `scripts/format_issues.py` | 已存在（同源，§2.6） | 无需动作；按需加闭环格式 |
| `memory/*.jsonl` + 合成 | `memory/learnings.jsonl` + `memory_synth.py` | 新写（TS/bash 侧工具） |
| `.github/workflows/evolve.yml` | `evolve-self.yml`（bot token/重试/排队） | 直移模板，命令换 npm |
| `IDENTITY.md` 自我宪法 | `SELF_IDENTITY.md` | 新写，门换 npm |
| `skill-evolve`/`trajectory` 技能 | `skills/skill-evolve/`、`skills/trajectory/` | 要点移植 |
| Rust agent（yoagent REPL） | **不移植**（我们使用外部 agent CLI 适配器） | — |

---

*SymEvo 本质：把"yoyo 只进化自己"与"code-evolve 只进化项目"两个单环，耦合成一个共享引擎的双环 —— 引擎在进化自己时产生的每一次改进，都通过 `templates/` 自动惠及所有被进化项目，形成可自我加速的正反馈。*
