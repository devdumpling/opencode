# OpenCode - Private Setup Guide

Personal setup documentation for running OpenCode locally with GitHub Copilot.

## Quick Start

```bash
# 1. Set environment variables (add to ~/.zshrc or ~/.bashrc)
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_DISABLE_AUTOUPDATE=true

# 2. Install dependencies (one-time)
cd /Users/devonwells/development/forks/opencode
bun install

# 3. Run OpenCode in your target project (works like Claude Code)
cd /path/to/your/project
bun --cwd /Users/devonwells/development/forks/opencode/packages/opencode dev
```

## GitHub Copilot Authentication

1. Start OpenCode in any project (see above)
2. Press `/` and type `connect`
3. Select "Login with GitHub Copilot"
4. Select "GitHub.com"
5. Complete OAuth flow in browser
6. Select model with `/model` command (e.g., `github-copilot/claude-sonnet-4`)

Auth tokens are stored in `~/.local/share/opencode/auth.json`

## Environment Variables

| Variable | Purpose | Recommended |
|----------|---------|-------------|
| `OPENCODE_DISABLE_MODELS_FETCH=true` | Disable fetching model definitions from models.dev | Yes |
| `OPENCODE_DISABLE_AUTOUPDATE=true` | Disable auto-update checks | Yes |
| `OPENCODE_DISABLE_LSP_DOWNLOAD=true` | Disable LSP server downloads | No (keep LSP) |

## Code Changes Made (Telemetry Hardening)

The following changes were made to this fork to disable external network calls:

### 1. Share System Disabled
**File:** `packages/opencode/src/project/bootstrap.ts`
- Commented out `Share.init()` and `ShareNext.init()`
- These sync session data to `api.opencode.ai`

### 2. Models.dev Periodic Refresh Disabled
**File:** `packages/opencode/src/provider/models.ts`
- Commented out `setInterval()` that fetches from `models.dev` every 60 minutes
- Still uses bundled/cached model data

### 3. Auto-Upgrade Check Disabled
**File:** `packages/opencode/src/cli/cmd/tui/thread.ts`
- Commented out `checkUpgrade()` call on startup
- This was calling GitHub/npm APIs even with env var set

## Running in Any Project

**OpenCode uses your current working directory** (same as Claude Code).

### Option 1: cd + run (recommended)

```bash
cd /path/to/your/project
bun --cwd /Users/devonwells/development/forks/opencode/packages/opencode dev
```

### Option 2: Pass project path as argument

```bash
bun --cwd /Users/devonwells/development/forks/opencode/packages/opencode dev /path/to/your/project
```

### Option 3: Shell alias (recommended for daily use)

Add to `~/.zshrc` or `~/.bashrc`:

```bash
alias oc='bun --cwd /Users/devonwells/development/forks/opencode/packages/opencode dev'
```

Then use just like Claude Code:
```bash
cd /path/to/your/project
oc
```

Or with a path:
```bash
oc /path/to/your/project
```

## Project-Specific Config (Optional)

Create `.opencode/opencode.json` in your target repo:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "disabled_providers": ["openrouter", "openai"],
  "model": {
    "default": "github-copilot/claude-sonnet-4"
  }
}
```

## Security Notes

### Network Calls Made (After Hardening)

| Endpoint | Purpose | Notes |
|----------|---------|-------|
| `github.com/login/*` | OAuth authentication | Required for Copilot |
| `api.githubcopilot.com` | LLM inference | Required for Copilot |

### Disabled by Code Changes (This Fork)

| Endpoint | What It Did | Status |
|----------|-------------|--------|
| `api.opencode.ai/share_*` | Session sharing/sync | **Removed** in bootstrap.ts |
| `api.dev.opencode.ai/*` | Dev share endpoint | **Removed** in bootstrap.ts |
| `models.dev/api.json` | Model definitions (60min refresh) | **Removed** interval in models.ts |
| `api.github.com/repos/.../releases` | Auto-update check | **Removed** call in thread.ts |

### Still Requires Environment Variables

Even with code changes, set these for defense in depth:
```bash
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_DISABLE_AUTOUPDATE=true
```

### Not Present (Verified)

- No Sentry, PostHog, Segment, Amplitude, or Google Analytics
- No frontend telemetry
- No feature flags or A/B testing
- Honeycomb telemetry is server-side only (Zen API), not in local TUI
- OpenTelemetry is opt-in only (disabled by default)

## Available Copilot Models

After connecting, use `/model` to select from available models:

- `github-copilot/gpt-4o`
- `github-copilot/gpt-4.1`
- `github-copilot/claude-sonnet-4`
- `github-copilot/o3-mini`
- (others depending on your Copilot subscription)

## Features to Avoid

| Feature | Why | How |
|---------|-----|-----|
| OpenCode Zen | Hosted proxy service | Use Copilot directly |
| Session Sharing | External URL generation | Don't use `/share` |
| OpenCode Plus | Not needed | Use Copilot subscription |

## Verification

Check for unwanted network calls:

```bash
# macOS
sudo lsof -i -P | grep opencode

# Verify auth storage
cat ~/.local/share/opencode/auth.json
```

## Key Source Files

- `packages/opencode/src/plugin/copilot.ts` - Copilot OAuth handling
- `packages/opencode/src/provider/provider.ts` - Provider system (lines 831-870 for Copilot)

---

# Architecture Overview

## High-Level Structure

```
┌─────────────────────────────────────────────────────────┐
│                    TUI (terminal UI)                     │
│                   React + @opentui                       │
└────────────────────────┬────────────────────────────────┘
                         │ RPC (not HTTP)
┌────────────────────────▼────────────────────────────────┐
│                   Worker Thread                          │
│              (Hono server, internal only)                │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   ┌─────────┐    ┌───────────┐    ┌───────────┐
   │  Agent  │    │  Session  │    │ Provider  │
   │  System │    │  Manager  │    │  System   │
   └────┬────┘    └─────┬─────┘    └─────┬─────┘
        │               │                │
        └───────►  LLM Stream  ◄─────────┘
                       │
                  ┌────▼────┐
                  │  Tools  │
                  └─────────┘
```

## The Agent Loop

When you send a message:

```
1. User message saved to session
           ↓
2. Build context (history + system prompt)
           ↓
3. Stream to LLM ──────────────────────────────┐
           ↓                                    │
4. LLM returns text OR tool call               │
           ↓                                    │
   ┌───────┴───────┐                           │
   │ Tool call?    │                           │
   └───────┬───────┘                           │
      YES  │  NO                               │
           │   └──► Done (show text)           │
           ▼                                    │
5. Check permissions (allow/deny/ask)          │
           ↓                                    │
6. Execute tool (bash, edit, read, etc.)       │
           ↓                                    │
7. Return result to LLM ───────────────────────┘
           (loop continues until no more tool calls)
```

## Core Components

### Agents (`src/agent/agent.ts`)
Define personas with specific capabilities:
- `build` - Primary coding agent
- `plan` - Planning mode agent
- `explore` - Read-only codebase exploration (subagent)
- `general` - General-purpose tasks (subagent)

Each agent has: name, prompt, model override, permission ruleset, max steps.

### Sessions (`src/session/`)
Container for a conversation. Messages split into **Parts**:
- `TextPart` - Assistant's text response
- `ToolPart` - Tool invocations with state (pending → running → completed)
- `ReasoningPart` - Extended thinking (if model supports)

Key files:
- `session/index.ts` - Session lifecycle
- `session/prompt.ts` - User input handling, agent loop
- `session/processor.ts` - Stream event handling
- `session/llm.ts` - LLM streaming, tool resolution

### Tools (`src/tool/`)
| Tool | Purpose |
|------|---------|
| `bash` | Shell commands (sandboxed) |
| `read` | File reading |
| `edit` | File editing (tree-sitter) |
| `write` | Create new files |
| `glob` | Find files by pattern |
| `grep` | Search file contents |
| `webfetch` | Fetch URLs |
| `task` | Spawn subagents |

### Providers (`src/provider/provider.ts`)
Abstraction over 26+ LLM backends via Vercel AI SDK.
Handles auth, model capabilities, cost tracking.

### Permissions (`src/permission/next.ts`)
Rules: `{ permission: "bash", pattern: "*", action: "allow|deny|ask" }`

Hierarchy: system defaults → user config → agent config → session overrides

## System Prompt Architecture

Prompts are assembled in layers (`src/session/llm.ts:68-94`):

```
1. Provider Header        ← Cache-friendly prefix
         +
2. Agent/Provider Prompt  ← Main instructions
         +
3. Custom system prompts  ← Per-call overrides
         +
4. Environment context    ← Working dir, platform, date
         +
5. Custom instructions    ← AGENTS.md, CLAUDE.md files
```

### Model-Specific Prompts (`src/session/system.ts`)

| Model | File | Style |
|-------|------|-------|
| Claude | `prompt/anthropic.txt` | Measured, TodoWrite emphasis |
| GPT-4/o1/o3 | `prompt/beast.txt` | Aggressive autonomy |
| Gemini | `prompt/gemini.txt` | Gemini-tailored |
| Others | `prompt/qwen.txt` | Generic fallback |

### Custom Instructions

Add your own via:
- **Project**: `AGENTS.md` or `CLAUDE.md` in repo root
- **Global**: `~/.config/opencode/AGENTS.md`
- **Config**: `instructions` array in `opencode.json`

## Data Flow

```
User types message
       ↓
TUI React component
       ↓
RPC call to worker
       ↓
SessionPrompt.command()
       ↓
Build messages + system prompt
       ↓
LLM.stream() → Provider SDK → API
       ↓
Stream events back
       ↓
SessionProcessor handles events
       ↓
Tool execution (if needed)
       ↓
Update message parts
       ↓
Publish events via Bus
       ↓
TUI re-renders
```

## Storage

- **Location**: `~/.local/share/opencode/` (global) or `.opencode/` (project)
- **Format**: JSON files keyed by `["session", projectID, sessionID]`
- **Auth**: `~/.local/share/opencode/auth.json`

## Key Architectural Decisions

1. **Worker-based** - Heavy lifting in worker thread, TUI stays responsive
2. **RPC not HTTP** - Direct IPC, no network exposure by default
3. **Event-driven** - Bus system for loose coupling
4. **Stream-based** - Incremental output for responsive UX
5. **Permission-first** - Fine-grained control at every level
6. **Provider-agnostic** - Same interface for all LLM backends
