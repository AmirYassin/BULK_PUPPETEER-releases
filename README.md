<p align="center">
  <img src="docs/assets/liquid_silver_logo.png" alt="BULK_PUPPETEER Logo" width="300">
</p>

<h1 align="center">BULK_PUPPETEER</h1>
<p align="center"><strong>v4.4.3</strong> — Task Orchestration Daemon for macOS Apple Silicon</p>

---

> [!CAUTION]
> **SPAWNED TASKS START IN DORMANT STATE — RESUME TO ACTIVATE!**

---

## Table of Contents

- [What is BULK_PUPPETEER?](#what-is-bulk_puppeteer)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [Security Architecture](#security-architecture)
- [API Reference](#api-reference)
- [CLI Reference](#cli-reference)
- [User Interface](#user-interface)
- [Architecture](#architecture)
- [CI/CD & Headless Mode](#cicd--headless-mode)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## What is BULK_PUPPETEER?

**BULK_PUPPETEER** is a framework-agnostic task orchestration daemon designed for macOS Apple Silicon. It runs as a background FastAPI service, managing concurrent shell tasks defined within Directed Acyclic Graphs (DAGs).

The system provides:
- A **native macOS Menu Bar app** (AppKit) for host-level management
- An **E2EE Command Line Interface** for task orchestration
- A **Web Dashboard** with real-time PTY streaming and a command palette
- An interactive **physics-based 3D logo** rendered via Three.js

---

## Installation

### Prerequisites

- macOS 13+ (Apple Silicon)
- Python 3.11+
- Xcode Command Line Tools (`xcode-select --install`)

### From DMG (macOS App)

1. Download [`BULK_PUPPETEER_v4.4.3.dmg`](https://github.com/AmirYassin/BULK_PUPPETEER-releases/releases/download/v4.4.3/BULK_PUPPETEER_v4.4.3.dmg)
2. Drag `BULK_PUPPETEER.app` to `/Applications`
3. Launch the app — on first run, it automatically injects a `bulk-cli` alias into your `~/.zshrc` and `~/.bash_profile`
4. Restart your terminal (or run `source ~/.zshrc`)

The CLI is available as `bulk-cli` when installed via DMG.

> [!NOTE]
> **CLI naming:** `swarm-cli` (from source) and `bulk-cli` (from DMG) are the same tool. All examples below use `swarm-cli` — substitute `bulk-cli` if you installed via DMG.

### From Source

```bash
git clone https://github.com/AmirYassin/BULK_PUPPETEER.git
cd BULK_PUPPETEER
/usr/bin/pip3 install -e ".[dev]"
```

The CLI is available as `swarm-cli` when running from source.

### Post-Install: Environment Setup

> [!WARNING]
> **`$PATH` Isolation:** BULK_PUPPETEER runs as a native macOS AppKit daemon and **does not inherit your terminal's `$PATH`** (e.g., `~/.zshrc` or `/opt/homebrew/bin`).

To ensure the Swarm Engine can locate your CLI tools (`gemini`, `python`, `node`, etc.), create system-level symlinks:

```bash
sudo ln -s /opt/homebrew/bin/gemini /usr/local/bin/gemini
```

Click **"Restart Daemon"** in the Menu Bar after creating symlinks for them to take effect.

### First-Run Setup

After launching the app for the first time, complete these steps from the Menu Bar icon:

1. **Set your WhatsApp number** — Menu Bar > **"Set WhatsApp Number…"** (or set `BP_USER_PHONE` env var before launch). The number is saved to `~/Library/Application Support/BULK_PUPPETEER/.user_jid` and takes effect immediately — no restart required.
2. **Set your Gemini API key** — Menu Bar > **"Set Gemini API Key…"** (or set `GEMINI_API_KEY` env var). If the key in the environment and the saved file differ, the app will alert you with a fingerprint comparison dialog.
3. **Check the log file** — all daemon output is written to `~/Library/Application Support/BULK_PUPPETEER/server.log`.

All writable runtime state (SQLite database, session token, agent profiles) is stored in `~/Library/Application Support/BULK_PUPPETEER/`.

### Build macOS App (DMG)

```bash
# Dev build (LTO disabled, Nuitka cache enabled — fast iteration)
./build_release_cached.sh

# Production build (full LTO + notarization)
PRODUCTION=1 ./build_release.sh
```

Output: `dist/BULK_PUPPETEER_v4.4.3.dmg`

### Run Tests

```bash
# Full suite (backend + Playwright E2E)
/usr/bin/python3 run_tests.py

# Backend only
/usr/bin/python3 -m pytest -v -p no:playwright -n auto --timeout=40 \
  --ignore=TESTS/AUTO_UI --ignore=TESTS/backend/test_macos_integration.py TESTS/backend/

# E2E only
/usr/bin/python3 -m pytest -v -n auto --timeout=40 TESTS/AUTO_UI/
```

---

## Quick Start

### 1. Launch the App

```bash
# From source
./start_daemon.sh [manifest_path] [port]

# Or from /Applications (if installed via DMG)
open -a BULK_PUPPETEER
```

The daemon binds to `127.0.0.1:8080` and generates a session token at `.daemon.token`.

### 2. Copy the Session Token

Click the Menu Bar icon and select **"Copy Session Token"**, or:

```bash
export SWARM_TOKEN=$(cat .daemon.token)
```

> **Token auto-detection:** The CLI automatically looks for `.daemon.token` in your current working directory. You only need to export the token if running from a different directory, or override it with `--token`.

### 3. Spawn Tasks

> [!CAUTION]
> **Spawned tasks start in DORMANT state. You MUST resume them to activate!** Use `--run` to start immediately.

```bash
# Set max concurrent tasks (default: 3)
swarm-cli set-workers 4

# Spawn independent agents (dormant by default — must resume)
swarm-cli add DATA_BOT_A "write a python script named 'a.py' that prints Hello"
swarm-cli add DATA_BOT_B "write a python script named 'b.py' that prints World"

# Spawn and immediately start (--run skips DORMANT state)
swarm-cli add DATA_BOT_C "verify that a.py and b.py produce Hello World" --deps DATA_BOT_A,DATA_BOT_B --run

# Spawn an agent using Claude with the Sonnet variant
swarm-cli add CODE_REVIEW "review the changes in src/" --model claude --model-variant sonnet

# Spawn a raw shell command (no model wrapping)
swarm-cli add HEALTH_CHECK "echo 'all good'" --model raw
```

The CLI automatically inherits your shell's working directory and preserves `.gemini` session history via dynamic `-r` flag injection. Think of spawning a task as "forking" your current terminal context.

### 4. Activate Tasks (Required)

Tasks are intentionally spawned in a dormant state to prevent sudden CPU overloads. You must trigger them to start:

- **Option A (Web Dashboard):** Open `http://127.0.0.1:8080` and click **Resume** on the task
- **Option B (CLI):**
  ```bash
  swarm-cli resume DATA_BOT_A
  ```
- **Option C (REST API):**
  ```bash
  curl -X POST -H "Authorization: Bearer $SWARM_TOKEN" \
    "http://127.0.0.1:8080/api/tasks/resume/DATA_BOT_A"
  ```

### 5. Monitor & Control

```bash
# Stream live PTY output (maintains WebSocket until process ends or you disconnect)
swarm-cli logs DATA_BOT_C

# Check swarm status (human-readable)
swarm-cli status

# Check swarm status (machine-readable JSON, for scripts/agents)
swarm-cli status --json

# Kill a task
swarm-cli kill DATA_BOT_C

# Pause the entire swarm
swarm-cli pause

# Resume the swarm
swarm-cli resume-swarm

# Resume all dormant tasks
swarm-cli resume-all

# Send input to a task's stdin
swarm-cli stdin MY_TASK "yes"

# Delete a completed task
swarm-cli delete OLD_TASK

# View backend logs
swarm-cli server-logs --tail 200
```

### 6. Stop the Daemon

```bash
./stop_daemon.sh
```

---

## What's New in v4.4.3

- **Chat history preservation** — the AI remembers conversation context within a session. History survives crash restarts automatically. Switching models clears the history to avoid cross-model context bleed.
- **Orchestrator agent spawning** — the `veteran-wa-orchestrator` swarm profile now ships with a full guide for spawning Gemini and Claude sub-agents from within an orchestration session.
- **User Identity (WhatsApp number)** — set your phone number via Menu Bar > **"Set WhatsApp Number…"**. Saved to `~/Library/Application Support/BULK_PUPPETEER/.user_jid`, active immediately in-process with no restart required. Also configurable via the `BP_USER_PHONE` env var.
- **SSL certificate fix for .app bundle** — `paths.py` monkey-patches `certifi` at startup so that `httpx` and `google-genai` resolve SSL correctly after installation from DMG, where the system Python cert bundle is not available.
- **Gemini API key mismatch detection** — if the `GEMINI_API_KEY` env var and the saved key file contain different keys, a tray dialog appears with a fingerprint comparison so you can resolve the conflict before the daemon starts making API calls.
- **WhatsApp bridge auth state machine** — bridge authentication is now driven by stdout parsing against a well-defined state machine, replacing the previous HTTP polling approach that produced false-positive auth failures.
- **Unified writable state directory** — SQLite database, session token, and agent profiles all live under `~/Library/Application Support/BULK_PUPPETEER/`, consistent with macOS sandboxing conventions.

## What's New in v4.0.0 (The SOTA iTerm & Rate-Limit Upgrade)

- **iTerm-in-Browser UI:** Complete aesthetic overhaul of the web terminal. The console now features a strict native monospace stack, dark-mode 12px rounded modal styling, and rigorous scroll-lock tolerances that replicate a native macOS iTerm2 experience directly inside the browser.
- **Concurrency Burst Mitigation (Execution Jitter):** The backend Execution Engine now employs a mathematically randomized `1.0-8.0s` startup jitter. When unleashing a swarm of agents simultaneously, this architectural mechanism cleanly staggers subprocess initialization, mathematically circumventing strict LLM API Request-Per-Second (429) rate limits.
- **JSON WebSocket Multiplexing:** Refactored the frontend-to-backend pipeline to use a scalable JSON message protocol (`{"type": "input", "data": "..."}` and `{"type": "resize"}`), breaking away from legacy raw-text buffering.
- **SIGWINCH Window Resizing:** The terminal viewport now actively synchronizes its dimensions with the underlying macOS kernel. Resizing the browser actively redraws terminal apps (like `top` or `vim`) in real-time.
- **TrueColor (256) PTY Injection:** The engine now strictly enforces `TERM=xterm-256color` and explicitly allocates a master/slave pseudo-terminal, ensuring that high-fidelity UI apps like the Gemini CLI render flawlessly.

## What's New in v3.8.0

- **DORMANT state** — dynamically added tasks now start in `DORMANT` instead of overloading `KILLED`. Clear semantic distinction between "not yet started" and "user terminated"
- **`--run` flag** — `swarm-cli add <id> <prompt> --run` starts the task immediately, skipping DORMANT state
- **Hot-resize concurrency** — `set-workers` swaps the semaphore in-place. Running tasks are no longer killed and the manifest is no longer reloaded
- **Shell escaping hardening** — `_escape_for_shell()` now handles `!` (bash history expansion), `\n`, `\r`, and null bytes
- **PTY EIO resilience** — graceful handling of `OSError(EIO)` when the PTY slave closes before the read loop
- **Exit code hints** — failed tasks with exit code 127 or 126 now include diagnostic hints ("command not found", "permission denied") in their log buffer
- **`SWARM_TOKEN` env var** — CLI checks `SWARM_TOKEN` environment variable before falling back to `.daemon.token` file
- **`--stable-token`** — daemon flag to reuse existing `.daemon.token` across restarts (avoids breaking existing agent connections)
- **DORMANT badge** — Web Dashboard renders DORMANT tasks with a teal status badge

### What's New in v3.7.0

- **Full CLI coverage** — all 16 REST API endpoints now have `swarm-cli` commands (was 8)
- **`swarm-cli resume`** — resume dormant/killed tasks from the CLI (previously dashboard-only)
- **Multi-model support** — `--model` and `--model-variant` flags on `swarm-cli add` (gemini, claude, aider, raw)
- **8 new commands** — `kill-all`, `pause`, `resume-swarm`, `resume-all`, `stdin`, `edit`, `delete`, `server-logs`
- **Security hardening** — expanded model variant validation, strict Claude allowlist

---

## Core Concepts

### Directed Acyclic Graph (DAG)

Tasks are not executed sequentially — they run dynamically based on dependency relationships. The engine uses **Kahn's Algorithm** to guarantee a cycle-free execution graph.

- A task with `depends_on` stays `BLOCKED` until all dependencies reach `COMPLETED`
- If a dependency fails (`FAILED`), dependent tasks cascade to failure automatically
- Tasks with no dependencies run immediately (subject to concurrency limits)

### Worker Concurrency

The engine uses an `asyncio.Semaphore` to limit how many tasks run simultaneously. **Default: 3 workers.** If 5 tasks are unblocked at once, 3 run immediately and 2 remain `QUEUED` until a worker slot opens.

To prevent high-concurrency swarms from tripping LLM API burst rate limits (e.g., `429 Too Many Requests`), the Execution Engine employs an automated `1.0-8.0s` startup jitter. This cleanly staggers the `fork/exec` calls across the time window, keeping burst spikes below the server's radar.

```bash
# View current limit
swarm-cli get-workers

# Hot-resize (running tasks are NOT killed — takes effect for new task scheduling)
swarm-cli set-workers 8
```

### Task Lifecycle (FSM)

Every task moves through a Finite State Machine with validated transitions:

```
DORMANT → QUEUED → IN_PROGRESS → COMPLETED
                               → FAILED
                               → KILLED
BLOCKED → QUEUED (when dependencies complete)
COMPLETED / FAILED / KILLED / DORMANT → QUEUED (restart/retry)
```

> **DORMANT** is the default state for dynamically added tasks. It indicates the task is registered but not yet scheduled. Use `resume` or add with `--run` to activate.

Illegal state jumps raise `IllegalStateTransitionError`.

---

## Security Architecture

All local communication is secured with a three-layer protocol:

1. **Bearer Token Auth** — 64-char hex token generated via `os.urandom(32).hex()`, stored in `.daemon.token` (chmod 600)
2. **AES-128-CBC Encryption (Fernet)** — all CLI-to-daemon payloads are encrypted end-to-end, keyed from the session token
3. **CORS restricted to localhost** — the API only accepts cross-origin requests from `http://127.0.0.1:8080` and `http://localhost:8080`; external origins are blocked

The bearer token is never logged. Even local loopback traffic intercepted by other processes is unreadable.

---

## API Reference

All endpoints require: `Authorization: Bearer <64_char_hex_token>`

### REST

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/tasks` | List all tasks with their current state |
| `GET` | `/api/status` | Global swarm status (concurrency, pause state, task count) |
| `POST` | `/api/tasks/add` | Inject a new task (DORMANT by default, or auto-start with `auto_start: true`) |
| `POST` | `/api/tasks/resume/{id}` | Resume/restart a single dormant task |
| `POST` | `/api/tasks/kill/{id}` | Send SIGKILL to a specific task |
| `POST` | `/api/tasks/kill_all` | Emergency kill all active tasks |
| `POST` | `/api/tasks/pause_all` | Globally pause the swarm |
| `POST` | `/api/tasks/resume_swarm` | Globally resume the swarm |
| `POST` | `/api/tasks/resume_all` | Resume all dormant tasks (DORMANT/FAILED/COMPLETED/KILLED) |
| `POST` | `/api/tasks/{id}/stdin` | Inject raw text into a task's stdin |
| `POST` | `/api/config/concurrency` | Hot-resize worker concurrency (no restart, running tasks unaffected) |
| `PUT` | `/api/tasks/{id}` | Replace the configuration of a dormant task |
| `DELETE` | `/api/tasks/{id}` | Purge a dormant task from memory |
| `GET` | `/api/tasks/{id}/logs` | Retrieve the 2000-line output buffer for a task |
| `GET` | `/api/server_logs` | Tail the backend diagnostic log file |

### Task Payload Schema

```json
{
  "id": "BUILD_CORE",
  "cmd": "python3 build.py",
  "cwd": "/opt/app",
  "env": {"PRODUCTION": "1"},
  "depends_on": ["LINT", "TEST"],
  "use_pty": true
}
```

### WebSocket

| Endpoint | Purpose |
|----------|---------|
| `ws://.../ws/console/{id}` | Bi-directional PTY channel (stdout/stderr + keystrokes) |
| `ws://.../ws/server_logs` | Read-only daemon event stream |

---

## CLI Reference

> **Reminder:** Use `swarm-cli` (from source) or `bulk-cli` (from DMG). They are identical.

### Global Arguments

| Argument | Default | Purpose |
|----------|---------|---------|
| `--port` | `8080` | Daemon port (`1024-65535`) |
| `--token` | Auto from `.daemon.token` | Session token for auth + E2EE key derivation |
| `--json` | `false` | Machine-readable JSON output for scripts, agents, and parsing tools (e.g., `bulk-cli status --json | jq '.tasks'`). |
| `--headless` | `false` | Run without macOS Menu Bar (for CI/CD and headless servers) |

### Commands

| Command | Arguments | Purpose |
|---------|-----------|---------|
| `status` | — | Swarm topology and task states |
| `add` | `<id> <prompt> [--cwd] [--deps] [--model] [--model-variant] [--profile] [--run]` | Spawn a task (DORMANT by default; `--run` starts immediately). Assign personas via `--profile`. |
| `kill` | `<id>` | Terminate a task via SIGKILL |
| `resume` | `<id>` | Resume a dormant/killed task |
| `kill-all` | — | Emergency kill all active tasks |
| `pause` | — | Globally pause the swarm |
| `resume-swarm` | — | Globally resume the swarm |
| `resume-all` | — | Resume all dormant tasks (DORMANT/KILLED/FAILED/COMPLETED) |
| `stdin` | `<id> <text>` | Send raw text to a task's stdin |
| `edit` | `<id> <json>` | Replace config of a dormant task |
| `delete` | `<id>` | Purge a dormant task from memory |
| `server-logs` | `[--tail N]` | Tail backend diagnostic logs |
| `logs` | `<id>` | Attach to live PTY stream (last 2000 lines circular buffer) |
| `get-workers` | — | Show concurrency limit |
| `set-workers` | `<N>` | Hot-resize max concurrent tasks (running tasks unaffected) |

### Supported Models

| Model | Binary | Variants | Description |
|-------|--------|----------|-------------|
| `gemini` (default) | `gemini` | Any Gemini API model ID (`gemini-2.5-flash`, `gemini-2.5-pro`, `flash`, `pro`, etc.) | Google Gemini CLI |
| `claude` | `claude` | Strict allowlist: `sonnet`, `opus`, `haiku`, `sonnet[1m]`, `opus[1m]`, `opusplan`, plus specific full IDs (`claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`, etc.) | Anthropic Claude Code |
| `aider` | `aider` | Any LiteLLM model string (`gemini/gemini-2.5-pro`, `claude-sonnet-4-6`, `gpt-4o`, etc.) | AI pair programming |
| `raw` | — | — | Raw shell command (no wrapping) |

> **Model string validation:** Gemini and Aider accept any model string their respective CLIs support — the orchestrator passes them through without an allowlist. Claude validates against a known model list because Claude Code is strict about model name format.

### CLI Output Examples

**Human-readable (default):**
```bash
swarm-cli status
# Swarm Status: ACTIVE
# Concurrency: 3
# Total Tasks: 12
```

**Machine-readable (for agents/scripts):**
```bash
swarm-cli status --json
# {
#   "tasks": {
#     "build_frontend": {"status": "COMPLETED", "cmd": "npm run build", "return_code": 0}
#   },
#   "swarm": {"is_paused": false, "concurrency_limit": 3, "task_count": 1}
# }
```

> **Note on prompts:** The `add` command wraps your prompt in an AES-encrypted JSON payload. For complex prompts with nested quotes or long file paths, keep them concise or use single quotes internally to prevent JSON escape errors during transmission.

---

## User Interface

### macOS Menu Bar

The daemon integrates with macOS WindowServer via AppKit:
- **Pause/Resume All** — global task controls
- **Copy Session Token** — copies to clipboard with native notification
- **Set WhatsApp Number…** — saves your phone number to `.user_jid`; takes effect immediately, no restart needed
- **Set Gemini API Key…** — stores your API key; detects mismatches between env var and saved file
- **Restart Daemon** — clean `os.execv` process replacement

### Web Dashboard

![Web Dashboard](docs/assets/dashboard.png)

- **Real-time PTY streaming** with scroll-lock for rapid output
- **Command Palette** — press `/` for keyboard-driven orchestration
- **3D Logo** — interactive physics-based marionette rendered in WebGL

---

## Architecture

```
src/bulk_puppeteer/core/
├── engine.py       # ExecutionEngine, TaskManager, polymorphic task hierarchy
├── api.py          # FastAPI app factory, REST + WebSocket, ASGI middleware
├── models.py       # Pydantic schemas (TaskDef, TaskState, TaskStatus)
├── config.py       # Immutable SwarmConfig (single source of truth)
├── fsm.py          # TaskStateMachine (legal transition adjacency list)
├── dag.py          # SwarmDAG (Kahn's Algorithm cycle detection)
├── events.py    # EventBus (Observer pattern, decouples I/O from WS)
├── tray.py         # macOS Menu Bar (rumps + AppKit)
├── react.py        # ReActAgent (LLM-driven Reason+Act loop)
├── paths.py        # Path resolver + certifi monkey-patch for .app bundle SSL
├── exceptions.py   # OrchestratorError → DAGCycleError, TaskExecutionError, PtyAllocationError
└── constants.py    # Shared constants
```

### Execution Flow

1. `__main__.py` parses args, builds `SwarmConfig`, calls `create_app(config)`, starts Uvicorn
2. `ExecutionEngine.load_manifest()` parses JSON manifest into `AbstractTask` objects, registers in `SwarmDAG`
3. `ExecutionEngine.start_all()` spawns one `asyncio.Task` per job
4. Each task: waits for DAG deps, acquires semaphore, executes via PTY or standard shell
5. All I/O routed through `EventBus` to WebSocket subscribers
6. State transitions validated by `TaskStateMachine` — illegal jumps raise exceptions

### Task Types

- **`PtyInteractiveTask`** — OS-level PTY allocation for TTY-aware tools (vim, gemini, etc.)
- **`StandardShellTask`** — `asyncio.create_subprocess_shell` with pipes (when `use_pty: false`)

---

## CI/CD & Headless Mode

For CI/CD pipelines and environments without a display, bypass the Menu Bar integration:

```bash
# Start daemon headless (no AppKit, no Window Server required)
nohup python3 -m bulk_puppeteer --headless --port 9090 > daemon.log 2>&1 &
```

Then point the CLI to the custom port:

```bash
swarm-cli --port 9090 status
swarm-cli --port 9090 add BUILD "run the build" --deps LINT,TEST
```

---

## Troubleshooting

### Why is my task showing as `DORMANT`?

`DORMANT` is the default state for dynamically added tasks. It means the task is registered but not yet scheduled.

| Scenario | Cause | Fix |
|----------|-------|-----|
| **Default state** | You added the task without `--run` | Hit **Resume** in the Dashboard, or `swarm-cli resume <id>`, or re-add with `--run` |
| **Intentional hold** | Task is parked for later activation | Resume when ready |

### Why does my task fail immediately after resume?

| Scenario | Cause | Fix |
|----------|-------|-----|
| **Exit code 127** | Command not found — the daemon can't locate the executable | Create a `/usr/local/bin` symlink and restart the daemon (see [Environment Setup](#post-install-environment-setup)) |
| **Exit code 126** | Permission denied on the executable | `chmod +x` the binary |
| **Subprocess boot failure** | The daemon can't find the binary (e.g., `gemini`) | Check `swarm-cli logs <id>` for the exit hint |

### Connection Refused / 500 Error

- **Symptom:** `[ERROR] API Request Failed: None`
- **Cause:** The daemon is not running on the expected port.
- **Fix:** Ensure you launched the `.app` or ran `./start_daemon.sh`. Verify with `swarm-cli --port <PORT> status`.

### Authentication Failure

- **Symptom:** `[ERROR] API Request Failed: Unauthorized`
- **Cause:** The `.daemon.token` in your working directory doesn't match the daemon's active token, or the file is missing.
- **Fix:** Run `swarm-cli` from the same directory where the daemon was launched, pass the correct token with `--token`, or set `export SWARM_TOKEN=$(cat .daemon.token)`. Use `--stable-token` on daemon start to reuse the same token across restarts.

### Gemini API Key Mismatch

- **Symptom:** A tray dialog appears on launch comparing two key fingerprints.
- **Cause:** The `GEMINI_API_KEY` environment variable and the key saved in `~/Library/Application Support/BULK_PUPPETEER/` do not match.
- **Fix:** Dismiss the dialog and decide which key is authoritative. Remove the env var to use the saved key, or update the saved key via Menu Bar > **"Set Gemini API Key…"**.

### SSL Errors After DMG Install

- **Symptom:** `httpx` or `google-genai` raises SSL certificate verification errors.
- **Cause:** The `.app` bundle uses a bundled Python that does not have access to the system cert store.
- **Fix:** This is handled automatically by `paths.py` on startup. If errors persist, ensure you are running the installed `.app` and not a manually invoked Python process.

### Task Not Found

- **Symptom:** `[ERROR] Failed to fetch logs for 'xyz'` (empty or 404)
- **Cause:** The task ID doesn't exist in the active DAG.
- **Fix:** Run `swarm-cli status` to see which task IDs are currently loaded.

---

## License

Copyright (c) 2026 Amir Yassin. All rights reserved.

---

```
>>>>>CHANGES<<<<<
- [Role]: Researcher/Doc Auditor
- [Action]: Fixed Quick Start resume curl — changed PUT /api/tasks/{id}?action=resume to POST /api/tasks/resume/{id} to match actual api.py route. (v2026-04-01)
- [Action]: Fixed FSM state diagram — removed fictional WAITING state; correct states are BLOCKED, QUEUED, IN_PROGRESS, COMPLETED, FAILED, KILLED per fsm.py. (v2026-04-01)
- [Action]: Expanded Security Architecture section — added CORS restriction to localhost note, noted bearer token is not logged. (v2026-04-01)
- [Action]: Replaced abbreviated REST API table with complete accurate endpoint list matching all routes registered in api.py (15 endpoints). (v2026-04-01)
- [Role]: Developer (Claude)
- [Action]: v3.8.0 — Updated all documentation for 9 peer review friction points: DORMANT state, --run flag, hot-resize concurrency, SWARM_TOKEN, --stable-token, exit code hints, PTY EIO, shell escaping. Added What's New v3.8.0 section. (v2026-04-01)
- [Action]: v4.4.3 — Updated version header, DMG filename, download URL; added What's New v4.4.3 section (chat history, agent spawning, user identity, SSL fix, API key mismatch, WA auth state machine, unified state dir); added First-Run Setup section; expanded Menu Bar UI list; updated build commands to build_release_cached.sh / PRODUCTION=1 build_release.sh; added paths.py to architecture tree; added Gemini API key mismatch and SSL troubleshooting entries. (v2026-04-10)
```
