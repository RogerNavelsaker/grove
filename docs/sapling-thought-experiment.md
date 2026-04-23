# Sapling: An Orchestration-Native Agent Runtime

> Thought experiment — what if os-eco had its own coding agent runtime, built
> from the ground up for multi-agent orchestration?

## The Mismatch

Every existing coding agent — Claude Code, Codex, Pi, Cline, OpenCode — was
built for **human-in-the-loop interactive use**, then retrofitted with
headless/RPC modes for programmatic control. Overstory spends enormous effort
working around this:

- **Hooks** to inject mail into a system that doesn't know about mail
- **TUI scraping** to detect readiness in a system that doesn't signal it
- **Shell script guards** to block tools in a system that doesn't expose a guard API
- **Overlay text** to enforce behavior that should be code
- **Transcript parsing** to extract metrics from a format we don't control
- **Beacon injection** to deliver startup instructions via tmux send-keys

An orchestration-native runtime flips the assumption: the agent is designed to
be *spawned and coordinated*, with human interaction as the optional mode.

## The Ecosystem Gap

```
os-eco today
├── overstory   orchestrates agents
├── mulch       stores expertise for agents
├── seeds       tracks issues agents work on
├── canopy      manages prompts agents use
└── ???         the agent itself
```

The four tools coordinate around an agent runtime they don't control. The
runtime is the most important piece of the system and it's the one piece that
isn't ours. Every integration point is a shim bolted onto someone else's
architecture.

A fifth tool — **Sapling** — would complete the picture. The thing that actually
grows in the overstory's canopy, fed by mulch, planted from seeds.

## What Disappears

The coupling between Overstory and Claude Code spans ~1500 lines across 15+
files. With a native runtime, most of it evaporates:

| Current complexity | Why it exists | Gone? |
|--------------------|--------------|-------|
| `hooks-deployer.ts` (634 lines) | Claude Code doesn't know about Overstory | Yes — native lifecycle events |
| `hooks.json.tmpl` (108 lines) | Translating needs into Claude Code's hook format | Yes — no translation layer |
| TUI readiness detection | Claude Code doesn't signal readiness | Yes — protocol signals `ready` |
| `ENV_GUARD`, `PATH_PREFIX` hacks | Claude Code strips PATH, hooks fire in wrong sessions | Yes — agent controls its own env |
| Tool blocklists (13 names) | Blocking Claude Code's native team/interactive tools | Yes — those tools don't exist |
| Overlay text for guards | "Don't push to main" is prose, not code | Yes — guards are TypeScript |
| Transcript parsing | Extracting tokens from Claude Code's JSONL | Yes — agent reports tokens natively |
| Beacon injection + verification | Delivering startup instructions via tmux send-keys | Yes — protocol delivers assignment |
| `settings.local.json` generation | Deploying hooks in Claude Code's config format | Yes — no config file needed |

## Architecture

```
sapling
├── llm/              LLM client (direct API: Anthropic, OpenAI, Google, local)
├── tools/            Tool system (read, write, edit, bash, grep, glob)
├── loop/             Conversation loop (prompt → tool calls → execute → repeat)
├── context/          Context management (compaction, caching, window tracking)
├── protocol/         JSON-RPC over stdin/stdout (orchestration interface)
├── guards/           Tool guard system (TypeScript, not shell scripts)
└── ecosystem/        Native os-eco integration
    ├── mail          Direct bun:sqlite access to mail.db
    ├── mulch         Prime on startup, record on completion
    ├── seeds         Update issue status as work progresses
    └── canopy        Load rendered prompts for system context
```

### Tech Stack

Same as the rest of os-eco:
- **Runtime:** Bun
- **Language:** TypeScript strict mode
- **Database:** bun:sqlite (WAL mode) for any local state
- **LLM access:** Direct API calls via official SDKs or raw fetch
- **No wrapping another CLI** — the runtime calls provider APIs directly

## How Things Change

### Spawning

**Today (Claude Code):**
```
ov sling
  → create worktree
  → generate overlay text
  → generate hooks.json shell scripts
  → deploy settings.local.json
  → create tmux session
  → launch `claude --model X --permission-mode bypassPermissions`
  → poll tmux capture-pane for "❯", 'Try "', "bypass permissions"
  → send beacon via tmux send-keys
  → poll again to verify beacon received
  → send dispatch mail via mail.db
  → hope UserPromptSubmit hook fires and injects it
```

**With Sapling:**
```
ov sling
  → create worktree
  → spawn `sapling --mode rpc` as subprocess
  → send JSON-RPC: { method: "start", params: { task, spec, files, model, guards } }
  → receive: { type: "ready" }
```

### Mail

**Today:**
```
Orchestrator                     Claude Code agent
     │                                │
     │── write to mail.db ──────────>│ (agent doesn't see this)
     │                                │
     │   ... wait for UserPromptSubmit hook to fire ...
     │   ... hook spawns `ov mail check --inject` ...
     │   ... subprocess reads mail.db, writes to stdout ...
     │   ... Claude Code prepends stdout to next prompt ...
     │                                │
     │<── agent eventually sees it ──│
```

**With Sapling:**
```
Orchestrator                     Sapling agent
     │                                │
     │── write to mail.db ──────────>│
     │                                │
     │   ... agent's built-in mail    │
     │       client reads mail.db     │
     │       directly via bun:sqlite  │
     │       (same WAL database)      │
     │                                │
     │<── agent reads immediately ───│
```

No hooks. No shell scripts. No stdout injection. No subprocess spawning. The
agent reads the same SQLite database. WAL mode handles concurrent access.

### Tool Guards

**Today:** Shell scripts that parse JSON on stdin and echo decisions to stdout:

```bash
#!/bin/sh
tool_name=$(echo "$1" | jq -r '.tool_name')
if [ "$tool_name" = "Bash" ]; then
  input=$(echo "$1" | jq -r '.tool_input.command // ""')
  if echo "$input" | grep -q "git push.*main"; then
    echo '{"decision":"block","reason":"Cannot push to main"}';
    exit 0
  fi
fi
```

**With Sapling:** TypeScript functions, type-checked and testable:

```typescript
const guards: ToolGuard[] = [
  {
    tool: "bash",
    check: (input) => {
      if (input.command?.match(/git push.*main/)) {
        return { block: true, reason: "Cannot push to main from worktree" };
      }
    },
  },
  {
    tool: "write",
    check: (input) => {
      if (!fileScope.includes(input.path)) {
        return { block: true, reason: `Outside file scope: ${input.path}` };
      }
    },
  },
];
```

### Metrics and Cost Tracking

**Today:** Parse Claude Code's JSONL transcript after the session ends. The
format is `{type:"assistant", message:{usage:{input_tokens, output_tokens,
cache_read_input_tokens}}}`. File location is
`~/.claude/projects/<encoded-path>/<session-id>.jsonl`. Pricing is a hardcoded
table matching model name substrings to Anthropic's published rates.

**With Sapling:** The API response includes token usage directly. The agent
writes to Overstory's metrics.db in real-time:

```typescript
const response = await llm.createMessage({ model, messages, tools });

metricsStore.recordSnapshot({
  agentName,
  inputTokens: response.usage.input_tokens,
  outputTokens: response.usage.output_tokens,
  model: response.model,
  timestamp: new Date().toISOString(),
});
```

No transcript file. No parsing. No path encoding. No pricing table lookup.
Costs are known at the moment they're incurred.

### System Prompt Construction

**Today:** Write a CLAUDE.md overlay file with markdown text, hope the runtime
reads it, hope it follows the instructions. Quality gates are text placeholders
(`{{QUALITY_GATE_BASH}}`). Agent identity is a YAML file that gets rendered
into prose. Mulch expertise is injected via a `SessionStart` hook that runs
`mulch prime` and prepends output to the session.

**With Sapling:** The system prompt is constructed programmatically:

```typescript
const systemPrompt = [
  agentDefinition.basePrompt,           // from agents/builder.md
  buildTaskAssignment(task, spec),       // structured, not prose
  buildFileScope(files),                 // enforced by guards, described in prompt
  buildQualityGates(config.qualityGates),// from config, not placeholders
  await mulch.prime({ files }),          // direct call, not subprocess
  buildEcosystemContext({ mail: true, seeds: true }),
].join("\n\n");

const agent = new Agent({ model, systemPrompt, tools, guards });
```

Every piece of context is code. Testable. Typed. No hoping the runtime reads a
file correctly.

## The LLM Layer

Instead of wrapping a CLI that wraps an API, the runtime calls provider APIs
directly:

```typescript
// Anthropic
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-5-20250514",
  system: systemPrompt,
  messages: history,
  tools: toolDefinitions,
  max_tokens: 8192,
});

// OpenAI
const response = await openai.chat.completions.create({
  model: "gpt-5-codex",
  messages: [{ role: "system", content: systemPrompt }, ...history],
  tools: toolDefinitions,
});
```

This gives:
- **Full prompt control** — no overlay that might be ignored
- **Direct token tracking** — usage in every API response
- **Streaming control** — decide per-turn whether to stream
- **Provider-specific features** — Anthropic's prompt caching, extended
  thinking, etc.
- **No CLI binary dependency** — the runtime is self-contained

### The Conversation Loop

The core loop is straightforward. Every coding agent implements the same
pattern:

```
1. Send messages + tools to LLM
2. LLM responds with text and/or tool calls
3. For each tool call:
   a. Check guards (block or allow)
   b. Execute tool
   c. Collect result
4. Append assistant message + tool results to history
5. If tool calls were made, go to 1 (LLM needs to see results)
6. If no tool calls, the turn is complete — agent is idle
7. Check for new mail / orchestrator commands
8. If new input, go to 1
9. If no input, wait
```

This is ~300 lines of TypeScript. Claude Code, Pi, Codex — they all implement
this loop. The complexity isn't in the loop itself; it's in the **tool
implementations** and **context management**.

## Honest Accounting: What You Need to Build

| Component | Lines (est.) | Difficulty | Notes |
|-----------|-------------|------------|-------|
| **Tool system** (read, write, edit, bash, grep, glob) | ~800 | Medium | Well-understood. Pi and Claude Code are references. |
| **LLM client** (multi-provider) | ~400 | Medium | Anthropic SDK + OpenAI SDK + provider routing. |
| **Conversation loop** | ~300 | Low | Send → tool calls → execute → repeat. Standard pattern. |
| **Context management** | ~500 | Hard | Compaction, caching, window tracking. This is where the dragons are. |
| **JSON-RPC protocol** | ~200 | Low | stdin/stdout. Pi's RPC is a reference. |
| **System prompt builder** | ~200 | Low | Assemble from agent def + mulch + task + gates. |
| **Guard system** | ~150 | Low | TypeScript functions, not shell scripts. |
| **Ecosystem clients** (mail, mulch, seeds) | ~300 | Low | Direct bun:sqlite + existing CLI wrappers. |
| **Token/cost tracking** | ~100 | Low | API responses include usage. |
| **Total** | **~2950** | | |

For reference, Overstory is ~15,000 lines. The hooks-deployer alone is 634.

### The Hard Part: Context Management

This deserves emphasis. Context management is the difference between a toy and a
production agent. It covers:

- **When to compact:** The context window fills up. When do you summarize old
  turns vs. dropping them? Claude Code has years of tuning on this.
- **What to preserve:** Tool call results, file contents read earlier, error
  messages — some of these are critical for the current task, some are noise.
- **Caching strategy:** Anthropic's prompt caching lets you cache the system
  prompt prefix. The runtime needs to structure messages so the cacheable
  prefix is stable across turns.
- **Multi-turn coherence:** After compaction, the agent needs to "remember"
  what it was doing. A bad summary loses critical context and the agent
  derails.

This is solvable — Pi solves it, Claude Code solves it, even Aider solves it
with its map/summary approach. But it's the component that takes the most
iteration to get right.

## What You Lose

Honest trade-offs:

1. **Claude Code's system prompt.** Years of empirical tuning on how to instruct
   an LLM to be a good coding agent. Tool descriptions, safety guidelines, edit
   strategies, error recovery. This is non-trivial intellectual property that
   would need to be rebuilt through iteration.

2. **MCP ecosystem.** Claude Code supports Model Context Protocol servers. A
   custom runtime would need its own extension mechanism or an MCP client
   implementation.

3. **Human takeover.** Sometimes the orchestrator needs a human to step in.
   Claude Code's TUI is that escape hatch. A headless-only runtime loses this
   unless you build a TUI (and then you're building Pi).

4. **Community momentum.** Claude Code, Codex, and Cline have large user bases
   reporting bugs, requesting features, and stress-testing edge cases. A custom
   runtime absorbs that maintenance burden internally.

5. **Provider quirks.** Each LLM provider has tool-use quirks — different
   response formats, different error modes, different rate limiting behavior.
   Claude Code and Pi absorb these. A custom runtime means you absorb them.

## The Hybrid Path

The most compelling option: **build the orchestration shell, embed an existing
LLM engine.**

Pi's monorepo separates concerns cleanly:
- `@mariozechner/pi-agent-core` — LLM interaction, tool execution, conversation management
- `@mariozechner/pi-coding-agent` — TUI, CLI, session management

You could import `pi-agent-core` as a library and wrap it:

```typescript
import { Agent, createTools } from "@mariozechner/pi-agent-core";

// Sapling = orchestration shell + Pi's LLM engine
const agent = new Agent({
  model: task.model,
  tools: [
    ...createTools({ cwd: worktreePath }),  // Pi's battle-tested tools
    createMailTool(mailStore, agentName),    // native mail.db access
    createMulchTool(worktreePath),           // native mulch integration
    createSeedsTool(worktreePath),           // native seeds integration
  ],
  systemPrompt: buildOverstoryPrompt(task, mulchContext),
});

// Orchestration protocol
const rpc = createRpcServer(process.stdin, process.stdout);

rpc.on("start", async (params) => {
  guards.load(params.guards);
  await agent.prompt(params.initialPrompt);
  rpc.emit("ready");
});

rpc.on("message", async (params) => {
  await agent.followUp(params.text);
});

rpc.on("abort", () => agent.abort());

// Native mail polling between turns
agent.on("idle", async () => {
  const messages = mailStore.check(agentName);
  if (messages.length) {
    await agent.followUp(formatMail(messages));
  }
  reportMetrics(agent.lastUsage);
});

// Tool guards — code, not shell scripts
agent.on("tool_call", (call) => {
  const result = guards.check(call);
  if (result?.block) return { type: "block", reason: result.reason };
  return { type: "allow" };
});
```

### What the hybrid gets you

| Dimension | Full custom | Hybrid (shell + pi-core) |
|-----------|------------|--------------------------|
| Context management | Must build (~500 lines, hard) | Pi's, battle-tested |
| Tool implementations | Must build (~800 lines) | Pi's, with custom tools added |
| Provider quirks | Must absorb | Pi absorbs |
| Ecosystem integration | Native | Native (custom tools + event handlers) |
| Orchestration protocol | Build it | Build it |
| Maintenance burden | Full | Shared with Pi |
| System prompt | Full control | Full control |
| Guard system | Full control | Full control (via tool_call event) |
| Upgrade path | Manual | `bun update @mariozechner/pi-agent-core` |

You keep full control over the parts that matter for orchestration (protocol,
guards, ecosystem integration, system prompt) and delegate the parts that are
commodity (tool implementations, context management, provider abstraction) to a
well-maintained open-source library.

### Dependency risk

The concern with embedding `pi-agent-core`: what if Pi's API changes, or the
project goes unmaintained?

Mitigations:
- Pi is MIT-licensed. Fork is always available.
- The dependency is on `pi-agent-core`, not the full CLI. The core is the
  stable, less-frequently-changed layer.
- The orchestration shell is yours. Swapping the LLM engine (to a different
  library, or to a custom implementation) means changing the inner layer, not
  the outer.
- Worst case: you've already got the orchestration shell built. Replacing the
  inner engine is the "full custom" path, but with all the orchestration work
  already done.

## How It Relates to the Runtime Abstraction

The runtime abstraction (Overstory issue #38) and Sapling are not mutually
exclusive. They're sequential:

```
Phase 0-2: Runtime abstraction
  → Extract AgentRuntime interface
  → Claude adapter (current behavior)
  → Codex adapter
  → Pi adapter

Phase 3+: Sapling (if/when warranted)
  → Build orchestration shell
  → Embed pi-agent-core (or custom LLM engine)
  → Sapling becomes another AgentRuntime adapter
  → But it's the one that doesn't need shims
```

The abstraction layer is the right next step regardless. It's prerequisite work
whether you end up using Codex, Pi-via-adapter, or Sapling. And if Sapling
materializes, it slots in as just another runtime — the one that happens to
understand the ecosystem natively.

## Summary

| Approach | Effort | Eliminates coupling? | Risk |
|----------|--------|---------------------|------|
| **Runtime abstraction** (#38) | Low | Isolates it behind interface | Low |
| **Full custom runtime** | High (~3000 lines) | Yes, completely | High — context management is hard |
| **Hybrid** (shell + pi-agent-core) | Medium (~1500 lines) | Yes, completely | Medium — Pi dependency |

The hybrid is the sweet spot. You get ecosystem-native orchestration, type-safe
guards, direct mail access, native metrics — without rebuilding the LLM engine.
The orchestration shell is ~1500 lines of code that replaces ~1500 lines of
coupling shims. Same effort, fundamentally better architecture.

Whether this is worth building depends on a simpler question: **does Overstory
spend more time fighting runtime integration than building orchestration
features?** If the answer is yes — and the hooks-deployer, TUI detection, beacon
system, and transcript parsing suggest it is — then the investment pays for
itself.
