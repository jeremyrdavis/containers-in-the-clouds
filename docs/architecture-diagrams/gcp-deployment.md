# GCP Deployment

GCP-specific deployment diagram: thoughtsapp running on both GKE and Cloud Run, behind a global
external Application Load Balancer, sharing one Cloud SQL database and one Kafka + evaluation
service.

```mermaid
flowchart TB
    Users((Users))

    subgraph GLB["Global external Application Load Balancer"]
        URLMap["URL map\nweightedBackendServices"]
        BSGKE["Backend service: bs-gke"]
        BSCR["Backend service: bs-cloudrun"]
        BSMIG["Backend service: bs-mig\n(optional)"]
        URLMap -- "50%" --> BSGKE
        URLMap -- "50%" --> BSCR
        URLMap -. "equal weight (optional)" .-> BSMIG
    end

    Users --> URLMap

    subgraph GKE["GKE"]
        NEG["Standalone zonal NEG\n(cloud.google.com/neg annotation)"]
        Pods["thoughtsapp pods\nDEPLOYMENT_TARGET=GKE"]
        NEG --> Pods
    end
    BSGKE --> NEG

    subgraph CloudRun["Cloud Run"]
        SNEG["Serverless NEG"]
        Service["thoughtsapp service\n(--set-env-vars\nDEPLOYMENT_TARGET=CLOUD_RUN)"]
        SNEG --> Service
    end
    BSCR --> SNEG

    subgraph MIGOpt["Compute Engine (optional third target)"]
        MIG["Managed instance group"]
    end
    BSMIG -.-> MIG

    DB[("Cloud SQL for PostgreSQL\n(pgvector)")]
    Kafka{{"Google Managed Service\nfor Apache Kafka\n(or Strimzi on GKE)"}}
    Eval["Evaluation service\n(Cloud Run worker pool)"]

    Pods --> DB
    Service --> DB
    MIG -.-> DB
    Pods --> Kafka
    Service --> Kafka
    MIG -.-> Kafka
    Kafka --> Eval
    Eval --> DB
```

The Compute Engine MIG backend is an optional third target at equal weight, not part of the
required 50/50 GKE/Cloud Run split — shown with dashed edges. Unlike the microservices evaluation
service on GKE, the Cloud Run evaluation deployment runs as a **worker pool**, not a
request-serving service, since it only needs to consume Kafka events.
