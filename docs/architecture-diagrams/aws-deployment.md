# AWS Deployment

AWS-specific deployment diagram: thoughtsapp running on both EKS and ECS on Fargate, behind one
ALB, sharing one RDS PostgreSQL database and one Kafka + evaluation service.

```mermaid
flowchart TB
    Users((Users))

    subgraph ALB["ALB"]
        Listener["Listener\nforward action: 50/50 split"]
        TGEKS["Target group: tg-eks\n(round robin)"]
        TGECS["Target group: tg-ecs\n(round robin)"]
        Listener -- "50%" --> TGEKS
        Listener -- "50%" --> TGECS
    end

    Users --> Listener

    subgraph EKS["EKS"]
        LBC["AWS Load Balancer Controller\n+ TargetGroupBinding"]
        Pods["thoughtsapp pods\nDEPLOYMENT_TARGET=EKS"]
    end
    LBC -- "registers (IP mode)" --> TGEKS
    TGEKS --> Pods

    subgraph ECS["ECS on Fargate"]
        Task["thoughtsapp service\n(task def env:\nDEPLOYMENT_TARGET=ECS)"]
    end
    Task -- registers --> TGECS
    TGECS --> Task

    DB[("Amazon RDS PostgreSQL\n(pgvector)")]
    MSK{{"Amazon MSK\n(or Strimzi on EKS)"}}
    Eval["Evaluation service"]

    Pods --> DB
    Task --> DB
    Pods --> MSK
    Task --> MSK
    MSK --> Eval
    Eval --> DB

    subgraph AppRunnerAlt["Optional third target (can't join the ALB)"]
        R53["Route 53\nweighted records"]
        AppRunner["App Runner instance"]
        R53 --> AppRunner
    end
    Users -. "alternative demo path" .-> R53
    AppRunner --> DB
    AppRunner --> MSK
```

App Runner cannot register with an ALB target group, so it isn't part of the 50/50 ALB split —
it's shown here as an optional separate path fronted by Route 53 weighted DNS records, or it can
simply be kept as its own standalone demo URL.
