# Long-Form Memory: Retaining and Recalling Information Across 1,000+ Turns in Real-Time AI Systems

> **Reference Articles**
> - [How Clawdbot Remembers Everything](https://manthanguptaa.in/posts/clawdbot_memory/)
> - [I Reverse Engineered Claude's Memory System](https://manthanguptaa.in/posts/claude_memory/)
> - [Towards Human-Like Memory for AI Agents](https://manthanguptaa.in/posts/towards_human_like_memory_for_ai_agents/)

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Foundational Philosophy](#2-foundational-philosophy)
3. [Architecture Overview](#3-architecture-overview)
4. [Memory Layers](#4-memory-layers)
5. [Storage Stack (Mid-Scale Production)](#5-storage-stack-mid-scale-production)
6. [Per-Turn Pipeline](#6-per-turn-pipeline)
7. [Extraction Algorithm](#7-extraction-algorithm)
8. [Retrieval Algorithm](#8-retrieval-algorithm)
9. [Prompt Composition & Injection](#9-prompt-composition--injection)
10. [Consolidation & Forgetting](#10-consolidation--forgetting)
11. [Edge Cases & Failure Handling](#11-edge-cases--failure-handling)
12. [Tunable Parameters](#12-tunable-parameters)
13. [Latency Budget](#13-latency-budget)
14. [Evaluation & Testing Strategy](#14-evaluation--testing-strategy)
15. [Cost Estimates](#15-cost-estimates)
16. [Implementation Phases](#16-implementation-phases)

---

## 1. Problem Statement

### The Challenge

Modern AI systems reason well within short context windows but fail when conversations span hundreds or thousands of turns. Critical information shared early is forgotten as the conversation grows.

**Example:**
- Turn 1: "My preferred language is Kannada"
- Turn 937: "Can you call me tomorrow?"
- The system must still recall the language preference, time constraints, and prior commitments — without replaying the full history.

### Why LLMs Alone Can't Solve This

- Limited context windows
- Cannot replay full conversation history at scale
- Forget early information as conversations grow
- Become slow and expensive when full history is repeatedly injected

### Constraints

- Full conversation replay is **not allowed**
- Unlimited prompt growth is **not allowed**
- Manual tagging is **not allowed**
- Must be **fully automated**
- Must support **1,000+ turns**
- Must operate in **real time** (< 100ms retrieval overhead)

### Objective

Build a system where information introduced at turn 1 can be accurately recalled and applied at turn 100 or turn 1,000, without replaying the full conversation and without increasing system latency.

---

## 2. Foundational Philosophy

### Key Insights from Reference Articles

**From Clawdbot's Memory System:**
- Memory should be **local, transparent, and human-editable**
- Store as readable files (Markdown), not opaque databases
- Give users full control and visibility over what the agent remembers

**From Claude's Memory (Reverse Engineered):**
- Memory retrieval should be **selective and on-demand**
- Not everything gets injected every turn — the system decides when to access memory
- Prompt structure: System Prompt → User Memories → Conversation History → Current Message

**From Human-Like Memory for AI Agents:**
- Most AI memory systems are caches, not true memory
- Human memory is a pipeline: **Encoding → Storage → Consolidation → Retrieval → Forgetting**
- All five stages must be implemented — most systems only do storage + retrieval
- **Forgetting is essential** — a system that remembers everything remembers nothing useful

### Core Principle

> **Context ≠ Memory**
> - Context is ephemeral — what the model sees this request (bounded by token window)
> - Memory is persistent — what survives across sessions, restarts, and model changes
> - They serve different purposes and must be designed separately

---

## 3. Architecture Overview

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   EXTRACT   │───▶│    STORE     │───▶│   RETRIEVE   │───▶│   INJECT    │───▶│   RESPOND   │
│  (per turn) │    │ (persistent) │    │ (per turn)   │    │ (to prompt) │    │  (LLM call) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

Five subsystems working in a pipeline:
1. **Extract** — Identify what's worth remembering from the current turn
2. **Store** — Persist memories across sessions and restarts
3. **Retrieve** — Find relevant memories at inference time
4. **Inject** — Compose memories into the prompt without overloading it
5. **Respond** — Generate the response with full memory context

A sixth background process runs asynchronously:
6. **Consolidate** — Merge, decay, and prune memories periodically

---

## 4. Memory Layers

The system uses five distinct memory layers, inspired by the human memory model:

### Layer 1: Core Memory (Persistent Identity)

- Stable, slowly-evolving representation of the user
- Contains: name, language, timezone, personality traits, long-term instructions
- Stored as human-readable Markdown files (Clawdbot-style)
- **Always injected** into every prompt — the agent's sense of "who it's talking to"
- Updated rarely, only with high confidence
- Size budget: ~200–500 tokens

### Layer 2: Sensory Memory (Per-Turn Filter)

- Lightweight, fast filter on every incoming message
- Decides: "Is there anything worth encoding?"
- Filters out greetings, acknowledgments, filler
- Eliminates ~60% of turns before any expensive processing
- Keeps extraction cost low at 1,000+ turns

### Layer 3: Short-Term Memory (Working Context)

- Current conversation window — last 5–10 turns
- Provides immediate context for reasoning
- Ephemeral, bounded by context window
- Exists only for the current request

### Layer 4: Long-Term Memory (Persistent Store)

- Information that survived the sensory filter and was deemed important
- Stored in two parallel structures:
  - **Key-value store** — for exact, factual lookups
  - **Vector store** — for semantic, meaning-based retrieval
- Persists across sessions, restarts, and model changes

### Layer 5: Forgetting & Consolidation (Memory Hygiene)

- Background process that merges duplicates, decays unused memories, prunes low-confidence records
- Promotes frequently-accessed memories to Core Memory
- Resolves contradictions
- Ensures the memory store gets **sharper** over time, not noisier

---

## 5. Storage Stack (Mid-Scale Production)

### Chosen Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Flat Files** (Markdown) | Local disk | Core memory — always injected, human-readable |
| **Redis** | Managed service | Structured key-value store — fast exact lookups, deduplication, recency tracking |
| **pgvector / Qdrant** | Managed service or Postgres extension | Vector store — semantic similarity search |

**Total infrastructure: 2 managed services** (Redis + Postgres/Qdrant)

### pgvector vs Qdrant Decision

| Factor | pgvector | Qdrant |
|---|---|---|
| Choose if | Already running Postgres | Want a dedicated vector engine |
| Infrastructure | Zero new services (just an extension) | One new service |
| Performance | Good up to ~1M vectors | Excellent, purpose-built |
| Filtering | Full SQL power | Rich metadata filtering |

**Rule of thumb:** Already using Postgres → pgvector. Not using Postgres → Qdrant.

### Flat File Structure

```
memory/
├── CORE.md              ← User identity (always injected)
├── PREFERENCES.md       ← Standing preferences
├── INSTRUCTIONS.md      ← Long-term behavioral rules
├── CONSTRAINTS.md       ← Hard constraints (always injected)
└── SESSION_LOG.md       ← Compressed session summaries
```

**Example CORE.md:**
```markdown
- Name: Gaurav
- Preferred Language: Kannada
- Timezone: IST (UTC+5:30)
- Role: Developer
- First interaction: 2026-01-15
```

### Redis Data Structures

| Redis Type | Purpose | Example |
|---|---|---|
| **Hash** | Full memory record | `HSET mem:mem_a1b2 type preference key call_time value "after 11 AM" ...` |
| **String** | Fast dedup lookup | `SET preference:call_time mem_a1b2` |
| **Sorted Set** | Recency-ordered index | `ZADD recent_memories <timestamp> mem_a1b2` |
| **Set** | Type-based index | `SADD type:constraint mem_a1b2 mem_c3d4` |

**Persistence:** Enable AOF (Append Only File) to survive Redis restarts.

### Vector Store Design

Each memory stored as:
- A **1536-dimensional embedding** (from embedding model)
- **Metadata** for filtering (memory type, source turn, confidence)
- A **reference ID** linking back to the full record in Redis

The vector store does NOT store full records — only enough to search and filter. Full records live in Redis. Each system does what it's best at.

### Memory Record Schema

```json
{
  "memory_id": "mem_a1b2c3d4",
  "type": "preference | fact | entity | constraint | commitment | instruction",
  "key": "call_time",
  "value": "after 11 AM",
  "source_turn": 1,
  "confidence": 0.93,
  "created_at": "2026-01-15T10:30:00Z",
  "last_accessed_turn": 412,
  "access_count": 87,
  "supersedes": null,
  "embedding": [0.12, -0.45, 0.78, ...]
}
```

---

## 6. Per-Turn Pipeline

### Write Path (Async — does not block response)

```
User Message (Turn N)
       │
       ▼
  Sensory Filter ──── "Worth remembering?" (heuristic, ~0ms)
       │ (if yes)
       ▼
  Extraction ──── Extract structured memories (small LLM, ~200ms)
       │
       ▼
  Deduplication ──── Already exists? Update vs Create (Redis, ~1ms)
       │
       ▼
  Storage ──── Write to Redis + pgvector/Qdrant (~10ms)
       │
       ▼
  (Optional) Promote to flat file if identity-level
```

### Read Path (Sync — blocks response, must be fast)

```
  1. Read CORE.md ────────────────────▶ Always injected (~1ms)
  2. Embed current message ────────────▶ Generate vector (~20ms)
  3. Query pgvector/Qdrant ────────────▶ Top-K semantic matches (~25ms)
  4. Fetch always-on from Redis ───────▶ Constraints + Instructions (~1ms)
  5. Fetch recency set from Redis ─────▶ Recently accessed memories (~1ms)
  6. Hydrate full records from Redis ──▶ Get complete memory data (~2ms)
  7. Merge & Rank ─────────────────────▶ Score, budget, select (~1ms)
  8. Compose prompt ───────────────────▶ Inject into LLM prompt
```

**Total read path latency: ~51ms**

---

## 7. Extraction Algorithm

### Three-Stage Extraction Funnel

Most messages don't contain memorable information. The funnel ensures only signal-rich turns get expensive LLM processing.

```
ALL MESSAGES (1,000+ turns)
       │
  STAGE 1: HEURISTIC GATE ──── Cost: ~0ms, no LLM call, eliminates ~60%
       │ (~40% pass)
  STAGE 2: CLASSIFIER ────────  Cost: ~5ms, pattern matching, eliminates ~20% more
       │ (~20% pass)
  STAGE 3: LLM EXTRACTOR ────  Cost: ~200ms, small LLM call, processes only ~20% of turns
       │
  MEMORY RECORDS (written to storage)
```

### Stage 1: Heuristic Gate

**Zero cost, zero latency.** Pure rule-based filtering.

**Auto-REJECT (never reach Stage 2):**

| Pattern | Examples |
|---|---|
| Ultra-short acknowledgments | "ok", "sure", "thanks", "yes", "no" |
| Greetings / farewells | "hi", "hello", "bye", "good morning" |
| Repetition of agent output | "yeah that's what I said" |
| Pure questions with no facts | "What time is it?", "How does this work?" |
| Short commands (< 4 words, no entities) | "do it", "go ahead", "next one" |

**Auto-PASS (always reach Stage 2):**

| Signal | Examples |
|---|---|
| Time/date references | "after 11 AM", "next Thursday", "every Monday" |
| Preference language | "I prefer", "I like", "I always", "don't ever" |
| Named entities | "my colleague Priya", "the dashboard project" |
| Self-referential facts | "I'm a developer", "I live in Bangalore" |
| Explicit memory instructions | "always respond in...", "remember that..." |
| Long messages (> 30 words) | Any substantial message |

### Stage 2: Lightweight Classifier

Runs on messages that passed Stage 1. Classifies the **memory type** before committing to LLM extraction.

**Pattern-based approach (~2ms, no ML):**

| Pattern Match | Classified As | Confidence |
|---|---|---|
| "I prefer / I like / I want / I always" | PREFERENCE | 0.8 |
| "Don't ever / Never / Always make sure" | CONSTRAINT | 0.85 |
| "I am / I'm a / I work at / I live in" | FACT | 0.8 |
| "Remember that / From now on / Going forward" | INSTRUCTION | 0.9 |
| "Call me at / Meet on / Deadline is" | COMMITMENT | 0.8 |
| Detected names, places, organizations | ENTITY | 0.7 |
| None of the above | UNCERTAIN | — |

If classified with confidence ≥ 0.7, pass to Stage 3 with the type hint. If UNCERTAIN, still pass — let the LLM decide.

### Stage 3: LLM Extractor

Only ~20% of messages reach here. Uses a **small, fast LLM** (GPT-4o-mini, Claude Haiku, or equivalent).

**Extractor input:**
- Current message
- Type hint from Stage 2
- Last 3 turns of conversation (for context)
- Existing memories with the same type hint (dedup awareness)

**Extractor output (structured JSON):**
```json
[
  {
    "type": "preference",
    "key": "call_time",
    "value": "after 11 AM",
    "confidence": 0.93,
    "is_update": false,
    "reasoning": "User explicitly stated call time preference"
  }
]
```

Returns `[]` if nothing is worth storing.

**Extraction rules:**

| Rule | Rationale |
|---|---|
| One memory per distinct fact | "I prefer Kannada and calls after 11 AM" → TWO records |
| Normalize the key | "call me after 11" and "phone calls after 11 AM" → same key: `call_time` |
| Capture semantic value, not raw text | "yeah so like don't call me before eleven ya know" → "Prefers calls after 11 AM" |
| Flag updates explicitly | User changing a preference → set `is_update: true` |
| Confidence based on explicitness | "I prefer X" (0.95) vs. "I guess X" (0.6) vs. inferred (0.5) |
| Return empty if uncertain | Better to miss a memory than hallucinate one |

### Post-Extraction: Deduplication

Before writing to storage:

```
New extraction: {type: "preference", key: "call_time", value: "after 2 PM"}
       │
  Query Redis: GET preference:call_time
       │
       ├── NOT FOUND → Create new record in Redis + vector store
       │
       └── FOUND (existing: "after 11 AM") → This is an UPDATE
              │
              ├── New is more recent AND confident → SUPERSEDE
              │   (update Redis, re-embed in vector store, mark old as superseded)
              │
              └── Ambiguous → Store BOTH, let consolidation worker resolve
```

---

## 8. Retrieval Algorithm

### Multi-Signal Retrieval

The retriever uses **three parallel signals** and merges them into a single ranked list.

### Signal 1: Semantic Similarity

- Embed the current user message
- Query pgvector/Qdrant for top 20 nearest neighbors (deliberate over-fetch)
- Each result returns a cosine similarity score (0.0 to 1.0)

**Thresholds:**

| Score | Interpretation | Action |
|---|---|---|
| 0.85 – 1.0 | Highly relevant | Strong candidate |
| 0.70 – 0.85 | Possibly relevant | Include if other signals agree |
| 0.50 – 0.70 | Weak match | Discard unless type-boosted |
| Below 0.50 | Irrelevant | Always discard |

### Signal 2: Type-Based Always-On

Certain memory types are **always retrieved**, every turn, regardless of similarity:

- All memories of type **CONSTRAINT** (safety-critical)
- All memories of type **INSTRUCTION** (behavioral rules)
- Core identity facts from **CORE.md** flat file

**Implementation:** Redis SET lookups (`SMEMBERS type:constraint`, `SMEMBERS type:instruction`)

**Budget:** Reserve 4–5 slots out of the total budget for always-on memories.

### Signal 3: Recency-Weighted

Boosts memories recently accessed or created. Uses Redis sorted sets ordered by `last_accessed_turn`.

**Recency decay formula:**
```
recency_score = 1.0 / (1.0 + α × (current_turn - last_accessed_turn))

α = 0.02 (tunable)

Examples:
  5 turns ago   → 0.91
  50 turns ago  → 0.50
  500 turns ago → 0.09
```

### Merge & Rank Algorithm

```
STEP 1: COLLECT candidates from all three signals
STEP 2: UNION and deduplicate by memory_id
STEP 3: COMPUTE final score for each candidate:

  final_score = (W_sem  × semantic_score)
              + (W_type × type_priority_score)
              + (W_rec  × recency_score)
              + (W_freq × frequency_score)
              + (W_conf × confidence)

STEP 4: WEIGHTS
  W_sem  = 0.40  (semantic similarity)
  W_type = 0.25  (type priority)
  W_rec  = 0.15  (recency)
  W_freq = 0.10  (access frequency)
  W_conf = 0.10  (extraction confidence)

STEP 5: TYPE PRIORITY SCORES
  CONSTRAINT  → 1.0  (highest)
  INSTRUCTION → 0.9
  COMMITMENT  → 0.8
  PREFERENCE  → 0.7
  FACT        → 0.5
  ENTITY      → 0.4

STEP 6: FREQUENCY SCORE
  frequency_score = min(1.0, access_count / 50)

STEP 7: SORT by final_score descending
STEP 8: BUDGET — keep top K (K = 10–12), hard cap ~500 tokens
```

### Worked Example: Turn 937

**User:** "Can you call me tomorrow?"

**Semantic results:**
| Memory | Score |
|---|---|
| "prefers calls after 11 AM" | 0.89 |
| "timezone is IST" | 0.82 |
| "demo scheduled for Thursday" | 0.71 |

**Always-on:** "always respond in Kannada", "use formal tone"

**Recency:** "demo scheduled for Thursday" (last accessed turn 920)

**After merge & rank (top 6 kept):**
1. "demo scheduled for Thursday" — 0.685
2. "prefers calls after 11 AM" — 0.672
3. "timezone is IST" — 0.615
4. "prefers Kannada language" — 0.551
5. "always respond in Kannada" — 0.495
6. "use formal tone" — 0.440

**Result:** Agent schedules call after 11 AM IST, avoids Thursday conflict, responds in Kannada with formal tone — all from a simple "Can you call me tomorrow?" at turn 937.

---

## 9. Prompt Composition & Injection

Following the structure reverse-engineered from Claude's memory system:

```
┌─────────────────────────────────────────────┐
│  1. SYSTEM PROMPT (static instructions)      │  ~200 tokens
├─────────────────────────────────────────────┤
│  2. CORE MEMORY (always present)             │  ~200 tokens
│     User identity, standing preferences      │
│     Active constraints & instructions        │
├─────────────────────────────────────────────┤
│  3. RETRIEVED MEMORIES (dynamic, per-turn)   │  ~400 tokens
│     Top-K relevant from retrieval algorithm  │
│     Formatted as concise bullet points       │
├─────────────────────────────────────────────┤
│  4. SHORT-TERM CONTEXT (last 5–10 turns)     │  ~500 tokens
├─────────────────────────────────────────────┤
│  5. CURRENT USER MESSAGE                     │  ~50 tokens
└─────────────────────────────────────────────┘
  TOTAL: ~1,350 tokens (stable regardless of turn count)
```

**Key design principle:** Memories are injected **on-demand, not all at once**. The model sees only 10–15 relevant memories, never the full store of hundreds.

---

## 10. Consolidation & Forgetting

Runs **asynchronously** (not on the inference path). Triggered every ~50 turns or on session end.

### Operations

| Operation | What It Does | Example |
|---|---|---|
| **Merge** | Combine overlapping memories | "likes coffee" + "prefers black coffee" → "prefers black coffee" |
| **Supersede** | Replace outdated info | "call after 11 AM" → "call after 2 PM" (newer) |
| **Decay** | Reduce confidence of unused memories | Not accessed in 200+ turns → confidence -= 0.1 |
| **Prune** | Delete memories below threshold | Confidence < 0.3 after decay → remove from all stores |
| **Promote** | Elevate to core memory | Accessed 50+ times, confidence > 0.9 → append to CORE.md |
| **Reindex** | Rebuild Redis sorted sets | Ensure recency ordering is accurate |

### Promotion Rules

A memory gets promoted from Redis → Flat File when:
- Accessed **50+ times**
- Confidence **> 0.9**
- Type is `preference`, `instruction`, or `constraint`
- It affects **every response** (language, tone, format)

---

## 11. Edge Cases & Failure Handling

### Contradictory Memories

```
mem_a1: "prefers calls after 11 AM"  (turn 1, confidence 0.93)
mem_x9: "prefers calls after 2 PM"   (turn 400, confidence 0.91)
```

**Resolution:** Prefer the more recent one, UNLESS the older has significantly higher confidence (> 0.15 gap). Inject only the winner. Flag for consolidation.

### No Relevant Memories Found

All semantic scores below threshold, no always-on memories exist yet.

**Resolution:** Inject only CORE.md content and proceed. Graceful degradation to memoryless response. Never hallucinate memories.

### Too Many Relevant Memories

15+ memories score above 0.85, but budget is 12.

**Resolution:**
1. Apply diversity penalty — same type + similar key → keep only the higher-scoring one
2. Cluster by topic — pick best representative per cluster
3. Hard cut at K=12

### User Requests Forgetting

"Forget what I said about call times"

**Resolution:**
1. Detect DELETION INTENT in extraction
2. Search for matching memories
3. Mark as DELETED in Redis (don't hard-delete — audit trail)
4. Remove vectors from pgvector/Qdrant
5. Memory is never retrieved again

### Hypothetical Statements

"If I were allergic to shellfish, that would be a problem"

**Resolution:** Hypotheticals should either not be extracted, or be extracted with very low confidence (< 0.4) and thus filtered during retrieval. The extraction LLM must distinguish hypothetical from factual.

---

## 12. Tunable Parameters

### Extraction Parameters

| Parameter | Default | Increase Effect | Decrease Effect |
|---|---|---|---|
| Heuristic word threshold | 4 words | Fewer reach Stage 2 | More reach Stage 2 |
| Extraction confidence threshold | 0.6 | Fewer, higher-quality memories | More, noisier memories |
| Deduplication similarity threshold | 0.92 | More duplicates allowed | More aggressive merging |

### Retrieval Parameters

| Parameter | Default | Increase Effect | Decrease Effect |
|---|---|---|---|
| Semantic top-K (over-fetch) | 20 | Wider candidate pool | Narrower, might miss |
| Semantic similarity threshold | 0.50 | Fewer candidates pass | More noise |
| Memory budget (final K) | 12 | More context, higher cost | Leaner prompts |
| Token budget | 500 | Richer memory context | Faster, cheaper |
| W_sem (semantic weight) | 0.40 | Meaning matters more | Other signals dominate |
| W_type (type weight) | 0.25 | Constraints dominate | Semantic dominates |
| W_rec (recency weight) | 0.15 | Recent memories favored | Old treated equally |
| Recency decay α | 0.02 | Faster forgetting | Slower forgetting |

---

## 13. Latency Budget

### Extraction (Async — invisible to user)

| Stage | Latency |
|---|---|
| Heuristic gate | ~0.1ms |
| Classifier | ~5ms |
| LLM extraction | ~200ms |
| Dedup check (Redis) | ~1ms |
| Write to Redis + vector store | ~10ms |
| **Total** | **~216ms (async)** |

### Retrieval (Sync — blocks response)

| Step | Latency |
|---|---|
| Read CORE.md | ~1ms |
| Embed current message | ~20ms |
| Query pgvector/Qdrant | ~25ms |
| Fetch always-on (Redis) | ~1ms |
| Fetch recency set (Redis) | ~1ms |
| Hydrate full records (Redis) | ~2ms |
| Merge & Rank | ~1ms |
| **Total** | **~51ms** |

**Key insight:** Extraction is fire-and-forget after response. Retrieval must complete before LLM generates. Extraction latency is invisible to the user.

---

## 14. Evaluation & Testing Strategy

### Five Evaluation Dimensions

| Dimension | Core Question |
|---|---|
| **Recall Accuracy** | Did it remember what it should? |
| **Precision & Noise** | Did it inject only relevant memories? |
| **Latency & Cost** | Was it fast and cheap enough? |
| **Safety & Trust** | Did it make things up or poison memories? |
| **Decay Quality** | Does it age well? |

### Test Design: Planted Memory Recall

```
Phase 1 — PLANT: Inject a verifiable fact at turn P
Phase 2 — NOISE: Simulate N unrelated turns
Phase 3 — PROBE: Ask a question requiring the planted fact at turn P+N+1
Phase 4 — EVALUATE: Was the memory retrieved correctly?
```

### Distance Sweep Tests

| Plant Turn | Probe Turn | Distance | Recall Target |
|---|---|---|---|
| 1 | 11 | 10 turns | ≥ 95% |
| 1 | 51 | 50 turns | ≥ 95% |
| 1 | 101 | 100 turns | ≥ 95% |
| 1 | 251 | 250 turns | ≥ 90% |
| 1 | 501 | 500 turns | ≥ 90% |
| 1 | 1001 | 1000 turns | ≥ 85% |

### Memory Type Coverage Tests

| Planted Memory | Type | Probe Question |
|---|---|---|
| "I prefer dark mode" | PREFERENCE | "Set up my workspace" |
| "I'm allergic to peanuts" | CONSTRAINT | "Suggest a snack" |
| "My manager is Priya" | ENTITY | "Draft a message to my manager" |
| "Always use bullet points" | INSTRUCTION | "Summarize this for me" |
| "I promised to deliver by Friday" | COMMITMENT | "What's pending this week?" |
| "I live in Bangalore" | FACT | "What's the weather like here?" |

### Precision Tests

```
Precision@K = (relevant memories in top K) / K

Target: Precision@5  ≥ 0.70
Target: Precision@10 ≥ 0.50
Cross-cluster contamination: < 15%
```

**Topic isolation test:** Plant 3 topic clusters (food, work, family). Probe one topic. Measure how much cross-cluster contamination occurs.

### Latency Scaling Tests

| Stored Memories | Retrieval Latency Target |
|---|---|
| ~5 (10 turns) | < 30ms |
| ~40 (100 turns) | < 50ms |
| ~150 (500 turns) | < 75ms |
| ~300 (1000 turns) | < 100ms |
| ~1200 (5000 turns) | < 150ms |

Latency growth should be **sublinear** (vector indices scale logarithmically).

### Safety Tests

**Hallucination test:**
- Run 200 turns, then ask about something NEVER mentioned
- Target: 0% hallucination rate (acceptable < 1%, failure > 2%)

**Hypothetical poisoning test:**
- User makes hypothetical statement; verify it's not stored as fact
- Target: 0/50 false positives

**Update correctness test:**
- Change a preference midway; verify only the new value is recalled
- Target: > 95% correctness

### Forgetting Quality Tests

- Transient facts should decay when unused
- Standing preferences should NOT decay
- Consolidation should reduce count while preserving information

### Noise Profiles for Synthetic Conversations

| Profile | Description | Purpose |
|---|---|---|
| Minimal | Mostly empty/trivial turns | Test recall with no interference |
| Topical | Related topics as noise | Test precision among similar facts |
| Adversarial | Similar but different entities | Test if similar names/facts confuse |
| High volume | Many extractable memories | Test scaling |
| Mixed | Realistic blend | Most representative |

### Regression Test Suites

| Suite | Trigger | Turns | Time | Purpose |
|---|---|---|---|---|
| **Smoke** | Every commit | 50 | ~30s | Basic extraction + retrieval |
| **Short-range** | Every PR | 100 | ~2min | Short-distance recall |
| **Mid-range** | Nightly | 500 | ~15min | Medium-distance recall |
| **Full-range** | Weekly | 1000 | ~45min | Full system validation |
| **Stress** | Pre-release | 5000 | ~4hr | Scaling and degradation |
| **Adversarial** | Pre-release | 1000 | ~1hr | Safety and hallucination |

### Evaluation Report Targets

| Metric | Target |
|---|---|
| Recall at distance 100 | ≥ 95% |
| Recall at distance 500 | ≥ 90% |
| Recall at distance 1000 | ≥ 85% |
| Precision@5 | ≥ 0.70 |
| Precision@10 | ≥ 0.50 |
| P99 retrieval latency | < 150ms |
| Hallucination rate | 0% |
| Update correctness | > 95% |
| Cost per turn (avg) | < $0.005 |

---

## 15. Cost Estimates

### Per-Turn Costs (1,000 turns/day)

| Component | Cost Driver | Estimate |
|---|---|---|
| Flat files | Local disk | Free |
| Redis | ~50MB memory | ~$10–15/month |
| pgvector | Postgres instance + storage | ~$15–25/month |
| Embedding calls | ~200 turns/day need embedding | ~$0.50–1/month |
| Extraction LLM | ~200 turns/day need extraction | ~$2–5/month |
| **Total** | | **~$30–45/month** |

### Comparison: Memory System vs Full Replay

| Turn Count | Full Replay Cost/Turn | Memory System Cost/Turn | Savings |
|---|---|---|---|
| 100 | ~$0.15 | ~$0.003 | 98% |
| 500 | ~$0.75 | ~$0.003 | 99.6% |
| 1000 | ~$1.85 | ~$0.003 | 99.8% |

---

## 16. Implementation Phases

| Phase | What to Build | Outcome | Timeline |
|---|---|---|---|
| **Phase 1** | Flat files + Redis + basic extraction (Stage 1 & 2 only) | Memory loop working end-to-end, no semantic search yet | Week 1–2 |
| **Phase 2** | Add pgvector/Qdrant + embedding + semantic retrieval | The "turn 1 → turn 1000" recall capability | Week 3–4 |
| **Phase 3** | Full extraction pipeline (Stage 3 LLM) + deduplication | Higher quality memories, fewer duplicates | Week 5–6 |
| **Phase 4** | Consolidation worker (merge, decay, prune, promote) | Memory quality improves over time | Week 7–8 |
| **Phase 5** | Evaluation framework + synthetic test generator | Automated testing, regression suites | Week 9–10 |
| **Phase 6** | Tuning (weights, thresholds, budgets) + monitoring | Optimized precision, latency, recall | Week 11–12 |

Each phase builds on the previous without requiring rewrites. You have a **working prototype after Phase 1** and a **production-grade system after Phase 6**.

---

## Summary

This system enables an AI agent to:

1. **Extract** important information from any turn using a three-stage funnel (heuristic → classifier → LLM)
2. **Store** memories in three complementary layers (flat files + Redis + vector store)
3. **Retrieve** relevant memories using multi-signal ranking (semantic + type-priority + recency)
4. **Inject** only the top K memories into each prompt (~500 tokens, ~51ms)
5. **Consolidate** memories in the background (merge, decay, prune, promote)
6. **Evaluate** with automated tests across five dimensions (recall, precision, latency, safety, decay)

The result: information introduced at turn 1 is accurately recalled at turn 1,000 — without replaying history, without growing the prompt, and without hallucinating memories.