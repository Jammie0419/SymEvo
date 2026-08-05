# AGENTS.md

Canonical, agent-agnostic guide for any AI coding agent (Claude Code, Codex,
OpenCode, Cursor, etc.) working **on this repository**. `CLAUDE.md` imports this
file — keep project guidance here, not duplicated there.

> This is guidance for developing the **symevo package itself**. It is not
> the runtime that symevo installs into a user's project (that lives in
> `templates/` and `.evolve/`).

## What This Is

An npm CLI package (`symevo`) that turns any project into a self-evolving
codebase. Users run `npx symevo init` in their project, fill in
`.evolve/vision.md` and `.evolve/spec.md`, and the framework autonomously builds
and improves the project session after session using an AI coding agent
(Claude Code by default; Codex, OpenCode, and Ollama are also supported).

## Repository Structure

```
symevo/                  # npm package source
├── package.json              # npm package config (bin: symevo, se)
├── tsconfig.json             # TypeScript config
├── jest.config.js            # Jest (ts-jest) config
├── src/                      # CLI source (TypeScript)
│   ├── cli.ts                # Entrypoint (commander)
│   ├── commands/             # One file per subcommand (see Commands below)
│   │   ├── init.ts           # symevo init (exports runInit)
│   │   ├── setup.ts          # symevo setup (guided onboarding wizard)
│   │   ├── start.ts          # symevo start (set up recurring cron)
│   │   ├── stop.ts           # symevo stop (remove cron)
│   │   ├── run.ts            # symevo run
│   │   ├── status.ts         # symevo status
│   │   ├── eject.ts          # symevo eject
│   │   ├── migrate.ts        # symevo migrate
│   │   ├── vision.ts         # symevo vision (guided interview)
│   │   ├── spec.ts           # symevo spec (guided interview)
│   │   └── proof.ts          # symevo proof (PROOF9 quality gates)
│   └── utils/
│       ├── paths.ts          # Path resolution
│       ├── checks.ts         # Dependency checking
│       ├── output.ts         # TTY/JSON output
│       ├── agent.ts          # Agent backend invocation
│       ├── config.ts         # Agent/mode selection + config I/O
│       ├── interview.ts      # Guided vision/spec interview engine
│       ├── migrate.ts        # Spec/vision document migration
│       ├── proof.ts          # PROOF9 requirements management
│       └── __tests__/        # Jest unit tests
├── templates/                # Files installed by `init` into a target project
│   ├── scripts/              # evolve.sh, detect_stack.sh, format_issues.py, agents/ (per-backend adapters)
│   ├── skills/               # Agent behavior definitions (5 skill dirs, each a SKILL.md)
│   ├── state/                # IDENTITY.md, JOURNAL.md, LEARNINGS.md, REQUIREMENTS.md, DAY_COUNT, vision.md, spec.md
│   └── workflows/            # GitHub Actions (evolve.yml, ci.yml)
├── docs/                     # INSTALL.md, STATUS.md, PROOF9.md
├── dist/                     # Compiled output (gitignored; built on publish)
└── .github/workflows/        # CI for this package
```

## Commands

The CLI ships 11 subcommands (registered in `src/cli.ts`; also available as `se`):

| Command | Purpose |
|---------|---------|
| `init` | Initialize `.evolve/` in the current project |
| `setup` | Guided onboarding wizard: agent → interview → mode → schedule → ready |
| `start` | Start the evolution engine (sets up a recurring local cron job) |
| `stop` | Stop the evolution engine (removes the local cron job) |
| `run` | Run one evolution cycle |
| `status` | Show current evolution state |
| `eject` | Remove symevo framework, keep project files |
| `migrate` | Convert an existing spec or vision document into symevo format |
| `vision` | Guided interview to generate `.evolve/vision.md` |
| `spec` | Guided interview to generate `.evolve/spec.md` |
| `proof` | PROOF9 quality gates and requirements management |

## Development

```bash
npm install           # Install dependencies
npm run build         # Compile TypeScript to dist/ (prepends shebang, chmods bin)
npm run dev           # Watch mode
npm run lint          # Type-check without emitting (tsc --noEmit)
npm test              # Run the Jest unit suite
```

Installation and publishing are documented in [INSTALL.md](INSTALL.md).

## How It Works (User Perspective)

1. User runs `symevo init` in their project
2. Creates `.evolve/` with scripts, skills, state files
3. Creates `vision.md` and `spec.md` templates in `.evolve/`
4. User fills in vision and spec
5. `symevo run` executes one evolution cycle via `.evolve/scripts/evolve.sh`
6. `evolve.sh` invokes the configured agent CLI to read vision/spec, build features, verify, commit
7. Optionally: `--with-ci` (or `--mode ci`) installs GitHub Actions for auto-evolution

## Key Design Decisions

- **`.evolve/` namespace**: All framework files live in `.evolve/` to avoid polluting the project root
- **`PROJECT_DIR=.`**: The project being built IS the repo root (not a subdirectory)
- **`EVOLVE_DIR` env var**: All paths in `evolve.sh` are relative to this variable
- **Templates shipped with npm package**: `init` copies from `node_modules/symevo/templates/`
- **vision.md and spec.md in .evolve/**: User documents live alongside other state files to avoid filename collisions

## Protected Files (in templates)

These must not be modified by the evolution agent:
- `.evolve/IDENTITY.md` — agent constitution
- `.evolve/scripts/evolve.sh` — orchestrator
- `.evolve/scripts/format_issues.py` — input sanitization
- `.github/workflows/evolve.yml` — auto-evolution workflow
- `.github/workflows/evolve-ci.yml` — CI/CD safety net (installed by `init --with-ci` from `templates/workflows/ci.yml`, renamed so it never clobbers the target repo's own `ci.yml`)
