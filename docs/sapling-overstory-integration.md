# Sapling × Overstory Integration Spec

> Comprehensive design for integrating Sapling as an agent runtime in Overstory — from initial adapter to becoming THE default worker runtime.

**Status:** Draft
**Date:** 2026-03-03
**Authors:** Orchestrator session

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Vision & Motivation](#2-vision--motivation)
3. [Architecture Overview](#3-architecture-overview)
4. [Phase 1: Runtime Adapter](#4-phase-1-runtime-adapter)
5. [Phase 2: Sapling-Native Hook System](#5-phase-2-sapling-native-hook-system)
6. [Phase 3: Native Ecosystem Integration](#6-phase-3-native-ecosystem-integration)
7. [Phase 4: Sapling as THE Default Worker Runtime](#7-phase-4-sapling-as-the-default-worker-runtime)
8. [Phase 5: Orchestrator Context Optimization](#8-phase-5-orchestrator-context-optimization)
9. [Technical Specifications](#9-technical-specifications)
10. [Migration Path](#10-migration-path)
11. [Open Questions](#11-open-questions)

---

## 1. Executive Summary

Sapling is a headless coding agent with proactive inter-turn context management. Overstory is a multi-agent orchestration system that spawns coding agents in git worktrees. Today, Overstory manages agents through tmux sessions — spawning Claude Code, Pi, Codex, Copilot, or Gemini in terminal panes and monitoring them via tmux capture-pane.

Sapling was built to avoid tmux entirely. It runs as a direct subprocess, manages its own context window, and communicates via structured NDJSON events. This makes it uniquely suited for fleet operations where dozens of agents run in parallel.

**The integration is phased:**

| Phase | What | Why |
|-------|------|-----|
| 1 | Runtime adapter (`--runtime sapling`) | Sapling is one option among many |
| 2 | Sapling-native hook system | Guards enforced inside sapling, configured by overstory |
| 3 | Native ecosystem integration | Direct mail.db/metrics.db/mulch/seeds access |
| 4 | Sapling as THE default worker runtime | Optimal for fleet ops by default |
| 5 | Orchestrator context optimization | Apply sapling's context techniques to the orchestrator itself |

Each phase builds on the previous. Phase 1 is the minimum viable integration. Phase 4 is the strategic goal.

---

## 2. Vision & Motivation

### The Problem with Current Runtimes

Overstory's existing runtimes (Claude Code, Pi, Codex, Copilot, Gemini) were built for **human-interactive** use, not orchestration. Using them for fleet operations requires:

- **Tmux management** — ~500 lines of session creation, pane capturing, keystroke injection, readiness detection
- **Hook/guard shims** — ~600+ lines of settings.local.json or guard extension generation to enforce security boundaries
- **Beacon verification** — Retransmit the initial prompt because TUIs sometimes swallow keystrokes during initialization
- **Transcript scraping** — Parse runtime-specific JSONL formats to extract token usage after the fact
- **Overlay text for enforcement** — Prose instructions ("you MUST NOT edit files outside your scope") instead of code-level enforcement

Each runtime adapter is ~200–400 lines of integration glue to bridge the gap between a human-facing tool and an orchestration system.

### What Sapling Changes

Sapling is **orchestration-native**:

| Concern | Current Runtimes | Sapling |
|---------|-----------------|---------|
| Process model | TUI in tmux pane | Direct subprocess, no tmux |
| Readiness | Scrape tmux pane for prompt indicators | Process start = ready, or NDJSON `started` event |
| Communication | Mail + tmux send-keys nudge | Mail (direct SQLite reads between turns) |
| Monitoring | tmux capture-pane | NDJSON event stream on stdout |
| Guards | External hooks/extensions | Native hook system inside sapling |
| Token tracking | Parse transcript files after completion | Real-time token usage in NDJSON events |
| Context management | Runtime-internal (opaque, uncontrollable) | Proactive 5-stage pipeline (measure → score → prune → archive → reshape) |
| Memory footprint | 250–400 MB per agent (TUI overhead) | 60–120 MB per agent (headless) |
| Billing | Varies (subscription for CC, per-token for SDK) | Per-token (Anthropic SDK direct) |

### The Strategic Arc

1. **Phase 1:** Sapling is *another* runtime option — `ov sling task-123 --runtime sapling --name alice`
2. **Phase 4:** Sapling is *THE* runtime — worker agents default to sapling; Claude Code/Pi reserved for interactive coordinators
3. **End state:** Two-tier execution model:
   - **Coordinator tier:** Claude Code or Pi (TUI, human-interactive)
   - **Worker tier:** Sapling (headless, lightweight, context-managed, scales to 20+ agents on 16 GB RAM)

---

## 3. Architecture Overview

### Current Overstory Agent Lifecycle

```
ov sling task-123 --capability builder --name alice
  │
  ├─ 1. Load config + manifest
  ├─ 2. Validate (hierarchy, name, concurrency limits)
  ├─ 3. Create git worktree → .overstory/worktrees/alice/
  ├─ 4. Generate overlay (CLAUDE.md with task assignment)
  ├─ 5. runtime.deployConfig() → write instructions + hooks/guards
  ├─ 6. Claim task in issue tracker
  ├─ 7. Create agent identity
  ├─ 8. ensureTmuxAvailable() ◄── TMUX DEPENDENCY
  ├─ 9. createSession() → tmux new-session running spawn command ◄──
  ├─ 10. waitForTuiReady() → poll tmux capture-pane ◄──
  ├─ 11. sendKeys() → inject beacon message via tmux ◄──
  ├─ 12. Record session in SessionStore (tmuxSession, pid)
  └─ 13. Return AgentSession
```

Steps 8–11 are tmux-specific. For sapling, these become:

```
  ├─ 8. (skip — no tmux needed)
  ├─ 9. Bun.spawn("sp", "run", ...) → direct subprocess
  ├─ 10. (skip — process starts immediately)
  ├─ 11. (skip — task is the prompt, no beacon injection needed)
```

### Proposed Architecture

```
                    ┌─────────────────────────────┐
                    │      Orchestrator            │
                    │   (Claude Code session)      │
                    └──────────┬──────────────────┘
                               │ ov sling
                    ┌──────────┴──────────────────┐
                    │     AgentRuntime dispatch     │
                    └──┬──────┬──────┬──────┬─────┘
                       │      │      │      │
              ┌────────┘  ┌───┘  ┌───┘  ┌───┘
              ▼           ▼      ▼      ▼
         ┌─────────┐ ┌──────┐ ┌────┐ ┌────────┐
         │ Sapling  │ │Claude│ │ Pi │ │Codex/  │
         │ (new)    │ │ Code │ │    │ │Gemini/ │
         │          │ │      │ │    │ │Copilot │
         └────┬─────┘ └──┬───┘ └─┬──┘ └───┬────┘
              │          │       │         │
         No tmux      Tmux    Tmux     Tmux
         Direct       pane    pane     pane
         subprocess
         NDJSON out
         Native hooks
```

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **No tmux for sapling** | Sapling is headless — tmux adds overhead and complexity with zero benefit. Process management via Bun.spawn directly. |
| **NDJSON event stream** | Sapling's `--json` flag emits structured events on stdout. Overstory reads these for monitoring, token tracking, and completion detection. |
| **Sapling-native hooks** | Guards are enforced *inside* sapling's tool registry, not via external hook injection. Overstory provides a guard config file that sapling reads at startup. |
| **Task-as-prompt** | The overlay content (task assignment, file scope, quality gates) becomes the prompt passed to `sp run`. No beacon injection needed. |
| **AgentSession.tmuxSession optional** | For sapling agents, `tmuxSession` is empty and `pid` is the process ID. Existing overstory monitoring code must handle this. |

---

## 4. Phase 1: Runtime Adapter

**Goal:** `ov sling task-123 --runtime sapling --name alice` works end-to-end.

### 4.1 New File: `overstory/src/runtimes/sapling.ts`

Implements `AgentRuntime` interface. ~200–300 lines.

```typescript
export class SaplingRuntime implements AgentRuntime {
  readonly id = "sapling";
  readonly instructionPath = "SAPLING.md";  // or configurable

  buildSpawnCommand(opts: SpawnOpts): string {
    // Returns: sp run "<prompt>" --backend sdk --model <model> --json --cwd <cwd>
    // --system-prompt-file <path> for the agent persona/overlay
  }

  buildPrintCommand(prompt: string, model?: string): string[] {
    // Returns: ["sp", "run", prompt, "--backend", "sdk", ...]
    // For one-shot AI calls (merge resolver, watchdog triage)
  }

  async deployConfig(
    worktreePath: string,
    overlay: OverlayContent | undefined,
    hooks: HooksDef
  ): Promise<void> {
    // 1. Write overlay to SAPLING.md (system prompt file)
    // 2. Write guard config to .sapling/guards.json (Phase 2)
    // Phase 1: overlay only, no guards
  }

  detectReady(_paneContent: string): ReadyState {
    return { phase: "ready" };  // Headless — always ready
  }

  requiresBeaconVerification(): boolean {
    return false;  // No TUI, no beacon needed
  }

  async parseTranscript(path: string): Promise<TranscriptSummary | null> {
    // Parse sapling NDJSON output (emitted via --json flag)
    // Look for "run" events with token counts
  }

  buildEnv(model: ResolvedModel): Record<string, string> {
    // ANTHROPIC_API_KEY for SDK backend
    // SAPLING_BACKEND=sdk (or configurable)
    // SAPLING_MODEL=<model>
    return { ...model.env };
  }
}
```

### 4.2 Registry Update: `overstory/src/runtimes/registry.ts`

Add sapling to the runtime map:

```typescript
["sapling", () => new SaplingRuntime()]
```

### 4.3 Spawn Path Changes: `overstory/src/commands/sling.ts`

The critical change: **bypass tmux for headless runtimes.**

Current flow (steps 8–11) calls:
- `ensureTmuxAvailable()`
- `createSession()` — tmux new-session
- `waitForTuiReady()` — poll tmux capture-pane
- `sendKeys()` — inject beacon

For sapling, this becomes:

```typescript
if (runtime.headless) {
  // Direct subprocess — no tmux
  const proc = Bun.spawn(["sp", "run", ...args], {
    cwd: worktreePath,
    stdout: "pipe",  // NDJSON events
    stderr: "pipe",  // Logs
    env: { ...process.env, ...runtimeEnv },
  });

  // Store PID, no tmux session
  session.pid = proc.pid;
  session.tmuxSession = "";

  // Redirect stdout to log file for later analysis
  // (or pipe to event ingestion)
  pipeToLogFile(proc.stdout, logPath);
  pipeToLogFile(proc.stderr, logPath);
} else {
  // Existing tmux path (unchanged)
  await ensureTmuxAvailable();
  const pid = await createSession(tmuxSessionName, worktreePath, spawnCmd, { ... });
  await waitForTuiReady(tmuxSessionName, ...);
  await sendKeys(tmuxSessionName, beacon);
}
```

**AgentRuntime interface addition** (non-breaking):

```typescript
export interface AgentRuntime {
  // ... existing methods ...

  /** Whether this runtime is headless (no tmux, direct subprocess). */
  readonly headless?: boolean;

  /**
   * Build the argv + options for direct subprocess spawn.
   * Only called when headless is true. Returns null to fall back to tmux.
   */
  buildDirectSpawn?(opts: SpawnOpts): {
    argv: string[];
    env?: Record<string, string>;
    stdin?: "pipe" | "ignore";
  };
}
```

Sapling implements `headless = true` and `buildDirectSpawn()`. All existing runtimes are unaffected (`headless` defaults to `undefined`/falsy).

### 4.4 Process Management

Without tmux, overstory needs to manage sapling processes directly:

| Concern | Tmux Model | Direct Subprocess Model |
|---------|-----------|------------------------|
| **Start** | `tmux new-session -d -s <name>` | `Bun.spawn(argv, { ... })` |
| **Stop** | `tmux kill-session -t <name>` | `process.kill(pid, "SIGTERM")` |
| **Monitor** | `tmux capture-pane -t <name>` | Read NDJSON from piped stdout |
| **Status** | `tmux has-session -t <name>` | Check if PID is alive |
| **Output** | `tmux capture-pane -p` | Read log file or NDJSON events |

**New file: `overstory/src/worktree/process.ts`** (~100 lines)

Subprocess management for headless runtimes:

```typescript
export async function spawnDirect(
  argv: string[],
  cwd: string,
  env: Record<string, string>,
  logDir: string,
): Promise<{ pid: number; proc: Subprocess }>;

export function isProcessAlive(pid: number): boolean;

export function killProcess(pid: number, signal?: string): boolean;

export async function pipeToFile(
  stream: ReadableStream<Uint8Array>,
  path: string,
): Promise<void>;
```

### 4.5 Monitoring Adaptation

Overstory's `ov status`, `ov dashboard`, `ov inspect`, and `ov feed` currently rely on:
- `tmux has-session` for liveness checks
- `tmux capture-pane` for output inspection
- Session store `tmuxSession` field for lookups

For sapling agents:
- **Liveness:** Check PID existence (`kill -0 <pid>`)
- **Output:** Read NDJSON log file (tail last N events)
- **Status:** Parse last NDJSON event for current state

The session store already has a `pid` field. The existing code paths that check `tmuxSession` need a guard:

```typescript
if (session.tmuxSession) {
  // tmux-based monitoring
} else if (session.pid) {
  // PID-based monitoring (headless)
}
```

### 4.6 System Prompt / Overlay Strategy

Current overstory flow:
1. Generate overlay content (markdown with task assignment, file scope, quality gates)
2. Write to `runtime.instructionPath` (e.g., `.claude/CLAUDE.md`)
3. Runtime reads it on startup

For sapling:
1. Generate overlay content (same as today)
2. Write to `SAPLING.md` in worktree root
3. Pass as `--system-prompt-file SAPLING.md` to `sp run`
4. The task prompt (first user message) is a concise directive: "Read SAPLING.md for your task assignment and begin immediately."

Alternatively, the overlay content could be the **prompt itself** (passed directly as the positional arg to `sp run`). But `--system-prompt-file` is better because:
- It separates system instructions from the task prompt
- It persists in the worktree for debugging/inspection
- It aligns with sapling's existing `--system-prompt-file` flag

### 4.7 Sapling Output / NDJSON Event Protocol

Sapling currently emits two NDJSON events (with `--json`):

```jsonl
{"success":true,"command":"response","response":"Final text from agent"}
{"success":true,"command":"run","exitReason":"task_complete","totalTurns":15,"inputTokens":45000,"outputTokens":12000}
```

**Phase 1 needs:** These two events are sufficient for:
- **Completion detection:** Process exits, read final NDJSON line
- **Token tracking:** Extract `inputTokens` / `outputTokens` from `run` event
- **Exit reason:** `task_complete`, `max_turns`, `error`

**Future phases will extend** the NDJSON protocol (see [Section 9.2](#92-ndjson-event-protocol-evolution)).

### 4.8 Completion & Cleanup

When a sapling agent finishes:
1. Process exits with code 0 (success) or 1 (error)
2. Overstory detects exit (Bun.spawn's `proc.exited` promise resolves)
3. Parse NDJSON log for final token counts
4. Update session store: `state = "completed"` or `"failed"`
5. Worktree remains for merge

**Detection options:**
- **Option A (passive):** Overstory's watchdog periodically checks PID liveness
- **Option B (active):** Await `proc.exited` in a background task, fire completion callback
- **Recommended: Option B** — Register `proc.exited.then(handleCompletion)` at spawn time. Immediate detection, no polling.

### 4.9 Scope of Phase 1

**Included:**
- `SaplingRuntime` adapter implementing `AgentRuntime`
- `headless` flag + `buildDirectSpawn()` on AgentRuntime interface
- Direct subprocess management in `sling.ts` (bypass tmux when headless)
- `process.ts` module for PID management
- NDJSON log capture to `.overstory/logs/<agent>/`
- `ov stop` support (SIGTERM to PID)
- Token tracking from NDJSON events
- `ov status` / `ov dashboard` handle tmux-less agents

**Excluded (deferred):**
- Guard/hook system (Phase 2)
- Direct mail.db reads from sapling (Phase 3)
- Direct metrics.db writes from sapling (Phase 3)
- Default runtime selection (Phase 4)
- Per-turn NDJSON event streaming (Phase 2+)

---

## 5. Phase 2: Sapling-Native Hook System

**Goal:** Overstory-configured guards enforced inside sapling's tool execution pipeline.

### 5.1 Design Principles

1. **Guards are code, not prose** — TypeScript functions that inspect tool calls and block/allow them, not CLAUDE.md instructions that the LLM might ignore.
2. **Overstory configures, sapling enforces** — Overstory writes a guard config file; sapling reads it at startup and applies it to every tool call.
3. **Event emission** — Every tool call emits a structured event (pre-call and post-call) that overstory can consume for monitoring.
4. **No runtime performance penalty** — Guards are synchronous checks, not network calls or subprocess invocations.

### 5.2 Guard Config File

Overstory writes `.sapling/guards.json` to the worktree during `deployConfig()`:

```json
{
  "version": 1,
  "agentName": "alice",
  "capability": "builder",
  "worktreePath": "/path/to/.overstory/worktrees/alice",
  "rules": {
    "fileScope": ["src/auth.ts", "src/auth.test.ts"],
    "readOnly": false,
    "blockedTools": [],
    "blockedBashPatterns": [
      "git push", "git checkout", "rm -rf",
      "curl", "wget", "npm publish"
    ],
    "pathBoundary": "/path/to/.overstory/worktrees/alice",
    "allowedPaths": ["/path/to/.overstory/worktrees/alice"]
  },
  "qualityGates": [
    { "command": "bun test", "description": "Tests pass" },
    { "command": "bun run lint", "description": "Lint clean" }
  ],
  "events": {
    "emitToolCalls": true,
    "emitFile": ".sapling/events.jsonl",
    "logToolArgs": true
  }
}
```

### 5.3 Sapling Hook Architecture

New module in sapling: `src/hooks/`

```
sapling/src/hooks/
  index.ts          # HookManager: load config, register pre/post hooks
  guards.ts         # Guard implementations (path boundary, bash patterns, etc.)
  events.ts         # Event emitter (NDJSON to file or stdout)
  types.ts          # Hook types (HookConfig, GuardResult, ToolEvent)
```

**Hook execution flow (integrated into tool dispatch in `loop.ts`):**

```
LLM returns tool_use blocks
  │
  ├─ For each tool call:
  │   ├─ hookManager.preToolCall(toolName, input)
  │   │   ├─ Check guards (path boundary, file scope, blocked patterns)
  │   │   ├─ If blocked: return { blocked: true, reason: "..." }
  │   │   └─ Emit pre-call event to NDJSON
  │   │
  │   ├─ If blocked: return blocked message as tool_result (no execution)
  │   │
  │   ├─ tool.execute(input, cwd)
  │   │
  │   └─ hookManager.postToolCall(toolName, input, result)
  │       ├─ Emit post-call event to NDJSON
  │       └─ Track activity (file modifications, etc.)
  │
  └─ Continue with tool results
```

### 5.4 Guard Types

| Guard | Applies to | Behavior |
|-------|-----------|----------|
| **Path boundary** | write, edit, bash | Block writes outside worktree path |
| **File scope** | write, edit | Only allow edits to files in `fileScope` list |
| **Read-only mode** | write, edit, bash (write commands) | Block all modifications (for scouts/reviewers) |
| **Blocked bash patterns** | bash | Regex match against command string |
| **Blocked tools** | any | Completely disable specific tools |

### 5.5 Event Protocol

Every tool call emits a structured event:

```jsonl
{"ts":"2026-03-03T10:15:00Z","type":"tool_call","agent":"alice","tool":"edit","args":{"file":"src/auth.ts"},"blocked":false,"turn":5}
{"ts":"2026-03-03T10:15:01Z","type":"tool_result","agent":"alice","tool":"edit","success":true,"turn":5}
{"ts":"2026-03-03T10:15:02Z","type":"tool_call","agent":"alice","tool":"bash","args":{"command":"git push"},"blocked":true,"reason":"blocked pattern: git push","turn":5}
```

Overstory reads these events via:
- `ov inspect alice` — tail the event file
- `ov feed` — aggregate events across agents
- `ov trace alice` — chronological timeline

### 5.6 Sapling Changes Required

| File | Change |
|------|--------|
| `src/types.ts` | Add `HookConfig`, `GuardResult`, `ToolEvent` types |
| `src/hooks/index.ts` | New: `HookManager` class (load config, pre/post dispatch) |
| `src/hooks/guards.ts` | New: Guard implementations |
| `src/hooks/events.ts` | New: NDJSON event emitter |
| `src/loop.ts` | Integrate `hookManager.preToolCall()` / `postToolCall()` into tool dispatch |
| `src/cli.ts` | Load guard config from `.sapling/guards.json` if present |
| `src/config.ts` | Add `guardsFile` config option |

### 5.7 Overstory Changes Required

| File | Change |
|------|--------|
| `src/runtimes/sapling.ts` | `deployConfig()` writes `.sapling/guards.json` |
| `src/agents/guard-rules.ts` | Export guard rules as JSON-serializable config (not bash patterns) |

---

## 6. Phase 3: Native Ecosystem Integration

**Goal:** Sapling agents directly read/write overstory databases and call ecosystem tools, eliminating subprocess overhead.

### 6.1 Direct Mail Access

Current model: Agents send mail via `ov mail send --to orchestrator --body "done"` (subprocess).

Native model: Sapling opens `.overstory/mail.db` directly via `bun:sqlite` between turns.

```typescript
// In sapling's agent loop, between turns:
const unread = mailDb.checkUnread(agentName);
if (unread.length > 0) {
  // Inject mail as a user message
  messages.push({
    role: "user",
    content: formatMailMessages(unread)
  });
}
```

**Benefits:**
- ~1ms SQLite read vs ~200ms subprocess spawn
- No shell escaping issues
- Bidirectional: agent can send replies in the same transaction

**New sapling module: `src/ecosystem/mail.ts`** (~80 lines)
- Opens `mail.db` (path from guard config or env var)
- `checkUnread(agentName)` — read unread messages
- `send(from, to, subject, body, type)` — insert message
- `markRead(id)` — mark as read
- Called between turns in the agent loop

### 6.2 Direct Metrics Writes

Current model: Token usage scraped from transcripts after completion.

Native model: Sapling writes to `.overstory/metrics.db` after each LLM call.

```typescript
// After each LLM response in loop.ts:
metricsDb.record({
  agentName,
  inputTokens: response.usage.inputTokens,
  outputTokens: response.usage.outputTokens,
  model: response.model,
  timestamp: new Date().toISOString(),
  turn: currentTurn,
});
```

**Benefits:**
- Real-time cost visibility in `ov costs --live`
- Per-turn granularity (not just totals)
- No transcript parsing needed

**New sapling module: `src/ecosystem/metrics.ts`** (~50 lines)

### 6.3 Direct Mulch Access

Current model: `mulch prime` called as subprocess during agent priming.

Native model: Sapling uses `@os-eco/mulch-cli` programmatic API (already an npm dep of overstory).

```typescript
import { prime } from "@os-eco/mulch-cli";
const expertise = await prime({ domains: config.mulchDomains, files: config.fileScope });
// Inject into system prompt
```

**Benefits:**
- ~10ms vs ~500ms subprocess
- Structured output (not stdout scraping)
- Can query mid-session (not just at startup)

### 6.4 Direct Seeds Access

Current model: `sd update <id> --status in_progress` as subprocess.

Native model: Direct JSONL file operations (seeds stores issues as `.seeds/issues.jsonl`).

```typescript
// On task claim:
seedsDb.updateStatus(taskId, "in_progress");

// On completion:
seedsDb.updateStatus(taskId, "done");
seedsDb.close(taskId, { reason: summary });
```

### 6.5 Configuration

Ecosystem paths are passed to sapling via environment variables or guard config:

```json
{
  "ecosystem": {
    "mailDb": "/path/to/.overstory/mail.db",
    "metricsDb": "/path/to/.overstory/metrics.db",
    "mulchDir": "/path/to/.mulch",
    "seedsDir": "/path/to/.seeds"
  }
}
```

### 6.6 Agent Loop with Ecosystem

The full agent loop with native ecosystem integration:

```
while turns < maxTurns:
  // 1. Check mail (direct SQLite read, ~1ms)
  mail = ecosystem.mail.checkUnread(agentName)
  if mail: inject as user message

  // 2. LLM call
  response = await client.call(request)

  // 3. Record metrics (direct SQLite write, ~1ms)
  ecosystem.metrics.record(agentName, response.usage)

  // 4. Extract tool calls
  if no tool calls: break (task complete)

  // 5. Execute tools with guards
  results = await Promise.all(toolCalls.map(call => {
    guard = hooks.preToolCall(call)
    if guard.blocked: return blockedResult(guard.reason)
    result = await tool.execute(call.input, cwd)
    hooks.postToolCall(call, result)
    return result
  }))

  // 6. Context management pipeline
  messages = contextManager.process(messages, response.usage, currentFiles)

  // 7. Update task status if needed
  if shouldUpdateStatus: ecosystem.seeds.update(taskId, status)
```

---

## 7. Phase 4: Sapling as THE Default Worker Runtime

**Goal:** Overstory defaults to sapling for worker agents. Claude Code/Pi reserved for coordinators and interactive roles.

### 7.1 Config-Driven Runtime Selection

In `.overstory/config.yaml`:

```yaml
runtime:
  default: sapling          # Changed from "claude"
  coordinator: claude       # Interactive roles stay on Claude Code
  lead: claude              # Leads may need interactive debugging
  builder: sapling          # Workers use sapling
  scout: sapling
  reviewer: sapling
  merger: sapling

  sapling:
    backend: sdk            # Direct API calls
    model: claude-sonnet-4-6
    contextWindow: 200000
    maxTurns: 200
```

### 7.2 Manifest Changes

The agent manifest maps capabilities to default runtimes:

```json
{
  "agents": {
    "builder": {
      "base": "agents/builder.md",
      "runtime": "sapling",
      "model": "sonnet",
      "depth": 2,
      "canSpawn": false
    },
    "scout": {
      "base": "agents/scout.md",
      "runtime": "sapling",
      "model": "haiku",
      "depth": 2,
      "canSpawn": false
    }
  }
}
```

### 7.3 Migration Strategy

- **Opt-in first:** Users explicitly use `--runtime sapling`
- **Config override:** Set `runtime.default: sapling` in config.yaml
- **Default flip:** After proving stability, change the hardcoded default from `"claude"` to `"sapling"`
- **Escape hatch:** `--runtime claude` always available for any agent

### 7.4 What Changes for the Orchestrator

When sapling is the default worker runtime:

1. **No tmux required** for worker agents — only coordinators need tmux
2. **Lower resource usage** — 20 sapling workers ≈ 1.2–2.4 GB vs 5–8 GB for Claude Code workers
3. **Faster spawn** — No TUI initialization, no beacon verification, no readiness polling
4. **Better monitoring** — Real-time NDJSON events vs tmux capture-pane scraping
5. **Precise cost tracking** — Per-turn token data vs post-hoc transcript scraping
6. **Proactive context management** — Agents maintain quality over long sessions instead of degrading

### 7.5 Sapling Agent Personas

Sapling already has `agents/{builder,reviewer,scout}.md` personas. These map directly to overstory capabilities:

| Overstory Capability | Sapling Persona | Key Behaviors |
|---------------------|-----------------|---------------|
| builder | builder.md | Writes code, runs quality gates, no file-scope violations |
| scout | scout.md | Read-only exploration, maps codebase structure |
| reviewer | reviewer.md | Reviews code, writes comments, no edits |
| merger | builder.md (variant) | Merge-focused builder with conflict resolution expertise |
| lead | N/A (uses Claude Code) | Needs interactive delegation, stays on Claude Code |

---

## 8. Phase 5: Orchestrator Context Optimization

**Goal:** Apply sapling's context management techniques to the orchestrator session itself.

### 8.1 The Problem

The orchestrator (your Claude Code session) manages multiple agents, reads status reports, reviews diffs, makes merge decisions. Over a long session, context accumulates:
- Status check outputs from 10+ agents
- Mail messages (dozens)
- Merge conflict diffs
- Issue tracker outputs
- Historical decisions

Claude Code's built-in context management is a blunt instrument — it compresses old messages when approaching the limit. It doesn't score relevance, prune stale agent status, or archive resolved decisions.

### 8.2 Possible Approaches

**Option A: Run orchestrator through sapling**
- Replace Claude Code as the orchestrator runtime with sapling
- Requires sapling to support interactive/TUI mode (significant effort)
- Breaks the "coordinator = interactive, worker = headless" model
- **Not recommended for v1**

**Option B: Extract context primitives as a library**
- Factor sapling's context pipeline into `@os-eco/sapling-context` package
- Overstory imports and applies context management to orchestrator hooks
- Claude Code's `PreCompact` hook could invoke sapling context analysis
- **Recommended approach**

**Option C: Sapling as context advisor**
- Orchestrator periodically calls `sp run "Analyze this conversation and suggest what to prune"`
- Advisory only — orchestrator decides what to act on
- Lightweight but indirect
- **Useful as an interim step**

### 8.3 Context Library Design (Option B)

```typescript
// @os-eco/sapling-context (extracted package)
import { createContextManager } from "@os-eco/sapling-context";

const ctx = createContextManager({
  contextWindow: 200_000,
  budget: { systemPrompt: 0.15, archive: 0.10, history: 0.40, current: 0.15, headroom: 0.20 },
});

// Between orchestrator turns (via PreCompact hook or custom integration):
const optimized = ctx.process(messages, tokenUsage, activeFiles);
```

### 8.4 Scope

Phase 5 is the most speculative. The key deliverable is:
- Prove sapling's context management works at scale for workers (Phases 1–4)
- Identify which context primitives are orchestrator-applicable
- Extract and package them
- Integrate via overstory hooks or a future orchestrator mode

---

## 9. Technical Specifications

### 9.1 AgentRuntime Interface Evolution

**Phase 1 additions** (non-breaking, all optional):

```typescript
export interface AgentRuntime {
  // ... existing 8 required methods + 2 optional ...

  /** Whether this runtime is headless (no tmux, direct subprocess). Default: false. */
  readonly headless?: boolean;

  /**
   * Build direct subprocess spawn options.
   * Called instead of buildSpawnCommand() when headless is true.
   * Falls back to buildSpawnCommand() + tmux if not implemented.
   */
  buildDirectSpawn?(opts: SpawnOpts): DirectSpawnOpts;

  /**
   * Parse NDJSON events from agent's stdout/log for real-time monitoring.
   * Called by ov inspect / ov feed for headless agents.
   */
  parseEvents?(logPath: string, since?: string): AsyncIterable<AgentEvent>;
}

export interface DirectSpawnOpts {
  /** argv for Bun.spawn (e.g., ["sp", "run", "..."]) */
  argv: string[];
  /** Additional environment variables */
  env?: Record<string, string>;
  /** Whether to pipe stdin (for RPC) or ignore it */
  stdin?: "pipe" | "ignore";
  /** Whether to pipe stdout for NDJSON events */
  stdout?: "pipe" | "ignore";
}

export interface AgentEvent {
  type: "started" | "tool_call" | "tool_result" | "turn_complete" | "response" | "run_complete" | "error";
  timestamp: string;
  data: Record<string, unknown>;
}
```

### 9.2 NDJSON Event Protocol Evolution

**Phase 1 (current sapling output):**

```jsonl
{"success":true,"command":"response","response":"..."}
{"success":true,"command":"run","exitReason":"task_complete","totalTurns":15,"inputTokens":45000,"outputTokens":12000}
```

**Phase 2 (with hooks + per-turn events):**

```jsonl
{"ts":"...","type":"started","agent":"alice","model":"claude-sonnet-4-6","maxTurns":200}
{"ts":"...","type":"turn_start","turn":1}
{"ts":"...","type":"tool_call","tool":"read","args":{"file":"src/auth.ts"},"blocked":false}
{"ts":"...","type":"tool_result","tool":"read","success":true,"tokens":450}
{"ts":"...","type":"turn_end","turn":1,"inputTokens":3200,"outputTokens":1800,"contextUtilization":0.12}
{"ts":"...","type":"mail_received","from":"orchestrator","subject":"priority change"}
{"ts":"...","type":"turn_start","turn":2}
...
{"ts":"...","type":"run_complete","exitReason":"task_complete","totalTurns":15,"inputTokens":45000,"outputTokens":12000}
```

**Phase 3 (with ecosystem events):**

```jsonl
{"ts":"...","type":"mail_sent","to":"orchestrator","subject":"task complete","mailType":"worker_done"}
{"ts":"...","type":"metrics_recorded","turn":5,"inputTokens":3200,"outputTokens":1800,"estimatedCost":0.012}
{"ts":"...","type":"tracker_updated","taskId":"task-123","status":"done"}
```

### 9.3 Guard Config Schema

```typescript
interface SaplingGuardConfig {
  version: 1;
  agentName: string;
  capability: string;
  worktreePath: string;
  rules: {
    /** Exclusive file list — only these files may be written/edited. Empty = unrestricted. */
    fileScope: string[];
    /** If true, all write/edit/bash-write operations are blocked. */
    readOnly: boolean;
    /** Tool names that are completely disabled. */
    blockedTools: string[];
    /** Regex patterns matched against bash command strings. */
    blockedBashPatterns: string[];
    /** Absolute path boundary — writes outside this path are blocked. */
    pathBoundary: string;
    /** Additional allowed paths (e.g., shared temp dirs). */
    allowedPaths: string[];
  };
  qualityGates: Array<{
    command: string;
    description: string;
  }>;
  events: {
    /** Emit per-tool-call events. */
    emitToolCalls: boolean;
    /** Path to write events (NDJSON). Relative to worktree. */
    emitFile: string;
    /** Include tool input arguments in events (may contain sensitive data). */
    logToolArgs: boolean;
  };
  ecosystem?: {
    mailDb: string;
    metricsDb: string;
    mulchDir: string;
    seedsDir: string;
    overstoryDir: string;
  };
}
```

### 9.4 Process Lifecycle State Machine

```
                      ┌─────────┐
                      │ spawned │
                      └────┬────┘
                           │ Bun.spawn() succeeds
                      ┌────▼────┐
                      │ running │◄──── NDJSON events flowing
                      └────┬────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         exit code 0  exit code 1  SIGTERM
              │            │            │
        ┌─────▼────┐ ┌────▼─────┐ ┌────▼──────┐
        │completed │ │  failed  │ │terminated │
        └──────────┘ └──────────┘ └───────────┘
```

Overstory maps these to `AgentState`:
- `spawned` → `booting`
- `running` → `working`
- `completed` → `completed`
- `failed` → `failed`
- `terminated` → `completed` (with note)

### 9.5 Cost Comparison Model

| Factor | Claude Code Worker | Sapling Worker (SDK) |
|--------|-------------------|---------------------|
| Billing | Subscription (fixed/month) | Per-token API |
| Memory | ~300 MB | ~80 MB |
| Spawn time | ~5–8s (TUI init + readiness) | ~1s (process start) |
| Context at turn 50 | Degraded (bloated window) | Managed (pruned + archived) |
| Token waste (stale context) | ~30–50% of input tokens | ~5–10% (context pipeline) |
| Guard enforcement | Prose instructions (LLM may ignore) | Code-level (guaranteed) |
| Monitoring granularity | Tmux capture (text scraping) | NDJSON events (structured) |

**Break-even analysis:**
- Subscription: ~$200/month for Claude Pro (5x rate limit)
- API pricing: Sonnet at $3/$15 per MTok (in/out)
- A typical builder agent session: ~100K input, ~30K output = ~$0.75/session
- At ~267 sessions/month, API costs exceed subscription
- But: sapling's context management reduces input tokens by ~30–50%, pushing break-even to ~380–530 sessions/month
- Fleet operations often run 50–100 sessions/day → API is cheaper with context management

---

## 10. Migration Path

### 10.1 Phase 1 → Phase 2

**Sapling side:**
1. Add `src/hooks/` module (guards, events)
2. Modify `loop.ts` to call hooks pre/post tool execution
3. Add `--guards-file` CLI option
4. Add per-turn NDJSON events

**Overstory side:**
1. Update `SaplingRuntime.deployConfig()` to write `.sapling/guards.json`
2. Update `ov inspect` to read sapling event files

### 10.2 Phase 2 → Phase 3

**Sapling side:**
1. Add `src/ecosystem/` module (mail, metrics, seeds, mulch clients)
2. Modify `loop.ts` to check mail between turns
3. Modify `loop.ts` to write metrics after each LLM call
4. Add ecosystem config to guard config schema

**Overstory side:**
1. Pass ecosystem paths in guard config
2. Remove transcript-parsing fallback for sapling agents (metrics are live)
3. Update `ov costs --live` to use real-time metrics.db data

### 10.3 Phase 3 → Phase 4

**Overstory side:**
1. Change default runtime from `"claude"` to `"sapling"` in `config.ts`
2. Update `agent-manifest.json` defaults
3. Update docs and `ov init` templates
4. Add migration guide for existing users

**No sapling changes needed** — Phase 4 is a config/default change.

### 10.4 Backwards Compatibility

Throughout all phases:
- `--runtime claude` always works (existing behavior unchanged)
- `--runtime pi`, `--runtime codex`, etc. unaffected
- Existing tmux-based monitoring continues for non-sapling agents
- Session store schema doesn't change (just uses `pid` without `tmuxSession`)
- All overstory commands (`ov status`, `ov merge`, `ov mail`, etc.) work with both sapling and tmux-based agents

---

## 11. Open Questions

### Resolved During Design

| Question | Decision |
|----------|----------|
| Tmux for sapling? | No — direct subprocess management |
| Hook system approach? | Sapling-native hooks, overstory provides config |
| Billing model coupling? | None — overstory doesn't care, sapling handles its own billing |
| Workers vs orchestrator? | Workers first, orchestrator optimization later (Phase 5) |

### Open for Future Phases

| Question | Phase | Notes |
|----------|-------|-------|
| Should sapling support `ov nudge`? | 2 | With no tmux, nudge becomes a mail message. Or: inject text into stdin if piped. |
| JSON-RPC for mid-task steering? | 3 | Pi-style RPC via stdin/stdout. Enables `ov coordinator ask` for sapling agents. |
| How does `ov inspect --follow` work for headless agents? | 2 | Tail NDJSON event file instead of tmux capture-pane. |
| Should sapling agents auto-send `worker_done` mail? | 3 | Native ecosystem integration makes this trivial. |
| Can sapling agents spawn sub-workers? | 4 | Only if sapling gains `ov sling` integration (lead capability). Likely stays Claude Code. |
| How does the context library (Phase 5) interact with Claude Code's built-in compression? | 5 | TBD — may need to hook into PreCompact or replace it entirely. |
| Should sapling support model fallback chains? | 3+ | E.g., try Opus, fall back to Sonnet on rate limit. Useful for fleet cost optimization. |
| Per-agent context budget tuning? | 3+ | Scouts need less history budget; builders need more. Configurable via guard config. |

---

## Appendix A: File Change Inventory

### Overstory Changes

| File | Phase | Change |
|------|-------|--------|
| `src/runtimes/sapling.ts` | 1 | **New** — SaplingRuntime adapter |
| `src/runtimes/types.ts` | 1 | Add `headless`, `buildDirectSpawn`, `parseEvents` optional fields |
| `src/runtimes/registry.ts` | 1 | Register sapling runtime |
| `src/commands/sling.ts` | 1 | Add headless spawn path (bypass tmux) |
| `src/worktree/process.ts` | 1 | **New** — Direct subprocess management |
| `src/commands/stop.ts` | 1 | Handle PID-based termination |
| `src/commands/status.ts` | 1 | Handle tmux-less agents |
| `src/commands/dashboard.ts` | 1 | Handle tmux-less agents |
| `src/commands/inspect.ts` | 2 | Read NDJSON events for sapling agents |
| `src/commands/feed.ts` | 2 | Aggregate sapling NDJSON events |
| `src/agents/guard-rules.ts` | 2 | Export guard rules as JSON config |

### Sapling Changes

| File | Phase | Change |
|------|-------|--------|
| `src/hooks/index.ts` | 2 | **New** — HookManager |
| `src/hooks/guards.ts` | 2 | **New** — Guard implementations |
| `src/hooks/events.ts` | 2 | **New** — NDJSON event emitter |
| `src/hooks/types.ts` | 2 | **New** — Hook/guard types |
| `src/loop.ts` | 2 | Integrate hook pre/post calls |
| `src/cli.ts` | 2 | Load guard config |
| `src/config.ts` | 2 | Add guardsFile option |
| `src/ecosystem/mail.ts` | 3 | **New** — Direct mail.db client |
| `src/ecosystem/metrics.ts` | 3 | **New** — Direct metrics.db client |
| `src/ecosystem/seeds.ts` | 3 | **New** — Direct seeds client |
| `src/ecosystem/mulch.ts` | 3 | **New** — Direct mulch client |
| `src/loop.ts` | 3 | Mail check between turns, metrics write per turn |

### Estimated Lines of Code

| Component | Lines | Phase |
|-----------|-------|-------|
| SaplingRuntime adapter | ~250 | 1 |
| process.ts (subprocess mgmt) | ~100 | 1 |
| sling.ts headless path | ~80 | 1 |
| Interface additions | ~40 | 1 |
| Registry + monitoring updates | ~60 | 1 |
| **Phase 1 total** | **~530** | |
| Hook manager + guards | ~300 | 2 |
| Event emitter | ~80 | 2 |
| Guard config deployment | ~60 | 2 |
| Loop integration | ~40 | 2 |
| NDJSON per-turn events | ~60 | 2 |
| **Phase 2 total** | **~540** | |
| Ecosystem clients (mail, metrics, seeds, mulch) | ~300 | 3 |
| Loop ecosystem integration | ~60 | 3 |
| Config / env wiring | ~40 | 3 |
| **Phase 3 total** | **~400** | |
| Config defaults + manifest | ~20 | 4 |
| **Phase 4 total** | **~20** | |
| **Grand total** | **~1,490** | |

---

## Appendix B: Relationship to Existing Design Docs

| Document | Relationship |
|----------|-------------|
| `docs/sapling.md` | Original vision. This spec **implements** the overstory integration section. |
| `docs/sapling-thought-experiment.md` | Strategic rationale. This spec **realizes** the "orchestration-native runtime" vision. |
| `docs/sapling-mvp-spec.md` | MVP blueprint. This spec **extends** the MVP with overstory-specific features (hooks, ecosystem, runtime adapter). |

Key alignment points:
- Two-tier execution model (coordinator = interactive, worker = headless) ✓
- TypeScript guards, not shell scripts ✓
- Native ecosystem integration (direct SQLite, not subprocess) ✓
- System prompt is code, not overlay text ✓
- Context management is the innovation ✓
- No TUI (headless, observability via overstory commands) ✓
