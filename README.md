# symevo

**Describe what you want. Walk away. Come back to working software.**

symevo is an autonomous project builder. Give it a vision and a technical spec, and it builds your project from scratch — then keeps improving it, session after session, commit after commit, day after day. You define the *what* and *why*; the agent figures out the *how*.

[![npm](https://img.shields.io/badge/npm-not%20published%20yet-orange.svg)](https://www.npmjs.com/package/symevo)
[![license](https://img.shields.io/github/license/Jammie0419/SymEvo.svg)](https://github.com/Jammie0419/SymEvo/blob/main/LICENSE)

<!-- After publishing to npm, replace the "not published yet" badge above with:
[![npm version](https://img.shields.io/npm/v/symevo.svg)](https://www.npmjs.com/package/symevo)
-->

> **中文版说明:** [README.zh-CN.md](README.zh-CN.md)

> **Status:** symevo is **not yet published to the npm registry**. `npx symevo` / `npm install -g symevo` will 404 until it is published — install from local source for now (see [Installation](#installation)).

---

## Table of Contents

- [What It Is](#what-it-is)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Commands](#commands)
- [How the Evolution Loop Works](#how-the-evolution-loop-works)
- [Project Layout](#project-layout)
- [The Spec Is the Source of Truth](#the-spec-is-the-source-of-truth)
- [Multi-Agent Support](#multi-agent-support)
- [Stack Detection](#stack-detection)
- [Local vs. Cloud](#local-vs-cloud)
- [CI Mode Checklist](#ci-mode-checklist-cloud-evolution-prerequisites)
- [Community Issues](#community-issues)
- [Safety](#safety)
- [Review Before You Ship](#review-before-you-ship)
- [Upgrading](#upgrading)
- [Ejecting](#ejecting)
- [Requirements](#requirements)
- [Background](#background)
- [Roadmap](#roadmap)
- [Acknowledgments](#acknowledgments)
- [License](#license)

---

## What It Is

Most AI coding tools wait for you to tell them what to do next. symevo doesn't wait. It reads your vision, checks the spec, looks at what's already built, decides what to work on, implements it, verifies the build passes, writes a journal entry about what it learned, and commits. Then it does it again. And again.

Every 4 hours, your project gets a little closer to matching your vision:

```
Day 0  — Reads your spec. Sets up the project scaffold. First test passes.
Day 1  — Implements the core feature. Writes integration tests.
Day 3  — Adds the CLI interface. Fixes a bug from Day 1.
Day 7  — Responds to a GitHub issue. Polishes error messages.
Day 14 — Your project works. You barely touched a keyboard.
```

---

## Installation

### From local source (current — package not yet on npm)

The package is not published, so this repo's checkout **is** the install source. Three ways, depending on whether you want a live link to your code or a fixed snapshot:

| Method | Command (run in the repo root) | What gets installed | Source edits affect it? | Best for |
|--------|-------------------------------|---------------------|-------------------------|----------|
| **npm link** | `npm link` | Junction/symlink pointing at the repo | **Yes** — after `npm run build` the new version is live immediately | Active development, iterating on the source |
| **Global folder install** | `npm install -g .` | A junction pointing at the repo (**not** a copy) | **Yes** — behaves the same as `npm link` | Not recommended — it is a live link, not a snapshot |
| **Tarball snapshot** | `npm pack` → `npm install -g symevo-x.y.z.tgz` | A true copy, fully independent of the source | **No** — modify the source freely; the installed version stays fixed | Stable per-version installs, copying to another machine |

> ⚠️ `npm install -g <folder>` does **not** copy the package — npm installs it as a junction pointing at your repo, so it behaves exactly like `npm link`. If you want a snapshot that ignores later source changes, always go through the tarball.

**Switching between development mode and snapshot mode:**

```bash
# Development mode (source edits go live after `npm run build`)
cd symevo
npm link

# Back to a stable snapshot
cd symevo
npm version patch                 # bump 0.2.0 → 0.2.1 (add --no-git-tag-version to skip the git commit/tag)
npm pack                          # creates symevo-0.2.1.tgz
npm install -g ./symevo-0.2.1.tgz # installs a real copy; replaces the previous link/install

# Re-link whenever you want live source again
cd symevo
npm link
```

`npm version <patch|minor|major>` bumps the SemVer version in `package.json` — `patch` is the last digit (`0.2.0 → 0.2.1`), `minor` (`0.2.0 → 0.3.0`), `major` (`0.2.0 → 1.0.0`). Inside a git repo it also commits and tags by default; pass `--no-git-tag-version` to change only the version number.

### After publishing to npm

Once the package is on the registry, the standard paths work everywhere:

```bash
npx symevo init          # one-off run in any project
# or
npm install -g symevo    # global install; use `se` as a shorthand: se init, se start, ...
```

---

## Quick Start

> **Detailed install, agent setup, verification, and troubleshooting:** see [INSTALL.md](INSTALL.md).

### New project

1. **Initialize**

   ```bash
   npx symevo init        # or: symevo init, if installed globally
   ```

   This creates `.evolve/` with templates for your vision and spec.

2. **Define your vision and spec** — three options:

   **Option A: Guided interview** (recommended for first-timers)

   ```bash
   symevo vision
   symevo spec
   ```

   `vision` runs five rounds of Socratic questions — what you're building, who it's for, what problem it solves, and what success looks like. `spec` then interviews you on tech stack, architecture, and a prioritized feature checklist. When an agent is configured (see [Multi-Agent Support](#multi-agent-support)), your configured LLM drafts the `.evolve/vision.md` / `.evolve/spec.md` from your answers and you can accept it or ask for a refinement; with no agent configured it falls back to a built-in template. Re-run either with `--refine` to revisit prior answers.

   **Option B: Write both files directly**

   Edit `.evolve/vision.md` and `.evolve/spec.md` using the templates as a guide. The templates include examples and comments explaining what each section needs.

3. **Start building**

   ```bash
   export ANTHROPIC_API_KEY=sk-...    # or the key for your chosen agent
   symevo start
   ```

   If you use a Claude subscription instead of an API key, initialize with `--auth-mode oauth` and run `claude login` — no `ANTHROPIC_API_KEY` needed.

   The engine runs on a schedule (every 4 hours by default) and starts building your project autonomously.

### Existing project

Already have a codebase and docs? symevo can adopt your project.

1. **Initialize**

   ```bash
   cd your-project
   npx symevo init
   ```

2. **Import your existing documents**

   ```bash
   # Convert an existing spec document into symevo format
   symevo migrate spec ./docs/technical-spec.md

   # AI-powered conversion (deeper analysis, cross-references your codebase)
   symevo migrate spec ./PRD.md --ai

   # Convert an existing overview into vision format
   symevo migrate vision ./docs/overview.md
   ```

   The `migrate` command extracts features, tech stack, and architecture from your existing docs and formats them for symevo. Use `--ai` for smarter conversion that checks which features are already implemented.

   You can also run the guided interview to refine an existing vision:

   ```bash
   symevo vision --refine
   ```

   This loads your current `.evolve/vision.md` and walks you through each section, showing your previous answers so you can update or keep them.

3. **Review and start**

   Check the generated files in `.evolve/`, make any adjustments, then:

   ```bash
   export ANTHROPIC_API_KEY=sk-...
   symevo start
   ```

   The agent picks up where your project left off — it reads the codebase, checks which spec features are already implemented, and starts working on what's missing.

---

## Commands

All commands are available as both `symevo <cmd>` and `se <cmd>`.

| Command | What it does |
|---------|-------------|
| `symevo setup` | Guided wizard: agent → interview → mode → schedule → ready (re-runnable) |
| `symevo init` | Scaffold `.evolve/` with vision and spec templates |
| `symevo vision` | Guided Socratic interview to generate `.evolve/vision.md` |
| `symevo spec` | Guided interview to generate `.evolve/spec.md` |
| `symevo migrate` | Convert an existing spec/vision document into symevo format |
| `symevo start` | Turn on the evolution engine (local cron) |
| `symevo stop` | Pause evolution |
| `symevo run` | Run one cycle manually |
| `symevo status` | Check progress — day count, features done, schedule |
| `symevo eject` | Remove the framework, keep everything the agent built |
| `symevo proof` | PROOF9 quality gates and requirements management ([docs](docs/PROOF9.md)) |

### `setup`

The one-command front door for new projects. On a terminal it chains the whole onboarding flow — agent + auth picker, vision + spec interview, execution mode (local/ci/both), and schedule — then installs everything and tells you you're live. Re-run it any time to reconfigure; it honors your existing config and preserves evolution history.

```bash
symevo setup            # guided: nothing → fully configured, ready to evolve
symevo setup --agent codex --mode ci --every 6   # same flags as init, non-interactive
```

### `init`

```bash
symevo init                          # on a terminal, prompts for agent + auth (Claude/api-key by default)
symevo init --agent codex            # skip the prompt; use Codex CLI
symevo init --auth-mode oauth        # use Claude subscription (claude login) instead of API key
symevo init --with-ci                # also install GitHub Actions for cloud evolution
symevo init --mode both              # choose where evolution runs: local | ci | both (persisted in config)
symevo init --mode ci --every 6      # run every 6 hours (applies to both the CI cron and the local job)
symevo init --force                  # upgrade framework files (preserves journal + learnings)
```

`--mode` is the unified execution-mode selector (also offered as an interactive prompt on a terminal). `local` installs a local cron job, `ci` installs the GitHub Actions workflows, `both` does both; the choice is persisted in `.evolve/config.json` and shown by `symevo status`. `--with-ci` remains as an alias for `--mode ci`. When `local`/`both` is chosen but the agent's API key isn't set yet, the cron job is deferred — set the key and run `symevo start`.

`--every <hours>` (1–24, default 4) sets the evolution cadence and is applied to **both** targets: the installed GitHub Actions `cron:` and the local cron entry. On a terminal it's also offered as a prompt when a schedule is being installed.

### `vision`

```bash
symevo vision           # guided interview to create .evolve/vision.md
symevo vision --refine  # revisit and improve an existing vision.md
```

### `spec`

```bash
symevo spec           # guided interview to create .evolve/spec.md
symevo spec --refine  # revisit and improve an existing spec.md
```

On a terminal, `symevo init` offers to run the `vision` and `spec` interviews for you right after install — decline to edit the files by hand instead.

### `migrate`

```bash
symevo migrate spec ./docs/technical-spec.md     # regex extraction (no API key needed)
symevo migrate spec ./PRD.md --ai                # AI-powered conversion via claude CLI
symevo migrate vision ./docs/overview.md         # convert to vision.md format
symevo migrate spec ./README.md --ai --yes       # skip confirmation prompt
```

### `start`

```bash
symevo start                # every 4 hours (default)
symevo start --every 2      # every 2 hours
symevo start --run-now      # start now, then repeat on schedule
symevo start --model claude-opus-4-6  # use a different model
```

---

## How the Evolution Loop Works

Each cycle is autonomous and self-correcting:

```
  Read vision + spec + journal
         |
         v
  Assess current state ---- "What exists vs. what's specified?"
         |
         v
  Prioritize work ---------- CI fix > bootstrap > next feature > bugs > issues
         |
         v
  Implement + test ---------- Write code, run build, verify
         |                         |
         |                    Build fails?
         |                         |
         |                    Fix it (up to 3 tries)
         |                         |
         |                    Still fails? Revert. Journal the failure.
         |
         v
  Journal entry ------------ Honest log: what worked, what didn't, what's next
         |
         v
  Commit + tag ------------- "Day 5 (09:00): add JWT auth with refresh tokens"
```

The journal is the agent's memory across sessions. It reads its own history to avoid repeating mistakes and to build on what worked.

---

## Project Layout

```
my-project/
├── .evolve/
│   ├── vision.md          ← you write this (or use `symevo vision`)
│   ├── spec.md            ← you write this (or use `symevo migrate`)
│   ├── config.json        ← agent and model settings
│   ├── scripts/           ← orchestration engine (protected)
│   ├── skills/            ← agent behaviors (protected)
│   ├── IDENTITY.md        ← agent constitution (protected)
│   ├── JOURNAL.md         ← the agent's memory
│   ├── LEARNINGS.md       ← cached research
│   └── DAY_COUNT          ← evolution counter
├── src/                   ← the agent builds this
├── tests/                 ← the agent writes these
└── .github/workflows/
    ├── evolve.yml         ← evolution workflow (won't touch yours)
    └── evolve-ci.yml      ← CI workflow (renamed so it won't clobber your ci.yml)
```

---

## The Spec Is the Source of Truth

Your `.evolve/spec.md` drives everything. Features are a prioritized checklist:

```markdown
## Features (Priority Order)
- [x] `api serve` — Start the HTTP server
- [x] `api health` — Health check endpoint
- [~] User authentication with JWT
- [ ] Rate limiting middleware
- [ ] WebSocket support for real-time updates
- [ ] Admin dashboard
```

The agent implements them top to bottom. `[x]` = done. `[~]` = in progress. `[ ]` = next up. The agent updates these checkboxes as it works.

---

## Multi-Agent Support

symevo works with multiple AI coding agents:

| Agent | CLI | Flag |
|-------|-----|------|
| Claude Code | `claude` | `--agent claude` (default) |
| Codex CLI | `codex` | `--agent codex` |
| OpenCode | `opencode` | `--agent opencode` |
| Ollama | `ollama` | `--agent ollama` |

```bash
symevo init --agent codex             # initialize with Codex
symevo run --agent ollama             # one-off run with Ollama
symevo start --agent opencode         # schedule with OpenCode
symevo init --auth-mode oauth         # Claude subscription (no API key required)
```

The `--agent` flag on `init` is stored in `.evolve/config.json`. Subsequent `run` and `start` commands read from config automatically. You can override with `--agent` on any command.

**Claude auth modes:**
- `api-key` (default) — set `ANTHROPIC_API_KEY` in your environment
- `oauth` — run `claude login` once; no API key needed (requires a Claude subscription)

The default model adapts to your agent (e.g., `llama3` for Ollama, `o4-mini` for Codex). Override with `--model`.

---

## Stack Detection

Drop symevo into any project. It figures out how to build and test it:

| Stack | Detected by | Build | Test | Lint |
|-------|------------|-------|------|------|
| TypeScript | `tsconfig.json` | `npm run build` | `npm run test` | `npm run lint` |
| Next.js | `"next"` in package.json | `npm run build` | `npm run test` | `npm run lint` |
| Python | `pyproject.toml` | `uv sync` | `uv run pytest` | `uv run ruff check .` |
| Rust | `Cargo.toml` | `cargo build` | `cargo test` | `cargo clippy` |
| Go | `go.mod` | `go build ./...` | `go test ./...` | `go vet ./...` |
| Deno | `deno.json` | — | `deno test` | `deno lint` |
| Java/Kotlin | `pom.xml` / `build.gradle` | `mvn compile` / `gradle build` | `mvn test` / `gradle test` | — |
| C#/.NET | `*.csproj` / `*.sln` | `dotnet build` | `dotnet test` | — |
| Ruby | `Gemfile` | `bundle install` | `rspec` / `rake` | `rubocop` |
| PHP | `composer.json` | `composer install` | `composer test` / `phpunit` | — |
| C/C++ | `CMakeLists.txt` | `cmake --build build` | `ctest` | — |
| Static | `index.html` | — | — | — |

Package managers (npm, yarn, pnpm, bun), Python tooling (uv, poetry, pip), and the Gradle wrapper (`./gradlew`) are detected automatically.

**Monorepos** are supported automatically. If no stack marker is found at the project root, symevo scans immediate subdirectories. When multiple stacks are found (e.g., `backend/` with Python and `frontend/` with Next.js), each substack is verified independently — build, test, and lint run in their respective directories. The post-session fix loop and CI workflow both handle monorepos.

---

## Local vs. Cloud

Run it however fits your workflow:

**Local** — `symevo start`
- Cron job on your machine
- API key stored securely in `.evolve/.env` (mode 600, gitignored)
- Logs in `.evolve/evolve.log`

**Cloud** — `symevo init --with-ci`
- GitHub Actions installed as `.github/workflows/evolve.yml` and `evolve-ci.yml`
- Runs every 4 hours with 3-attempt retry logic
- The workflow is templated for your `--agent`: it installs the matching CLI, sets `AGENT`/`MODEL`, and wires the right secret. `init` prints the exact `gh secret set …` command for your backend (`ANTHROPIC_API_KEY` for Claude, `OPENAI_API_KEY` for Codex, the provider key for opencode). OAuth is local-only, so CI always uses `api-key` mode regardless of your local `--auth-mode`.
- `ollama` runs models locally and isn't supported on hosted CI runners, so `--with-ci` is skipped for it — use **Local** execution.

Both run the same engine. Mix and match.

---

## CI Mode Checklist (Cloud Evolution Prerequisites)

`--mode ci` (alias `--with-ci`) runs the engine on GitHub's own servers, so it can reach GitHub without anything installed on your machine — no local cron, bash, or API key needed locally. Before it can run, your GitHub repo needs four things:

1. **The repo must be on GitHub.** Push the project (including the `.evolve/` scaffold and `.github/workflows/` created by `init`) to a GitHub repo. The workflow only runs from a GitHub-hosted repository.

2. **Actions must be enabled.** Repo → `Settings` → `Actions` → `General` → make sure "Allow all actions" is on. This is the default for new repos, but it is occasionally disabled on private repos.

3. **The agent's API key must exist as a repository secret.** Repo → `Settings` → `Secrets and variables` → `Actions` → `New repository secret`:
   - Claude Code → name `ANTHROPIC_API_KEY`
   - Codex → name `OPENAI_API_KEY`
   - OpenCode → name the provider key your config expects (`ANTHROPIC_API_KEY` or `OPENAI_API_KEY`)
   - Ollama → not supported on hosted CI runners (use Local execution)
   `init --mode ci` also prints the equivalent `gh secret set <NAME>` command for the CLI route. OAuth auth is local-only, so CI always uses api-key mode.

4. **Push once more after `init --mode ci`** so the workflow file is actually on GitHub. A workflow that was never pushed simply doesn't exist — the `Actions` tab stays empty.

Once all four are done, the first run starts on the next cron tick (default every 4h). To kick it off immediately: open the `Actions` tab, select the `Evolution` workflow, and click **Run workflow**.

> Troubleshooting: `Actions` tab empty → step 4 (workflow never pushed). Run fails at the agent step → step 3 (secret name doesn't match, or the key is invalid).

---

## Community Issues

The agent reads GitHub issues tagged with special labels:

| Label | What it does |
|-------|-------------|
| `agent-input` | Feature requests and bug reports from users — agent prioritizes by vote count |
| `agent-self` | Issues the agent filed for itself — its own backlog for future sessions |
| `agent-help-wanted` | Questions the agent couldn't solve alone — it checks for human replies |

Issue content is treated as untrusted input. The agent analyzes intent but writes its own implementation — it never executes code from issues.

---

## Safety

The agent is powerful but constrained:

- **Protected files** — `IDENTITY.md`, `scripts/`, `workflows/` cannot be modified by the agent
- **Build verification** — every change must pass build + tests or it gets reverted
- **Automatic rollback** — 3 failed fix attempts = full revert to pre-session state
- **Prompt injection defense** — random boundary markers, HTML comment stripping, body truncation on all issue content
- **Honest journaling** — the agent can't hide failures; the journal is append-only

---

## Review Before You Ship

symevo is powered by AI, and AI-generated code requires human oversight before production use. The agent does its best — it writes tests, verifies builds, and journals its decisions — but it can introduce bugs, security vulnerabilities, or architectural choices that don't fit your context.

**Before deploying or publishing anything the agent built:**
- Review the code changes (`git log`, `git diff`)
- Run your own security review, especially for auth, input handling, and data access
- Test edge cases the agent may not have considered
- Check dependency choices — the agent may pull in packages you haven't vetted
- Read the journal (`.evolve/JOURNAL.md`) to understand *why* decisions were made

The evolution engine is a powerful accelerator, not a replacement for engineering judgment. Treat its output the way you'd treat a pull request from a junior developer: assume good intent, verify thoroughly.

---

## Upgrading

```bash
npm update -g symevo
symevo init --force     # updates engine, preserves your evolution history
# or: se init --force
# Note: --force migrates root-level vision.md/spec.md into .evolve/ automatically
```

> For a local tarball snapshot, `npm update -g` doesn't apply — re-pack (`npm version patch` → `npm pack`) and `npm install -g ./symevo-x.y.z.tgz` instead. For an `npm link` install, rebuild (`npm run build`) and the change is live.

---

## Ejecting

```bash
symevo eject    # or: se eject
```

Stops the engine, removes `.evolve/` and workflows. Your `vision.md` and `spec.md` are copied to the project root. Everything the agent built — your code, tests, docs — stays exactly where it is.

---

## Requirements

- Node.js >= 18
- Python 3
- Git
- An AI coding agent: [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://github.com/openai/codex), [OpenCode](https://github.com/opencode-ai/opencode), or [Ollama](https://ollama.ai)
- API key for your chosen agent (not needed for Ollama)

**Windows notes:**
- symevo's dependency check looks for a `python3` command. If only `python` exists on your machine, create a `python3.exe` copy next to `python.exe` (or install Python with the launcher) so the check passes.
- PATH resolution order on Windows: **system PATH entries win over user PATH entries**, and only freshly opened terminals pick up PATH changes. To make a specific Python version global, put its directory at the *front of the system PATH*.

---

## Background

Why I built this: [I Built a Codebase That Builds Itself](https://lab.frankbria.com/posts/i-built-a-codebase-that-builds-itself/).

## Roadmap

- **Skill/plugin format** — install as a Claude Code skill, Codex plugin, etc.
- **GitHub Action** — `uses: Jammie0419/SymEvo@v1` for zero-install cloud evolution
- **AI video demos** — auto-generate video walkthroughs of each evolution session ([#8](https://github.com/Jammie0419/SymEvo/issues/8))

## Acknowledgments

Built on the architecture pioneered by [yoyo-evolve](https://github.com/yologdev/yoyo-evolve) by [yologdev](https://github.com/yologdev). The core concepts — autonomous evolution loops, journal-driven memory, spec-driven feature prioritization, and build verification with automatic rollback — originate from that project. symevo packages these ideas into a drop-in CLI tool for any project.

## License

[MIT](LICENSE)
