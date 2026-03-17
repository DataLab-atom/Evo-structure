# Evo-structure Plugin — Git-Native MCTS Structural Code Search Engine

Evo-structure is the engineering implementation of **"[Paper title pending]"**. It transforms the Monte Carlo Tree Search (MCTS) Beam Search into a **git-native asynchronous search engine**, using LLM-driven **structural operators** to automatically explore optimal code structure variants in any git repository toward better benchmark performance.

> **Paper:** *(link pending)*

## Key Features

A structural evolution engine designed as the MCTS counterpart to Evo-anything, focusing on **MCTS search over the code structure space**:

| Feature | Description |
|---------|-------------|
| **Git-native search nodes** | Each search node = git branch; `ancestor_patches` vanishes, replaced by `git log` |
| **Persistent state machine** | mcts-engine MCP Server — crash-recoverable, process-independent |
| **Async Human Gate** | Synchronous blocking → message push + cron auto-continue on timeout; decide from your phone |
| **Cross-run memory** | Op experience across runs to avoid repeating failed directions |
| **8 structural operators** | insert / merge / decouple / split / extract / parallelize / pipeline / stratify |

## Installation

### Prerequisites

**Required:**
- Python >= 3.11
- Git

**Optional (automatically enabled when installed):**
- `oracle` CLI — MapAgent whole-repo context analysis (`npm install -g oracle`)
- `claude` CLI — ComboAgent complex structural rewrites using Claude Code instead of direct edits
- `codex` CLI — alternative for ComboAgent complex variant generation
- `lobster` CLI — atomic setup workflows + PR approval gate
- `tmux` — non-blocking background execution for long benchmarks
- `pyflakes` — static import/name checks before committing variants (`pip install pyflakes`)
- OpenClaw skills: `canvas`, `session-logs` (install via `clawhub install <slug>`)

### Option 1: npm (recommended)

```bash
npm install -g evo-structure
```

This automatically installs the Python MCP server via `pip` during the npm postinstall step.

After installation, configure your AI IDE:

```bash
# Configure all supported platforms (Claude Code, Cursor, Windsurf, OpenClaw)
npx evo-structure setup

# Or configure a specific platform
npx evo-structure setup --platform claude
npx evo-structure setup --platform cursor
npx evo-structure setup --platform windsurf
npx evo-structure setup --platform openclaw
```

---

### Option 2: Manual

#### Step 1: Install mcts-engine (required for all platforms)

```bash
git clone https://github.com/DataLab-atom/Evo-structure.git
cd Evo-structure/plugin/mcts-engine
pip install .
```

---

### OpenClaw

<details>
<summary>CLI one-liner (recommended)</summary>

```bash
openclaw plugins install openclaw-evo-structure
openclaw gateway restart
openclaw plugins doctor   # verify
```

</details>

<details>
<summary>Local development mode</summary>

```bash
openclaw plugins install -l ./plugin
openclaw gateway restart
```

</details>

<details>
<summary>Manual install</summary>

Copy the plugin to the extensions directory and register it in `~/.openclaw/openclaw.json`:

```bash
cp -r plugin/ ~/.openclaw/extensions/openclaw-evo-structure/
```

```json
{
  "plugins": {
    "entries": {
      "openclaw-evo-structure": {
        "enabled": true,
        "config": {}
      }
    }
  },
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "args": [],
      "env": {}
    }
  }
}
```

```bash
openclaw gateway restart
```

</details>

**Verify:** Type `/mcts-status` in a conversation. Seeing "Search not initialized" means the install succeeded.

---

### Claude Code

Add the MCP server to your project root or global `.claude/settings.json`:

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

Link skills to Claude Code:

```bash
ln -s $(pwd)/plugin/skills/* ~/.claude/skills/
```

Restart Claude Code and you're ready.

---

### Cursor

Add to `.cursor/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

Cursor will auto-discover MCP tools (`mcts_init`, `mcts_step`, etc.). Import the agent protocol as a Cursor Rule:

```bash
cp plugin/AGENTS.md .cursor/rules/evo-structure-agents.md
```

---

### Windsurf

Add to `~/.codeium/windsurf/mcp_config.json`:

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

---

### Any Other MCP-Compatible Client

Evo-structure's core is a standard [MCP](https://modelcontextprotocol.io) server. Any client that supports MCP stdio transport can connect:

```bash
# Start the server directly (stdio mode)
mcts-engine
```

Available MCP tools: `mcts_init`, `mcts_register_targets`, `mcts_step`, `mcts_check_cache`, `mcts_get_lineage`, `mcts_get_status`, `mcts_freeze_branch`, `mcts_boost_branch`, `mcts_record_synergy`.

---

### Optional Configuration

Search state is stored in `~/.openclaw/mcts-state/` by default. Override with an environment variable:

```bash
export COG_STATE_DIR=/path/to/your/state
```

Or configure via `openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "openclaw-evo-structure": {
        "enabled": true,
        "config": {
          "statePath": "/path/to/your/state"
        }
      }
    }
  }
}
```

## Quick Start

```
You send on Telegram: Help me optimize this feature extraction module
         ↓
  /search project_root "python main.py" triggers
         ↓
  MapAgent analyzes repo → identifies optimization target functions
         ↓
  mcts_init → run baseline to confirm it works → tag seed-baseline
         ↓
  MCTS search loop: each generation spawns beam_width structural variants
    ├── ComboAgent × N (in parallel)
    │     Critic → Engineer → AST check → git worktree → benchmark
    ├── UCB selects best nodes
    ├── Every gate_interval gens: push tree snapshot to your phone
    │     (auto-continue if no response within 30 min)
    └── ReflectAgent writes memory
         ↓
  Push best branch + send search report when done
```

## How It Works

Evo-structure maps each **search node** of the traditional MCTS Beam Search to a **git branch**, completely eliminating in-memory state dependency:

| MCTS Concept | Git Mapping | Advantage |
|-------------|-------------|-----------|
| search node | git branch + commit | Naturally persistent, crash-recoverable |
| root node | `seed-baseline` tag | Immutable baseline anchor |
| ancestor_patches | `git log` (commit chain diff) | Auto-recorded, no manual tracking |
| score | commit message | Persists with the commit |
| tried_combos | branch naming (enumerable) | `git branch --list 'mcts/*/gen-N/*'` |
| rollback | `git checkout` parent branch | Natural support, no side effects |
| apply best | `git checkout best-overall` | Idempotent, undoable |
| tree structure | `git log --graph --all` | Full lineage graph |
| parallel eval | git worktree | True isolation |

### 8 Structural Operators

| Type | Operator | Semantics |
|------|----------|-----------|
| Binary | `insert` | Insert new node C between A→B |
| Binary | `merge` | Merge A→B into a single node |
| Binary | `decouple` | Break the direct A→B dependency |
| Binary | `split` | Split B into B1, B2 |
| Binary | `extract` | Extract common component C from A→B |
| N-ary | `parallelize` | Convert sequential intermediate nodes to concurrent |
| N-ary | `pipeline` | Convert chain to streaming processing |
| N-ary | `stratify` | Reorganize cross-layer dependencies into ordered layers |

### Search Loop

Each MCTS generation has six phases:

1. **Planning** — `mcts_step("begin_generation")`: UCB selects frontier, assigns beam_width op tasks
2. **Generation** — ComboAgent in parallel: Critic → Engineer → static check → git commit
3. **Evaluation** — git worktree isolation → run benchmark → report fitness
4. **Selection** — `mcts_step("select")`: UCB keeps top-K, prunes eliminated branches
5. **Human Gate** — every N generations, push search tree snapshot to phone; wait for decision (auto-continue on timeout)
6. **Reflection** — ReflectAgent extracts lessons into structured memory

### Comparison with Evo-anything

| Dimension | Evo-anything | Evo-structure |
|-----------|-------------|---------------|
| Search scope | Global multi-target evolution | Tree-shaped MCTS search |
| Core mapping | individual → git branch | search node → git branch |
| State machine | evo-engine MCP Server | mcts-engine MCP Server |
| Selection strategy | Population selection + Synergy | UCB + Beam Search |
| Human interaction | Progress notifications | **Async Human Gate** (per-generation decisions) |
| Memory system | per-target + global | per-op + per-project |

**Key differentiator**: The Async Human Gate is unique to Evo-structure — it transforms MCTS from "must babysit while it runs" into "check your phone before bed and decide".

## Skills

| Command | Description |
|---------|-------------|
| `/search <project_path> <benchmark_cmd>` | Full search entry: init + baseline + start MCTS loop |
| `/mcts-status` | View current search progress, tree state, best score |
| `/mcts-report` | Generate search report (lineage + op analysis + diff) |
| `/mcts-stop` | Stop search, apply current best node |
| `/mcts-rollback` | Roll back to previous generation frontier |
| `/boost <branch>` | Increase UCB priority of a branch |
| `/freeze <branch>` | Freeze a branch, stop expanding from it |

## Repository Structure

```
Evo-structure/
├── LICENSE
├── README.md
├── README_EN.md
├── research/                      # Design docs and analysis
│   └── mcts-engine/
│       ├── README.md              # Document index
│       ├── 01_current_analysis.md # Deep analysis of original MCTS implementation
│       ├── 02_platform_mapping.md # Platform capability mapping
│       └── DESIGN.md              # Full design document
└── plugin/
    ├── openclaw.plugin.json       # Plugin definition
    ├── AGENTS.md                  # Search protocol (core loop)
    ├── SOUL.md                    # Agent persona
    ├── TOOLS.md                   # Tool usage conventions
    ├── agents/                    # Per-agent behavior specs
    │   ├── orchestrator.md        # OrchestratorAgent (with canvas dashboard)
    │   ├── combo_agent.md         # ComboAgent (with static checks, tmux, coding-agent)
    │   ├── gate_agent.md          # GateAgent (async Human Gate)
    │   ├── policy_agent.md        # PolicyAgent
    │   ├── reflect_agent.md       # ReflectAgent (with cross-run memory)
    │   └── map_agent.md           # MapAgent (with oracle whole-repo analysis)
    ├── mcts-engine/               # Search engine (MCP server)
    │   ├── server.py              # MCP tool interface + state machine
    │   ├── models.py              # Data models
    │   └── selection.py           # UCB selection algorithms
    ├── skills/                    # User-invocable skills
    │   ├── search/                # Start a search
    │   ├── mcts-status/           # Check progress
    │   ├── mcts-report/           # Generate report
    │   ├── mcts-stop/             # Stop search
    │   ├── mcts-rollback/         # Roll back
    │   ├── boost/                 # Boost priority
    │   └── freeze/                # Freeze node
    └── workflows/                 # Lobster declarative workflows
        ├── mcts-setup.lobster     # Atomic setup (validate→baseline→tag→mkdir)
        └── mcts-finish.lobster    # Finish flow (tag→push→approval gate→PR)
```

## Search Memory

Evo-structure maintains structured memory in the target repository to avoid exploring failed directions again:

```
memory/
├── global/
│   ├── long_term.md              # Cross-project lessons
│   └── op_lessons.md            # Cross-project op summaries
├── projects/{project_hash}/
│   ├── long_term.md              # Accumulated wisdom for this project
│   └── runs/{run_id}/
│       ├── gen_{N}.md            # Per-generation reflection
│       ├── tree_final.md         # Final search tree snapshot
│       └── best_diff.md          # Best node vs baseline diff summary
└── ops/
    ├── insert.md                 # Global insert op success/failure log
    ├── merge.md
    └── ...  (remaining operators)
```

## Branch Naming

```
mcts/{run_id}/gen-{N}/{op}-{uuid8}           # standard search node
mcts/{run_id}/synergy/{opA}+{opB}-{uuid8}   # cross-op combination (Synergy)
```

Tags: `seed-baseline`, `best-gen-{N}`, `best-mcts-{run_id}`, `best-overall`

Commit message format (machine-parseable):
```
mcts(score=0.8423,op=insert,gen=3,run=a3f9b2): insert cache layer between loader→extractor
```
