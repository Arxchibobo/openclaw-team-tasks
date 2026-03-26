# openclaw-team-tasks

**Multi-agent pipeline coordination for OpenClaw — orchestrate dev teams across Linear, DAG, and Debate workflows.**

![Python](https://img.shields.io/badge/python-3.12%2B-blue)
![Dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)

## Overview

`openclaw-team-tasks` is a standalone Python CLI tool that lets an orchestrating AI agent (the "AGI coordinator") manage multi-agent development workflows through shared JSON state files. It is designed as an [OpenClaw](https://github.com/openclaw/openclaw) skill and integrates with OpenClaw's `sessions_send` API to dispatch tasks to worker agents (e.g., `code-agent`, `test-agent`, `docs-agent`, `monitor-bot`).

All state is stored locally as JSON — no server, no external dependencies, no pip install required.

## Features

Three coordination modes for different workflows:

| Mode | Description | Best For |
|------|-------------|----------|
| **Linear** | Sequential pipeline with auto-advance | Bug fixes, simple features, step-by-step workflows |
| **DAG** | Dependency graph with parallel dispatch | Large features, spec-driven dev, complex task graphs |
| **Debate** | Multi-agent positions + cross-review + synthesis | Code reviews, architecture decisions, competing hypotheses |

- **Zero dependencies** — pure Python 3.12+ stdlib
- **JSON-backed state** — inspect, edit, or version-control task files directly
- **Cycle detection** — DAG mode rejects circular dependencies on `add`
- **Auto-unblock notifications** — completing a DAG task reports which downstream tasks are now ready
- **`--json` output** — machine-readable output for programmatic dispatch loops
- **Workspace tracking** — optional shared directory path threaded through all task contexts

## Requirements

- Python 3.12+
- No external packages

## Installation

```bash
git clone https://github.com/Arxchibobo/openclaw-team-tasks.git
cd openclaw-team-tasks

# No pip install needed — run directly
python3 scripts/task_manager.py --help
```

For OpenClaw skill integration, copy to your skills directory:

```bash
cp -r openclaw-team-tasks/ /path/to/clawd/skills/team-tasks/
```

## Quick Start

### Mode A: Linear Pipeline

Agents execute sequentially; completing one stage auto-advances to the next.

```bash
TM="python3 scripts/task_manager.py"

# Create project with pipeline order
$TM init my-api -g "Build REST API with tests and docs" \
  -p "code-agent,test-agent,docs-agent,monitor-bot"

# Assign a task to each stage
$TM assign my-api code-agent "Implement Flask REST API: GET/POST/DELETE /items"
$TM assign my-api test-agent "Write pytest tests, target 90%+ coverage"
$TM assign my-api docs-agent "Write README with API docs and examples"
$TM assign my-api monitor-bot "Security audit and deployment readiness check"

# Check what's next
$TM next my-api
# ▶️  Next stage: code-agent

# Dispatch → work → save result → mark done
$TM update my-api code-agent in-progress
$TM result my-api code-agent "Created app.py with 3 endpoints"
$TM update my-api code-agent done
# ▶️  Next: test-agent  (auto-advance!)

$TM status my-api
```

**Status output:**
```
📋 Project: my-api
🎯 Goal: Build REST API with tests and docs
📊 Status: active  |  Mode: linear
▶️  Current: test-agent

  ✅ code-agent: done
     Task: Implement Flask REST API
     Output: Created app.py with 3 endpoints
  🔄 test-agent: in-progress
     Task: Write pytest tests, target 90%+ coverage
  ⬜ docs-agent: pending
  ⬜ monitor-bot: pending

  Progress: [██░░] 2/4
```

---

### Mode B: DAG (Dependency Graph)

Tasks declare dependencies and are dispatched in parallel when their deps are satisfied.

```bash
TM="python3 scripts/task_manager.py"

$TM init my-feature -m dag -g "Build search feature with parallel workstreams"

# Add tasks with dependencies
$TM add my-feature design      -a docs-agent  --desc "Write API spec"
$TM add my-feature scaffold    -a code-agent  --desc "Create project skeleton"
$TM add my-feature implement   -a code-agent  -d "design,scaffold" --desc "Implement API"
$TM add my-feature write-tests -a test-agent  -d "design"          --desc "Write test cases from spec"
$TM add my-feature run-tests   -a test-agent  -d "implement,write-tests" --desc "Run all tests"
$TM add my-feature write-docs  -a docs-agent  -d "implement"       --desc "Write final docs"
$TM add my-feature review      -a monitor-bot -d "run-tests,write-docs"  --desc "Final review"

$TM graph my-feature
```

**Graph output:**
```
📋 my-feature — DAG Graph

├─ ⬜ design [docs-agent]
│  ├─ ⬜ implement [code-agent]
│  │  ├─ ⬜ run-tests [test-agent]
│  │  │  └─ ⬜ review [monitor-bot]
│  │  └─ ⬜ write-docs [docs-agent]
│  └─ ⬜ write-tests [test-agent]
└─ ⬜ scaffold [code-agent]
   └─ ⬜ implement (↑ see above)

  Progress: [░░░░░░░] 0/7
```

```bash
# Get all tasks ready to dispatch simultaneously
$TM ready my-feature
# 🟢 Ready to dispatch (2 tasks):
#   📌 design → agent: docs-agent
#   📌 scaffold → agent: code-agent

$TM update my-feature design done
# 🟢 Unblocked: write-tests  ← auto-detected!
```

**Key DAG behaviors:**
- `ready` returns all tasks whose dependencies are satisfied — dispatch them in parallel
- `ready --json` includes `depOutputs` — previous stage results passed to the next agent
- Cycle detection on `add` — circular dependencies are rejected immediately
- Partial failure: unrelated branches continue; only downstream tasks block

---

### Mode C: Debate (Multi-Agent Deliberation)

Send the same question to multiple agents, collect positions, cross-review, and synthesize.

```bash
TM="python3 scripts/task_manager.py"

$TM init security-review --mode debate \
  -g "Review auth module for security vulnerabilities"

# Add debaters with roles
$TM add-debater security-review code-agent  --role "security expert focused on injection attacks"
$TM add-debater security-review test-agent  --role "QA engineer focused on edge cases"
$TM add-debater security-review monitor-bot --role "ops engineer focused on deployment risks"

# Round 1: initial positions
$TM round security-review start
$TM round security-review collect code-agent  "Found SQL injection in login()"
$TM round security-review collect test-agent  "Missing input validation on email field"
$TM round security-review collect monitor-bot "No rate limiting on auth endpoints"

# Round 2: cross-review
$TM round security-review cross-review
$TM round security-review collect code-agent  "Agree on validation. Rate limiting is critical."
$TM round security-review collect test-agent  "SQL injection is most severe. Adding rate limit tests."
$TM round security-review collect monitor-bot "Both findings valid. Recommending WAF as additional layer."

# Synthesize all positions
$TM round security-review synthesize
```

**Debate workflow:**
```
Question → [Agent A] → Position A ─┐
         → [Agent B] → Position B ─┤── Cross-Review ── Synthesis
         → [Agent C] → Position C ─┘
```

## CLI Reference

| Command | Mode | Usage | Description |
|---------|------|-------|-------------|
| `init` | all | `init <project> -g "goal" [-m linear\|dag\|debate]` | Create project |
| `add` | dag | `add <project> <task-id> -a <agent> [-d <deps>] [--desc "..."]` | Add task with deps |
| `add-debater` | debate | `add-debater <project> <agent-id> [-r "role"]` | Add debater |
| `round` | debate | `round <project> start\|collect\|cross-review\|synthesize` | Debate actions |
| `status` | all | `status <project> [--json]` | Show progress |
| `assign` | linear/dag | `assign <project> <stage> "desc"` | Set task description |
| `update` | linear/dag | `update <project> <stage> <status>` | Change status |
| `next` | linear | `next <project> [--json]` | Get next stage |
| `ready` | dag | `ready <project> [--json]` | Get dispatchable tasks |
| `graph` | dag | `graph <project>` | Show dependency tree |
| `log` | linear/dag | `log <project> <stage> "msg"` | Add log entry |
| `result` | linear/dag | `result <project> <stage> "output"` | Save stage output |
| `reset` | linear/dag | `reset <project> [stage] [--all]` | Reset to pending |
| `history` | linear/dag | `history <project> <stage>` | Show log history |
| `list` | all | `list` | List all projects |

### Status Values

| Status | Icon | Meaning |
|--------|------|---------|
| `pending` | ⬜ | Waiting for dispatch |
| `in-progress` | 🔄 | Agent is working |
| `done` | ✅ | Completed |
| `failed` | ❌ | Failed (blocks downstream tasks) |
| `skipped` | ⏭️ | Intentionally skipped |

### `init` Options

```bash
python3 scripts/task_manager.py init <project> \
  --goal "Project description" \
  --mode linear|dag|debate \
  --pipeline "agent1,agent2,agent3"  # linear only \
  --workspace "/path/to/shared/dir" \
  --force  # overwrite existing project
```

## Integration with OpenClaw

This tool is designed as an OpenClaw skill. The orchestrating agent tracks all state via the CLI and dispatches work to named worker agents via `sessions_send`.

**Linear dispatch loop:**
```
1. next <project> --json           → get next stage info
2. update <project> <agent> in-progress
3. sessions_send(agent, task)      → dispatch to worker agent
4. Wait for agent reply
5. result <project> <agent> "..."  → save output
6. update <project> <agent> done   → auto-advances to next stage
7. Repeat
```

**DAG dispatch loop:**
```
1. ready <project> --json          → get all dispatchable tasks (with depOutputs)
2. For each ready task (parallel):
   a. update <project> <task> in-progress
   b. sessions_send(agent, task + depOutputs)
3. On reply: result → update done → check newly unblocked tasks
4. Repeat until all tasks complete
```

## Common Pitfalls

**Linear: stage ID is the agent name, not a number**
```bash
# ❌ Wrong
python3 scripts/task_manager.py assign my-project 1 "Build API"

# ✅ Correct
python3 scripts/task_manager.py assign my-project code-agent "Build API"
```

**DAG: dependencies must be added before they can be referenced**
```bash
# ❌ Wrong — "dependency 'design' not found"
$TM add my-project implement -a code-agent -d "design"

# ✅ Correct — add the dependency first
$TM add my-project design -a docs-agent --desc "Write spec"
$TM add my-project implement -a code-agent -d "design" --desc "Implement"
```

**Debate: all debaters must be added before starting a round**
```bash
# ❌ Wrong — cannot add debaters after rounds start
$TM round my-debate start
$TM add-debater my-debate new-agent

# ✅ Correct
$TM add-debater my-debate agent-a
$TM add-debater my-debate agent-b
$TM round my-debate start
```

## Configuration

### Data Directory

Project files are stored as JSON at:
```
/home/ubuntu/clawd/data/team-tasks/<project>.json
```

Override with an environment variable:
```bash
export TEAM_TASKS_DIR=/custom/path
```

## Project Structure

```
openclaw-team-tasks/
├── README.md                         # This file
├── SKILL.md                          # OpenClaw skill definition
├── SPEC.md                           # Enhancement spec (debate + workspace)
├── scripts/
│   └── task_manager.py               # Main CLI (Python 3.12+, stdlib only)
└── docs/
    ├── GAP_ANALYSIS.md               # Comparison with Claude Code Agent Teams
    ├── AGENT_TEAMS_OFFICIAL_DOCS.md  # Reference documentation
    ├── three-modes-overview.svg
    ├── dev-team-agents-detail.svg
    └── debate-mode-flow.svg
```

## License

MIT
