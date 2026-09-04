# Azure Deployment

Azure-specific deployment diagram: thoughtsapp running on both AKS and Azure Container Apps,
fronted by one of two alternative load-balancing paths, sharing one PostgreSQL database and one
Kafka + evaluation service.

```mermaid
flowchart TB
    Users((Users))

    subgraph LB["Load balancer (pick one path)"]
        direction LR
        AppGW["Application Gateway\nBackend pools: AKS ingress + ACA FQDN\nRound robin; host-header\noverride for ACA pool"]
        TM["Traffic Manager\nWeighted profile, 50/50\n(DNS-based — client DNS\ncaching caveat)"]
    end

    Users --> AppGW
    Users -. "alternative path" .-> TM

    subgraph AKS["AKS"]
        AKSIngress["Internal ingress"]
        AKSPods["thoughtsapp pods\n(manifests set\nDEPLOYMENT_TARGET=AKS)"]
        AKSIngress --> AKSPods
    end

    subgraph ACA["Azure Container Apps"]
        ACAApp["Container App\n(--env-vars DEPLOYMENT_TARGET=ACA)\nApp FQDN"]
    end

    AppGW --> AKSIngress
    AppGW -- "host-header override" --> ACAApp
    TM --> AKSIngress
    TM --> ACAApp

    AppGW -. "probe /q/health/ready" .-> AKSPods
    AppGW -. "probe /q/health/ready" .-> ACAApp
    TM -. "probe /q/health/ready" .-> AKSPods
    TM -. "probe /q/health/ready" .-> ACAApp

    DB[("Azure Database for PostgreSQL\nFlexible Server + pgvector")]
    EH{{"Event Hubs (Kafka protocol)\nor Strimzi on AKS"}}
    Eval["Evaluation service"]

    AKSPods --> DB
    ACAApp --> DB
    AKSPods --> EH
    ACAApp --> EH
    EH --> Eval
    Eval --> DB
```

The Application Gateway and Traffic Manager paths are alternatives, not run simultaneously — pick
one per deployment. Traffic Manager's 50/50 weighted routing is DNS-based, so clients may keep hitting the
same target until their resolver's cached record expires, unlike Application Gateway's per-request load
balancing.
