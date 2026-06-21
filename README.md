# CRM Agentic Reasoning Engine

**Tool-use agent (Claude Sonnet) backed by a standalone MCP server that reasons across CRM data, product documentation, and a knowledge graph — with grounded answers, traceable evidence, and quality gates.**

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/)
[![MCP](https://img.shields.io/badge/protocol-MCP-purple)](https://modelcontextprotocol.io/)
[![LangGraph](https://img.shields.io/badge/orchestration-LangGraph-purple)](https://github.com/langchain-ai/langgraph)
[![LlamaIndex](https://img.shields.io/badge/RAG-LlamaIndex-orange)](https://www.llamaindex.ai/)
[![Neo4j](https://img.shields.io/badge/GraphRAG-Neo4j-green)](https://neo4j.com/)
[![RAGAS](https://img.shields.io/badge/eval-RAGAS-yellow)](https://github.com/explodinggradients/ragas)

---

## Contents

- **[The Problem](#the-problem)**
- **[Architecture](#architecture)** · [deployment topology](#deployment--system-topology)
- **[ReAct + Reflexion](#react--reflexion--the-two-named-patterns)** · [why 6 turns](#react-loop-inner--yao-et-al-2022) · [why max 2 repairs (measured)](#why-max-2-repairs-not-more)
- **[The 6 Tools](#the-6-tools)**
- **[Why MCP, not in-process tools](#why-mcp-not-in-process-tools)**
- **[Grounding, Safety & Reliability](#grounding-safety-and-reliability)** · [PII & guardrails](#pii-protection-and-input-sanitization) · [prompt caching ~85%](#cost-engineering--prompt-caching-cuts-token-cost-85)
- **[Evaluation Framework](#evaluation-framework)** · [why correctness 0.35](#evaluation-framework) · [RAG config comparison](#rag-retrieval-strategy-comparison)
- **[Tech Stack](#tech-stack)** · [provider-portable / Bedrock](#multi-model-strategy)

---

## The Problem

An account manager prepping Acme's renewal asks: *"Acme's renewal is at risk — which Act! Marketing Automation features are they under-using, what does the documentation recommend for those features, and who on their buying committee should I prioritize, excluding anyone connected to our competitors?"*

This one question touches three sources — CRM data (feature usage, renewal status), product docs (recommendations for under-utilized features), and the knowledge graph (buying committee, competitor connections). Each baseline falls short for a different reason — pure LLMs hallucinate, LLM + RAG over a single corpus misses cross-cutting context, and direct SQL can't reason over docs or graph relationships.

**This system is a LangGraph ReAct (Reason+Act) tool-use agent backed by a standalone MCP server.** The agent (Claude Sonnet 4.6) decomposes cross-source questions via parallel tool calls, broadens retrieval when evidence is thin, and synthesizes one coherent answer with traceable per-source citations (`[E#]` for SQL, `[D#]` for docs, `[G#]` for graph). A deterministic Validate gate enforces an evidence-tag contract with Reflexion-style repair on any ungrounded claim.

---

## Architecture

```mermaid
flowchart TB
    Q(("User<br/>Question")) --> AGENT["Agent<br/>(Claude Sonnet 4.6 — ReAct loop)<br/>≤ 6 turns"]

    AGENT -->|"tool_use<br/>(single / parallel)"| MCP_CLIENT["MCP Client<br/>(JSON-RPC over HTTPS)"]
    MCP_CLIENT -.->|"6 JSON-Schema-<br/>contracted tools"| MCPSRV["standalone<br/>MCP server"]
    MCP_CLIENT -->|tool_result| AGENT

    AGENT -->|"emits candidate answer<br/>with [E#]/[D#]/[G#] tags"| VAL["Validate<br/>(deterministic<br/>regex + Pydantic)"]
    VAL -->|"fail · retries left<br/>(Reflexion repair, max 2)"| AGENT
    VAL -->|"fail · max repairs hit"| FB["Fallback"]
    VAL -->|"pass → final answer"| RESP
    VAL -->|pass| ACT["Action"] & FU["Followup"]
    ACT -->|"suggested next step"| RESP
    FU -->|"follow-up prompts"| RESP
    FB -->|"evidence-only degraded answer"| RESP

    RESP(["Response"])

    MCPSRV -.->|sql_query<br/>sql_compare<br/>sql_trend<br/>sql_health| DB[("SQL<br/>CRM data · E#")]
    MCPSRV -.->|rag_search| VS[("Qdrant · LlamaIndex<br/>hybrid: vector + BM25 · D#")]
    MCPSRV -.->|graph_query| N4[("Neo4j<br/>graph · G#")]

    classDef agent fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef tool fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef response fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef boundary fill:#f3f4f6,stroke:#6b7280,color:#374151

    class AGENT agent
    class MCP_CLIENT,MCPSRV tool
    class VAL,ACT,FU,FB,RESP response
    class Q,DB,VS,N4 boundary
```

<details>
<summary><b>Architecture at a glance</b> — the spine (full detail above)</summary>

```mermaid
flowchart TB
    Q(("User")) --> AGENT["Agent"]

    AGENT <--> MCP_CLIENT["MCP Client"]
    MCP_CLIENT --> MCPSRV["MCP server"]

    AGENT --> VAL["Validate"]
    VAL -->|"repair ×2"| AGENT
    VAL --> FB["Fallback"]
    VAL -->|pass| RESP
    VAL --> ACT["Action"] & FU["Followup"]
    ACT --> RESP
    FU --> RESP
    FB --> RESP

    RESP(["Response"])

    MCPSRV --> DB[("SQL")]
    MCPSRV --> VS[("Qdrant")]
    MCPSRV --> N4[("Neo4j")]
```

</details>

**5-node LangGraph + 1 standalone MCP server.** Two products from one codebase: the engine (web UI surface) consumes the MCP server over HTTP/SSE; the same MCP server is independently demoable in Claude Desktop via stdio. Six JSON-Schema-contracted tools serve as the protocol surface — `sql_query`, `sql_compare`, `sql_trend`, `sql_health`, `rag_search`, `graph_query` — over Postgres, LlamaIndex, and Neo4j.

> Examples: "Show Q1 deals" → `sql_query` · "Q1 vs Q2 revenue" → `sql_compare` · "Revenue trend by month" → `sql_trend` · "Acme's health score" → `sql_health` · "How do I import contacts?" → `rag_search` · "Who is connected to our competitors?" → `graph_query` · "Acme renewal: under-used features + doc recommendations + committee minus competitors?" → **parallel calls to sql_query + rag_search + graph_query** (cross-source).

### Deployment & system topology

The same logical design, deployed: the engine and the MCP server run as **separate cloud services** (process isolation, independent scale); retrieval, graph, and SQL live in **managed Qdrant Cloud, Neo4j Aura, and Neon Postgres**. The MCP boundary is JSON-RPC over HTTPS, carrying tool calls one way and `{result, citations[E#/D#/G#]}` back.

```mermaid
flowchart TB
    User(("User"))

    subgraph BROWSER["Browser"]
        UI["React UI — chat + data panels"]
    end

    subgraph RW1["Engine service · Cloud"]
        API["FastAPI · /api/chat/stream"]
        AG["5-node LangGraph agent<br/>Agent · Validate · Action · Followup · Fallback"]
        MC["MCP client"]
        API --> AG --> MC
    end

    subgraph RW2["MCP server · Cloud (separate service)"]
        TL["6 JSON-Schema tools · guards · per-tool TTL cache"]
    end

    subgraph DATA["Managed data stores"]
        PG[("Neon<br/>Postgres · E#")]
        QDR[("Qdrant Cloud<br/>doc vectors · D#")]
        NEO[("Neo4j Aura<br/>graph · G#")]
    end

    subgraph SHARED["Model & observability services"]
        CLA["Claude Sonnet 4.6 (agent)<br/>· 4.5 (SQL·Cypher gen)"]
        GPT["GPT-5.4-nano<br/>Action · Followup"]
        EMB["OpenAI<br/>text-embedding-3-small"]
        OBS["LangSmith<br/>tracing · latency · SLOs"]
    end

    User --> UI
    UI -->|"HTTPS · SSE"| API
    MC -->|"JSON-RPC / HTTPS"| TL
    RW2 -->|"SQL · RAG · Graph"| DATA
    RW1 -.->|"LLM · traces"| SHARED
    RW2 -.->|"LLM · embed · traces"| SHARED

    style RW1 fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    style RW2 fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    style DATA fill:#f3f4f6,stroke:#6b7280,color:#374151
    style SHARED fill:#f3f4f6,stroke:#6b7280,color:#374151
    style BROWSER fill:#dcfce7,stroke:#16a34a,color:#14532d
```

<details>
<summary><b>Deployment at a glance</b> — the spine (full detail above)</summary>

```mermaid
flowchart TB
    User(("User"))

    subgraph BROWSER["Browser"]
        UI["React UI"]
    end

    subgraph RW1["Engine service · Cloud"]
        API["FastAPI"]
        AG["LangGraph agent"]
        MC["MCP client"]
        API --> AG --> MC
    end

    subgraph RW2["MCP server · Cloud"]
        TL["6 tools"]
    end

    subgraph DATA["Managed data stores"]
        PG[("Neon")]
        QDR[("Qdrant")]
        NEO[("Neo4j")]
    end

    subgraph SHARED["Model & observability services"]
        CLA["Claude"]
        GPT["GPT"]
        EMB["Embeddings"]
        OBS["LangSmith"]
    end

    User --> UI
    UI --> API
    MC --> TL
    RW2 --> DATA
    RW1 -.-> SHARED
    RW2 -.-> SHARED
```

</details>

---

## ReAct + Reflexion — the two named patterns

The agent operates as two cooperating loops drawn from published research.

### ReAct loop (inner — Yao et al., 2022)

The agent **reasons** → **acts** (emits a tool call) → **observes** → reasons again. It emits parallel `tool_use` blocks for independent tools, and **broadens** retrieval (`top_k`, `hop_depth`) when evidence is thin.

**Why a 6-turn cap?** A turn is one model round-trip. The cap is **3 sources (SQL, RAG, graph) × 2 passes** (a lookup + one wider retry). Normal questions finish in **1–2 turns** — independent sources fire in parallel; 6 is the ceiling for the worst case: three sources forced sequential, each needing a retry (e.g. *"committee minus competitors"* — find competitors first, then filter). Past 6 it's **looping, not gathering** → it falls back, staying inside **p95 ≤ 8s**. It's a **coverage bound, not a tuned number** — generous headroom that rarely binds; hitting it is a fall-back signal, not a reason to raise it.

```mermaid
flowchart LR
    R[Reason] --> A[Act<br/>tool_use]
    A --> O[Observe<br/>tool_result]
    O --> R
    R -.->|enough evidence| FA[Candidate answer]
```

### Reflexion loop (outer — Shinn et al., 2023)

After the agent produces a candidate answer, a deterministic Validate gate (regex + Pydantic, sub-millisecond) checks:
- Structure: `Answer` + `Evidence` sections parse
- Tag presence (no naked claims): every substantive sentence (≥ 5 words) ends with a `[E#]/[D#]/[G#]` tag
- Tag-to-citation cross-check: every cited id matches a tool-returned citation

On failure, the gate emits a structured critique that names every failure mode and lists the only citations the agent may use. The agent re-invokes **without tools bound** (so it physically cannot emit another `tool_use`) and rewrites the answer using only valid evidence. Max 2 repair retries; then fallback to a degraded evidence-only answer.

```mermaid
flowchart LR
    FA[Candidate answer] --> V{Deterministic<br/>Validate gate}
    V -->|ok| AF[Action + Followup]
    V -->|fail| C[Critic emits<br/>structured critique]
    C --> R["Agent revises<br/>(no tools)"]
    R --> V
    V -->|max 2 repairs<br/>exhausted| FB[Fallback<br/>evidence-only answer]
```

**Why deterministic Validate?** The agent that just wrote the answer is the worst auditor of its own work — same context, same blind spots. A deterministic critic catches mechanical failures (missing tags, invented citations, naked claims) the LLM would miss. Sub-millisecond cost; no token spend; fully testable.

### Why max 2 repairs (not more)?

*Why not repair more to get a better answer?* Because **more repairs can't make the answer better.** Repair runs **tools-unbound** — it only re-words the *existing* evidence; it cannot gather new evidence. So an ungrounded claim has exactly two causes, and only one is a repair's job:

```mermaid
flowchart TB
    Q["Claim fails Validate<br/>(ungrounded)"] --> WHY{"Why ungrounded?"}
    WHY -->|"wording doesn't match<br/>the retrieved evidence"| REP["Repair — re-word to match<br/>evidence (tools unbound)"]
    WHY -->|"the evidence is<br/>missing entirely"| RET["Needs more *retrieval* —<br/>the ReAct loop's job<br/>(broaden top_k / hop_depth)"]
    REP -->|"fixed in ≤ 2 tries"| OK["Grounded"]
    REP -->|"still ungrounded after 2"| FB["Fallback — evidence-only.<br/>A 3rd/4th re-word cannot add<br/>evidence that isn't there."]

    classDef gate fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef ok fill:#dcfce7,stroke:#16a34a,color:#14532d
    class REP,RET gate
    class OK ok
```

So 2 captures the recoverable grounding errors; beyond that the system **falls back honestly** rather than burn latency — each repair is a full model round-trip against the p95 ≤ 8s SLO.

**Measured (controlled experiment).** 10 crafted non-conforming answers — missing tags and invalid citations — run through the *real* Validate gate + repair loop (Claude Sonnet 4.6, tools unbound), with the cap raised to 4 to observe the tail:

| Grounded at | Cases | |
|---|---|---|
| 0 — already valid | 3 | `███` |
| **repair 1** | **7** | `███████` |
| repair 2 | 0 | (none) |
| repair 3 | 0 | (none) |
| repair 4 | 0 | (none) |

**Every case that tripped the gate recovered on the 1st repair; none ever needed a 2nd, 3rd, or 4th.** Recovery saturates at repair 1, so the production cap of **2 is a safety margin** — attempts 3–4 are never reached. That's the quantitative half of *"more repairs can't help"*: there's nothing left to fix.

---

## The 6 Tools

Each tool returns a uniform shape: `{result, citations: [{id, source, excerpt}], debug}`. Citation ids are prefixed `E#` (SQL), `D#` (RAG), `G#` (graph) so the agent cites uniformly across sources.

| Tool | Purpose | Use for | Don't use for |
|---|---|---|---|
| **sql_query** | Single-source CRM lookups | lists, counts, sums, filters | comparisons, trends, health scores |
| **sql_compare** | A-vs-B analysis with deterministic math | "Q1 vs Q2 revenue", "Acme vs TechCorp" | single-snapshot fetches |
| **sql_trend** | Time-series with granularity detection | "revenue trend by month", "growth rate" | snapshots or A-vs-B |
| **sql_health** | Weighted 6-factor account health score | "Acme's health", "at-risk accounts" | plain deal lookups |
| **rag_search** | Hybrid retrieval over Act! CRM docs | "how do I import contacts?" | CRM data |
| **graph_query** | Multi-hop Neo4j traversal | "who is connected to", "buying committee" | single-table lookups |

Tool descriptions include explicit **anti-examples** (each enumerates every other tool as a "do NOT use" case). This is the primary driver of tool-pick accuracy — a measured SLO (≥ 0.85).

### Specialized SQL math stays in the tools (not the LLM)

The four `sql_*` tools each carry deterministic post-processing rather than returning raw rows for the LLM to compute on:
- `sql_compare` returns `{entity_a, entity_b, metrics: {col: {diff, percent_change}}}`
- `sql_trend` returns `{direction, percent_change, volatility, period_changes}`
- `sql_health` returns `{score, grade, components, insights}` from a weighted 6-factor formula (deal value [log], deal count, win rate, activity recency, pipeline coverage, renewal status)

LLMs are bad at arithmetic over JSON rows; this delegation pattern keeps the math reliable — and leaves the agent to do what it's good at: weaving findings into a narrative.

---

## Why MCP (not in-process tools)

The MCP server is a **separate process** (Anthropic's Model Context Protocol, JSON-RPC over stdio or HTTP(S)). Two real engineering wins drove the choice:

| Property | What it buys |
|---|---|
| **Protocol-level contracts (JSON Schema)** | Wire-level schema drift fails loudly at the protocol boundary, not silently inside Python. Bad parameters get caught before tool execution, not deep in code paths. |
| **Process isolation** | A tool crash (Neo4j hiccup, LlamaIndex OOM) doesn't take down the agent. Memory leaks in one tool don't degrade the whole system. Restart the MCP server — deploy a tool fix or clear a leak — without dropping the agent's live sessions. |

Secondary benefits — portability across MCP-speaking clients (future optionality), independent deploy/scale, an auth boundary — are real but weren't the drivers; the immediate engineering value is reliability + correctness at the tool boundary.

The cost is one network round-trip per tool call (or ~1 ms over stdio in local / Claude Desktop mode) and a separate process lifecycle to manage. Both are well-amortized: the per-tool TTL LRU cache skips redundant calls, and a round-trip is small next to the actual SQL/Cypher/retrieval work — so most turns spend their latency budget in the real work, not the protocol.

---

## Grounding, Safety, and Reliability

### Evidence-grounded responses

Every claim cites its source with a traceable tag. No tag → claim is dropped.

```
Acme is using Act! Marketing Automation [D1] with 3 active campaigns [E1].
Their renewal is due Feb 15 [E2] and health score is at-risk [E3].
Maria (VP Sales) is connected to competitor Globex through a
shared board membership [G1].

Evidence:
- E1: activities.type = "AMA Campaign" WHERE company = "Acme" (3 rows)
- E2: companies.renewal_date = "2026-02-15" (row 12)
- E3: companies.health_status = "at-risk" (row 12)
- D1: act_docs/marketing_automation.pdf § "Campaign Management"
- G1: (Maria)-[:BOARD_MEMBER]->(TechBoard)<-[:BOARD_MEMBER]-(Globex VP)
```

This grounding discipline drives the faithfulness SLO (≥ 0.85 RAGAS) — responses are measured against retrieved context; claims without evidence are penalized.

### Safety guards on LLM-generated queries

```mermaid
flowchart LR
    LLM["LLM generates<br/>SQL or Cypher<br/>inside tool"] --> PARSE["AST parse<br/>(sqlglot / Cypher regex)"]
    PARSE -->|parse error| BLOCK["Block"]
    PARSE --> CHK{"Write ops?"}
    CHK -->|yes| BLOCK
    CHK -->|no| LIM["Cap rows<br/>(LIMIT 1000)"]
    LIM --> EXEC["Execute against<br/>Postgres / Neo4j"]
```

- **SQL guard (sqlglot AST)**: blocks INSERT/DELETE/DROP/UPDATE/CREATE/ALTER/GRANT, blocks file-reading TVFs (`read_csv`), auto-injects `LIMIT 1000`
- **Cypher guard (read-only enforcement)**: blocks CREATE/DELETE/DETACH/SET/REMOVE/MERGE/DROP/CALL/FOREACH; auto-injects `LIMIT 1000`

*Why `LIMIT 1000`?* It caps the blast radius of a mis-generated query — without it, a bad `WHERE` could dump an entire table into the model's context (cost + latency blowup) or the response. 1000 rows is generous for analytical CRM questions while bounding the worst case, and the LLM reasons better over a bounded result set than tens of thousands of rows.

### PII protection and input sanitization

These layers keep **PII from leaking** into the model's context or its answers:

```mermaid
flowchart TB
    U(("User<br/>Question")) --> SAN["Input sanitization<br/>prompt-injection neutralize"]
    SAN --> AG["Agent<br/>(ReAct loop)"]

    AG <-->|"tool_use / result"| MC["MCP Client"]
    MC -.->|"6 tools"| SRV["standalone<br/>MCP server"]

    SRV -.->|"sql_* tools"| SQLG["SQL boundary<br/>read-only guard (sqlglot)<br/>+ PII masking"]
    SQLG -.-> DB[("CRM SQL · E#")]

    DOCS[("Product docs")] --> ING["Ingestion<br/>PII redaction"]
    ING -.->|"scrub before embed"| VS[("Qdrant · D#")]
    SRV -.->|"rag_search"| VS

    AG -->|"candidate answer<br/>+ citations"| VAL["Validate<br/>output gate (citations)"]
    VAL --> RESP(["Response"])

    classDef guard fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef agent fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef tool fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef response fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef boundary fill:#f3f4f6,stroke:#6b7280,color:#374151

    class SAN,SQLG,ING,VAL guard
    class AG agent
    class MC,SRV tool
    class RESP response
    class U,DB,VS,DOCS boundary
```

| Layer | Where | What it does |
|---|---|---|
| **Input sanitization** | engine input boundary | neutralizes prompt-injection patterns before the agent sees the input |
| **Ingestion redaction** | MCP server, at ingestion | scrubs embedded PII (emails, phones) from docs before embedding — PII never enters the vector store |
| **SQL boundary masking** | MCP server, SQL boundary | policy-driven masking of email / phone + free-text scrub, before rows reach the LLM |

**Policy-driven, not blanket redaction.** In a CRM, contact *names* are often the legitimate answer ("who's on Acme's buying committee?") — so the default masks contact-*channel* PII (email, phone) the model never needs, scrubs free-text, and keeps names. The policy is **data, not code** — masking names too is a one-line change. Detection is **hybrid**: regex for pattern-based PII (email, phone) plus **Microsoft Presidio** NER for entities embedded in free text (names, locations, organizations).

---

### Cost engineering — prompt caching cuts token cost ~85%

The Messages API is **stateless** — every turn re-sends the **system prompt + 6 tool schemas**, which are large and identical each time. Caching **writes** that static prefix to cache once (~1.25× the **normal** price), then **reads** it back at **~0.1× (~90% off)** on every later turn — so it pays for itself by the 2nd turn. With the **5-minute cache TTL**, an active session reuses that one prefix across many turns *and* questions: **~70% on a single 6-turn question, ~85% sustained.**

A separate **per-tool TTL cache** on the MCP server skips redundant *tool execution* when the same parameters recur in a session.

---

## Evaluation Framework

A curated grounded evaluation set — **manually authored, not synthetic** — covering every tool path. Each question is paired with **expected source IDs** (gold labels) so retrieval scoring can be ground-truthed. RAGAS metrics, LLM-as-Judge for terminal-node quality, regression gate on every CI run.

```mermaid
flowchart LR
    Q[("Grounded<br/>Questions")] --> PIPE["Full Pipeline<br/>(agent → tools → validate)"]
    PIPE --> RAGAS["RAGAS<br/>faithfulness<br/>relevancy<br/>correctness"]
    PIPE --> AJ["Action Judge<br/>relevance<br/>actionability<br/>appropriateness"]
    PIPE --> FJ["Followup Judge<br/>relevance<br/>grounding<br/>diversity"]
    PIPE --> TP["Tool-pick<br/>accuracy"]
    RAGAS --> SLO{"SLO Gate"}
    AJ --> SLO
    FJ --> SLO
    TP --> SLO
    SLO -->|pass| JSON["JSON Baseline"]
    SLO -->|fail| CI["CI Blocks Merge"]
    JSON --> REG["Regression<br/>Detection"]
```

### Quality Gates & SLOs

| Category | SLO | Threshold | Scope |
|----------|-----|-----------|-------|
| **Answer Quality (RAGAS)** | Faithfulness | ≥ 0.85 | Final answer |
| | Answer Relevancy | ≥ 0.85 | Final answer |
| | Answer Correctness | ≥ 0.35 | Final answer |
| **Tool-pick accuracy** | Agent picks correct tool(s) per question | ≥ 0.85 | Agent loop |
| **Output Quality (LLM-as-Judge)** | Action: relevance, actionability, appropriateness | Pass rate ≥ 80% | Action node |
| | Followup: relevance, grounding, diversity | Pass rate ≥ 80% | Followup node |
| **Performance** | p50 Latency | ≤ 3s | Full pipeline (warm cache) |
| | p95 Latency | ≤ 8s | Full pipeline |

**Regression gate:** CI fails when RAGAS composite drops more than 0.05 absolute vs the baseline JSON.

**Why correctness gates at 0.35, not 0.85.** Deliberate. RAGAS `answer_correctness` scores against a **single** reference answer, so it penalizes valid paraphrases and extra-but-correct detail — a high bar would fail CI on answers that are *right, just worded differently*. So **faithfulness (≥ 0.85) is the hard gate**; correctness is a **directional signal** watched by the regression gate. **Grounding is enforced; exact-wording correctness is measured, not guaranteed.**

### Why LLM-as-Judge over hard rules

Hard rules can't capture nuance — questions like "is this well-organized?" or "is the explanation clear?" don't reduce to keyword matches. Human review is ground truth but doesn't scale. Hybrid: LLM-as-judge runs on the full eval set; humans calibrate on a subset and spot-check judge drift.

### Retrieval Diagnostic

Recall@k / Precision@k / MRR run offline as a diagnostic lens — not part of the CI gate. When RAGAS regresses, these isolate the failure layer:

- Retrieval stable + RAGAS down → synthesis problem (prompt / LLM)
- Retrieval down + RAGAS down → retrieval problem (chunking, embeddings, reranker)

RAGAS leads config selection; retrieval metrics diagnose.

### Cost Considerations

RAGAS uses LLM-as-judge internally, so eval cost per CI run is non-trivial. Mitigated by:

- Full eval on PR
- Smaller smoke set on push
- Sampled scoring on production traffic (not every request)

### Drift Detection & Production Monitoring

Embedding-space drift detection compares target corpus to current corpus and flags when drift exceeds threshold. SLO burn-rate alerts fire when production latency, RAGAS faithfulness, or validate-repair fallback rate breach their baselines — catching regressions in production, not just at merge time.

### RAG Retrieval Strategy Comparison

Automated pipeline comparing **6 retrieval configurations** across the grounded question set using RAGAS metrics:

| Config | Retriever | Top-K | Reranker |
|--------|-----------|-------|----------|
| vector_top5 | Vector | 5 | None |
| vector_top10 | Vector | 10 | None |
| bm25_top5 | BM25 | 5 | None |
| hybrid_top5 | Vector + BM25 (RRF) | 5 | None |
| **vector_top10_rerank5** ← winner | Vector | 10 | SentenceTransformer |
| hybrid_top10_rerank5 | Vector + BM25 | 10 | SentenceTransformer |

Winner selected by composite score: `0.4 × relevancy + 0.4 × faithfulness + 0.2 × correctness`. Weights reflect the priority: the answer must address the question (relevancy) and stay grounded in retrieved context (faithfulness); correctness is weighted lower because semantic similarity to a reference answer is phrasing-sensitive.

---

## Tech Stack

### Multi-Model Strategy

Different models for different task complexities:

| Task | Where | Model | Why |
|---|---|---|---|
| **Agent tool-use loop** | Engine — ReAct + Reflexion | Claude Sonnet 4.6 | Strong tool-use, parallel calls, native prompt caching |
| **SQL generation** | MCP — inside sql_* tools | Claude Sonnet 4.5 | Precise structured SQL with fewer hallucinated columns |
| **Cypher generation** | MCP — inside graph_query | Claude Sonnet 4.5 | Same structured-output advantage for graph queries |
| **Action suggestions** | Engine — Action node | GPT-5.4-nano | Simple creative output, low cost |
| **Followup generation** | Engine — Followup node | GPT-5.4-nano | Question generation, low cost |
| **Embeddings** | MCP — rag_search | text-embedding-3-small | Vector similarity for retrieval |
| **Eval judges** | CI eval harness | GPT-5.4-mini | Structured scoring on Action / Followup quality |

**Provider-portable.** The agent model is a config value (`AGENT_MODEL` env var) behind LangChain's chat abstraction — so it runs on the Anthropic API today and ports to another backend (e.g. **AWS Bedrock**) with a config swap, not a redesign. The MCP tool server, guardrails, Validate gate, and eval harness are all **model-agnostic** — they don't change when the model backend does.

### Stack

| Component | Technology | Why |
|---|---|---|
| **Orchestration** | LangGraph | Stateful workflows, conditional edges, checkpointing |
| **Agent LLM bindings** | LangChain `ChatAnthropic` + `bind_tools()` | Tool-calling binding + provider-swappable chat abstraction (Anthropic API ↔ Bedrock) |
| **Tool protocol** | Model Context Protocol (MCP) | Industry-standard agent-tool boundary; portable to Claude Desktop |
| **RAG** | LlamaIndex (vector + BM25 + reranker) | Hybrid retrieval winner from automated comparison |
| **Vector DB** | Qdrant | Fast hybrid (dense + sparse) search, lean ops |
| **Graph DB** | Neo4j | Multi-hop entity traversal, Cypher queries |
| **SQL DB** | Postgres (Neon) | Managed relational store for CRM data; read-only, sqlglot-guarded |
| **Evaluation** | RAGAS + LLM-as-Judge | Faithfulness, relevancy, correctness, action/followup quality |
| **Backend (engine)** | FastAPI | Async, OpenAPI docs, Pydantic validation |
| **Streaming** | Server-Sent Events | Real-time token + tool_call delta streaming |
| **Tracing** | LangSmith | Per-turn observability — agent → tool → Validate, with cost + latency per span |
| **Checkpointing** | LangGraph `MemorySaver` | In-process state across turns within a session |

---

## Documentation

- **[Architecture Decisions](docs/ARCHITECTURE.md)** — design rationale, trade-offs, ReAct + Reflexion deep-dive
- **[Data Flow](docs/data-flow.md)** — stage-by-stage request lifecycle
- **[LangGraph Diagram](docs/LANGGRAPH_DIAGRAM.md)** — visual agent topology + MCP boundary

---

<p align="center">
  <strong>Built by <a href="https://github.com/sazzad-kamal">Sazzad Kamal</a></strong>
</p>
