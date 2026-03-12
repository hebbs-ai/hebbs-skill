---
name: hebbs
description: Cognitive memory engine — remember, recall, reflect, and forget knowledge with HEBBS.
homepage: https://hebbs.dev
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

## Before every command

Check the server is reachable before running any `hebbs-cli` command:

```
hebbs-cli recall "test" --format json 2>&1
```

If you see a connection error, start the server in the background:

```
HEBBS_AUTH_ENABLED=false nohup hebbs-server --data-dir ~/.hebbs/data > /tmp/hebbs-server.log 2>&1 &
```

Wait 2 seconds, then retry. Do not proceed until the server responds.

## Prerequisites

Install HEBBS via Homebrew or the install script:

```
brew install hebbs-ai/tap/hebbs
```

Or on any platform (Linux, macOS):

```
curl -sSf https://hebbs.ai/install | sh
```

The server must be running before using any `hebbs-cli` command. Start it with auth disabled:

```
HEBBS_AUTH_ENABLED=false hebbs-server --data-dir ~/.hebbs/data
```

To run in the background:

```
HEBBS_AUTH_ENABLED=false nohup hebbs-server --data-dir ~/.hebbs/data > /tmp/hebbs-server.log 2>&1 &
```

This starts the gRPC server on port 6380 and HTTP on port 6381. Data is stored in `~/.hebbs/data`.

Before running commands, verify the server is reachable: `hebbs-cli recall "test" --format json 2>&1`. If connection is refused, the server is not running.

## When to use HEBBS

- **Remember** facts, decisions, preferences, observations, or any text the user might want retrieved later.
- **Recall** relevant context before answering questions or making decisions.
- **Reflect** periodically to consolidate raw memories into durable insights.
- **Forget** outdated or irrelevant memories.

## Commands

### Remember — store a memory

```
hebbs-cli remember "The user prefers dark mode in all applications" --importance 0.8 --entity-id user_prefs
```

Flags:
- `--importance <0.0-1.0>` — how important this memory is (default 0.5). Use 0.8+ for user preferences, decisions, corrections. Use 0.3 for transient observations.
- `--entity-id <id>` — group memories by entity (e.g. `user_prefs`, `project_alpha`, a person's name). Omit for global context.
- `--context <json>` — arbitrary metadata as JSON object (e.g. `'{"source":"email","topic":"billing"}'`).
- `--edge <TARGET_ID:EDGE_TYPE[:CONFIDENCE]>` — link to another memory (repeatable). Types: `caused_by`, `related_to`, `followed_by`, `revised_from`, `insight_from`. Use to build causal chains and lineage.

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

Use this when enough memories have accumulated and you want to consolidate them into insights. This is a two-step process where HEBBS does the math and you do the thinking.

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
- `source_memory_ids` — hex-encoded IDs from the cluster (must be from the prepare output)
- `tags` — categorical labels

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

1. **Start of conversation**: `hebbs-cli prime <entity>` or `hebbs-cli recall` with the user's first message.
2. **User shares a fact/preference/decision**: `hebbs-cli remember` with appropriate importance.
3. **Before answering a question**: `hebbs-cli recall` with the question as cue.
4. **After 20+ new memories on an entity**: `hebbs-cli reflect-prepare` + `reflect-commit` to consolidate.
5. **User corrects something**: `hebbs-cli remember` the correction with high importance (0.9). Old conflicting memories will naturally decay.
6. **Periodic maintenance**: `hebbs-cli insights` to review, `hebbs-cli forget` to clean stale data.

## Output format

Always use `--format json` when parsing output programmatically. Human format is for display only.

## Connection

Default endpoint: `localhost:6380` (gRPC). Override with `--endpoint <host:port>`.
HTTP endpoint (metrics/health): `localhost:6381`.
