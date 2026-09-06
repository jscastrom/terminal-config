# Gentleman.Dots AI Agent Skills

> **Single Source of Truth** - This file is the master for all AI assistants.
> Run `./skills/setup.sh` to sync to Claude, Gemini, Copilot, and Codex formats.

This repository provides AI agent skills for Claude Code, OpenCode, and other AI assistants.
Skills provide on-demand context and patterns for working with this codebase.

## Quick Start

When working on this project, Claude Code automatically loads relevant skills based on context.
For manual loading, read the SKILL.md file directly.

## Available Skills

### Gentleman.Dots Specific (Repository Skills)

| Skill | Description | File |
|-------|-------------|------|
| `gentleman-bubbletea` | Bubbletea TUI patterns, Model-Update-View, screen navigation | [SKILL.md](skills/gentleman-bubbletea/SKILL.md) |
| `gentleman-trainer` | Vim Trainer RPG system, exercises, progression, boss fights | [SKILL.md](skills/gentleman-trainer/SKILL.md) |
| `gentleman-installer` | Installation steps, interactive/non-interactive modes | [SKILL.md](skills/gentleman-installer/SKILL.md) |
| `gentleman-e2e` | Docker-based E2E testing, multi-platform validation | [SKILL.md](skills/gentleman-e2e/SKILL.md) |
| `gentleman-system` | OS detection, command execution, cross-platform support | [SKILL.md](skills/gentleman-system/SKILL.md) |
| `go-testing` | Go testing patterns, table-driven tests, Bubbletea testing | [SKILL.md](skills/go-testing/SKILL.md) |

> **Note:** User-facing AI skills (React 19, TypeScript, SDD workflow, etc.) are now managed by [gentle-ai](https://github.com/Gentleman-Programming/gentle-ai).

## Auto-invoke Skills

When performing these actions, **ALWAYS** invoke the corresponding skill FIRST:

| Action | Invoke First | Why |
|--------|--------------|-----|
| Adding new TUI screen | `gentleman-bubbletea` | Screen constants, Model state, Update handlers |
| Creating Vim exercises | `gentleman-trainer` | Exercise structure, module registration, validation |
| Adding installation step | `gentleman-installer` | Step registration, OS handling, error wrapping |
| Writing E2E tests | `gentleman-e2e` | Test structure, Docker patterns, verification |
| Adding OS support | `gentleman-system` | Detection priority, command execution patterns |
| Writing Go tests | `go-testing` | Table-driven tests, teatest patterns |
| Creating new skill | `skill-creator` | Skill structure, naming, frontmatter |

## How Skills Work

1. **Auto-detection**: Claude Code reads CLAUDE.md which contains skill triggers
2. **Context matching**: When editing Go/TUI code, gentleman-bubbletea loads
3. **Pattern application**: AI follows the exact patterns from the skill
4. **First-time-correct**: No trial and error - skills provide exact conventions

## Skill Structure

```
skills/                              # Repository-specific skills
├── setup.sh                         # Sync script
├── gentleman-bubbletea/SKILL.md     # TUI patterns
├── gentleman-trainer/SKILL.md       # Vim trainer
└── ...
```

> User-installable skills (React 19, TypeScript, SDD, etc.) are now managed by [gentle-ai](https://github.com/Gentleman-Programming/gentle-ai).

## Contributing

### Adding a Repository Skill (for this codebase)
1. Read the `skill-creator` skill first
2. Create skill directory under `skills/`
3. Add SKILL.md following the template
4. Register in this file under "Gentleman.Dots Specific"
5. Run `./skills/setup.sh --all` to regenerate

## Project Overview

**Gentleman.Dots** is two things living in one repo:

1. **Dotfiles** — plain config trees (`GentlemanNvim/`, `GentlemanZsh/`, `GentlemanFish/`,
   `GentlemanNushell/`, `GentlemanTmux/`, `GentlemanZellij/`, `GentlemanGhostty/`,
   `GentlemanKitty/`, `herdr/`, plus root-level `alacritty.toml`, `.wezterm.lua`,
   `starship.toml`).
2. **`installer/`** — a Go + Bubbletea TUI that copies those trees into `~/.config` and
   installs the tools around them, on macOS, Linux (Debian/Ubuntu, Fedora, Arch) and Termux.

Contributors change `installer/`; users receive the `Gentleman*` directories. See
[README.md](README.md).

> The installer does **not** embed the configs. `stepCloneRepo` clones `Gentleman.Dots` from
> GitHub at install time and `stepCleanup` deletes the clone afterwards. A local edit under
> `GentlemanNvim/` is therefore **not** picked up by a local installer run until it lands on
> `main`.

## Commands

Go commands run from `installer/`.

| Task | Command |
|------|---------|
| Build | `go build -o gentleman-dots ./cmd/gentleman-installer` |
| All tests | `go test ./...` |
| Tests the way Linux CI runs them | `go test ./... -skip Golden` |
| One package | `go test ./internal/tui/trainer/ -v` |
| One test | `go test ./internal/tui/ -run TestWelcomeScreenGolden -v` |
| Regenerate golden snapshots | `go test ./internal/tui/ -run Golden -update` |
| Format | `gofmt -w .` |

Golden files in `internal/tui/testdata/` are **rendered on macOS**. CI runs `-skip Golden` on
Linux and the full suite only on `macos-latest`, so regenerate them on a Mac or the snapshots
drift.

Running the installer while developing, without touching your real setup:

```bash
./gentleman-dots --test      # redirects HOME to a temp dir
./gentleman-dots --dry-run   # prints what would run, executes nothing
./gentleman-dots --non-interactive --shell=fish --wm=tmux --nvim --terminal=ghostty
```

Env vars the code reads: `GENTLEMAN_TEST_MODE`, `GENTLEMAN_DRY_RUN`, `GENTLEMAN_VERBOSE`
(prints step logs in non-interactive mode).

E2E, from `installer/e2e/`:

```bash
./docker-test.sh              # interactive menu
./docker-test.sh e2e          # every image
./docker-test.sh e2e ubuntu   # one of: ubuntu, debian, fedora, alpine, termux
./docker-test.sh shell alpine # debug shell inside an image
```

`e2e_test.sh` and `e2e_test_termux.sh` must stay POSIX — Alpine runs them under `ash`,
Debian under `dash`.

## Architecture

### One Bubbletea model, with an escape hatch

`internal/tui` is a single `Model` (`model.go`) plus a `Screen` enum covering ~30 screens:
the install flow, the learn/keymaps browsers, backup/restore, and the Vim trainer.
`update.go` dispatches keys per screen and `view.go` renders per screen — a new screen means
touching those three files plus `GetCurrentOptions()`.

Installation itself is a **list of `InstallStep` built from `UserChoices`**, and there are
two builders that must agree:

- `Model.SetupInstallSteps()` (`model.go`) — TUI path; also decides `Interactive` per step.
- `buildStepsForChoices()` (`non_interactive.go`) — `--non-interactive` path.

Both feed one executor: `executeStep(stepID, *Model)` in `installer.go`, a switch from step
ID to a `stepXxx` function. **A new step has to be registered in both builders and in that
switch**, otherwise it silently exists in only one mode.

`Interactive: true` is the escape hatch. Those steps (`homebrew`, `deps`, `terminal`,
`setshell`) need a real TTY for a sudo password or `chsh`, so rather than running in-process
they emit a shell script (`interactive.go`) that Bubbletea runs through `tea.ExecProcess`,
suspending the TUI and handing over the terminal. Anything that prompts the user must go
this way — `system.Run` has no TTY.

### Progress reporting

Steps execute off the Bubbletea goroutine, so they report through the package-level
`globalProgram` via `SendLog`, which sends a `stepProgressMsg`. In non-interactive mode the
same call prints to stdout when `GENTLEMAN_VERBOSE=1`. That global is what lets
`RunNonInteractive` reuse every step function with a stub `Model`.

### `internal/system` is the only code that touches the OS

`detect.go` produces one `SystemInfo` (OS type, Termux, WSL, presence of brew/xcode/pkg);
detection order matters — Termux is checked before generic Linux. `exec.go` holds everything
else: command execution (`Run`, `RunSudo`, `RunBrew`, `RunPkg` and the `*WithLogs` variants),
file and directory copying, the backup system (`ConfigPaths`, `CreateBackup`, `ListBackups`,
`RestoreBackup`, into `~/.gentleman-backup-<timestamp>`), and the shell patchers
(`PatchZshForWM`, `PatchFishForWM`, `PatchNushellForWM`) that inject the multiplexer
auto-start block while guarding against nested sessions.

### Vim trainer (`internal/tui/trainer`)

Self-contained — no dependency on the TUI package. `types.go` defines `ModuleID` and the
unlock chain `moduleUnlockOrder`: a module unlocks when the *previous* module's boss is
defeated. Exercises live one file per module (`exercises_*.go`) and reach the app through
three dispatchers in `exercises.go`: `GetLessons`, `GetPracticeExercises`, `GetBoss`. A new
module needs a new `exercises_<name>.go`, an entry in each dispatcher, plus `GetAllModules()`
and `moduleUnlockOrder`.

`simulator.go` is the core: it executes the user's Vim motions against the exercise buffer to
compute the resulting cursor position and selection, so answers are graded by **effect**, not
by string comparison — `validation.go` then grades optimality and accepts alternatives.
`gamestate.go` holds the session; `stats.go` persists to
`~/.config/gentleman-trainer/stats.json` (tests override the directory via `statsConfigPath`).

## Conventions

- Step failures are wrapped with `wrapStepError` into a `StepError` carrying step ID, name
  and a user-facing description — the TUI renders that description, so keep it meaningful.
- Table-driven tests; `teatest` for whole-program TUI tests.
- Shell scripts under `installer/e2e/` are POSIX-only and `shellcheck`-clean.
- `CLAUDE.md`, `GEMINI.md`, `CODEX.md` and `.github/copilot-instructions.md` are **generated
  from this file and gitignored**. Edit `AGENTS.md`, then run `./skills/setup.sh --all`.

---

## Spec-Driven Development (SDD) Orchestrator

### Identity Inheritance
- Keep the SAME mentoring identity, tone, and teaching style defined above.
- Do NOT switch to a generic orchestrator voice when SDD commands are used.
- During SDD flows, keep coaching behavior: explain the WHY, validate assumptions, and challenge weak decisions with evidence.
- Apply SDD rules as an overlay, not a personality replacement.

You are the ORCHESTRATOR for Spec-Driven Development. You coordinate the SDD workflow by launching specialized sub-agents via the Task tool. Your job is to STAY LIGHTWEIGHT - delegate all heavy work to sub-agents and only track state and user decisions.

### Operating Mode
- Delegate-only: You NEVER execute phase work inline.
- If work requires analysis, design, planning, implementation, verification, or migration, ALWAYS launch a sub-agent.
- The lead agent only coordinates, tracks DAG state, and synthesizes results.

### Artifact Store Policy
- `artifact_store.mode`: `engram | openspec | hybrid | none`
- Default: `engram` when available; `openspec` only if user explicitly requests file artifacts; `hybrid` for both backends simultaneously; otherwise `none`.
- `hybrid` persists to BOTH Engram and OpenSpec. Provides cross-session recovery + local file artifacts. Consumes more tokens per operation.
- In `none`, do not write project files. Return results inline and recommend enabling `engram` or `openspec`.

### SDD Commands
- `/sdd-init` - Initialize orchestration context
- `/sdd-explore <topic>` - Explore idea and constraints
- `/sdd-new <change-name>` - Start change proposal flow
- `/sdd-continue [change-name]` - Run next dependency-ready phase
- `/sdd-ff [change-name]` - Fast-forward planning artifacts
- `/sdd-apply [change-name]` - Implement tasks in batches
- `/sdd-verify [change-name]` - Validate implementation
- `/sdd-archive [change-name]` - Close and persist final state
- `/sdd-new`, `/sdd-continue`, and `/sdd-ff` are meta-commands handled by YOU (the orchestrator). Do NOT invoke them as skills.

### Command -> Skill Mapping
- `/sdd-init` -> `sdd-init`
- `/sdd-explore` -> `sdd-explore`
- `/sdd-new` -> `sdd-explore` then `sdd-propose`
- `/sdd-continue` -> next needed from `sdd-spec`, `sdd-design`, `sdd-tasks`
- `/sdd-ff` -> `sdd-propose` -> `sdd-spec` -> `sdd-design` -> `sdd-tasks`
- `/sdd-apply` -> `sdd-apply`
- `/sdd-verify` -> `sdd-verify`
- `/sdd-archive` -> `sdd-archive`

### Orchestrator Rules
1. NEVER read source code directly - sub-agents do that
2. NEVER write implementation code directly - `sdd-apply` does that
3. NEVER write specs/proposals/design directly - sub-agents do that
4. ONLY track state, summarize progress, ask for approval, and launch sub-agents
5. Between sub-agent calls, show what was done and ask to proceed
6. Keep context minimal - pass file paths, not full file content
7. NEVER run phase work inline as lead; always delegate

### Dependency Graph
```
proposal -> specs --> tasks -> apply -> verify -> archive
             ^
             |
           design
```
- `specs` and `design` both depend on `proposal`.
- `tasks` depends on both `specs` and `design`.

### Sub-Agent Context Protocol

Sub-agents get a fresh context with NO memory. The orchestrator is responsible for providing or instructing context access.

#### Non-SDD Tasks (general delegation)

- **Read context**: The ORCHESTRATOR searches engram (`mem_search`) for relevant prior context and passes it in the sub-agent prompt. The sub-agent does NOT search engram itself.
- **Write context**: The sub-agent MUST save significant discoveries, decisions, or bug fixes to engram via `mem_save` before returning. It has the full detail — if it waits for the orchestrator, nuance is lost.
- **When to include engram write instructions**: Always. Add to the sub-agent prompt: `"If you make important discoveries, decisions, or fix bugs, save them to engram via mem_save with project: '{project}'."`

#### SDD Phases

Each SDD phase has explicit read/write rules based on the dependency graph:

| Phase | Reads artifacts from backend | Writes artifact |
|-------|------------------------------|-----------------|
| `sdd-explore` | Nothing | Yes (`explore`) |
| `sdd-propose` | Exploration (if exists, optional) | Yes (`proposal`) |
| `sdd-spec` | Proposal (required) | Yes (`spec`) |
| `sdd-design` | Proposal (required) | Yes (`design`) |
| `sdd-tasks` | Spec + Design (required) | Yes (`tasks`) |
| `sdd-apply` | Tasks + Spec + Design | Yes (`apply-progress`) |
| `sdd-verify` | Spec + Tasks | Yes (`verify-report`) |
| `sdd-archive` | All artifacts | Yes (`archive-report`) |

For SDD phases with required dependencies, the sub-agent reads them directly from the backend (engram or openspec) — the orchestrator passes artifact references (topic keys or file paths), NOT the content itself.

#### Engram Topic Key Format

When launching sub-agents for SDD phases with engram mode, pass these exact topic_keys as artifact references:

| Artifact | Topic Key |
|----------|-----------|
| Project context | `sdd-init/{project}` |
| Exploration | `sdd/{change-name}/explore` |
| Proposal | `sdd/{change-name}/proposal` |
| Spec | `sdd/{change-name}/spec` |
| Design | `sdd/{change-name}/design` |
| Tasks | `sdd/{change-name}/tasks` |
| Apply progress | `sdd/{change-name}/apply-progress` |
| Verify report | `sdd/{change-name}/verify-report` |
| Archive report | `sdd/{change-name}/archive-report` |
| DAG state | `sdd/{change-name}/state` |

Sub-agents retrieve full content via two steps:
1. `mem_search(query: "{topic_key}", project: "{project}")` → get observation ID
2. `mem_get_observation(id: {id})` → full content (REQUIRED — search results are truncated)

### Sub-Agent Launch Pattern
ALL sub-agent launch prompts (SDD and non-SDD) MUST include this SKILL LOADING section:
```
  SKILL LOADING (do this FIRST):
  Check for available skills:
    1. Try: mem_search(query: "skill-registry", project: "{project}")
    2. Fallback: read .atl/skill-registry.md
  Load and follow any skills relevant to your task.
```

### Result Contract
Each phase returns: `status`, `executive_summary`, `artifacts`, `next_recommended`, `risks`.

### State & Conventions (source of truth)
Use shared convention files installed under skills:
- `_shared/engram-convention.md` for artifact naming + two-step recovery
- `_shared/persistence-contract.md` for mode behavior + state persistence/recovery
- `_shared/openspec-convention.md` for file layout when mode is `openspec`

### Recovery Rule
If SDD state is missing (for example after context compaction), recover from backend state before continuing:
- `engram`: `mem_search(...)` then `mem_get_observation(...)`
- `openspec`: read `openspec/changes/*/state.yaml`
- `none`: explain that state was not persisted

### Multi-Agent Mode

This repository ships with a **single-agent** OpenCode configuration by default. For **multi-agent mode** (dedicated sub-agent per SDD phase with individual model routing), see the [gentle-ai installer](https://github.com/gentleman-programming/gentle-ai) which supports both modes.
