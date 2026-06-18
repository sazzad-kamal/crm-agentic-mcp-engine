# Scratch — whiteboard practice (NOT part of the docs)

Generic agentic re-skin + HITL action tool. Same shape as the main D1 diagram, with Action/Followup dropped and an action tool gated by human approval. Example domain: IT / access copilot.

## Full

```mermaid
flowchart TB
    Q(("User<br/>Question")) --> AGENT["Agent<br/>(ReAct loop)<br/>≤ 6 turns"]

    AGENT -->|"tool_use<br/>(single / parallel)"| MCP_CLIENT["MCP Client<br/>(JSON-RPC over HTTPS)"]
    MCP_CLIENT -.->|"JSON-Schema-<br/>contracted tools"| MCPSRV["standalone<br/>MCP server"]
    MCP_CLIENT -->|tool_result| AGENT

    AGENT -->|"emits candidate answer<br/>with citations"| VAL["Validate<br/>(deterministic<br/>regex + Pydantic)"]
    VAL -->|"fail · retries left<br/>(Reflexion repair, max 2)"| AGENT
    VAL -->|"fail · max repairs hit"| FB["Fallback"]
    VAL -->|"pass → final answer"| RESP
    FB -->|"evidence-only degraded answer"| RESP

    RESP(["Response"])

    MCPSRV -.->|"ticket_lookup<br/>ticket_analytics"| DB[("Ticket / Asset DB")]
    MCPSRV -.->|"kb_search"| VS[("IT Knowledge Base")]
    MCPSRV -.->|"access_query"| N4[("Access Graph")]

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
    class Q,DB,VS,N4,WR boundary
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

    MCPSRV --> DB[("Ticket DB")]
    MCPSRV --> VS[("Knowledge Base")]
    MCPSRV --> N4[("Access Graph")]

    MCPSRV --> GATE{"HITL"}
    GATE --> WR[("Access · write")]
```

**Base** = D1 minus Action/Followup, guardrails intact. **Re-skin** = swap sources/tools, split SQL into query + analytics, drop graph if no relationships. **Variation** = if it acts, add action tool → HITL → write (reads stay ungated).
