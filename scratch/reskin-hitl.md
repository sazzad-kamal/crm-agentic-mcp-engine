# Scratch — whiteboard practice (NOT part of the docs)

Default generic template: same as D1, with **Action/Followup removed**, **graph/Neo4j removed** (add only if the domain has relationships), and a **HITL action tool + write target** added.

## Full

```mermaid
flowchart TB
    Q(("User<br/>Question")) --> AGENT["Agent<br/>(Claude Sonnet 4.6 — ReAct loop)<br/>≤ 6 turns"]

    AGENT -->|"tool_use<br/>(single / parallel)"| MCP_CLIENT["MCP Client<br/>(JSON-RPC over HTTPS)"]
    MCP_CLIENT -.->|"JSON-Schema-<br/>contracted tools"| MCPSRV["standalone<br/>MCP server"]
    MCP_CLIENT -->|tool_result| AGENT

    AGENT -->|"emits candidate answer<br/>with [E#]/[D#] tags"| VAL["Validate<br/>(deterministic<br/>regex + Pydantic)"]
    VAL -->|"fail · retries left<br/>(Reflexion repair, max 2)"| AGENT
    VAL -->|"fail · max repairs hit"| FB["Fallback"]
    VAL -->|"pass → final answer"| RESP
    FB -->|"evidence-only degraded answer"| RESP

    RESP(["Response"])

    MCPSRV -.->|sql_query<br/>sql_compare<br/>sql_trend<br/>sql_health| DB[("SQL<br/>CRM data · E#")]
    MCPSRV -.->|rag_search| VS[("Qdrant · LlamaIndex<br/>hybrid: vector + BM25 · D#")]

    MCPSRV -.->|"grant_access (write)"| GATE{"HITL<br/>human approval"}
    GATE -->|approved| WR[("Identity / Access<br/>system · write")]

    classDef agent fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef tool fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef response fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef boundary fill:#f3f4f6,stroke:#6b7280,color:#374151
    classDef gate fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    class AGENT agent
    class MCP_CLIENT,MCPSRV tool
    class VAL,FB,RESP response
    class Q,DB,VS,WR boundary
    class GATE gate
```

## Skeleton (spine)

```mermaid
flowchart TB
    Q(("User")) --> AGENT["Agent"]

    AGENT <--> MCP_CLIENT["MCP Client"]
    MCP_CLIENT --> MCPSRV["MCP server"]

    AGENT --> VAL["Validate"]
    VAL -->|"repair ×2"| AGENT
    VAL --> FB["Fallback"]
    VAL -->|pass| RESP
    FB --> RESP

    RESP(["Response"])

    MCPSRV --> DB[("SQL")]
    MCPSRV --> VS[("Qdrant")]

    MCPSRV --> GATE{"HITL"}
    GATE --> WR[("Identity / Access · write")]
```

Edits vs D1: **removed** Action + Followup; **removed** graph/Neo4j (add back only if relationships matter); **added** `grant_access` write tool → **HITL** gate → **Identity / Access · write**. Default sources = SQL + RAG. Reads ungated; only the write passes the human gate.
