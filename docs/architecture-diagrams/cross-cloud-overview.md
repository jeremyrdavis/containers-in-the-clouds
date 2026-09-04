# Cross-Cloud Overview

Top-level architecture diagram showing the entire demo at a glance: the same thoughtsapp image set
deployed to multiple container targets in Azure, AWS, and GCP, each cloud fronted by its own load
balancer, each cloud with exactly one shared PostgreSQL database and one Kafka + evaluation service.

```mermaid
flowchart LR
    Users((Users))

    subgraph Azure["Azure"]
        AzureLB["Application Gateway /\nTraffic Manager"]
        AKS["AKS\nDEPLOYMENT_TARGET=AKS"]
        ACA["Container Apps\nDEPLOYMENT_TARGET=ACA"]
        AzureDB[("PostgreSQL + pgvector")]
        AzureKafka{{"Kafka"}}
        AzureEval["Evaluation service"]

        AzureLB --> AKS
        AzureLB --> ACA
        AKS --> AzureDB
        ACA --> AzureDB
        AKS --> AzureKafka
        ACA --> AzureKafka
        AzureKafka --> AzureEval
        AzureEval --> AzureDB
    end

    subgraph AWS["AWS"]
        AWSLB["ALB\n(weighted target groups)"]
        EKS["EKS\nDEPLOYMENT_TARGET=EKS"]
        ECS["ECS on Fargate\nDEPLOYMENT_TARGET=ECS"]
        AWSDB[("PostgreSQL + pgvector")]
        AWSKafka{{"Kafka"}}
        AWSEval["Evaluation service"]

        AWSLB --> EKS
        AWSLB --> ECS
        EKS --> AWSDB
        ECS --> AWSDB
        EKS --> AWSKafka
        ECS --> AWSKafka
        AWSKafka --> AWSEval
        AWSEval --> AWSDB
    end

    subgraph GCP["GCP"]
        GCPLB["Global external\nApplication Load Balancer"]
        GKE["GKE\nDEPLOYMENT_TARGET=GKE"]
        CloudRun["Cloud Run\nDEPLOYMENT_TARGET=CLOUD_RUN"]
        GCPDB[("PostgreSQL + pgvector")]
        GCPKafka{{"Kafka"}}
        GCPEval["Evaluation service"]

        GCPLB --> GKE
        GCPLB --> CloudRun
        GKE --> GCPDB
        CloudRun --> GCPDB
        GKE --> GCPKafka
        CloudRun --> GCPKafka
        GCPKafka --> GCPEval
        GCPEval --> GCPDB
    end

    Users --> AzureLB
    Users --> AWSLB
    Users --> GCPLB
```
