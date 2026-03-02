# Jamph-MLFlow-PostgreSQL
This is our custom MLFlow setup with PostgreSQL for tracking and evaluating JAMPH-trained OllAMA AI models. It will be hosted at NAIS eventually, so you don't need to install it locally, but you can run a local instance for testing if desired.

## MLflow PostgreSQL Backend Installation

This is a concise setup guide based on the Community Charts MLflow PostgreSQL backend docs.

### Quick Start (Local Development)

#### Prerequisites
- Docker Desktop

#### Step 1: Start PostgreSQL in Docker
Create a Docker network for MLflow services:
```bash
docker network create mlflow-network
```

Start PostgreSQL container:
```bash
docker run --name mlflow-postgres \
  --network mlflow-network \
  -e POSTGRES_USER=mlflow_user \
  -e POSTGRES_PASSWORD=your_secure_password \
  -e POSTGRES_DB=mlflow \
  -p 5432:5432 \
  -d postgres:16
```

Or start existing container:
```bash
docker start mlflow-postgres
```

#### Step 2: Build MLflow Image with PostgreSQL Support
Build custom image with psycopg2:
```bash
docker build -t mlflow-postgres:latest .
```

#### Step 3: Start MLflow Server in Docker
Run MLflow server container connected to PostgreSQL:
```bash
docker run --name mlflow-server \
  --network mlflow-network \
  -p 5001:5000 \
  -v $(pwd)/mlartifacts:/mlflow/artifacts \
  -d mlflow-postgres:latest \
  --host 0.0.0.0 \
  --port 5000 \
  --backend-store-uri postgresql://mlflow_user:your_secure_password@mlflow-postgres:5432/mlflow \
  --default-artifact-root /mlflow/artifacts
```

Or start existing container:
```bash
docker start mlflow-server
```

Access the UI at: **http://localhost:5001**

#### Step 4: Connect Jamph RAG Umami API to MLflow
In your Kotlin application, configure the MLflow tracking URI:

**Environment variable:**
```bash
MLFLOW_TRACKING_URI=http://localhost:5001
```

**Or in application.properties/application.yml:**
```properties
mlflow.tracking.uri=http://localhost:5001
```

**Connection details:**
- PostgreSQL stores experiment metadata, parameters, and metrics
- MLflow server exposes REST API at port 5001
- Artifacts (models, files) are stored in `./mlartifacts` volume
- Both containers communicate via `mlflow-network` Docker network



