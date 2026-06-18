# Scratch — whiteboard practice (NOT part of the docs)

Generic agentic re-skin + HITL action tool. Example domain: IT / access copilot (answers helpdesk questions, can grant access with human approval).

```mermaid
flowchart TB
    U(("User")) --> AG["Agent<br/>ReAct · max 6 turns"]

    AG <-->|"tool_use / result"| MC["MCP Client"]
    MC --> SRV["MCP server"]

    AG -->|"candidate answer<br/>+ citations"| VAL["Validate<br/>regex + Pydantic"]
    VAL -->|"repair x2"| AG
    VAL -->|"max repairs"| FB["Fallback"]
    VAL -->|"pass"| RESP(["Response"])
    FB --> RESP

    SRV -->|"ticket_lookup<br/>ticket_analytics"| DB[("Ticket / Asset DB")]
    SRV -->|"kb_search"| KB[("IT Knowledge Base")]
    SRV -->|"access_query"| GR[("Access Graph")]

    SRV -->|"grant_access (write)"| GATE{"HITL<br/>human approval"}
    GATE -->|"approved"| WR[("Identity / Access<br/>system - write")]

    classDef agent fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef tool fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef response fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef boundary fill:#f3f4f6,stroke:#6b7280,color:#374151
    classDef gate fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    class AG agent
    class MC,SRV tool
    class VAL,FB,RESP response
    class U,DB,KB,GR,WR boundary
    class GATE gate
```

**Base** = D1 minus Action/Followup, guardrails intact. **Re-skin** = swap sources/tools, split SQL into query + analytics, drop graph if no relationships. **Variation** = if it acts, add action tool -> HITL -> write (reads stay ungated).
