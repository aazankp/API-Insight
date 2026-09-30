# System Diagrams

This directory contains standalone Mermaid diagram source files representing the architecture, protocols, and data models of **ApiMonitor**:

| File | Type | Description |
| :--- | :--- | :--- |
| [`architecture-overview.mermaid`](architecture-overview.mermaid) | Flowchart | Multi-tier distributed system architecture from presentation to targets. |
| [`agent-protocol.mermaid`](agent-protocol.mermaid) | Sequence Diagram | Private agent token handshake, heartbeat pulse, and job execution cycle. |
| [`incident-lifecycle.mermaid`](incident-lifecycle.mermaid) | State Diagram | Incident states, AI Detective invocation, and MTTR recovery cycle. |
| [`database-er.mermaid`](database-er.mermaid) | Entity-Relationship | Core entities, foreign key relationships, and multi-tenant scoping. |

These diagrams can be rendered using any Mermaid viewer, GitHub markdown rendering, or the [Mermaid Live Editor](https://mermaid.live).
