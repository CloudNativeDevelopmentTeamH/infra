# VM-Installationsanleitung - Focusboard Application Stack

Diese Anleitung beschreibt, wie der komplette Focusboard-Application-Stack auf einer
Linux-VM **Bare-Metal (ohne Docker / Kubernetes)** installiert und konfiguriert wird.

## Voraussetzungen

- Debian-basierte Linux-Distribution (z. B. Ubuntu 22.04 oder 24.04).
- Root- oder `sudo`-Zugriff.
- **Nach außen (Firewall) muss nur Nginx erreichbar sein:** Port `80`/`443`.
  Alle übrigen Dienste binden nur an `localhost` und brauchen nicht freigegeben zu werden —
  sie müssen lediglich lokal **unbelegt** sein: `3000` (Frontend), `4000`/`50051` (Auth),
  `8088` (Focus), `8080`/`9090` (Analytics), `5672`/`15672` (RabbitMQ), `9092` (Redpanda),
  `5432` (PostgreSQL).

## Systemarchitektur

Der Stack besteht aus fünf Anwendungsdiensten und drei Infrastruktur-Komponenten:

**Anwendungsdienste**
- **Frontend**: Next.js-Anwendung (Port 3000)
- **Auth Service**: Node.js/Express + gRPC (Port 4000 HTTP, 50051 gRPC) — publiziert `user.registered` an RabbitMQ
- **Focus Service**: Quarkus/Java (Port **8088**) — gRPC-Client zu Auth (50051) & Analytics (9090), publiziert `focus.events` an Kafka
- **Analytics Service**: Go (Port 8080 HTTP, 9090 gRPC) — konsumiert `focus.events` aus Kafka
- **Email Service**: Node.js/TypeScript-Worker (kein HTTP-Port) — konsumiert `user.registered`, versendet per SMTP

**Infrastruktur**
- **PostgreSQL**: drei separate Datenbanken (`auth`, `focus`, `analytics`)
- **RabbitMQ**: Message Broker (Port 5672, Management-UI 15672) — Auth → Email Events
- **Redpanda / Kafka**: Event-Streaming (Port 9092) — Focus → Analytics Events
- **Nginx**: Reverse Proxy für HTTPS & Routing

> Da alle Dienste auf derselben VM laufen, kommunizieren sie direkt über `localhost`.
> Das Service-zu-Service-gRPC (Focus → Auth, Focus → Analytics) wird daher **nicht**
> über Nginx exponiert, sondern nur die HTTP-APIs für das Frontend.

---

## Schritt 1: Erforderliche Software installieren

System aktualisieren und Basis-Pakete (inkl. Java 21, Maven, Go und PostgreSQL) installieren:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git gnupg nginx \
  postgresql postgresql-contrib \
  openjdk-21-jdk maven golang-go
```

Node.js (über `nvm`) installieren:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20  # LTS-Version verwenden
nvm use 20
npm install -g pm2  # Prozessmanager für Node.js im Hintergrund
```

### RabbitMQ installieren

```bash
sudo apt install -y rabbitmq-server
sudo systemctl enable --now rabbitmq-server

# Benutzer anlegen und Berechtigungen setzen (Werte müssen zu den .env-Dateien passen)
sudo rabbitmqctl add_user user password
sudo rabbitmqctl set_permissions -p / user ".*" ".*" ".*"

# Optionale Management-UI auf Port 15672
sudo rabbitmq-plugins enable rabbitmq_management
```

### Redpanda (Kafka-kompatibel) installieren

```bash
curl -1sLf 'https://dl.redpanda.com/nzc4ZYQK3WRGd9sy/redpanda/cfg/setup/bash.deb.sh' | sudo -E bash
sudo apt install -y redpanda
sudo systemctl enable --now redpanda

# Event-Topic anlegen (von Focus produziert, von Analytics konsumiert)
rpk topic create focus.events -p 3
```

---

## Schritt 2: Datenbanken einrichten

PostgreSQL-Nutzer und die **drei** Datenbanken anlegen. Die DB-Namen müssen exakt
`auth`, `focus` und `analytics` lauten (so erwarten es die Anwendungen):
```bash
sudo -u postgres psql <<EOF
-- Gemeinsamer Login-Rolle für Auth & Focus
CREATE USER focus_user WITH PASSWORD 'sicheres_passwort';

-- Analytics verwendet standardmäßig einen eigenen Login (postgres://analytics:analytics@...)
CREATE USER analytics WITH PASSWORD 'analytics';

CREATE DATABASE auth      OWNER focus_user;
CREATE DATABASE focus     OWNER focus_user;
CREATE DATABASE analytics OWNER analytics;
EOF
```

> Passe Benutzer/Passwort bei Bedarf an — sie müssen mit den jeweiligen `.env`-Dateien
> bzw. der `POSTGRES_DSN` des Analytics-Service übereinstimmen.

---

## Schritt 3: Repositories klonen

Jeder Service liegt in einem **eigenen Repository**. Alle in ein gemeinsames
Arbeitsverzeichnis nebeneinander klonen:
```bash
mkdir focusboard && cd focusboard

git clone <auth-repository-url>      auth
git clone <focus-repository-url>     focus
git clone <frontend-repository-url>  frontend
git clone <email-repository-url>     email
git clone <analytics-repository-url> analytics
```

Die folgenden Schritte gehen davon aus, dass die Service-Ordner
(`auth/`, `focus/`, `frontend/`, `email/`, `analytics/`) so nebeneinander vorliegen.

---

## Schritt 4: Anwendungen einrichten & starten

### 1. Auth Service (Node.js/Express)

```bash
cd auth
npm install

# .env anlegen (siehe Variablen unten)
nano .env

# TypeScript bauen
npm run build

# Datenbankschema migrieren (Drizzle)
npx drizzle-kit push --config=drizzle.config.ts

# RabbitMQ-Topologie (Exchanges/Queues) einrichten
npm run setup:rabbitmq

# Mit pm2 starten
pm2 start dist/server.js --name "auth-service"
cd ..
```

Benötigte `.env`-Variablen (Auth):
```env
PORT=4000
GRPC_PORT=50051
PEPPER=<base64-pepper>
JWT_SECRET=<base64-secret>
DB_HOST=localhost
DB_PORT=5432
DB_NAME=auth
DB_USERNAME=focus_user
DB_PASSWORD=sicheres_passwort
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=user
RABBITMQ_PASSWORD=password
CORS_ORIGIN=https://meine-domain.de
```

### 2. Email Service (Node.js/TypeScript-Worker)

```bash
cd email
npm install

# .env anlegen (RABBITMQ_* und SMTP_*)
nano .env

npm run build

# RabbitMQ-Topologie einrichten (Consumer-Queue)
npm run setup:rabbitmq

pm2 start dist/server.js --name "email-service"
cd ..
```

Benötigte `.env`-Variablen (Email):
```env
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=user
RABBITMQ_PASSWORD=password
SMTP_HOST=sandbox.smtp.mailtrap.io
SMTP_PORT=2525
SMTP_USER=<smtp-user>
SMTP_PASS=<smtp-pass>
SMTP_FROM=noreply@focusboard.app
```

### 3. Focus Service (Java/Quarkus)

Flyway führt die DB-Migrationen automatisch beim Start aus. Focus benötigt zur Laufzeit
einen erreichbaren Auth-gRPC (50051), Analytics-gRPC (9090) und Kafka (9092).

```bash
cd focus

# Anwendung kompilieren (lädt Maven-Abhängigkeiten)
./mvnw clean package -DskipTests

# Backend starten (Konfiguration via Umgebungsvariablen, siehe unten)
nohup java -jar target/quarkus-app/quarkus-run.jar > focus.log 2>&1 &
cd ..
```

Wichtige Umgebungsvariablen (z. B. in einem Wrapper-Script oder via systemd `Environment=`):
```env
QUARKUS_HTTP_PORT=8088
QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://localhost:5432/focus
QUARKUS_DATASOURCE_USERNAME=focus_user
QUARKUS_DATASOURCE_PASSWORD=sicheres_passwort
QUARKUS_GRPC_CLIENTS_AUTH_HOST=localhost
QUARKUS_GRPC_CLIENTS_AUTH_PORT=50051
QUARKUS_GRPC_CLIENTS_ANALYTICS_HOST=localhost
QUARKUS_GRPC_CLIENTS_ANALYTICS_PORT=9090
FOCUS_EVENTS_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
FOCUS_EVENTS_KAFKA_TOPIC=focus.events
QUARKUS_HTTP_CORS_ORIGINS=https://meine-domain.de
```

### 4. Analytics Service (Go)

Konsumiert `focus.events` aus Kafka und schreibt in die `analytics`-Datenbank.

```bash
cd analytics/backend   # ggf. Pfad zum Analytics-Quellcode anpassen
go build -o analytics-service ./...

# Mit Umgebungsvariablen starten
pm2 start ./analytics-service --name "analytics-service"
cd ../..
```

Benötigte Umgebungsvariablen (Analytics):
```env
PORT=8080
GRPC_PORT=9090
KAFKA_BROKERS=localhost:9092
KAFKA_TOPIC=focus.events
KAFKA_GROUP_ID=analytics-service
STORE_BACKEND=postgres
POSTGRES_DSN=postgres://analytics:analytics@localhost:5432/analytics?sslmode=disable
```

### 5. Frontend (Next.js)

> **Wichtig:** `NEXT_PUBLIC_*`-Variablen sind **Build-Zeit-Variablen** und müssen
> **vor** `npm run build` gesetzt sein. Sie müssen auf die öffentlich erreichbaren
> URLs zeigen (über Nginx, siehe Schritt 5).

```bash
cd frontend
npm install

# .env für Build-Zeit-Variablen anlegen
nano .env

npm run build
pm2 start npm --name "frontend" -- start
cd ..
```

Benötigte `.env`-Variablen (Frontend):
```env
NEXT_PUBLIC_AUTH_API_BASE_URL=https://meine-domain.de/api/auth
NEXT_PUBLIC_API_BASE_URL=https://meine-domain.de/api/focus
APP_VERSION=1.0.0
```

### pm2 für Autostart sichern

```bash
pm2 startup   # gibt einen sudo-Befehl aus → ausführen
pm2 save
```

---

## Schritt 5: Nginx Reverse Proxy konfigurieren

```bash
sudo nano /etc/nginx/sites-available/focusboard
```

Folgende Konfiguration einfügen (Pfad-Präfixe `/api/auth/` und `/api/focus/` werden
auf die jeweiligen Backends abgebildet und müssen zu den `NEXT_PUBLIC_*`-URLs des
Frontends passen):
```nginx
server {
    listen 80;
    server_name meine-domain.de;

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Auth API
    location /api/auth/ {
        proxy_pass http://localhost:4000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Focus API (Port 8088)
    location /api/focus/ {
        proxy_pass http://localhost:8088/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Analytics API (optional)
    location /api/analytics/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> Die `CORS_ORIGIN` (Auth) bzw. `QUARKUS_HTTP_CORS_ORIGINS` (Focus) müssen auf
> `https://meine-domain.de` gesetzt sein, damit Browser-Requests zugelassen werden.

Aktivieren und neu starten:
```bash
sudo ln -s /etc/nginx/sites-available/focusboard /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## Schritt 6: TLS (empfohlen)

SSL-Zertifikate per Let's Encrypt / Certbot hinzufügen:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d meine-domain.de
```

---

## Startreihenfolge & Abhängigkeiten

Beim (Neu-)Start sollte die Reihenfolge die Abhängigkeiten respektieren:

1. PostgreSQL, RabbitMQ, Redpanda (Infrastruktur)
2. Auth Service (richtet RabbitMQ-Topologie ein, stellt gRPC bereit)
3. Analytics Service (stellt gRPC bereit, konsumiert Kafka)
4. Email Service (konsumiert RabbitMQ)
5. Focus Service (benötigt Auth- & Analytics-gRPC sowie Kafka)
6. Frontend
