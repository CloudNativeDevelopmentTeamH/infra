# infra

## Running with Docker Compose

### Prerequisites

1. **Set up environment variables** - Create a `.env` file in the `docker` directory:

```env
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=your_password
DB_NAME=auth_db
JWT_SECRET=your_jwt_secret_key
PEPPER=your_password_pepper
GRPC_PORT=50051
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

4. **(!not necessary!) Initial Database Setup:**

After the services are running, set up the database schema using Drizzle:

```bash
# Navigate to the auth service directory
# Push the schema to the database
npx drizzle-kit push
```

**Note**: ECR authentication tokens are valid for 12 hours. You'll need to re-authenticate after that period.