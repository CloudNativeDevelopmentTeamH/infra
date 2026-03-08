# infra

## Running with Docker Compose

### Prerequisites

1. **Set up environment variables** - Create a `.env` file in the `docker` directory:

```env
JWT_SECRET=your_jwt_secret_key
PEPPER=your_password_pepper

DB_NAME=auth
DB_USERNAME=test
DB_PASSWORD=test

RABBITMQ_USER=test
RABBITMQ_PASSWORD=test
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