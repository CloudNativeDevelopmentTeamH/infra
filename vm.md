# VM-Installationsanleitung - Focusboard Application Stack

Diese Anleitung beschreibt, wie der komplette Focusboard-Application-Stack (Frontend, Focus Service, Auth Service) auf einer Linux-VM (Bare-Metal ohne Docker) installiert und konfiguriert wird.

## Voraussetzungen

- Debian-basierte Linux-Distribution (z. B. Ubuntu 22.04 oder 24.04).
- Root- oder `sudo`-Zugriff.

## Systemarchitektur

Die Anwendung besteht aus vier Hauptdiensten:
- **Frontend**: Next.js-Anwendung (Port 3000)
- **Auth Service**: Node.js/Express mit gRPC (Ports: 4000 HTTP, 50051 gRPC)
- **Focus Service**: Quarkus/Java-Anwendung (Port 8080)
- **Analytics Service**: Go Fiber Anwendung
- **PostgreSQL**: Zwei separate Datenbanken (`auth_db`, `focus_db`)
- **Nginx**: Reverse Proxy für HTTPS & Routing

---

## Schritt 1: Erforderliche Software installieren

System aktualisieren und benötigte Pakete (inkl. Java und Postgres) installieren:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nginx postgresql postgresql-contrib openjdk-21-jdk maven
```

Node.js (über `nvm`) installieren:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20  # LTS-Version verwenden
nvm use 20
npm install -g pm2  # Prozessmanager für Node.js im Hintergrund
```

---

## Schritt 2: Datenbank einrichten

PostgreSQL-Nutzer und die beiden Datenbanken anlegen:
```bash
sudo -u postgres psql <<EOF
CREATE USER focus_user WITH PASSWORD 'sicheres_passwort';
CREATE DATABASE auth_db OWNER focus_user;
CREATE DATABASE focus_db OWNER focus_user;
EOF
```

---

## Schritt 3 & 4: Anwendungen einrichten & DB initialisieren

Repository klonen:
```bash
# git clone <deine-repository-url> focusboard
# cd focusboard
```

### 1. Auth Service
```bash
cd auth
npm install
cp .env.example .env  # Datenbank-Passwort und Einstellungen anpassen

# Schema in die Datenbank pushen (Drizzle)
npm run db:push

# Bauen und mit pm2 starten
npm run build
pm2 start dist/server.js --name "auth-service"
cd ..
```

### 2. Focus Service (Java/Quarkus)
```bash
cd focus
# Anwendung kompilieren (lädt Maven-Abhängigkeiten)
./mvnw clean package -DskipTests

# Backend starten (Flyway führt die DB-Migrationen automatisch beim Start aus)
nohup java -jar target/quarkus-app/quarkus-run.jar > focus.log 2>&1 &
cd ..
```

### 3. Frontend
```bash
cd frontend
npm install
# Erstelle die .env-Datei für notwendige Umgebungsvariablen (API-URLs etc.)

# Bauen und starten
npm run build
pm2 start npm --name "frontend" -- start
cd ..
```

---

## Schritt 5: Nginx Reverse Proxy konfigurieren

Erstellen einer Nginx-Konfiguration, um den Traffic weiterzuleiten:
```bash
sudo nano /etc/nginx/sites-available/focusboard
```

Füge folgende Konfiguration ein:
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

    # Focus API
    location /api/focus/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Aktivieren und neu starten:
```bash
sudo ln -s /etc/nginx/sites-available/focusboard /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## Schritt 6: TLS (optional)

SSL-Zertifikate hinzufügen per Let's Encrypt / Certbot:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d meine-domain.de
```