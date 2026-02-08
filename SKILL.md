---
name: browser-agent-server
description: Deploy a full-stack multi-model Browser Agent system with FastAPI server, real-time dashboard, VNC streaming, and LLM Council mode. Use when the user asks to set up browser automation, build a browser agent, deploy an AI web agent, create a browser-use server, or needs multi-model browser automation with strategies like council, consensus, fallback chain, or planner-executor.
---

# Browser Agent Server

Full-stack AI browser automation system: FastAPI backend + real-time dashboard + Xvfb/VNC display + multi-model LLM strategies.

## Architecture

```
agent_server.py  (FastAPI + WebSocket + browser-use 0.11.9)
dashboard.html   (single-file SPA: dark UI, live logs, VNC embed, model picker)
start.sh         (startup script with prerequisite checks)
```

**5 Strategies**: single, fallback_chain, planner_executor, consensus (per-step judge), council (multi-model failure recovery with loop detection)

**Display**: Xvfb :99 -> x11vnc :5999 -> noVNC/websockify :6080

## Setup

### 1. Install system dependencies

```bash
sudo apt-get update && sudo apt-get install -y xvfb x11vnc x11-apps imagemagick novnc
```

### 2. Create Python venv and install packages

```bash
python3 -m venv /home/node/browser-agent-venv
/home/node/browser-agent-venv/bin/pip install browser-use==0.11.9 fastapi uvicorn[standard] websockets
/home/node/browser-agent-venv/bin/python3 -m playwright install chromium
```

### 2b. CRITICAL: Install Chromium shared library dependencies

Without this step, Chromium will fail with `libatk-1.0.so.0: cannot open shared object file` or similar errors, causing a 30-second timeout on browser launch.

```bash
/home/node/browser-agent-venv/bin/python3 -m playwright install-deps chromium
```

This installs ~40 system libraries (libatk, libasound, libxkbcommon, fonts, etc.) that Chromium requires at runtime. **This is separate from `playwright install chromium`** which only downloads the browser binary.

### 3. Deploy application files

Copy bundled scripts to a project directory:

```bash
DEST="./outputs/browser-agent"
mkdir -p "$DEST"
cp ~/.claude/skills/browser-agent-server/scripts/agent_server.py "$DEST/"
cp ~/.claude/skills/browser-agent-server/scripts/dashboard.html "$DEST/"
cp ~/.claude/skills/browser-agent-server/scripts/start.sh "$DEST/"
chmod +x "$DEST/start.sh"
```

### 4. Configure environment

```bash
# Required: LLM API key (OpenAI-compatible gateway)
export AI_GATEWAY_API_KEY="your-key"

# Optional: default model
export BROWSER_AGENT_MODEL="openai/gpt-4o"

# Optional: custom port
export AGENT_PORT=8888
```

### 5. Start

```bash
cd "$DEST"
DISPLAY=:99 /home/node/browser-agent-venv/bin/python3 agent_server.py
# Or use start.sh for full prerequisite checks:
# ./start.sh
```

### 6. Export ports (sandbox environments)

```bash
/app/export-port.sh $AGENT_PORT   # Dashboard (default 8888, or whatever you set)
/app/export-port.sh 6080          # noVNC live view
```

**Note**: Port 3001 is reserved. Do not use it. If port 8888 is already in use, set `AGENT_PORT` to another value (e.g., 9222).

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Dashboard HTML |
| `/api/models` | GET | Available models + strategies |
| `/api/agent/start` | POST | Start task (JSON body below) |
| `/api/agent/stop` | POST | Stop running task |
| `/api/agent/status` | GET | Current status, action_log, result |
| `/ws` | WebSocket | Real-time updates (step, status, screenshot, judge_verdict, council_verdict) |

### Start task body

```json
{
  "task": "Go to google.com and search for AI",
  "max_steps": 50,
  "model_config_data": {
    "strategy": "council",
    "primary_model": "openai/gpt-4o",
    "secondary_model": "",
    "council_members": ["moonshotai/kimi-k2.5", "google/gemini-2.5-flash", "google/gemini-2.5-pro"]
  }
}
```

### WebSocket start (from dashboard)

```json
{
  "type": "start_task",
  "task": "...",
  "max_steps": 50,
  "model_config": { "strategy": "council", "primary_model": "openai/gpt-4o", "council_members": [...] }
}
```

## Strategy Details

| Strategy | How it works | When to use |
|----------|-------------|-------------|
| `single` | One model, all steps | Simple tasks, cost-sensitive |
| `fallback_chain` | Primary runs; switches to secondary on error/rate-limit | Reliability |
| `planner_executor` | Strong model plans first; fast model executes | Complex multi-step |
| `consensus` | Primary acts; judge model validates every step in real-time | Quality-critical |
| `council` | Primary runs; on repeated failure/loop/stall, ALL council models convene to diagnose, advise, and replan | Hard tasks, anti-stall |

### Council Mode Details

- **Failure trigger**: `consecutive_failures >= 2`
- **Loop trigger (3-tier)**: Strict fingerprint match (3 repeats), loose action-type match (4 repeats), same-URL stall with no progress (5 repeats)
- **Stall trigger**: Single step running > 60 seconds
- **Feedback injection**: Council verdict injected via `ActionResult.long_term_memory` (agent sees it next step)
- **Replan**: Council can replace `agent.state.plan` with revised steps
- **Cooldown**: 3 steps between loop-triggered councils to prevent meta-loops

## Available Models (AI Gateway)

Configure in `AVAILABLE_MODELS` list in agent_server.py:

```python
AVAILABLE_MODELS = [
    {"id": "openai/gpt-4o", "name": "GPT-4o", "tier": "fast", "vision": True},
    {"id": "moonshotai/kimi-k2.5", "name": "Kimi K2.5", "tier": "fast", "vision": True},
    {"id": "google/gemini-2.5-flash", "name": "Gemini 2.5 Flash", "tier": "fast", "vision": True},
    {"id": "google/gemini-2.5-pro", "name": "Gemini 2.5 Pro", "tier": "reasoning", "vision": True},
]
```

To add models: add to this list and they appear in dashboard dropdown + available as council members.

## Troubleshooting

### Browser launch timeout (`BrowserStartEvent timed out after 30.0s`)

Chromium is missing shared libraries. Fix:

```bash
/home/node/browser-agent-venv/bin/python3 -m playwright install-deps chromium
```

This installs libatk, libasound, libxkbcommon, fonts, etc. **Must run after `playwright install chromium`.**

### Verify Chromium works

```bash
DISPLAY=:99 /home/node/.cache/ms-playwright/chromium-*/chrome-linux64/chrome --version
```

If it prints a version, it's working. If it errors with `cannot open shared object file`, run `install-deps` above.

### Xvfb lock file error (`Server is already active for display :99`)

```bash
rm -f /tmp/.X99-lock
pkill -9 Xvfb
```

### Port already in use

```bash
export AGENT_PORT=9222   # or any free port (avoid 3001 - reserved)
```

### noVNC not loading

Ensure `novnc` system package is installed (`/usr/share/novnc/` must exist):

```bash
sudo apt-get install -y novnc
```

## Key Implementation Notes

- `browser-use` ChatOpenAI returns `ChatInvokeCompletion` with `.completion` field (NOT `.content`)
- `agent.state.plan` is mutable from `on_step_end` hook -- changes affect next step
- `ActionResult.long_term_memory` gets injected into next step's context via MessageManager
- `agent.state.consecutive_failures` tracks errors; reset on success
- The `on_step_end` hook signature: `AgentHookFunc = Callable[['Agent'], Awaitable[None]]`
- Dashboard is a single HTML file with inline CSS/JS (no build step)
- noVNC served from system install at `/usr/share/novnc/`
