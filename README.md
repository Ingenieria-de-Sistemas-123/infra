# infra
## System Architecture – Snippet Searcher

Below is the current component diagram for our microservices architecture:

```mermaid
flowchart LR
  %% External clients
  subgraph Client[UI futura / Postman]
  end

  Client -->|JWT| BFF[API Gateway / BFF]

  %% Core microservices
  subgraph Core
    SN[Snippet Service]
    LG[Language Service]
    TR[Test Runner Service]
    JB[Jobs Orchestrator]
  end

  BFF --> SN
  BFF --> LG
  BFF --> TR
  JB --> LG
  SN -- "SnippetUpdated evt" --> TR
  LG -- "ParserVersionChanged evt" --> JB

  %% Infrastructure
  subgraph Infra[Infrastructure]
    PG[(PostgreSQL)]
    S3[(S3 / MinIO)]
    RD[(Redis)]
    MQ[(RabbitMQ / SQS)]
    AUTH[(Auth0 / IdP)]
  end

  SN --> PG
  SN --> S3
  TR --> PG
  JB --> MQ
  LG --> RD
  BFF --> AUTH
