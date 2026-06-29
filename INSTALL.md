# Installing code-evolve

This guide covers installing the `code-evolve` CLI, wiring up an AI agent, and
verifying it works. For what the tool does and how the evolution loop runs, see
the [README](README.md).

---

## Prerequisites

| Requirement | Why | Check |
|-------------|-----|-------|
| **Node.js ≥ 18** | Runs the CLI | `node -v` |
| **Git** | The engine commits each cycle | `git --version` |
| **Python 3** | `format_issues.py` sanitizes GitHub issue input | `python3 --version` |
| **An AI coding agent CLI** | Does the actual building (see below) | `claude --version` |
| **An API key** | Auth for your agent (not needed for Ollama, or for Claude in OAuth mode) | — |

### Supported agents

Install **one** of these and make sure its CLI is on your `PATH`:

| Agent | CLI | Install |
|-------|-----|---------|
| Claude Code (default) | `claude` | https://docs.anthropic.com/en/docs/claude-code |
| Codex | `codex` | https://github.com/openai/codex |
| OpenCode | `opencode` | https://github.com/opencode-ai/opencode |
| Ollama (local models) | `ollama` | https://ollama.ai |

---

## Install the CLI

Pick the method that fits how you'll use it.

### Option A — `npx` (no install, recommended to start)

Run it on demand; npm fetches the latest published version each time:

```bash
npx code-evolve init
npx code-evolve setup
```

### Option B — Global install

Installs `code-evolve` and its short alias `ce` on your `PATH`:

```bash
npm install -g code-evolve

code-evolve --version    # or: ce --version
ce init
```

Upgrade later with:

```bash
npm update -g code-evolve
ce init --force          # refresh framework files, preserves journal + learnings
```

### Option C — From source (contributors)

```bash
git clone https://github.com/frankbria/code-evolve.git
cd code-evolve
npm install
npm run build            # compiles TypeScript to dist/
npm link                 # symlinks `code-evolve` and `ce` to this checkout
```

Now `code-evolve` runs your local build. Use `npm run dev` for watch-mode
recompilation while you work. `npm unlink -g code-evolve` removes the symlink.

---

## Set up your agent

### Claude Code

Two auth modes:

- **API key** (default):
  ```bash
  export ANTHROPIC_API_KEY=sk-...
  ```
- **OAuth** (Claude subscription, no API key) — local execution only:
  ```bash
  code-evolve init --auth-mode oauth
  claude login
  ```

### Codex / OpenCode

```bash
export OPENAI_API_KEY=sk-...        # Codex
# OpenCode: set the provider key your OpenCode config expects
code-evolve init --agent codex      # or --agent opencode
```

### Ollama (no API key)

```bash
ollama pull llama3
code-evolve init --agent ollama
```

The chosen `--agent` is saved to `.evolve/config.json`; later `run`/`start`
commands read it automatically.

---

## First run

From inside the project you want to evolve:

```bash
cd your-project

# Guided front door: agent → interview → mode → schedule → ready
code-evolve setup

# …or do it step by step:
code-evolve init           # scaffold .evolve/
code-evolve vision         # interview → .evolve/vision.md
code-evolve spec           # interview → .evolve/spec.md
code-evolve run            # run one cycle manually to verify
code-evolve start          # turn on the scheduler (every 4h by default)
```

Check progress any time:

```bash
code-evolve status
```

---

## Verify the install

```bash
code-evolve --version          # prints the version
code-evolve --help             # lists all 11 commands
code-evolve run                # runs a single evolution cycle end-to-end
```

A successful `run` reads your vision/spec, makes a change, verifies the build,
writes a journal entry (`.evolve/JOURNAL.md`), and commits.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `code-evolve: command not found` after global install | Ensure npm's global bin is on `PATH` (`npm bin -g`); reopen your shell. |
| `npx code-evolve` fetches an old version | `npx code-evolve@latest …` or install globally. |
| Agent CLI not found | Install the agent (see table above) and confirm `claude`/`codex`/`opencode`/`ollama` runs in your shell. |
| Session aborts immediately on a brand-new repo | A repo with **zero commits** currently aborts on start ([#19](https://github.com/frankbria/code-evolve/issues/19)). Make one commit (`git commit --allow-empty -m init`) first. |
| Missing API key | `export ANTHROPIC_API_KEY=…` (or your agent's key), or use Claude `--auth-mode oauth`. |
| Ollama with `--with-ci` | Not supported — Ollama runs models locally and isn't available on hosted CI runners. Use local execution. |

---

## For maintainers — publishing to npm

The package builds on publish (`prepublishOnly` runs `npm run build`), and only
`dist/` and `templates/` are shipped (see the `files` field in `package.json`).

```bash
npm run lint && npm test         # gate
npm pack --dry-run               # inspect exactly what ships
npm version patch                # or minor / major — bumps + tags
npm publish                      # builds via prepublishOnly, then publishes
git push --follow-tags
```

To test the tarball locally before publishing:

```bash
npm pack                         # produces code-evolve-<version>.tgz
npm install -g ./code-evolve-<version>.tgz
```
