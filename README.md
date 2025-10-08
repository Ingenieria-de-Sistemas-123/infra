# infra

flowchart LR
  subgraph Client[UI futura / Postman]
  end

  Client -->|JWT| BFF[API Gateway/BFF]

  subgraph Core
    SN[snippet-service]
    LG[language-service]
    TR[test-runner-service]
    JB[jobs-orchestrator]
  end

  BFF --> SN
  BFF --> LG
  BFF --> TR
  JB --> LG
  SN -- "SnippetUpdated evt" --> TR
  LG -- "ParserVersionChanged evt" --> JB

  subgraph Infra
    PG[(Postgres)]
    S3[(S3/MinIO)]
    RD[(Redis)]
    MQ[[RabbitMQ/SQS]]
    AUTH[(Auth0/IdP)]
  end

  SN -- CRUD/versión/compartir --> PG
  SN -- artefactos --> S3
  TR -- resultados --> PG
  JB -- colas --> MQ
  LG -- cache --> RD
  BFF --> AUTH
