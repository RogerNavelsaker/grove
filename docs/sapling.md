# Sapling

> The agent runtime that completes the os-eco ecosystem.

**Status:** Design  
**CLI:** `sapling` / `sp`  
**npm:** `@os-eco/sapling-cli`  
**Sub-repo:** `sapling/`  
**See also:** [sapling-thought-experiment.md](sapling-thought-experiment.md) — original design exploration

---

## The Gap

The four existing os-eco tools all coordinate *around* an agent runtime:

```
os-eco today
├── overstory   orchestrates agents
├── mulch       stores expertise for agents
├── seeds       tracks issues agents work on
├── canopy      manages prompts agents use
└── ???         the agent itself
```

Overstory currently uses Claude Code or Pi as the agent runtime. Both were built
for human-in-the-loop interactive use and retrofitted for programmatic control.
The result is a layer of shims: shell-script guards, tmux pane scraping, beacon
injection, transcript parsing, overlay text hoping the runtime reads it correctly.

Sapling is the fifth tool. The thing that actually grows in the overstory, fed by
mulch, planted from seeds, instructed by canopy. An agent runtime designed from
the start to be spawned and coordinated — with human interaction as the optional
mode, not the assumed one.

---

## What This Unlocks

When Sapling is the worker runtime, the shim layer in Overstory largely disappears:

| Current shim | Why it exists | Gone with Sapling |
|---|---|---|
| `hooks-deployer.ts` (634 lines) | Claude Code doesn't know about Overstory | Yes — native lifecycle events |
| Shell-script guards | Can't intercept tool calls in TypeScript | Yes — guards are TypeScript functions |
| TUI readiness detection | TUI doesn't signal readiness | Yes — protocol sends `{"type":"ready"}` |
| Beacon injection via tmux | No structured startup handshake | Yes — JSON-RPC `start` method |
| Transcript parsing + path encoding | Token data buried in an opaque JSONL format | Yes — agent writes to metrics.db directly |
| `settings.local.json` generation | Deploying hooks in Claude Code's config format | Yes — no config file needed |
| Overlay text for guards | "Don't push to main" is prose, not code | Yes — guards are enforced in code |

---

## The Two-Tier Model

Sapling is the worker runtime. The coordinator tier — the agent the human interacts
with — remains Pi, Claude Code, or any TUI agent the user prefers. Overstory doesn't
dictate what the human uses to orchestrate.

```
┌─────────────────────────────────────────────────────────┐
│  COORDINATOR TIER  — Pi / Claude Code / any TUI agent   │
│  Runs in tmux. Human can drop in and steer.             │
│  Capabilities: coordinator, supervisor, lead            │
└────────────────────────┬────────────────────────────────┘
                         │  ov sling
                         ▼
┌─────────────────────────────────────────────────────────┐
│  WORKER TIER  — Sapling                                 │
│  Headless Bun process. No tmux. No TUI.                 │
│  Streams NDJSON events. Reads mail directly via SQLite. │
│  Capabilities: builder, scout, reviewer, merger         │
└─────────────────────────────────────────────────────────┘
```

This split is what makes swarms scale on constrained hardware. A TUI agent
(Claude Code, Pi) consumes 250–400 MB of RAM for rendering infrastructure that
workers don't need. Sapling workers run at 60–120 MB. On a 16 GB machine, that's
the difference between running 5 workers and running 20.

---

## Architecture

Sapling is a standalone Bun tool — same stack as the rest of os-eco, no dependency
on Pi or any other agent runtime. It is heavily informed by Pi's design (the agent
loop, tool system, and context management are all well-understood patterns that Pi
executes well), but it owns every line of code.

```
sapling/
  src/
    index.ts            CLI entry point + JSON-RPC server
    loop.ts             Turn loop: idle check → LLM call → tool dispatch → repeat
    client.ts           Anthropic API client (@anthropic-ai/sdk, Bun-native)
    prompt.ts           System prompt builder (agent def + mulch + task + gates)
    guards.ts           TypeScript guard functions (path, bash, capability rules)
    tools/
      index.ts          Tool registry + dispatch
      bash.ts           Bash execution (Bun.spawn, timeout, streaming, truncation)
      read.ts           File read (Bun.file, offset/limit, truncation)
      write.ts          File write (Bun.write, atomic)
      edit.ts           In-place edit (oldText → newText, exact match)
      grep.ts           Ripgrep wrapper
      glob.ts           File discovery
    protocol/
      rpc.ts            JSON-RPC over stdin/stdout (overstory's control interface)
      events.ts         NDJSON event emitter to stdout
    context/
      compaction.ts     Summarize old turns when context window fills
      window.ts         Token usage tracking per turn
    ecosystem/
      mail.ts           Direct bun:sqlite read/write to .overstory/mail.db
      mulch.ts          ml prime on startup, ml record on completion
      seeds.ts          sd update as work progresses
      metrics.ts        Write token usage to .overstory/metrics.db in real time
```

### The Agent Loop

The core loop is the same pattern every coding agent implements. Pi is a good
reference implementation — Sapling's loop follows the same structure:

```typescript
async function runLoop(task: Task, agent: AgentState): Promise<void> {
  while (true) {
    // Before each LLM call: inject unread mail and any steer messages
    const mail = mailDb.checkUnread(agentName);
    if (mail.length) agent.messages.push(formatMailMessage(mail));

    const steer = controlChannel.drain();
    if (steer) agent.messages.push({ role: "user", content: steer });

    // LLM call
    const response = await llm.createMessage({
      model: agent.model,
      system: agent.systemPrompt,
      messages: agent.messages,
      tools: toolDefinitions,
    });

    metricsDb.record(agentName, response.usage);
    agent.messages.push({ role: "assistant", content: response.content });

    // Tool dispatch
    const toolCalls = response.content.filter(b => b.type === "tool_use");
    if (toolCalls.length === 0) break; // agent is done

    const results = await Promise.all(toolCalls.map(async (call) => {
      const blocked = guards.check(call);
      if (blocked) return toolBlockedResult(call, blocked.reason);
      emit({ type: "tool_execution_start", toolName: call.name, args: call.input });
      const result = await tools.execute(call.name, call.input);
      emit({ type: "tool_execution_end", toolName: call.name });
      return result;
    }));

    agent.messages.push({ role: "user", content: results });

    // Compact if approaching context limit
    if (window.isNearLimit(agent.messages)) {
      agent.messages = await compact(agent.messages, agent.systemPrompt);
    }
  }
}
```

### The LLM Client

Sapling calls the Anthropic API directly via `@anthropic-ai/sdk` — Anthropic's
official Bun-compatible SDK, no Pi involved. Additional providers (OpenAI, Google)
follow the same pattern as needed, behind a thin provider interface in `client.ts`.

### The Protocol

Overstory controls Sapling workers via JSON-RPC over stdin/stdout. This replaces
tmux `send-keys` for worker communication:

```jsonc
// Overstory → Sapling (stdin)
{"method": "start",    "params": {"task": "...", "model": "...", "guards": [...]}}
{"method": "steer",    "params": {"content": "Change approach: ..."}}
{"method": "followUp", "params": {"content": "After finishing, also..."}}
{"method": "abort"}

// Sapling → Overstory (stdout, NDJSON)
{"type": "ready"}
{"type": "agent_start", "agentName": "builder-1", "taskId": "TASK-42"}
{"type": "tool_execution_start", "toolName": "bash", "args": {...}}
{"type": "tool_execution_end",   "toolName": "bash", "durationMs": 4200}
{"type": "mail_received",        "from": "lead-1",  "subject": "..."}
{"type": "agent_end",            "exitReason": "task_complete"}
```

No tmux. No pane capture. No readiness polling. Structured communication throughout.

### System Prompt Construction

The system prompt is assembled programmatically from typed sources — not from an
overlay text file that the runtime might interpret loosely:

```typescript
const systemPrompt = [
  agentDef.basePrompt,                    // from sapling/agents/builder.md
  buildTaskAssignment(task, spec),         // structured, not prose
  buildFileScope(files),                   // described; enforced by guards
  buildQualityGates(config.qualityGates),  // from config, not placeholders
  await mulch.prime({ files }),            // direct call, not subprocess
].join("\n\n");
```

### Ecosystem Integration

Sapling reads and writes Overstory's SQLite databases directly using `bun:sqlite`
with WAL mode — the same approach the rest of os-eco uses. No subprocess round-trips:

- **Mail:** reads `.overstory/mail.db` between turns; marks messages read on receipt
- **Metrics:** writes token usage to `.overstory/metrics.db` after each API response
- **Mulch:** calls `ml prime` on startup for domain expertise; `ml record` on completion
- **Seeds:** calls `sd update` as task status changes

---

## Honest Accounting

| Component | Lines (est.) | Difficulty | Notes |
|---|---|---|---|
| Tool system (bash, read, write, edit, grep, glob) | ~800 | Medium | Pi is a solid reference implementation |
| Anthropic API client | ~400 | Medium | `@anthropic-ai/sdk` + streaming, retries, error handling |
| Agent loop | ~300 | Low | Standard pattern; Pi's loop is a good reference |
| Context management + compaction | ~500 | **Hard** | The part that takes iteration to get right |
| JSON-RPC protocol + event stream | ~200 | Low | |
| System prompt builder | ~200 | Low | |
| Guard system | ~150 | Low | Port logic from Overstory's `guard-rules.ts` |
| Ecosystem clients (mail, mulch, seeds, metrics) | ~300 | Low | Direct `bun:sqlite`, familiar pattern |
| **Total** | **~2,850** | | |

For reference: `hooks-deployer.ts` alone is 634 lines and eliminates only one of
the integration concerns Sapling replaces. Sapling is about twice the investment
of hooks-deployer and eliminates all of them.

### The hard part: context management

Everything except context management is well-understood. The tool implementations,
the API client, the protocol — these are mechanical. Context management is where the
dragons live:

- **When to compact:** context windows fill up mid-task. Compact too early and you
  lose information; too late and the API rejects the request.
- **What to preserve:** the task spec must always survive compaction. Tool results
  from earlier in the task may be critical. Recent conversation is most important.
- **Prompt caching:** Anthropic's prompt caching requires a stable cacheable prefix.
  Sapling's message structure needs to be designed around this from the start, not
  bolted on later.
- **Multi-turn coherence:** after compaction, the agent needs to know what it was
  doing. A bad summary causes derailment.

Pi solves this well. Sapling should study Pi's compaction implementation closely
and build something equivalent, not try to invent a novel approach.

---

## Relationship to Overstory

Sapling is an `AgentRuntime` adapter in Overstory, the same interface as
`ClaudeRuntime` and `PiRuntime`. Overstory doesn't need to know how Sapling works
internally — it spawns the process, sends JSON-RPC messages, and reads the event
stream.

The capability-to-runtime mapping in `config.yaml` makes Sapling the default for
leaf agents while leaving coordinator-tier agents on Pi or Claude Code:

```yaml
runtime:
  coordinator: pi      # human interacts with this — needs TUI
  supervisor: pi
  lead: pi
  builder: sapling     # fleet workers — headless, lightweight
  scout: sapling
  reviewer: sapling
  merger: sapling
```

See [`overstory/docs/sapling.md`](../overstory/docs/sapling.md) for overstory-
specific integration details (session store schema, tmux bypass, watchdog changes).

---

## Relationship to Pi

Pi remains the recommended runtime for coordinator-tier agents — it has a mature TUI,
excellent extensibility, and active maintenance.

Sapling is heavily informed by Pi's design. The agent loop, tool system, context
management strategy, and NDJSON event format are all patterns Pi executes well.
Sapling studies these and implements equivalents natively in Bun, with no runtime
dependency on Pi.

Sapling and Pi are not competitors. Pi is what the human uses. Sapling is what the
fleet uses. They run at different tiers of the same hierarchy.

---

## Relationship to the Rest of os-eco

```
Ecosystem today          With Sapling
──────────────           ──────────────────────
overstory                overstory
  └─ uses Claude Code      └─ uses Sapling (workers)
                                └─ reads mail.db        ← mulch, seeds, canopy:
                                └─ writes metrics.db       already exist
                                └─ calls ml prime
                                └─ calls sd update
                                └─ loads cn emit output
```

Sapling doesn't add new ecosystem concerns. It consumes the infrastructure the
other four tools have already built, natively, without subprocesses or shims.

---

## What Sapling Deliberately Excludes

**A TUI.** Sapling workers are headless. Observability comes from `ov dashboard`,
`ov feed`, and `ov trace` — not from watching tmux panes. If a specific worker needs
interactive debugging, `ov sling --runtime pi` runs that worker in Pi's TUI.

**Multi-provider at launch.** Anthropic-first via `pi-agent-core`. Other providers
follow as needed.

**A general extension system.** Sapling is purpose-built for overstory. The guard
layer, ecosystem integration, and protocol are built in. There's no plugin surface
because there's no use case for one.

---

## Implementation Path

Sapling can be bootstrapped with Overstory's own swarm once the Sapling spec is
well-defined. The implementation is straightforward enough that well-scoped workers
can build it in parallel:

1. **JSON-RPC protocol + event stream** — the communication layer; no LLM required to build and test
2. **System prompt builder + guard system** — port from Overstory's existing logic
3. **Ecosystem clients** — direct SQLite wrappers; straightforward
4. **`pi-agent-core` wiring** — connect the engine to the orchestration shell
5. **Overstory adapter** (`src/runtimes/sapling.ts`) — wire the protocol into `AgentRuntime`
6. **Integration tests** — spawn Sapling via Overstory, run a real task end-to-end
