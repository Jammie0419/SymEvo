# symevo

**描述你想要什么，走开，回来收获能用的软件。**

symevo 是一个自主项目构建器。给它一份愿景（vision）和一份技术规格（spec），它会从零开始构建你的项目——然后一个会话接一个会话、一次提交接一次提交、一天接一天地持续改进它。你定义*做什么*和*为什么*，agent 负责*怎么做*。

[![npm](https://img.shields.io/badge/npm-not%20published%20yet-orange.svg)](https://www.npmjs.com/package/symevo)
[![license](https://img.shields.io/github/license/Jammie0419/SymEvo.svg)](https://github.com/Jammie0419/SymEvo/blob/main/LICENSE)

<!-- 发布到 npm 后，把上面 "not published yet" 徽章替换为：
[![npm version](https://img.shields.io/npm/v/symevo.svg)](https://www.npmjs.com/package/symevo)
-->

> **English version:** [README.md](README.md)

> **当前状态：** symevo **尚未发布到 npm registry**，`npx symevo` / `npm install -g symevo` 会 404 —— 目前请用本地安装方式（见[安装](#安装)）。本机当前通过 tarball 快照安装（v0.2.0）。

---

## 目录

- [它是什么](#它是什么)
- [当前使用状态](#当前使用状态)
- [安装](#安装)
- [快速上手](#快速上手)
- [命令](#命令)
- [进化循环如何工作](#进化循环如何工作)
- [项目结构](#项目结构)
- [spec 是唯一事实来源](#spec-是唯一事实来源)
- [多 Agent 支持](#多-agent-支持)
- [技术栈自动检测](#技术栈自动检测)
- [本地 vs 云端](#本地-vs-云端)
- [CI 模式清单（云端进化的前置条件）](#ci-模式清单云端进化的前置条件)
- [社区 Issue 处理](#社区-issue-处理)
- [安全机制](#安全机制)
- [发布前人工审查](#发布前人工审查)
- [升级](#升级)
- [卸载（eject）](#卸载eject)
- [环境要求](#环境要求)
- [背景](#背景)
- [路线图](#路线图)
- [致谢](#致谢)
- [许可证](#许可证)

---

## 它是什么

大多数 AI 编程工具都在等你告诉它下一步做什么。symevo 不等。它读取你的 vision，检查 spec，看看已经建成了什么，决定接下来做什么，实现它，验证构建通过，写下本次会话学到了什么的日志，然后提交。接着再来一轮。循环往复。

每 4 小时，你的项目就朝你的愿景靠近一步：

```
第 0 天 — 读取 spec。搭建项目骨架。第一个测试通过。
第 1 天 — 实现核心功能。写集成测试。
第 3 天 — 添加 CLI 界面。修复第 1 天留下的 bug。
第 7 天 — 响应 GitHub issue。打磨错误信息。
第 14 天 — 你的项目能用了。你几乎没碰过键盘。
```

---

## 当前使用状态

> 本节记录本仓库在本机上的实际使用情况，随使用更新。

| 项目 | 状态 |
|---|---|
| 版本 | 0.2.0（未发布 npm，见[安装](#安装)）|
| 本机安装方式 | tarball 快照（`symevo-0.2.0.tgz`），与源码脱钩 |
| 源码与全局命令关系 | 改源码不影响已装的全局 `symevo`；想联动用 `npm link` 切回开发模式 |
| 示例项目 | `wechat-assistant-nanobot`（`E:\lzm\2026\projects\wechat-assistant-nanobot`，已 `symevo init`）|
| 运行环境 | Windows · Node 24 · Python 3.12（全局默认，3.10/3.7 并存）|
| 默认 Agent | Claude Code（`claude` 2.1.218，api-key 模式）|

---

## 安装

### 本地源码安装（当前方式 —— 包还没上 npm）

包未发布，所以这个仓库的检出**就是**安装源。三种方式，区别在于你想要"和源码联动"还是"固定快照"：

| 方式 | 命令（在仓库根目录执行） | 装的是什么 | 改源码会影响吗？ | 适合 |
|---|---|---|---|---|
| **npm link** | `npm link` | 指向仓库的 Junction/符号链接 | **会** — `npm run build` 后新版立即生效 | 开发中、反复改源码 |
| **全局文件夹安装** | `npm install -g .` | 指向仓库的 Junction（**不是**副本）| **会** — 和 `npm link` 行为一样 | 不推荐，它不是快照 |
| **tarball 快照** | `npm pack` → `npm install -g symevo-x.y.z.tgz` | 真副本，与源码完全独立 | **不会** — 随便改源码，已装版本固定 | 稳定版本安装、拷到别的机器 |

> ⚠️ `npm install -g <文件夹>` **并不是拷贝** —— npm 会装成一个指向你仓库的 Junction，行为和 `npm link` 一样。想要不受源码影响的快照，务必走 tarball。

**开发模式 ↔ 快照模式切换：**

```powershell
# 开发模式（改源码后 `npm run build` 立即生效）
cd E:\lzm\2026\research\SymEvo
npm link

# 切回稳定快照
cd E:\lzm\2026\research\SymEvo
npm version patch                 # 0.2.0 → 0.2.1（加 --no-git-tag-version 可跳过 git 提交/打标签）
npm pack                          # 生成 symevo-0.2.1.tgz
npm install -g .\symevo-0.2.1.tgz # 装的是真副本，替换之前的 link/安装

# 想再回到开发模式
cd E:\lzm\2026\research\SymEvo
npm link
```

`npm version <patch|minor|major>` 自动升 `package.json` 里的版本号 — `patch` 是最后一位（`0.2.0 → 0.2.1`）、`minor`（`0.2.0 → 0.3.0`）、`major`（`0.2.0 → 1.0.0`）。在 git 仓库里默认还会自动提交并打标签；加 `--no-git-tag-version` 则只改版本号。

> 打包生成的 `symevo-*.tgz` 已被 `.gitignore` 忽略，不会误提交。

### 发布到 npm 之后

包上了 registry 后，标准方式即可到处使用：

```bash
npx symevo init          # 任意项目里一次性运行
# 或
npm install -g symevo    # 全局安装；`se` 是缩写：se init、se start、...
```

---

## 快速上手

> **详细安装、agent 配置、验证和排错：** 见 [INSTALL.md](INSTALL.md)。

### 新项目

1. **初始化**

   ```bash
   npx symevo init        # 或已全局安装时直接: symevo init
   ```

   这会创建带 vision/spec 模板的 `.evolve/`。

2. **填写 vision 和 spec** —— 三选一：

   **方式 A：引导式访谈**（新手推荐）

   ```bash
   symevo vision
   symevo spec
   ```

   `vision` 会进行五轮苏格拉底式提问 —— 你在做什么、给谁用、解决什么问题、成功长什么样。`spec` 接着访谈技术栈、架构和排好优先级的特性清单。当配置了 agent（见[多 Agent 支持](#多-agent-支持)）时，你的 LLM 会根据回答起草 `.evolve/vision.md` / `.evolve/spec.md`，你可以接受或要求改写；没配 agent 时退回内置模板。加 `--refine` 可重新审视之前的回答。

   **方式 B：直接手写两个文件**

   参照模板编辑 `.evolve/vision.md` 和 `.evolve/spec.md`。模板带示例和注释，说明每个章节需要什么。

3. **开始构建**

   ```bash
   export ANTHROPIC_API_KEY=sk-...    # 或你选的 agent 的 key
   symevo start
   ```

   用的是 Claude 订阅而不是 API key 的话，用 `--auth-mode oauth` 初始化并运行 `claude login` —— 不需要 `ANTHROPIC_API_KEY`。

   引擎按计划（默认每 4 小时）自动开始构建你的项目。

### 已有项目

已经有代码库和文档？symevo 可以接管你的项目。

1. **初始化**

   ```bash
   cd your-project
   npx symevo init
   ```

2. **导入现有文档**

   ```bash
   # 把现有 spec 文档转成 symevo 格式
   symevo migrate spec ./docs/technical-spec.md

   # AI 转换（分析更深，会交叉参考你的代码库）
   symevo migrate spec ./PRD.md --ai

   # 把现有概述转成 vision 格式
   symevo migrate vision ./docs/overview.md
   ```

   `migrate` 从现有文档中提取特性、技术栈和架构，格式化成 symevo 需要的样子。加 `--ai` 做更智能的转换，会检查哪些特性已经实现。

   也可以用引导式访谈来改进现有 vision：

   ```bash
   symevo vision --refine
   ```

   这会加载你当前的 `.evolve/vision.md`，逐节带你看，展示之前的回答供你更新或保留。

3. **检查并启动**

   检查 `.evolve/` 里生成的文件，做必要的调整，然后：

   ```bash
   export ANTHROPIC_API_KEY=sk-...
   symevo start
   ```

   agent 会接着你项目的现状继续 —— 读代码库、检查 spec 里哪些特性已实现、开始做缺的部分。

---

## 命令

所有命令都同时支持 `symevo <cmd>` 和 `se <cmd>`。

| 命令 | 作用 |
|---|---|
| `symevo setup` | 引导向导：agent → 访谈 → 模式 → 计划 → 就绪（可重复运行）|
| `symevo init` | 生成带 vision/spec 模板的 `.evolve/` |
| `symevo vision` | 苏格拉底式引导访谈，生成 `.evolve/vision.md` |
| `symevo spec` | 引导式访谈，生成 `.evolve/spec.md` |
| `symevo migrate` | 把现有 spec/vision 文档转成 symevo 格式 |
| `symevo start` | 打开进化引擎（本地 cron）|
| `symevo stop` | 暂停进化 |
| `symevo run` | 手动跑一个周期 |
| `symevo status` | 查看进度 —— 天数、已完成特性、计划 |
| `symevo eject` | 移除框架，保留 agent 构建的一切 |
| `symevo proof` | PROOF9 质量门禁和需求管理（[文档](docs/PROOF9.md)）|

### `setup`

新项目的一站式入口。在终端上会把整个 onboarding 流程串起来 —— agent + 认证选择、vision + spec 访谈、执行模式（local/ci/both）和计划 —— 然后装好一切并告诉你已就绪。任何时候重跑都会重新配置；它尊重已有配置并保留进化历史。

```bash
symevo setup            # 引导：什么都没有 → 完全配置好，可以进化
symevo setup --agent codex --mode ci --every 6   # 与 init 相同的参数，非交互
```

### `init`

```bash
symevo init                          # 终端下会询问 agent + 认证（默认 Claude/api-key）
symevo init --agent codex            # 跳过提问；用 Codex CLI
symevo init --auth-mode oauth        # 用 Claude 订阅（claude login）代替 API key
symevo init --with-ci                # 同时安装 GitHub Actions 用于云端进化
symevo init --mode both              # 选择进化跑在哪：local | ci | both（持久化到 config）
symevo init --mode ci --every 6      # 每 6 小时跑一次（同时作用于 CI cron 和本地任务）
symevo init --force                  # 升级框架文件（保留 journal 和 learnings）
```

`--mode` 是统一的执行模式选择器（终端下也会以交互提示形式出现）。`local` 安装本地 cron，`ci` 安装 GitHub Actions workflow，`both` 两者都装；选择持久化在 `.evolve/config.json`，`symevo status` 会显示。`--with-ci` 仍是 `--mode ci` 的别名。选了 `local`/`both` 但 agent 的 API key 还没设置时，cron 任务会延后 —— 设置 key 后运行 `symevo start`。

`--every <hours>`（1–24，默认 4）设置进化频率，同时作用于**两个**目标：GitHub Actions 的 `cron:` 和本地 cron 条目。终端下安装计划时也会作为提示出现。

### `vision`

```bash
symevo vision           # 引导访谈创建 .evolve/vision.md
symevo vision --refine  # 重新审视并改进已有的 vision.md
```

### `spec`

```bash
symevo spec           # 引导访谈创建 .evolve/spec.md
symevo spec --refine  # 重新审视并改进已有的 spec.md
```

在终端下，`symevo init` 会询问是否立即为你跑 `vision` 和 `spec` 访谈 —— 拒绝的话就自己手写文件。

### `migrate`

```bash
symevo migrate spec ./docs/technical-spec.md     # 正则提取（不需要 API key）
symevo migrate spec ./PRD.md --ai                # AI 转换（走 claude CLI）
symevo migrate vision ./docs/overview.md         # 转成 vision.md 格式
symevo migrate spec ./README.md --ai --yes       # 跳过确认提示
```

### `start`

```bash
symevo start                # 每 4 小时（默认）
symevo start --every 2      # 每 2 小时
symevo start --run-now      # 立即开始，然后按计划重复
symevo start --model claude-opus-4-6  # 换一个模型
```

---

## 进化循环如何工作

每个周期都是自主且自纠错的：

```
  读取 vision + spec + journal
         |
         v
  评估当前状态 ---- "现在有什么 vs. 规格要求什么？"
         |
         v
  排优先级 ---------- CI 修复 > 搭建骨架 > 下一个特性 > bug > issues
         |
         v
  实现 + 测试 ---------- 写代码，跑构建，验证
         |                         |
         |                    构建失败？
         |                         |
         |                    修复（最多 3 次）
         |                         |
         |                    还失败？回滚。记入 journal。
         |
         v
  写 journal ------------ 诚实的日志：什么有效、什么无效、下一步
         |
         v
  提交 + 打标签 ---------- "Day 5 (09:00): add JWT auth with refresh tokens"
```

journal 是 agent 跨会话的记忆。它读自己的历史来避免重复犯错、在有效经验上继续构建。

---

## 项目结构

```
my-project/
├── .evolve/
│   ├── vision.md          ← 你写的（或用 `symevo vision`）
│   ├── spec.md            ← 你写的（或用 `symevo migrate`）
│   ├── config.json        ← agent 和模型设置
│   ├── scripts/           ← 编排引擎（受保护）
│   ├── skills/            ← agent 行为定义（受保护）
│   ├── IDENTITY.md        ← agent 宪章（受保护）
│   ├── JOURNAL.md         ← agent 的记忆
│   ├── LEARNINGS.md       ← 研究缓存
│   └── DAY_COUNT          ← 进化计数
├── src/                   ← agent 构建的代码
├── tests/                 ← agent 写的测试
└── .github/workflows/
    ├── evolve.yml         ← 进化 workflow（不会动你的）
    └── evolve-ci.yml      ← CI workflow（改名后不会覆盖你自己的 ci.yml）
```

---

## spec 是唯一事实来源

你的 `.evolve/spec.md` 驱动一切。特性是一个排好优先级的清单：

```markdown
## Features (Priority Order)
- [x] `api serve` — Start the HTTP server
- [x] `api health` — Health check endpoint
- [~] User authentication with JWT
- [ ] Rate limiting middleware
- [ ] WebSocket support for real-time updates
- [ ] Admin dashboard
```

agent 从上到下实现它们。`[x]` = 完成。`[~]` = 进行中。`[ ]` = 下一个。agent 干活时会更新这些勾选框。

---

## 多 Agent 支持

symevo 支持多种 AI 编程 agent：

| Agent | CLI | 参数 |
|---|---|---|
| Claude Code | `claude` | `--agent claude`（默认）|
| Codex CLI | `codex` | `--agent codex` |
| OpenCode | `opencode` | `--agent opencode` |
| Ollama | `ollama` | `--agent ollama` |

```bash
symevo init --agent codex             # 用 Codex 初始化
symevo run --agent ollama             # 用 Ollama 跑一次
symevo start --agent opencode         # 用 OpenCode 定时
symevo init --auth-mode oauth         # Claude 订阅（不需要 API key）
```

`init` 上的 `--agent` 会存进 `.evolve/config.json`。之后的 `run` 和 `start` 自动从 config 读取。任何命令都可以用 `--agent` 覆盖。

**Claude 认证方式：**
- `api-key`（默认）— 在环境变量里设置 `ANTHROPIC_API_KEY`
- `oauth` — 运行一次 `claude login`；不需要 API key（需要 Claude 订阅）

默认模型会随 agent 自适应（如 Ollama 用 `llama3`，Codex 用 `o4-mini`）。用 `--model` 覆盖。

---

## 技术栈自动检测

把 symevo 扔进任何项目，它自己会搞清楚怎么构建和测试：

| 技术栈 | 检测依据 | 构建 | 测试 | Lint |
|---|---|---|---|---|
| TypeScript | `tsconfig.json` | `npm run build` | `npm run test` | `npm run lint` |
| Next.js | package.json 里有 `"next"` | `npm run build` | `npm run test` | `npm run lint` |
| Python | `pyproject.toml` | `uv sync` | `uv run pytest` | `uv run ruff check .` |
| Rust | `Cargo.toml` | `cargo build` | `cargo test` | `cargo clippy` |
| Go | `go.mod` | `go build ./...` | `go test ./...` | `go vet ./...` |
| Deno | `deno.json` | — | `deno test` | `deno lint` |
| Java/Kotlin | `pom.xml` / `build.gradle` | `mvn compile` / `gradle build` | `mvn test` / `gradle test` | — |
| C#/.NET | `*.csproj` / `*.sln` | `dotnet build` | `dotnet test` | — |
| Ruby | `Gemfile` | `bundle install` | `rspec` / `rake` | `rubocop` |
| PHP | `composer.json` | `composer install` | `composer test` / `phpunit` | — |
| C/C++ | `CMakeLists.txt` | `cmake --build build` | `ctest` | — |
| 静态 | `index.html` | — | — | — |

包管理器（npm、yarn、pnpm、bun）、Python 工具（uv、poetry、pip）和 Gradle wrapper（`./gradlew`）都能自动检测。

**Monorepo** 自动支持。项目根目录找不到技术栈标记时，symevo 会扫描直接子目录。发现多个技术栈时（如 `backend/` 是 Python、`frontend/` 是 Next.js），每个子栈独立验证 —— 构建、测试、lint 在各自目录里跑。会话后的修复循环和 CI workflow 都支持 monorepo。

---

## 本地 vs 云端

按你的工作流选：

**本地** — `symevo start`
- 机器上的 cron 任务
- API key 安全存在 `.evolve/.env`（权限 600，gitignored）
- 日志在 `.evolve/evolve.log`

**云端** — `symevo init --with-ci`
- 安装 GitHub Actions 为 `.github/workflows/evolve.yml` 和 `evolve-ci.yml`
- 每 4 小时运行一次，带 3 次重试逻辑
- workflow 会按你的 `--agent` 模板化：安装对应 CLI、设置 `AGENT`/`MODEL`、接好对应的 secret。`init` 会打印对应后端的 `gh secret set …` 命令（Claude 用 `ANTHROPIC_API_KEY`，Codex 用 `OPENAI_API_KEY`，opencode 用对应 provider 的 key）。OAuth 只支持本地，所以 CI 永远用 `api-key` 模式，不管你本地 `--auth-mode` 是什么。
- `ollama` 在本地跑模型，不支持托管 CI runner，所以对它 `--with-ci` 会被跳过 —— 用**本地**执行。

两者跑同一个引擎。可以混用。

---

## CI 模式清单（云端进化的前置条件）

`--mode ci`（别名 `--with-ci`）把引擎放在 GitHub 自己的服务器上跑，所以它能直接访问 GitHub，本地什么都不用装 —— 不需要本地 cron、bash 或 API key。跑起来之前，你的 GitHub 仓库需要四样东西：

1. **仓库必须在 GitHub 上。** 把项目（包括 `init` 生成的 `.evolve/` 骨架和 `.github/workflows/`）推到一个 GitHub 仓库。workflow 只在 GitHub 托管的仓库里运行。

2. **Actions 必须启用。** 仓库 → `Settings` → `Actions` → `General` → 确认 "Allow all actions" 打开。新仓库默认开启，但私有仓库偶尔会被关掉。

3. **agent 的 API key 必须是仓库 secret。** 仓库 → `Settings` → `Secrets and variables` → `Actions` → `New repository secret`：
   - Claude Code → 名字 `ANTHROPIC_API_KEY`
   - Codex → 名字 `OPENAI_API_KEY`
   - OpenCode → 你的 config 期望的 provider key（`ANTHROPIC_API_KEY` 或 `OPENAI_API_KEY`）
   - Ollama → 不支持托管 CI runner（用本地执行）
   `init --mode ci` 也会打印对应的 `gh secret set <NAME>` 命令走 CLI 路线。OAuth 认证仅限本地，所以 CI 永远用 api-key 模式。

4. **`init --mode ci` 之后再推一次**，让 workflow 文件真的到 GitHub 上。从来没推上去的 workflow 就等于不存在 —— `Actions` 标签页是空的。

四样齐了之后，第一次运行在下一个 cron 点（默认每 4 小时）开始。想立即触发：打开 `Actions` 标签页，选 `Evolution` workflow，点 **Run workflow**。

> 排错：`Actions` 标签页空 → 第 4 步（workflow 没推上去）。在 agent 步骤失败 → 第 3 步（secret 名字不对，或 key 无效）。

---

## 社区 Issue 处理

agent 会读取带特殊标签的 GitHub issue：

| 标签 | 作用 |
|---|---|
| `agent-input` | 用户的功能请求和 bug 报告 —— agent 按投票数排优先级 |
| `agent-self` | agent 给自己提的 issue —— 未来会话的自己的 backlog |
| `agent-help-wanted` | agent 独自解决不了的问题 —— 它会检查有没有人回复 |

issue 内容视为不可信输入。agent 分析意图但自己写实现 —— 它从不执行 issue 里的代码。

---

## 安全机制

agent 很强大，但受约束：

- **受保护文件** — agent 不能修改 `IDENTITY.md`、`scripts/`、`workflows/`
- **构建验证** — 每个改动必须通过构建 + 测试，否则回滚
- **自动回滚** — 3 次修复失败 = 完全恢复到会话前状态
- **Prompt 注入防御** — 随机边界标记、HTML 注释剥离、所有 issue 内容截断
- **诚实日志** — agent 无法隐藏失败；journal 只追加不覆盖

---

## 发布前人工审查

symevo 由 AI 驱动，AI 生成的代码在用于生产之前需要人工把关。agent 尽力而为 —— 写测试、验证构建、记录决策 —— 但它也可能引入 bug、安全漏洞或不符合你上下文的架构选择。

**部署或发布 agent 构建的任何东西之前：**
- 审查代码改动（`git log`、`git diff`）
- 自己跑安全审查，尤其是认证、输入处理和数据处理
- 测试 agent 可能没想到的边界情况
- 检查依赖选择 —— agent 可能引入你没审查过的包
- 读 journal（`.evolve/JOURNAL.md`）理解*为什么*做这些决策

进化引擎是强大的加速器，不是工程判断的替代品。把它的产出当成初级开发者的 PR 来对待：假设意图良好，但严格验证。

---

## 升级

```bash
npm update -g symevo
symevo init --force     # 更新引擎，保留进化历史
# 或: se init --force
# 注意: --force 会自动把根目录的 vision.md/spec.md 迁移进 .evolve/
```

> 本地 tarball 快照不适用 `npm update -g` —— 需要重新打包（`npm version patch` → `npm pack`）再 `npm install -g ./symevo-x.y.z.tgz`。`npm link` 安装的话，`npm run build` 后改动即生效。

---

## 卸载（eject）

```bash
symevo eject    # 或: se eject
```

停止引擎，移除 `.evolve/` 和 workflows。你的 `vision.md` 和 `spec.md` 会被复制到项目根目录。agent 构建的一切 —— 你的代码、测试、文档 —— 原样保留。

---

## 环境要求

- Node.js >= 18
- Python 3
- Git
- 一个 AI 编程 agent：[Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Codex](https://github.com/openai/codex)、[OpenCode](https://github.com/opencode-ai/opencode) 或 [Ollama](https://ollama.ai)
- 你选的 agent 的 API key（Ollama 不需要）

**Windows 注意事项（本机实测）：**
- symevo 的依赖检查找的是 `python3` 命令。如果机器上只有 `python`，在 `python.exe` 旁边放一个 `python3.exe` 副本（或装带启动器的 Python）即可通过检查
- Windows 的 PATH 解析顺序：**系统 PATH 优先于用户 PATH**，且只有新开的终端才会读取 PATH 变更。想让某个 Python 版本全局生效，把它的目录放到**系统 PATH 最前面**

---

## 背景

为什么做这个：[I Built a Codebase That Builds Itself](https://lab.frankbria.com/posts/i-built-a-codebase-that-builds-itself/)（作者 Frank Bria）。

## 路线图

- **Skill/plugin 格式** — 装成 Claude Code skill、Codex plugin 等
- **GitHub Action** — `uses: Jammie0419/SymEvo@v1` 零安装云端进化
- **AI 视频演示** — 自动生成每次进化会话的视频走查（[#8](https://github.com/Jammie0419/SymEvo/issues/8)）

## 致谢

架构源自 [yoyo-evolve](https://github.com/yologdev/yoyo-evolve)（作者 [yologdev](https://github.com/yologdev)）。核心概念 —— 自主进化循环、journal 驱动记忆、spec 驱动特性排序、带自动回滚的构建验证 —— 都来自那个项目。symevo 把这些想法打包成一个即插即用的 CLI 工具。

## 许可证

[MIT](LICENSE)
