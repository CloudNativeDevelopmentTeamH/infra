# infra

## Running with Docker Compose

### Prerequisites

1. **Set up environment variables** - Create a `.env` file in the `docker` directory:

```env
JWT_SECRET=your_jwt_secret_key
PEPPER=your_password_pepper

DB_USERNAME=user
DB_PASSWORD=password

RABBITMQ_USER=user
RABBITMQ_PASSWORD=password

SMTP_HOST=sandbox.smtp.mailtrap.io
SMTP_PORT=2525
SMTP_USER=user
SMTP_PASS=password
SMTP_FROM=noreply@focusboard.app
```

2. **Authenticate with ECR** (required before pulling images):

```bash
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 758280076486.dkr.ecr.eu-central-1.amazonaws.com
```

3. **Start the services**:

```bash
cd docker
docker compose up
```

**Note**: ECR authentication tokens are valid for 12 hours. You'll need to re-authenticate after that period.


## Running with Kubernetes

The project includes a unified Helm chart for deploying the backend services (auth and email) to Kubernetes clusters with production-ready configurations.

### Helm Chart Structure

The Helm chart provides a complete Kubernetes deployment including:

- **Application Deployments**: Deployments for the auth and email service
- **PostgreSQL StatefulSet**: Persistent database with persistent volume storage
- **RabbitMQ StatefulSet**: Message broker for event publishing with persistent volume storage
- **Services**: ClusterIP services for app, database, and RabbitMQ
- **ConfigMaps**: Environment configuration management (DB settings, SMTP settings)
- **Secrets**: Sensitive data management (encrypted)
- **Networking**: Services and Ingress configuration

### Prerequisites

- Kubernetes cluster (tested on k3s)
- Helm 3.x installed
- `kubectl` configured to access your cluster
- Container images available in the container registry

### Configuration

The [values.yaml](helm/values.yaml) file contains all configurable parameters for all services:

### Secrets Management

The project includes an encrypted secrets file [secret_values.enc.yaml](helm/secret_values.enc.yaml) managed with [SOPS](https://github.com/getsops/sops).

**Decrypt secrets before deployment:**

```bash
# Decrypt the encrypted secrets file
export SOPS_AGE_KEY="age1..." # Use your actual age key
sops -d ./helm/secret_values.enc.yaml > ./helm/secret_values.yaml
```

### Deploy to Kubernetes

```bash
helm install backend ./helm -n backend --create-namespace -f ./helm/secret_values.yaml
```

### Update Deployment

To update the deployment with new configurations or image versions:

```bash
helm upgrade backend ./helm -n backend -f ./helm/secret_values.yaml
```

### Uninstall

To remove the deployment:

```bash
helm uninstall backend -n backend
kubectl delete namespace backend
```
