# Round-Robin Request Flow

Sequence diagram of a user's browser cycling between deployment targets on refresh, and the UI
displaying which platform actually served each page load.

```mermaid
sequenceDiagram
    actor Browser
    participant LB as Cloud Load Balancer
    participant A as Target A (e.g. AKS)
    participant B as Target B (e.g. ACA)
    participant DB as Shared PostgreSQL

    Note over LB: No sticky sessions needed — LB may route each fresh page load to any target

    rect rgb(240, 248, 255)
    Note over Browser,A: Initial page load — routed to Target A
    Browser->>LB: GET /
    LB->>A: GET /
    A-->>Browser: static frontend
    Browser->>LB: GET /info
    LB->>A: GET /info (same LB hostname → same target)
    A-->>Browser: {"deploymentTarget": "AKS"}
    Browser->>Browser: render badge "Served from: AKS"
    end

    Note over Browser,B: /, /thoughts, and /info route to the SAME target within one page load (path-based routing) — the badge never lies about which target served the page

    rect rgb(255, 250, 240)
    Note over Browser,B: User refreshes — LB routes to a different target
    Browser->>LB: GET /
    LB->>B: GET /
    B-->>Browser: static frontend
    Browser->>LB: GET /info
    LB->>B: GET /info
    B-->>Browser: {"deploymentTarget": "ACA"}
    Browser->>Browser: render badge "Served from: ACA"
    Browser->>LB: POST /thoughts
    LB->>B: POST /thoughts (same target as this page load)
    B->>DB: write thought
    DB-->>B: ack
    B-->>Browser: 201 Created
    end

    rect rgb(240, 248, 255)
    Note over Browser,A: Next refresh — LB routes back to Target A
    Browser->>LB: GET /thoughts
    LB->>A: GET /thoughts
    A->>DB: read thoughts
    DB-->>A: same rows written via Target B
    A-->>Browser: thoughts list (data persisted regardless of target)
    end
```

Because both targets read and write the same shared PostgreSQL, and neither the load balancer nor
the app tier keeps any per-user state, no sticky sessions are required — the load balancer is free
to route every fresh page load to whichever target it likes.
