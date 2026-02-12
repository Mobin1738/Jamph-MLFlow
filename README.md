# Jamph-MLFlow-PostgreSQL
This a custom MLflow with PostgreSQL used for monitoring and evaluating performance and of JAMPH trained OllAMA AI models
## MLflow PostgreSQL Backend Installation

This is a concise setup guide based on the Community Charts MLflow PostgreSQL backend docs.

### Quick Start (Local Development)

#### Prerequisites
- Docker Desktop (for PostgreSQL container)
- Python 3.10+
- macOS, Linux, or Windows with WSL2

#### Step 1: Set up Python virtual environment
```bash
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
python -m pip install --upgrade pip setuptools wheel
pip install mlflow psycopg2-binary
```

#### Step 2: Start PostgreSQL in Docker (Skip over this step if you have an existing mflow-postgres container in docker)
```bash
docker run --name mlflow-postgres \
  -e POSTGRES_USER=mlflow_user \
  -e POSTGRES_PASSWORD=your_secure_password \
  -e POSTGRES_DB=mlflow \
  -p 5432:5432 \
  -d postgres:16
```

#### Optional: Start PostgreSQL if you have an existing container
```bash
docker start mlflow-postgres

```

#### Step 3: Start MLflow server
```bash
mlflow server \
  --host 0.0.0.0 \
  --port 5001 \
  --backend-store-uri postgresql://mlflow_user:your_secure_password@localhost:5432/mlflow \
  --default-artifact-root ./mlartifacts
```

Access the UI at: **http://localhost:5001**

#### Step 4: Connect Kotlin API (Jamph RAG Umami)?
Set the tracking URI in your application:
```
MLFLOW_TRACKING_URI=http://localhost:5001
```

---

### Prerequisites (for Kubernetes deployment)
- PostgreSQL database (external or in-cluster)
- Storage class for PVC (if using in-cluster PostgreSQL)

Optional PostgreSQL client:
```bash
brew install postgresql
```

#### Windows (Winget)

Optional PostgreSQL client:
```powershell
winget install PostgreSQL.PostgreSQL
```

#### Linux (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```

Optional PostgreSQL client:
```bash
sudo apt-get install -y postgresql-client
```

### Option 1: External PostgreSQL (recommended for production)

#### Step 1: Create DB and user
```sql
CREATE DATABASE mlflow;
CREATE USER mlflow_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE mlflow TO mlflow_user;
```

#### Step 2: Install MLflow with external DB
```bash
helm install mlflow community-charts/mlflow \
	--namespace mlflow \
	--set backendStore.databaseMigration=true \
	--set backendStore.postgres.enabled=true \
	--set backendStore.postgres.host=your-postgres-host \
	--set backendStore.postgres.port=5432 \
	--set backendStore.postgres.database=mlflow \
	--set backendStore.postgres.user=mlflow_user \
	--set backendStore.postgres.password=your_secure_password
```

WARNING: Do not hardcode passwords on the command line. Use Kubernetes secrets instead.

### Option 2: PostgreSQL in Kubernetes (dev/test only)

#### Step 1: Install PostgreSQL (Bitnami chart)
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres bitnami/postgresql \
	--namespace mlflow \
	--set auth.postgresPassword=mlflow_password \
	--set auth.database=mlflow \
	--set primary.persistence.size=10Gi
```

#### Step 2: Get connection details
```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace mlflow postgres -o jsonpath="{.data.postgres-password}" | base64 -d)
export POSTGRES_HOST=$(kubectl get svc --namespace mlflow postgres -o jsonpath="{.spec.clusterIP}")
```

#### Step 3: Install MLflow using in-cluster DB
```bash
helm install mlflow community-charts/mlflow \
	--namespace mlflow \
	--set backendStore.databaseMigration=true \
	--set backendStore.postgres.enabled=true \
	--set backendStore.postgres.host=$POSTGRES_HOST \
	--set backendStore.postgres.port=5432 \
	--set backendStore.postgres.database=mlflow \
	--set backendStore.postgres.user=postgres \
	--set backendStore.postgres.password=$POSTGRES_PASSWORD
```

### Option 3: Use existing database secret

#### Step 1: Create secret
```bash
kubectl create secret generic postgres-database-secret \
	--namespace mlflow \
	--from-literal=username=mlflow_user \
	--from-literal=password=your_secure_password
```

#### Step 2: Configure MLflow (values.yaml)
```yaml
backendStore:
	databaseMigration: true
	postgres:
		enabled: true
		host: postgresql-instance1.cg034hpkmmjt.eu-central-1.rds.amazonaws.com
		port: 5432
		database: mlflow
		existingDatabaseSecret:
			name: postgres-database-secret
			usernameKey: username
			passwordKey: password
```

### Configuration with values.yaml (example)
```yaml
backendStore:
	databaseMigration: true
	databaseConnectionCheck: true
	postgres:
		enabled: true
		host: postgresql-instance1.cg034hpkmmjt.eu-central-1.rds.amazonaws.com
		port: 5432
		database: mlflow
		user: mlflowuser
		password: Pa33w0rd!

artifactRoot:
	defaultArtifactRoot: ./mlruns
	defaultArtifactsDestination: ./mlartifacts

service:
	type: ClusterIP
	port: 5000

ingress:
	enabled: false

persistence:
	enabled: true
	size: 10Gi

extraEnvVars:
	MLFLOW_SQLALCHEMYSTORE_POOL_SIZE: "10"
	MLFLOW_SQLALCHEMYSTORE_MAX_OVERFLOW: "20"
	MLFLOW_SQLALCHEMYSTORE_POOL_RECYCLE: "3600"
	MLFLOW_SQLALCHEMYSTORE_ECHO: "false"
```

Install with:
```bash
helm install mlflow community-charts/mlflow --namespace mlflow -f values.yaml
```

### Database migration features
- Automatic schema migrations: `backendStore.databaseMigration: true`
- Connection checks before startup: `backendStore.databaseConnectionCheck: true`

Manual migration example:
```bash
mlflow db export-sqlite mlflow.db > mlflow_export.sql
mlflow db upgrade postgresql://mlflow_user:password@host:5432/mlflow
```

### Verification
1. `kubectl get pods -n mlflow`
2. `kubectl logs deployment/mlflow -n mlflow | grep -i "database\|postgres"`
3. `kubectl logs deployment/mlflow -c init-mlflow -n mlflow`
4. `kubectl port-forward svc/mlflow -n mlflow 5000:5000`
5. Visit `http://localhost:5000`

### Troubleshooting (common issues)
1. Connection refused: check service and network policy
2. Authentication failed: verify username/password and permissions
3. Database not found: ensure DB exists and user has access
4. Migration failures: check init container logs

### Production considerations
- Prefer managed PostgreSQL (RDS, Cloud SQL, Azure Database)
- Enable SSL/TLS
- Configure backups and monitoring
- Use connection pooling for high traffic

Source: https://community-charts.github.io/docs/charts/mlflow/postgresql-backend-installation
