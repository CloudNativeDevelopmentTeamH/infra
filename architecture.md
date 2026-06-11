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

    style Users fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
    style Traefik fill:#ff9800,stroke:#e65100,stroke-width:3px
    style NextJS fill:#42a5f5,stroke:#0d47a1,stroke-width:3px
    style AuthHTTP fill:#66bb6a,stroke:#1b5e20,stroke-width:3px
    style AuthGRPC fill:#66bb6a,stroke:#1b5e20,stroke-width:3px
    style FocusApp fill:#26a69a,stroke:#00695c,stroke-width:3px
    style EmailApp fill:#90caf9,stroke:#1565c0,stroke-width:3px
    style AnalyticsApp fill:#ab47bc,stroke:#6a1b9a,stroke-width:3px
    style AuthDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style FocusDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style AnalyticsDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style RabbitMQ fill:#ffcc80,stroke:#e65100,stroke-width:3px
    style Redpanda fill:#ffcc80,stroke:#e65100,stroke-width:3px
    style ExternalSMTP fill:#cfd8dc,stroke:#607d8b,stroke-width:3px
    style LocalPath fill:#d84315,stroke:#bf360c,stroke-width:3px
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

    style Dev fill:#ce93d8,stroke:#6a1b9a,stroke-width:3px
    style Browser fill:#ab47bc,stroke:#4a148c,stroke-width:3px
    style LocalFrontend fill:#42a5f5,stroke:#0d47a1,stroke-width:3px
    style LocalAuth fill:#66bb6a,stroke:#1b5e20,stroke-width:3px
    style LocalFocus fill:#26a69a,stroke:#00695c,stroke-width:3px
    style LocalEmail fill:#90caf9,stroke:#1565c0,stroke-width:3px
    style LocalAnalytics fill:#ab47bc,stroke:#6a1b9a,stroke-width:3px
    style LocalAuthDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style LocalFocusDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style LocalAnalyticsDB fill:#e53935,stroke:#b71c1c,stroke-width:3px
    style LocalRabbitMQ fill:#ffcc80,stroke:#e65100,stroke-width:3px
    style LocalRedpanda fill:#ffcc80,stroke:#e65100,stroke-width:3px
    style ExternalSMTP fill:#cfd8dc,stroke:#607d8b,stroke-width:3px
    style AuthData fill:#d84315,stroke:#bf360c,stroke-width:3px
    style FocusData fill:#d84315,stroke:#bf360c,stroke-width:3px
    style AnalyticsData fill:#d84315,stroke:#bf360c,stroke-width:3px
    style RedpandaData fill:#d84315,stroke:#bf360c,stroke-width:3px
```
