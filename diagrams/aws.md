In this diagram, Microservices A, B, and C act as the writers (producers) on the left, and Microservices AB and BC act as the consumers of enriched data on the right.

AWS Managed CDC & Enrichment Architecture
```
graph LR
    subgraph Producers [Microservices - Write]
        A[Microservice A]
        B[Microservice B]
        C[Microservice C]
    end

    subgraph Storage [Database Layer]
        Aurora[(Amazon Aurora<br/>PostgreSQL)]
    end

    subgraph Streaming [Ingestion & Transport]
        DMS[AWS DMS<br/>CDC / Replication]
        MSK{{Amazon MSK<br/>Managed Kafka}}
    end

    subgraph Processing [Compute & Schema]
        Flink{Managed Service for<br/>Apache Flink}
        Glue[AWS Glue<br/>Schema Registry]
    end

    subgraph Consumers [Microservices - Read]
        AB[Microservice AB<br/>Enriched A+B]
        BC[Microservice BC<br/>Enriched B+C]
    end

    %% Data Flow
    A --> Aurora
    B --> Aurora
    C --> Aurora

    Aurora -.->|WAL Logs| DMS
    DMS --> MSK
    
    MSK <--> Flink
    Flink --- Glue
    
    Flink -->|Topic AB| AB
    Flink -->|Topic BC| BC

    %% Styling
    style Aurora fill:#f90,stroke:#232f3e,stroke-width:2px,color:#fff
    style MSK fill:#3b48cc,stroke:#232f3e,stroke-width:2px,color:#fff
    style Flink fill:#e13238,stroke:#232f3e,stroke-width:2px,color:#fff
    style DMS fill:#3b48cc,stroke:#232f3e,stroke-width:1px,color:#fff
```

Key Component Mapping
| On-Prem Stack | AWS Managed Equivalent | Role in the Architecture |
| :--- | :--- | :--- |
| **PostgreSQL** | **Amazon Aurora (PG)** | High-performance relational storage for services A, B, and C. |
| **Debezium** | **AWS Database Migration Service (DMS)** | Captures row-level changes (CDC) from Aurora and streams them. |
| **Apache Kafka** | **Amazon MSK** | The central message bus for raw and enriched events. |
| **Apache Flink** | **Managed Service for Apache Flink** | Joins streams (A+B and B+C) and performs real-time enrichment. |
| **Apicurio** | **AWS Glue Schema Registry** | Manages Avro/JSON schemas for serialization consistency. |

Why this works for your use case:
Decoupling: Microservices A, B, and C don't need to know about the enrichment logic; they just write to the DB.

Operational Excellence: AWS DMS eliminates the need to manage a Kafka Connect cluster manually.

Scalability: Managed Service for Apache Flink (formerly Kinesis Data Analytics) handles the state management (RocksDB) and checkpoints in S3 automatically, so you don't have to tune JVM parameters or disk space for the Flink JobManagers/TaskManagers.