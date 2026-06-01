# jaaicode — Just Another AI Coder

**jaaicode** is an AI coding assistant that runs from your terminal, TUI, web browser, or daemon client. It combines an autonomous ReAct agent loop with one of the largest built-in toolbelts in the OSS field (~63 tools), per-role model routing, persistent sessions, skills, recipes, multi-day resumable workplans, and enterprise-ready deployment controls.

It is designed for developers who want an AI pair programmer that can plug into any frontier model (Claude, GPT, Gemini) through their native API, an OpenAI-compatible endpoint, or a LiteLLM proxy, and operate across real codebases without being locked to a single model vendor or IDE.

> Current release: **0.42.3**. Wheels are published for Linux, macOS, and Windows.

---

## What jaaicode does

- **Understands and edits real projects**: read/write files, apply diffs, batch edits, AST-aware refactors (rename / extract / inline), symbol search, LSP diagnostics, tests, git operations, notebooks, shell commands, headless browser, and Jupyter execution.
- **Runs as an agent**: bounded ReAct loop (auto-extending, default cap 25) with pause, queue, three independent kill switches (token budget, hard iteration cap, **LoopGuard** with 3 detectors), and review-loop workflows.
- **Works across vendors**: first-party clients for OpenAI, Anthropic, Google/Gemini, Bedrock, Vertex, and Azure OpenAI, plus any OpenAI-compatible API or LiteLLM proxy, Copilot/OpenAI Responses-only models, and fallback chains.
- **Controls cost automatically**: 3 routing profiles (`speed` / `balanced` / `cost`) with **mid-turn escalation** that swaps a heavier model in transparently when a cheaper model loops, plus per-session token and dollar tracking and `max_cost` budgets.
- **Keeps project context useful**: auto-context (TF-IDF), code maps, PageRank-ranked symbol maps (`repomap`), optional Graphify knowledge-graph integration, persistent memory, and compacting summaries that preserve live IDs.
- **Supports power workflows**: planner → coder → reviewer orchestration, 6 specialised sub-agent types, cron prompts, hooks, skills, plugins, recipes, worktrees, session branching, bookmarks, and transcript export.
- **Runs autonomously for hours or days**: shadow-git checkpoints, auto-compaction, background monitors, cron-driven prompts, sub-agents, resumable sessions, and the `--workplan` engine (multi-unit, worktree-isolated, SIGINT-safe, per-unit checkpoint commits) keep large migrations, benchmark runs, and refactor campaigns moving without supervision.

---

## Interfaces

One engine, many front-ends. All driven from the same `AppState` and tool registry:

| Interface | Status | Notes |
|---|---:|---|
| CLI REPL | ✅ | Rich-rendered streaming, Tokyo Night / Latte themes, command queue, history, tab-complete, slash commands, shell escape. |
| Textual TUI | ✅ | Fullscreen terminal app with input footer, status bar, collapsible tool cards, modals, and diff views. Palette-driven theming via `tui/palette.py`. |
| Web UI | ✅ | `--serve` browser chat with live streaming, typed events, resumable event replay, tool cards, approvals, and inline diffs. |
| Daemon | ✅ | Long-running `jaaicode.daemon` over Unix-domain-socket JSON-RPC for shared clients and automation. |
| MCP server | ✅ | `--mcp` exposes jaaicode's own tool surface to other MCP-aware agents. |
| ACP server | ✅ | `--acp` agent-to-agent protocol bridge (rare in OSS). |
| Pipe mode | ✅ | Scriptable `--pipe` / `-p` operation, including Claude Code-compatible `stream-json` output. |

---

## Model and provider support

- **First-party clients**: OpenAI (Chat + Responses), Anthropic, Google/Gemini, Bedrock, Vertex, Azure OpenAI.
- **OpenAI-compatible APIs**: OpenAI, LiteLLM, OpenRouter-style routes, and compatible self-hosted gateways.
- **OpenAI Responses API routing**: Responses-only models such as `gpt-5.5`, `gpt-5.2-codex`, `gpt-5.3-codex`, and `codex-mini` are routed through `/v1/responses` instead of `/v1/chat/completions`.
- **Native + XML tool calling**: native function calling where supported; XML tool calls for simpler endpoints that lack a function-calling schema.
- **Runtime switching**: `/model`, `/models`, `/provider`, `/compare` (run the same prompt through two models side-by-side — unique among OSS agents), and endpoint-aware switching.
- **Fallback clients**: try another configured model/provider when one fails or overloads.
- **Auto model router**: speed/cost/balanced profiles with availability checks, **mid-turn escalation**, optional long-context tier, and **vision-capable gating** (warns and drops images on text-only models instead of returning a 400).
- **Per-role model binding**: pin different models to `planner` / `coder` / `reviewer` / `refiner` / `router` roles.

### Cheap-model strategy

A typical balanced profile uses a cheaper Claude tier for routine turns and only escalates to a heavier model when needed:

```yaml
balanced:
  fast: claude-haiku-4.5         # routine work, fastest + cheapest
  capable: claude-sonnet-4.7     # harder cases, default workhorse
  long_context: claude-opus-4.7  # very large contexts / hardest tasks
  escalation_threshold: 3
```

Cheap-tier Claude models handle the bulk of routine turns; the router escalates to Sonnet or Opus only when LoopGuard signals trouble or when context grows past the cheaper tier's sweet spot.

---

## Agent capabilities

- **Bounded autonomous loop** with XML or native tool-call parsing.
- **Auto-extending iteration cap** so long but productive tasks continue without false warnings; only hard-caps when LoopGuard says stop.
- **LoopGuard** (3 detectors): repeated text, repeat-write storms, no-progress patterns.
- **Esc pause** that stops at safe iteration boundaries without corrupting tool-call/tool-result pairs.
- **Mid-turn user queue**: clarifications typed during a task are folded into the next iteration as one merged user message.
- **Auto-context**: query-aware TF-IDF file injection skips generic prompts and loads relevant code only when useful.
- **Post-write diagnostics**: LSP/linter feedback is deferred per-iteration and fed back into the loop after edits.
- **Review loop**: builder → reviewer → fixer rounds with structured findings and convergence checks.
- **Sub-agents (6 types)**: `general`, `explore`, `planner`, `test-runner`, `reviewer`, `builder` — including background mode.
- **Planner → coder → reviewer orchestration** with role-specific models for larger tasks.
- **Workplan engine (`--workplan`)**: multi-day, multi-unit, resumable; isolates each unit in a worktree, file-locks, SIGINT-safe, with per-unit checkpoint commits.
- **Auto-worktree isolation (`--auto-worktree`)**: detects when a repo is contended by other workers — live session leases (heartbeat files under `.jaaicode/leases/`), sibling git worktrees, an upstream branch that moved, or a dirty tree — and automatically branches edits into a sibling worktree only when needed; otherwise edits in place while still publishing a lease so concurrent sessions can detect each other.
- **Reflection memory** on tool errors so the agent doesn't repeat a failed call verbatim.

---

## Toolbelt highlights (~63 tools)

The newer polymorphic dispatchers keep the model schema compact while preserving legacy compatibility.

### Files, diffs, and editing

- `read_file` with PDF support and page ranges.
- `write_file`, `write_files`, `edit_file` (with `replace_all`), `apply_diff`, `batch_edit`.
- Rich diff previews in CLI/TUI/Web with source-line gutters.
- Undo stack for file edits, plus **shadow-git checkpoints** for whole-session rewind.
- Secret detection (17 patterns) and destructive-operation confirmations before writes/commits.
- `_safe_path` default-deny (since 1.1.0) and destructive-git block.

### Search and code intelligence

- `search` dispatcher: grep (with context/modes), glob, `search_symbols` (tree-sitter), `find_callers`, `list_dir`.
- `lsp_query` for definition, references, hover/type info, and diagnostics.
- `repomap` for PageRank-ranked project symbol maps.
- `graph` tool over a [Graphify](https://github.com/safishamsi/graphify) knowledge-graph index (`op=query|path|explain|affected`) for structural questions like "where is X?", "what calls Y?", "what breaks if I rename Z?". Opt-in: most models won't reach for it by default, but it can be useful when prompted explicitly or wired into a skill.
- AST/code maps and structural file chunking.
- Respect for `.gitignore` by default; skips for `target/`, `build/`, `dist/`, `node_modules/`, caches, and binary artefacts.

### Analysis and refactoring

- `analyze` dispatcher (9 analyzers, unique in OSS): dependency graph, dead code, complexity, duplication, architecture map, type coverage, migration plan, diff impact, and code quality.
- `refactor` for project-wide rename, extract-function, and inline operations (AST, two-phase commit).
- `test_generate` and `test_failure` for test creation and structured failure analysis.
- Built-in benchmark/refactor harness for multi-file refactor evaluation.

### Shell, git, notebooks, web, and runtime

- `shell` with rlimits / sandbox backends, syntax-highlighted in CLI, plain in TUI/Web tool cards.
- `monitor` for long-running shell jobs (start/check/stop) and `watch` to block on conditions (PID exit, file growth, URL OK, regex match) without LLM round-trips.
- `git` dispatcher for status, diff, commit, log, blame, and pickaxe / history search.
- Notebook edit/run support for `.ipynb` files.
- `web` dispatcher for fetch, DuckDuckGo search (no API key), and headless Playwright browser (text + screenshot).
- `docker_exec` containerised execution across 25 languages.
- `diagnostics` (cross-file linter aggregation) and profiling tools available for advanced workflows.
- Image generation via ComfyUI (unique among CLI agents).

### Tasks, memory, cron, and docs

- `task` dispatcher for create / get / list / update with **`blocks` / `blocked_by` dependency graph** (unique).
- `memory` dispatcher (`remember` / `recall` / `forget`) for persistent project and global facts.
- `propose_doctrine` writes section-aware rules to **`JAAICODE.md`**.
- `cron` dispatcher for scheduled prompts on a cadence.
- `todo_write` for checklist-to-task conversion.

---

## Slash commands and workflows

Common commands include:

| Command | Purpose |
|---|---|
| `/help` / `/help advanced` | Essentials-first help, with full power-user command list on demand. |
| `/model`, `/models`, `/provider`, `/compare` | Browse, switch, and compare configured models/providers side-by-side. |
| `/config`, `/config verbose`, `/config find` | Essentials-first config display and searchable full config. |
| `/stats`, `/stats verbose` | Token/cost/context/session statistics. |
| `/session save`, `/session load` | Save/load sessions; nameless save/load supported. |
| `/compact`, `/clear`, `/retry`, `/regen` | Conversation lifecycle controls. |
| `/add`, `/drop`, `/files`, `/undo`, `/history` | Context-file and edit-history controls. |
| `/plan`, `/review`, `/review-loop`, `/test`, `/autotest` | Plan-mode, code-review, and test workflows. |
| `/skill`, `/style`, `/hooks`, `/quality`, `/audit` | Skills, response styles, lifecycle hooks, outcome labelling, and audit logs. |
| `/worktree`, `/branch`, `/bookmark`, `/goto`, `/switch` | Git/worktree/session navigation and conversation branching (`--worktree` always isolates; `--auto-worktree` isolates only when the repo is contended). |
| `/workplan` | Multi-day, multi-unit, resumable execution. |
| `/checkpoints gc` | Clean old checkpoint shadow repos safely. |
| `/doctor`, `/doctor slow` | Environment and slow-folder diagnostics. |
| `/mcp`, `/plugins`, `/recipes` | Inspect and manage MCP servers, plugins, and recipe configs. |

---

## Sessions, memory, and context

- Named and nameless session save/load.
- Autosave/autoload for recovering recent work.
- Session snapshots include messages, context files, active mode/style, undo stack, token counters, auto-context selections, and model name.
- **Conversation branching, bookmarks, replay, and HTML/Markdown transcript export** (unique among CLI agents).
- Persistent memory stored outside the repo (`~/.jaaicode/memory.md`).
- Project-specific doctrine stored in `JAAICODE.md`.
- Auto-compact and manual `/compact` preserve live-state identifiers such as PIDs, run IDs, task IDs, paths, and tool-call invariants.

---

## Skills, hooks, plugins, recipes, and customization

- **Skills**: `SKILL.md` directories discovered from project, user, and built-in paths; auto-activation uses strict trigger gates and body-size caps to avoid context bloat. **51 builtin** skills covering languages, cloud, testing, security, architecture, performance, and platform topics.
- **Skill benchmarking**: A/B testing and heuristic quality scoring for skills, plus cross-tool export/import to Claude Code, Codex, Copilot, Cursor, and Gemini formats.
- **Styles**: response style is orthogonal to mode (`concise`, `educational`, `business`, `security-first`, plus user/project styles).
- **Modes / personas**: `chat`, `code`, `plan` plus `architect`, `ask`, `debug`, `review`; one-turn `@mode` invocation is supported.
- **Hooks**: lifecycle hook points including SessionStart, SessionEnd, UserPromptSubmit, PreToolUse, PostToolUse, PreCompact, PostCompact, Stop.
- **Plugins / custom tools**: drop `.py` or `.sh` scripts into `~/.jaaicode/plugins/` or `~/.jaaicode/tools/` to extend the tool surface.
- **Recipes**: YAML agent configs with `on_success` / `on_failure` / `on_loop_detected` action items (`kind: notify | webhook | log | email`).
- **MCP**: stdio + SSE MCP tool and resource integration; jaaicode is also a self-hosting MCP **server** (`--mcp`) and supports the ACP agent-to-agent protocol (`--acp`).

---

## Quality, testing, and benchmarks

- Auto-test detection for Python, Rust, Go, JavaScript/TypeScript, Java, Scala, Make, and more (10+ frameworks).
- Structured test failure parsing for self-correction loops.
- Outcome labelling via `/quality`, with optional LLM judge for ambiguous cases.
- Bench runners for **SWE-bench, polyglot multi-language tasks, multi-agent matrices, refactor, bugfix-cache, canonical, frontier-9way**.
- Uniform benchmark output schema: `benchmark_results/<benchmark>/<run-id>/run.json` conforming to the frozen `jaaicode.bench.run/v1` schema.
- Public benchmark report tooling through `scripts/bench-run.py` and `scripts/bench-report.py`.
- Auto-published badge JSON on tagged release.

---

## Security, permissions, and enterprise posture

- Per-tool permission model with `allow` / `ask` / `deny` behaviour.
- YOLO / autorun modes for trusted local automation, with a clear footer indicator when active.
- Destructive operation confirmations when not bypassed.
- **Secret-pattern checks** (17 regex patterns) before writes/commits.
- **Cost budgets and token guards**: `max_cost`, per-role token budgets, and three independent kill switches (token, iteration, loop).
- Compliance mode to disable external network tools for regulated deployments.
- Hash-chained audit log with redacted tool arguments and `/audit verify`.
- **MCP supply-chain check** before loading third-party MCP servers.
- Docker base image for self-hosted deployments.
- Private enterprise tier lives outside the public repo; public MIT code uses a runtime extension seam (`jaaicode.ext.require_premium()`) so the core stays clean.

---

## Where jaaicode leads

- **Broadest protocol surface in OSS**: CLI + TUI + Web + MCP server + ACP server + daemon, all from one engine.
- **Most built-in tools (~63)** of any OSS agent.
- **No vendor lock-in**: any frontier model via native client, OpenAI-compatible API, or LiteLLM proxy.
- **Auto model routing** with mid-turn escalation and `/compare` side-by-side runs — unique.
- **9-analyzer quality suite** (`dependency_graph`, `dead_code`, `complexity`, `duplication`, `architecture_map`, `type_coverage`, `migration_plan`, `diff_impact`, `code_quality`) — unique.
- **AST refactor primitives** (project-wide rename / extract / inline, two-phase commit) — unique as agent tools.
- **Workplan engine** for multi-day, worktree-isolated, resumable runs — no real peer in OSS.
- **Knowledge-graph (Graphify) integration** as an opt-in shortcut for structural questions.
- **Conversation branching, bookmarks, autosave/autoload, HTML/MD transcript export**.
- **Task dependency graph** (`blocks` / `blocked_by`) — unique.
- **Cross-tool doctrine compatibility**: reads `JAAICODE.md`, `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursorrules`, and `.github/copilot-instructions.md` natively.
- **Skill benchmarking and cross-tool skill export** to Claude / Codex / Copilot / Cursor / Gemini formats.
- **Image generation via ComfyUI** — unique among CLI agents.
- **First-class benchmarks** with a frozen schema and a public dispatcher/report tool.

---

## Known gaps and tradeoffs

- No first-party IDE extension or inline Tab autocomplete yet.
- No cloud-hosted background-agent service in the public core (Claude Routines, Copilot Cloud Agent, Cursor Background Agents all cover this).
- Semantic vector codebase indexing is not the default; jaaicode currently emphasises lightweight TF-IDF and the Graphify knowledge graph.
- Public SWE-Bench Verified score not yet posted (harness exists; number unclaimed).
- Per-model edit-format selection table is scaffolded but not yet populated; today the agent uses one strong default format across models.
- Mobile / voice / canvas-style editor experiences are outside the current core product.
- Some advanced tools require optional system dependencies such as Docker, Playwright, language servers, or provider-specific API access.

---

## Quick start

### Install from a published wheel

```bash
# pick the wheel matching your platform (Linux / macOS / Windows)
pip install jaaicode-0.42.3-<platform>.whl
```

### Install from a source checkout

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

### Run

```bash
# interactive CLI
jaaicode

# Textual TUI
jaaicode --tui

# Web UI / server mode
jaaicode --serve

# MCP server
jaaicode --mcp

# Pipe / script mode
printf 'summarize this repo' | jaaicode --pipe

# Multi-day resumable workplan
jaaicode --workplan plan.yaml --resume

# Auto-isolate into a git worktree only when the repo is busy with others
jaaicode --auto-worktree
```

Configure a provider with an API key for OpenAI, Anthropic, Google/Gemini, Bedrock, Vertex, or Azure OpenAI — or point at an OpenAI-compatible endpoint or LiteLLM proxy — then select models with `/provider`, `/model`, or auto-routing.

---

## Project status

This document reflects the feature set on the active development branch as of May 2026 (`jaaicode 0.42.3`). Active development happens on private local-network remotes; the GitHub mirror is treated as informational and publish-only.
