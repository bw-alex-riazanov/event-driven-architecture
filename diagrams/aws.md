In this diagram, Microservices A, B, and C act as the writers (producers) on the left, and Microservices AB and BC act as the consumers of enriched data on the right.

AWS Managed CDC & Enrichment Architecture
```mermaid
graph LR
    subgraph Producers [Microservices - Write]
        A[Microservice A]
        B[Microservice B]
        C[Microservice C]
    end

    subgraph Storage [Database Layer]
        DB_A[(Aurora A)]
        DB_B[(Aurora B)]
        DB_C[(Aurora C)]
    end

    subgraph Ingestion [CDC Layer]
        DMS[AWS DMS<br/>CDC / Replication]
    end

    subgraph Streaming [Messaging Layer]
        subgraph MSK [Amazon MSK]
            T_Raw[Raw Topics<br/>A, B, C]
            T_Enriched[Enriched Topics<br/>AB, BC]
        end
    end

    subgraph Processing [Compute & State]
        subgraph FlinkApp [Managed Service for Apache Flink]
            Flink{Flink Engine}
            Rocks[(RocksDB<br/>Local State)]
            Flink <--> Rocks
        end
        S3[(Amazon S3<br/>Checkpoints)]
        Rocks -.->|Snapshots| S3
    end

    subgraph Consumers [Microservices - Read]
        MS_AB[Microservice AB]
        MS_BC[Microservice BC]
    end

    %% Data Flow
    A --> DB_A
    B --> DB_B
    C --> DB_C

    DB_A & DB_B & DB_C -.-> DMS
    DMS --> T_Raw
    
    T_Raw --> Flink
    Flink --> T_Enriched
    
    T_Enriched -->|Topic AB| MS_AB
    T_Enriched -->|Topic BC| MS_BC

    %% Styling
    style DB_A fill:#f90,stroke:#23
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

Estimated Monthly Cost (100 TPS / ~260M events per month)

### Estimated Monthly Cost (AWS Managed Stack)
**Region:** US East (N. Virginia)  
**Throughput:** ~100 writes/sec (~260M events/month)

| Service | Component | Monthly Cost (Serverless) | Monthly Cost (Provisioned) | Cost Driver |
| :--- | :--- | :--- | :--- | :--- |
| **Amazon Aurora** | Database (PostgreSQL) | $106.00 | $85.00 | Storage + I/O + Instance |
| **AWS DMS** | CDC Replication | $63.00 | $63.00 | dms.t3.medium instance |
| **Amazon MSK** | Kafka Streaming | $584.00 | $180.00 | Cluster hours + Throughput |
| **Managed Flink** | Stream Processing | $160.00 | $160.00 | 2 KPU (Processing Units) |
| **Glue Registry** | Schema Management | $0.00 | $0.00 | Free Tier (up to 1M req) |
| **TOTAL** | | **~$913.00** | **~$488.00** | |

---

#### 💡 Key Takeaways for Optimization:
* **MSK Serverless vs. Provisioned:** Serverless is great for "zero-management," but it carries a high base fee of ~$0.75/hr. Switching to **Provisioned m7g.large** instances can save you over **$400/month** if your load is steady.
* **Aurora Serverless v2:** For 100 TPS, Aurora is highly efficient, scaling down to 0.5 ACU during idle periods.
* **Managed Flink:** The minimum cost is 2 KPUs (1 for the application and 1 for orchestration). This is the "luxury" part of the stack that replaces manual Flink cluster maintenance.
