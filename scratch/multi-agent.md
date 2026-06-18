# Scratch — multi-agent variant (NOT part of the docs)

Same scaffold, expanded: a **Planner** delegates to **scoped React worker agents** that share an **MCP client** into the **one shared MCP server**. Guardrails (Validate → Fallback) unchanged. This is the Q18 "how I'd go multi-agent" picture.

## Full

```mermaid
flowchart TB
    Q(("User<br/>Question")) --> PL["Planner<br/>(decompose · delegate · synthesize)"]

    PL -->|"sub-task A"| W1["SQL Worker<br/>(ReAct loop)"]
    PL -->|"sub-task B"| W2["Docs Worker<br/>(ReAct loop)"]
    W1 -->|results| PL
    W2 -->|results| PL

    W1 <-->|"tool_use / result"| MC["MCP Client"]
    W2 <-->|"tool_use / result"| MC
    MC -.->|JSON-RPC / HTTPS| MCPSRV["shared<br/>MCP server"]

    PL -->|"synthesized answer<br/>with [E#]/[D#]"| VAL["Validate<br/>(deterministic<br/>regex + Pydantic)"]
    VAL -->|"fail · retries left<br/>(repair, max 2)"| PL
    VAL -->|"fail · max repairs hit"| FB["Fallback"]
    VAL -->|"pass → final answer"| RESP
    FB -->|"evidence-only degraded answer"| RESP

    RESP(["Response"])

    MCPSRV -.->|sql_query<br/>sql_compare<br/>sql_trend<br/>sql_health| DB[("SQL<br/>CRM data · E#")]
    MCPSRV -.->|rag_search| VS[("Qdrant · LlamaIndex<br/>hybrid: vector + BM25 · D#")]

    classDef agent fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef tool fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef response fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef boundary fill:#f3f4f6,stroke:#6b7280,color:#374151

    class PL,W1,W2 agent
    class MC,MCPSRV tool
    class VAL,FB,RESP response
    class Q,DB,VS boundary
```

## Skeleton (spine)

```mermaid
flowchart TB
    Q(("User")) --> PL["Planner"]

    PL --> W1["SQL Worker"]
    PL --> W2["Docs Worker"]
    W1 --> PL
    W2 --> PL

    W1 <--> MC["MCP Client"]
    W2 <--> MC
    MC --> MCPSRV["MCP server"]

    PL --> VAL["Validate"]
    VAL -->|"repair ×2"| PL
    VAL --> FB["Fallback"]
    VAL -->|pass| RESP
    FB --> RESP

    RESP(["Response"])

    MCPSRV --> DB[("SQL")]
    MCPSRV --> VS[("Qdrant")]
```

## The points this diagram makes (Q18)
- **What makes it multi-agent = multiple ReAct loops** (the workers), not the tools. Each worker is its own reasoning loop, scoped to a tool subset.
- **Planner** decomposes → delegates → synthesizes. A *router* if routing is simple; a *ReAct planner* if the plan must adapt to results.
- **Shared MCP client + server.** Multi-agent is **additive** — add agents in front of the same tool layer, don't rebuild the tools. (This is why the standalone MCP server makes it cheap.)
- **Repair stays in the reasoning layer** — Validate sends failures back to the *planner* (tools unbound, re-synthesis only), never back to the tools.
- **Same guardrails** — Validate → Fallback unchanged; the planner's synthesized answer still passes the deterministic gate.
- **When to use** — only when one agent can't reliably pick among too many tools, or the task fans out into independent deep sub-tasks. Otherwise stay single-agent.
