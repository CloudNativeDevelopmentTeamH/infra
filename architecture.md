# FocusBoard - Complete System Architecture

## 🏗️ Kubernetes Architecture (k3s)

This configuration models the localized k3s architecture, utilizing Helm for direct deployments.

```mermaid
graph TB
    subgraph Internet["🌐 Internet"]
        Users[End Users<br/>Web Browsers]
    end

    subgraph Infrastructure["☁️ Local Infrastructure"]
        subgraph K3s["k3s Cluster"]
            subgraph Ingress["Ingress Layer"]
                Traefik[Traefik Ingress Controller]
            end

            subgraph Frontend["Frontend Service"]
                NextJS[Next.js App<br/>Port 3000]
            end

            subgraph AuthService["Auth Microservice"]
                AuthHTTP[Auth Service<br/>HTTP REST API<br/>Port 4000]
                AuthGRPC[Auth Service<br/>gRPC API<br/>Port 50051]
            end

            subgraph FocusService["Focus Microservice"]
                FocusApp[Focus Service<br/>Quarkus Java<br/>Port 8088]
            end

            subgraph AnalyticsService["Analytics Microservice"]
                AnalyticsApp[Analytics Service<br/>HTTP 8080 / gRPC 9090]
            end

            subgraph EmailService["Email Microservice"]
                EmailApp[Email Service]
            end

            subgraph MessageBroker["Message Brokers"]
                RabbitMQ[(RabbitMQ<br/>Port 5672 / Mgmt 15672)]
                Redpanda[(Redpanda / Kafka<br/>Port 9092)]
            end

            subgraph Storage["Persistent Storage"]
                LocalPath[Local Path Provisioner<br/>Volumes]
            end

            subgraph Data["Data Layer"]
                AuthDB[(Auth Database<br/>PostgreSQL)]
                FocusDB[(Focus Database<br/>PostgreSQL)]
                AnalyticsDB[(Analytics Database)]
            end
        end

        subgraph Management["Management"]
            Dev[Local Developer<br/>Helm]
        end
    end

    subgraph GitOps["🔄 CI/CD & Registry"]
        CI[CI Pipeline]
        Registry[Git Container Registry<br/>GHCR]
    end

    %% User flows
    Users -->|HTTPS| Traefik
    Traefik -->|frontend.localhost| NextJS
    Traefik -->|auth.localhost| AuthHTTP
    Traefik -->|focus.localhost| FocusApp
    Traefik -->|analytics.localhost| AnalyticsApp
    Traefik -.->|rabbitmq.localhost<br/>Mgmt UI| RabbitMQ

    %% Service-to-service communication (gRPC)
    FocusApp -.->|gRPC Authentication| AuthGRPC
    FocusApp -.->|gRPC Analytics| AnalyticsApp

    %% Async Communication (RabbitMQ)
    AuthHTTP -->|user.registered| RabbitMQ
    RabbitMQ -->|consume| EmailApp
    EmailApp -->|SMTP| ExternalSMTP[External SMTP]

    %% Async Communication (Kafka / Redpanda)
    FocusApp -->|publish focus.events| Redpanda
    AnalyticsApp -->|consume focus.events| Redpanda

    %% Data layer
    AuthHTTP --> AuthDB
    AuthGRPC --> AuthDB
    FocusApp --> FocusDB
    AnalyticsApp --> AnalyticsDB

    AuthDB --> LocalPath
    FocusDB --> LocalPath
    RabbitMQ --> LocalPath
    Redpanda --> LocalPath
    AnalyticsDB --> LocalPath

    %% Deployment & CI Flow
    CI -->|Build & Push Images| Registry
    K3s -.->|Pulls Images| Registry
    Dev -.->|Deploy locally via Helm| K3s

    style Users fill:#8e24aa,stroke:#ce93d8,stroke-width:2px,color:#fff
    style Traefik fill:#e65100,stroke:#ffb74d,stroke-width:2px,color:#000
    style NextJS fill:#1565c0,stroke:#64b5f6,stroke-width:2px,color:#fff
    style AuthHTTP fill:#2e7d32,stroke:#81c784,stroke-width:2px,color:#fff
    style AuthGRPC fill:#2e7d32,stroke:#81c784,stroke-width:2px,color:#fff
    style FocusApp fill:#00695c,stroke:#4db6ac,stroke-width:2px,color:#fff
    style EmailApp fill:#00838f,stroke:#4dd0e1,stroke-width:2px,color:#fff
    style AnalyticsApp fill:#6a1b9a,stroke:#ce93d8,stroke-width:2px,color:#fff
    style AuthDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style FocusDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style AnalyticsDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style RabbitMQ fill:#ffb300,stroke:#ffe082,stroke-width:2px,color:#000
    style Redpanda fill:#ffb300,stroke:#ffe082,stroke-width:2px,color:#000
    style ExternalSMTP fill:#455a64,stroke:#90a4ae,stroke-width:2px,color:#fff
    style LocalPath fill:#bf360c,stroke:#ff8a65,stroke-width:2px,color:#fff
```


## 👨‍💻 Local Development Architecture

This setup is defined in [docker/compose.yml](docker/compose.yml). All containers share a
single `default_network` (bridge).

```mermaid
graph TB
    subgraph Developer["Local Development Environment"]
        Dev[Developer<br/>Laptop/Workstation]
        Browser[Web Browser]
    end

    subgraph Compose["🐳 Docker Compose — default_network (bridge)"]
        subgraph Services["Application Services"]
            LocalFrontend[frontend<br/>Next.js<br/>Port 3000]
            LocalAuth[auth_app<br/>Node.js/TypeScript<br/>HTTP 4000 / gRPC 50051]
            LocalFocus[focus<br/>Quarkus/Java<br/>Port 8088]
            LocalAnalytics[analytics<br/>Go<br/>gRPC 9090]
            LocalEmail[auth_email<br/>Node.js/TypeScript<br/>Worker - no port]
        end

        subgraph Brokers["Message Brokers"]
            LocalRabbitMQ[(auth_queue<br/>RabbitMQ<br/>5672 / Mgmt 15672)]
            LocalRedpanda[(redpanda<br/>Kafka API<br/>Port 9092)]
        end

        subgraph Databases["PostgreSQL 16"]
            LocalAuthDB[(auth_database<br/>DB: auth)]
            LocalFocusDB[(postgres<br/>DB: focus)]
            LocalAnalyticsDB[(analytics-postgres<br/>DB: analytics)]
        end

        subgraph Volumes["Docker Volumes"]
            AuthData[auth_db_data]
            FocusData[postgres_data]
            AnalyticsData[analytics_db_data]
            RedpandaData[redpanda_data]
        end
    end

    ExternalSMTP[External SMTP<br/>Mailtrap Sandbox]

    %% Developer workflow
    Dev -->|opens| Browser

    %% Published host ports — browser connects directly (no proxy)
    Browser -->|localhost:3000| LocalFrontend
    Browser -->|localhost:4000 REST| LocalAuth
    Browser -->|localhost:8088 REST| LocalFocus
    Browser -.->|localhost:15672 Mgmt UI| LocalRabbitMQ

    %% Service-to-service communication (gRPC)
    LocalFocus -.->|gRPC Authentication 50051| LocalAuth
    LocalFocus -.->|gRPC Analytics 9090| LocalAnalytics

    %% Async Communication (RabbitMQ)
    LocalAuth -->|user.registered| LocalRabbitMQ
    LocalRabbitMQ -->|consume| LocalEmail
    LocalEmail -->|SMTP| ExternalSMTP

    %% Async Communication (Kafka / Redpanda)
    LocalFocus -->|publish focus.events| LocalRedpanda
    LocalAnalytics -->|consume focus.events| LocalRedpanda

    %% Database connections
    LocalAuth -->|SQL| LocalAuthDB
    LocalFocus -->|SQL| LocalFocusDB
    LocalAnalytics -->|SQL| LocalAnalyticsDB

    %% Storage (RabbitMQ is ephemeral - no volume)
    LocalAuthDB --> AuthData
    LocalFocusDB --> FocusData
    LocalAnalyticsDB --> AnalyticsData
    LocalRedpanda --> RedpandaData

    style Dev fill:#8e24aa,stroke:#ce93d8,stroke-width:2px,color:#fff
    style Browser fill:#6a1b9a,stroke:#ce93d8,stroke-width:2px,color:#fff
    style LocalFrontend fill:#1565c0,stroke:#64b5f6,stroke-width:2px,color:#fff
    style LocalAuth fill:#2e7d32,stroke:#81c784,stroke-width:2px,color:#fff
    style LocalFocus fill:#00695c,stroke:#4db6ac,stroke-width:2px,color:#fff
    style LocalEmail fill:#00838f,stroke:#4dd0e1,stroke-width:2px,color:#fff
    style LocalAnalytics fill:#6a1b9a,stroke:#ce93d8,stroke-width:2px,color:#fff
    style LocalAuthDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style LocalFocusDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style LocalAnalyticsDB fill:#c62828,stroke:#ef9a9a,stroke-width:2px,color:#fff
    style LocalRabbitMQ fill:#ffb300,stroke:#ffe082,stroke-width:2px,color:#000
    style LocalRedpanda fill:#ffb300,stroke:#ffe082,stroke-width:2px,color:#000
    style ExternalSMTP fill:#455a64,stroke:#90a4ae,stroke-width:2px,color:#fff
    style AuthData fill:#bf360c,stroke:#ff8a65,stroke-width:2px,color:#fff
    style FocusData fill:#bf360c,stroke:#ff8a65,stroke-width:2px,color:#fff
    style AnalyticsData fill:#bf360c,stroke:#ff8a65,stroke-width:2px,color:#fff
    style RedpandaData fill:#bf360c,stroke:#ff8a65,stroke-width:2px,color:#fff
```
