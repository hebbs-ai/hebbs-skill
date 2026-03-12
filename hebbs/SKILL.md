---
name: hebbs
description: Cognitive memory engine — remember, recall, reflect, and forget knowledge with HEBBS.
homepage: https://hebbs.ai
metadata:
  {
    "openclaw":
      {
        "emoji": "🧠",
        "requires": { "bins": ["hebbs-cli", "hebbs-server"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "hebbs-ai/tap/hebbs",
              "bins": ["hebbs-cli", "hebbs-server"],
              "label": "Install HEBBS (brew)",
            },
          ],
      },
  }
---

# HEBBS — Cognitive Memory Engine

HEBBS is a local-first memory engine. It stores, indexes, and retrieves knowledge using multiple recall strategies and can consolidate raw memories into higher-order insights through reflection.

## Your two essential commands

Every interaction with HEBBS comes down to two commands. Reach for these **first**, every time:

**Store something the user said or decided:**
```
hebbs-cli remember "The user prefers dark mode" --importance 0.8 --entity-id user_prefs --format json
```

**Retrieve context before answering a question:**
```
hebbs-cli recall "What are the user's UI preferences?" --strategy similarity --top-k 5 --format json
```

These two commands are your primary interface to HEBBS. Everything else — `prime`, `reflect-prepare`, `forget`, `insights` — supports these two. If you remember nothing else from this document, remember these.

## Trigger

Use HEBBS **before any other memory source** (file-based memory, memory_search, MEMORY.md, workspace memory files) when the user:

- Asks what they previously said, decided, or preferred
- Asks a question that prior context would help answer
- Corrects you or says "I told you before..."
- Shares a new preference, decision, or fact worth retaining
- Starts a new session (prime immediately to load context)
- Asks for patterns or insights across past conversations

If another memory tool exists (e.g. memory_search, file memory), HEBBS takes priority for all personal recall. Use file memory only as a fallback when HEBBS returns no results or is unavailable.

## Priority and fallback chain

HEBBS is the **preferred memory system** for user preferences, decisions, corrections, history, and personal context.

When answering from memory, follow this order:

1. **HEBBS insights** (`hebbs-cli insights`) for consolidated, high-confidence knowledge
2. **HEBBS recall** (`hebbs-cli recall`) for raw memories matching the query
3. **File memory** (memory_search, MEMORY.md, memory/*.md) only if HEBBS returns nothing or is unreachable
4. If all sources are empty, say so. Do not guess or hallucinate history.

When storing new information:

1. Write to HEBBS immediately (`hebbs-cli remember`)
2. Optionally mirror critical facts to file memory for durability

## Capability tiers

HEBBS works in layers. Each layer works independently:

| Tier | What works | Requirements |
|---|---|---|
| **Basic memory** | `remember`, `recall`, `forget`, `prime` | Server running |
| **Semantic recall** | Similarity, temporal, causal, analogical strategies | Server running (ONNX embedder included) |
| **Reflection / insights** | `reflect-prepare` + `reflect-commit`, `insights` | Server running + agent acts as the LLM (no API key needed) |

All tiers work out of the box with just the server running. No external LLM or API key is required — the agent (you) provides the reasoning for reflection.

## Before every command

Check the server is healthy before running any `hebbs-cli` command:

```
hebbs-cli status --format json 2>&1
```

If `status` shows `SERVING`, proceed normally.

If `status` fails with a connection error or shows `NOT_SERVING`, follow this startup sequence:

1. Ensure the data directory exists:
```
mkdir -p ~/.hebbs/data
```

2. Start the server (prefer `brew services` if available, otherwise nohup):
```
brew services start hebbs 2>/dev/null || HEBBS_AUTH_ENABLED=false nohup hebbs-server start --data-dir ~/.hebbs/data > /tmp/hebbs-server.log 2>&1 &
```

3. The ONNX embedding model takes time to initialize on first start. Retry status up to 5 times with 5-second intervals:
```
for i in 1 2 3 4 5; do sleep 5; hebbs-cli status --format json 2>&1 && break; done
```

4. If still not serving after 25 seconds, check the log:
```
cat /tmp/hebbs-server.log | tail -20
```

Common causes: data directory missing or unwritable, port 6380 already in use.

Do not proceed with other HEBBS commands until `status` shows `SERVING`.

## Prerequisites

Install HEBBS via Homebrew or the install script:

```
brew install hebbs-ai/tap/hebbs
```

Or on any platform (Linux, macOS):

```
curl -sSf https://hebbs.ai/install | sh
```

## Quick setup (one command)

Install, start, and verify in one shot:

```
brew install hebbs-ai/tap/hebbs && brew services start hebbs && sleep 10 && hebbs-cli status --format json
```

If `brew services` is not available (e.g., non-Homebrew install), use:

```
mkdir -p ~/.hebbs/data && HEBBS_AUTH_ENABLED=false nohup hebbs-server start --data-dir ~/.hebbs/data > /tmp/hebbs-server.log 2>&1 & sleep 10 && hebbs-cli status --format json
```

The server listens on gRPC port 6380 and HTTP port 6381. Data is stored in `~/.hebbs/data` (Homebrew) or the configured `--data-dir`.

## Operations

| Situation | Operation | Command |
|---|---|---|
| User shares a fact, preference, or decision | Store it | `hebbs-cli remember` |
| User asks a question about past context | Retrieve it | `hebbs-cli recall` |
| User corrects something you said or stored | Store the correction (importance 0.9) | `hebbs-cli remember` |
| Start of a new conversation | Load context | `hebbs-cli prime` |
| Want consolidated patterns from many memories | Get distilled knowledge | `hebbs-cli insights` |
| 20+ raw memories accumulated for an entity | Consolidate into insights | `hebbs-cli reflect-prepare` + `reflect-commit` |
| Outdated or wrong memories need cleanup | Remove them | `hebbs-cli forget` |

## Commands

### Remember — store a memory

```
hebbs-cli remember "The user prefers dark mode in all applications" --importance 0.8 --entity-id user_prefs --format json
```

> **Always use `--format json` when you need the memory ID** (e.g. for `--edge` on a subsequent `remember`). Extract the ID with: `jq -r '.memory_id'`
>
> **Warning:** Capture the memory ID from `--format json` output **before** referencing it in `--edge`. Do not parse IDs from human-format output.

Flags:
- `--importance <0.0-1.0>` — how important this memory is (default 0.5). Use 0.8+ for user preferences, decisions, corrections. Use 0.3 for transient observations.
- `--entity-id <id>` — group memories by entity (e.g. `user_prefs`, `project_alpha`, a person's name). Omit for global context.
- `--context <json>` — arbitrary metadata as JSON object (e.g. `'{"source":"email","topic":"billing"}'`).
- `--edge <TARGET_ID:EDGE_TYPE[:CONFIDENCE]>` — link to another memory (repeatable). Types: `caused_by`, `related_to`, `followed_by`, `revised_from`, `insight_from`. Use to build causal chains and lineage. **Shell quoting:** use `"${MEM_ID}:edge_type"` — bare `$MEM_ID:edge_type` triggers zsh variable modifier expansion.

### Recall — retrieve relevant memories

```
hebbs-cli recall "What does the user prefer for UI themes?" --strategy similarity --top-k 5 --format json
```

Four strategies — pick based on what you need:

| Strategy | When to use | Example |
|---|---|---|
| `similarity` | Find memories related to a topic | "What do we know about deployment?" |
| `temporal` | Get recent activity for an entity | "What happened today with project X?" |
| `causal` | Trace cause-effect chains from a memory | "What led to this decision?" |
| `analogical` | Find structurally similar patterns | "Have we seen a problem like this before?" |

**Core flags:**
- `--strategy <similarity|temporal|causal|analogical>` — recall strategy (default: similarity).
- `--top-k <n>` — max results (default 10).
- `--entity-id <id>` — scope to entity (required for temporal).
- `--format json` — machine-readable output.

**Scoring weights** — control how results are ranked. The composite score blends four signals: `relevance × recency × importance × reinforcement`. Default weights are `0.5:0.2:0.2:0.1`.
- `--weights <R:T:I:F>` — four colon-separated floats.
- `1:0:0:0` — pure semantic similarity (ignore recency, importance, reinforcement).
- `0.2:0.8:0:0` — heavily favor recent memories.
- `0.3:0.1:0.5:0.1` — prioritize high-importance memories (user preferences, decisions).

Only `cue` and `--strategy` are required. All other flags use smart defaults suitable for most workloads — tune only when you have a specific reason.

**Strategy-specific flags:**

| Flag | Strategy | Default | Description |
|---|---|---|---|
| `--ef-search <n>` | similarity | 50 | HNSW search quality. Higher = more accurate, slower. |
| `--time-range <START:END>` | temporal | unbounded | Microsecond timestamps. Omit for newest-first up to top_k. |
| `--seed <hex_id>` | causal | auto-detect | Starting memory for graph traversal. Omit to auto-pick by cue. |
| `--max-depth <n>` | causal | 5 (max 10) | Maximum hops from seed memory. |
| `--edge-types <types>` | causal | all | Comma-separated: `caused_by,followed_by,related_to,revised_from,insight_from`. |
| `--analogical-alpha <0-1>` | analogical | 0.5 | 0.0 = pure structural similarity, 1.0 = pure embedding similarity. |

### Reflect (two-step, agent-driven)

HEBBS supports a single `reflect` command that runs the full reflection cycle server-side (clustering → LLM proposal → validation → commit). However, OpenClaw exposes the two-step `reflect-prepare` + `reflect-commit` flow so that **you (the agent) are the LLM**. This lets users keep control over which model reasons about their memories, rather than requiring a server-side LLM configuration.

No server-side LLM is needed for this flow. HEBBS does the clustering and prompt construction; you read the clusters, reason about them, and commit insights.

**Step 1: Prepare**

```
hebbs-cli reflect-prepare --entity-id user_prefs --format json
```

Returns JSON with:
- `session_id` — pass this to step 2
- `clusters` — groups of related memories, each with:
  - `memories` — full memory content for this cluster (id, content, importance, entity_id, created_at). **Read these to understand what the cluster is about.**
  - `proposal_system_prompt` + `proposal_user_prompt` — pre-built prompts you can send to your LLM to generate insight candidates
  - `memory_ids` — source memory IDs (hex-encoded)
  - `validation_context` — additional data for validating proposed insights

**Step 2: Reason and commit**

After calling your LLM with the proposal prompts and optionally validating the results:

```
hebbs-cli reflect-commit --session-id <id> --insights '[{"content":"Users consistently prefer dark themes","confidence":0.9,"source_memory_ids":["aabb...","ccdd..."],"tags":["preference","ui"]}]'
```

Each insight needs:
- `content` — the consolidated insight text
- `confidence` — 0.0 to 1.0
- `source_memory_ids` — hex-encoded IDs. **Use the `memory_ids` array from the cluster**, not `memories[].memory_id` (which is a ULID and will be rejected).
- `tags` — categorical labels

Reflection requires at least 5 memories for an entity to produce clusters. If `clusters` is empty, accumulate more memories before retrying.

Sessions expire after 10 minutes.

### Insights — retrieve consolidated knowledge

```
hebbs-cli insights --entity-id user_prefs --max-results 10 --min-confidence 0.7 --format json
```

Flags:
- `--entity-id <id>` — filter by entity.
- `--max-results <n>` — maximum insights to return.
- `--min-confidence <0.0-1.0>` — only return insights above this confidence threshold.

Check insights before recalling raw memories — they represent distilled, validated knowledge.

### Forget — remove memories

```
hebbs-cli forget --ids <hex_id1> --ids <hex_id2>
hebbs-cli forget --entity-id old_project
hebbs-cli forget --staleness-us 2592000000000  # older than 30 days
hebbs-cli forget --kind insight --decay-floor 0.1  # remove low-value decayed insights
```

Flags (combine as needed — at least one filter required):
- `--ids <id>` — specific memory IDs to forget (repeatable).
- `--entity-id <id>` — scope to entity.
- `--staleness-us <microseconds>` — remove memories older than this.
- `--kind <episode|insight|revision>` — filter by memory kind.
- `--decay-floor <0.0-1.0>` — remove memories with decay score below this.
- `--access-floor <n>` — remove memories with access count below this.

### Prime — warm context for an entity

```
hebbs-cli prime user_prefs --max-memories 20 --similarity-cue "project status and preferences"
```

Flags:
- `--max-memories <n>` — maximum memories to return.
- `--similarity-cue <text>` — bias the selection toward memories related to this text. Very useful for loading context relevant to a specific topic rather than just recent activity.
- `--recency-us <microseconds>` — only include memories within this time window.
- `--context <json>` — additional context as JSON.

Returns a blend of recent + relevant memories for an entity. Use at the start of a conversation to load context.

## Decision guide

1. **Start of conversation**: Always `hebbs-cli prime <entity>` to load context. Do this before the first reply.
2. **Before answering any question about past context**: `hebbs-cli recall` with the question as cue. Do not answer from general knowledge when HEBBS might have the answer.
3. **User shares a fact, preference, or decision**: `hebbs-cli remember` immediately with appropriate importance (0.8+ for preferences and decisions).
4. **User corrects something**: `hebbs-cli remember` the correction with importance 0.9. Old conflicting memories will naturally decay.
5. **After 20+ new memories on an entity**: `hebbs-cli reflect-prepare` + `reflect-commit` to consolidate into insights.
6. **Periodic maintenance**: `hebbs-cli insights` to review, `hebbs-cli forget` to clean stale data.

## Output format

Always use `--format json` when parsing output programmatically. Human format is for display only.

## Connection

Default endpoint: `localhost:6380` (gRPC). Override with `--endpoint <host:port>`.
HTTP endpoint (metrics/health): `localhost:6381`.
