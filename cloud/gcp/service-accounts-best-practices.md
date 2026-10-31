# GCP Service Accounts: Complete Guide and Best Practices

## Overview

Service accounts are special Google accounts that represent applications or compute workloads, not human users. They enable secure, automated access to GCP resources without sharing personal credentials.

## What Are Service Accounts?

**Service Account** = Identity for applications and services (not humans)

**Key Characteristics:**
- Email format: `service-name@project-id.iam.gserviceaccount.com`
- Authenticates using keys (JSON or P12) instead of passwords
- Can be assigned IAM roles for resource access
- Can be used by applications, VMs, containers, or CI/CD pipelines

**When to Use:**
- Applications running on GCP or external platforms
- CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins)
- Automated scripts and cron jobs
- Cross-project or cross-service authentication
- Docker/container image management

## Creating Service Accounts

### Using gcloud CLI

```bash
# Set variables
PROJECT_ID="my-project"
SA_NAME="my-application-sa"
SA_DISPLAY_NAME="My Application Service Account"

# Create service account
gcloud iam service-accounts create ${SA_NAME} \
  --display-name="${SA_DISPLAY_NAME}" \
  --project=${PROJECT_ID}

# Get the full email
SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
echo "Service Account: ${SA_EMAIL}"
```

### Using Terraform

```hcl
resource "google_service_account" "application_sa" {
  account_id   = "my-application-sa"
  display_name = "My Application Service Account"
  project      = var.project_id
}

output "service_account_email" {
  value       = google_service_account.application_sa.email
  description = "Service account email address"
}
```

## Granting Permissions

Service accounts need IAM roles to access resources. Follow the principle of least privilege.

### Project-Level Roles

```bash
# Grant a single role
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectViewer"

# Grant multiple roles
ROLES=(
  "roles/storage.admin"
  "roles/artifactregistry.writer"
  "roles/logging.logWriter"
)

for ROLE in "${ROLES[@]}"; do
  gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="${ROLE}"
done
```

### Resource-Level Roles (More Secure)

Grant access to specific resources instead of entire project:

```bash
# Grant access to specific bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"

# Grant access to specific Artifact Registry repo
gcloud artifacts repositories add-iam-policy-binding my-repo \
  --location=us-central1 \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/artifactregistry.writer"
```

### Terraform IAM Example

```hcl
# Project-level role
resource "google_project_iam_member" "sa_storage_admin" {
  project = var.project_id
  role    = "roles/storage.admin"
  member  = "serviceAccount:${google_service_account.application_sa.email}"
}

# Bucket-level role (more secure)
resource "google_storage_bucket_iam_member" "sa_bucket_admin" {
  bucket = google_storage_bucket.my_bucket.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.application_sa.email}"
}

# Multiple roles for comprehensive access
locals {
  sa_roles = [
    "roles/storage.admin",
    "roles/artifactregistry.writer",
    "roles/artifactregistry.reader",
    "roles/logging.logWriter",
    "roles/aiplatform.user",
  ]
}

resource "google_project_iam_member" "sa_roles" {
  for_each = toset(local.sa_roles)
  project  = var.project_id
  role     = each.key
  member   = "serviceAccount:${google_service_account.application_sa.email}"
}
```

## Common IAM Roles

| Role | Purpose | Use Case |
|------|---------|----------|
| `roles/storage.admin` | Full Cloud Storage access | Application managing buckets and objects |
| `roles/storage.objectViewer` | Read-only object access | Application reading files from buckets |
| `roles/storage.objectCreator` | Upload objects only | Log aggregation service |
| `roles/artifactregistry.writer` | Push Docker images | CI/CD pipeline |
| `roles/artifactregistry.reader` | Pull Docker images | Kubernetes cluster, Cloud Run |
| `roles/logging.logWriter` | Write logs | Application logging |
| `roles/aiplatform.user` | Access Vertex AI APIs | ML applications |
| `roles/secretmanager.secretAccessor` | Read secrets | Application accessing API keys |
| `roles/browser` | Read-only project viewer | Monitoring, inventory tools |

## Creating and Managing Keys

Service account keys are JSON files containing credentials for authentication.

### Create Key

```bash
# Create and download key
gcloud iam service-accounts keys create ~/sa-key.json \
  --iam-account=${SA_EMAIL}

# Secure the key file
chmod 600 ~/sa-key.json

# Verify key structure
cat ~/sa-key.json | python -m json.tool | head -20
```

### List Keys

```bash
# List all keys for service account
gcloud iam service-accounts keys list \
  --iam-account=${SA_EMAIL}

# Output shows key ID, creation date, and expiration
```

### Delete Keys

```bash
# Delete specific key
KEY_ID="abc123..."
gcloud iam service-accounts keys delete ${KEY_ID} \
  --iam-account=${SA_EMAIL}

# Confirm deletion
# Type: y
```

## Authentication Methods

### Method 1: Application Default Credentials (ADC)

Best for local development and cloud environments.

```bash
# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/sa-key.json"

# Applications automatically use this credential
```

**Python Example:**

```python
from google.cloud import storage
import os

# Set credentials
os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = '/path/to/sa-key.json'

# Client automatically uses service account
client = storage.Client(project='my-project')

# List buckets
for bucket in client.list_buckets():
    print(bucket.name)
```

**Node.js Example:**

```javascript
const {Storage} = require('@google-cloud/storage');

// Set credentials
process.env.GOOGLE_APPLICATION_CREDENTIALS = '/path/to/sa-key.json';

// Client automatically uses service account
const storage = new Storage({
  projectId: 'my-project'
});

// List buckets
async function listBuckets() {
  const [buckets] = await storage.getBuckets();
  buckets.forEach(bucket => console.log(bucket.name));
}

listBuckets();
```

### Method 2: Activate in gcloud

Best for CLI operations and testing.

```bash
# Activate service account
gcloud auth activate-service-account --key-file=/path/to/sa-key.json

# Verify active account
gcloud auth list

# Set project
gcloud config set project my-project

# Now all gcloud commands use this service account
gcloud storage buckets list
```

### Method 3: Explicit Credentials in Code

Best when you need multiple service accounts or dynamic credential loading.

```python
from google.cloud import storage
from google.oauth2 import service_account

# Load credentials explicitly
credentials = service_account.Credentials.from_service_account_file(
    '/path/to/sa-key.json'
)

# Create client with specific credentials
client = storage.Client(
    credentials=credentials,
    project='my-project'
)

# Use client
bucket = client.bucket('my-bucket')
blob = bucket.blob('file.txt')
blob.upload_from_filename('local-file.txt')
```

## Docker & Artifact Registry Integration

### Configure Docker Authentication

```bash
# Activate service account
gcloud auth activate-service-account --key-file=/path/to/sa-key.json

# Configure Docker to use gcloud for authentication
gcloud auth configure-docker us-central1-docker.pkg.dev

# Verify configuration
cat ~/.docker/config.json | grep -A5 credHelpers
```

### Build and Push Docker Images

```bash
# Set variables
REGION="us-central1"
PROJECT_ID="my-project"
REPO_NAME="docker-repo"
IMAGE_NAME="my-app"
IMAGE_TAG="v1.0.0"

# Full image path
IMAGE_PATH="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:${IMAGE_TAG}"

# Build image
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

# Tag for Artifact Registry
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_PATH}

# Push to Artifact Registry
docker push ${IMAGE_PATH}

# Verify
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}
```

### Pull Docker Images

```bash
# Pull from Artifact Registry
docker pull ${IMAGE_PATH}

# Run container
docker run -p 8080:8080 ${IMAGE_PATH}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1

      - name: Configure Docker
        run: gcloud auth configure-docker us-central1-docker.pkg.dev

      - name: Build and Push
        run: |
          docker build -t my-app:${{ github.sha }} .
          docker tag my-app:${{ github.sha }} \
            us-central1-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }}
          docker push us-central1-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }}

      - name: Upload to GCS
        run: |
          echo "Build ${{ github.sha }}" > build-info.txt
          gcloud storage cp build-info.txt gs://my-bucket/builds/
```

**Setup:**
1. Create service account key
2. Go to GitHub repo → Settings → Secrets and variables → Actions
3. Create new secret: `GCP_SA_KEY`
4. Paste entire contents of `sa-key.json`

### GitLab CI/CD

```yaml
variables:
  GCP_PROJECT: my-project
  GCP_REGION: us-central1
  DOCKER_REPO: my-repo
  IMAGE_NAME: my-app

before_script:
  # Decode and authenticate
  - echo $GCP_SA_KEY | base64 -d > sa-key.json
  - gcloud auth activate-service-account --key-file=sa-key.json
  - gcloud config set project $GCP_PROJECT
  - gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev

build:
  stage: build
  image: google/cloud-sdk:alpine
  services:
    - docker:dind
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA
        ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${DOCKER_REPO}/${IMAGE_NAME}:$CI_COMMIT_SHA
    - docker push ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${DOCKER_REPO}/${IMAGE_NAME}:$CI_COMMIT_SHA
```

**Setup:**
1. Create service account key
2. Base64 encode: `cat sa-key.json | base64 > sa-key-base64.txt`
3. Go to GitLab project → Settings → CI/CD → Variables
4. Create variable: `GCP_SA_KEY` (type: File, protected)
5. Paste base64 content

### Cloud Build

```yaml
# cloudbuild.yaml
steps:
  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA'
      - '.'

  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA'

  # Deploy to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA'
      - '--region=us-central1'
      - '--platform=managed'

images:
  - 'us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA'

# Service account is automatically attached to Cloud Build
```

## Security Best Practices

### 1. Use Short-Lived Credentials

Instead of creating keys, use Workload Identity (for GKE) or service account impersonation:

```bash
# Impersonate service account (no key needed)
gcloud auth application-default login --impersonate-service-account=${SA_EMAIL}
```

### 2. Rotate Keys Regularly

```bash
# Create new key
gcloud iam service-accounts keys create new-sa-key.json \
  --iam-account=${SA_EMAIL}

# Update application to use new key
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/new-sa-key.json"

# Test new key
gcloud storage buckets list

# Delete old key
gcloud iam service-accounts keys delete ${OLD_KEY_ID} \
  --iam-account=${SA_EMAIL}
```

**Recommended Schedule:** Rotate every 90 days

### 3. Use Secret Manager for Keys

Never commit keys to version control. Store them in Secret Manager:

```bash
# Store key in Secret Manager
gcloud secrets create service-account-key \
  --data-file=sa-key.json \
  --replication-policy=automatic

# Grant access to application
gcloud secrets add-iam-policy-binding service-account-key \
  --member="serviceAccount:app-runner@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Retrieve in application
gcloud secrets versions access latest --secret=service-account-key > /tmp/sa-key.json
export GOOGLE_APPLICATION_CREDENTIALS="/tmp/sa-key.json"
```

### 4. Restrict Key File Permissions

```bash
# Owner read-only
chmod 600 sa-key.json

# Verify permissions
ls -la sa-key.json
# Should show: -rw------- (600)
```

### 5. Never Commit Keys to Git

Add to `.gitignore`:

```
# Service account keys
*-key.json
*.json
sa-key.json
credentials.json

# Application credentials
.env
.env.*
!.env.example
```

### 6. Use Least Privilege

Grant only necessary permissions:

```bash
# Bad: Too permissive
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/owner"

# Good: Specific permissions
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"

# Better: Resource-level permissions
gcloud storage buckets add-iam-policy-binding gs://specific-bucket \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

### 7. Monitor Service Account Activity

```bash
# View service account activity
gcloud logging read \
  "protoPayload.authenticationInfo.principalEmail=${SA_EMAIL}" \
  --limit=50 \
  --format=json

# Monitor specific resource access
gcloud logging read \
  "resource.type=gcs_bucket AND protoPayload.authenticationInfo.principalEmail=${SA_EMAIL}" \
  --limit=20
```

### 8. Disable Unused Service Accounts

```bash
# Disable (not delete) service account
gcloud iam service-accounts disable ${SA_EMAIL}

# Re-enable if needed
gcloud iam service-accounts enable ${SA_EMAIL}

# Delete permanently (careful!)
gcloud iam service-accounts delete ${SA_EMAIL}
```

## Troubleshooting

### Permission Denied Errors

```bash
# Check service account roles
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_EMAIL}" \
  --format="table(bindings.role)"

# Check bucket-level permissions
gcloud storage buckets get-iam-policy gs://my-bucket | grep ${SA_EMAIL}

# Verify service account exists
gcloud iam service-accounts describe ${SA_EMAIL}
```

### Invalid Credentials

```bash
# Verify key is valid JSON
cat sa-key.json | python -m json.tool

# Check key expiration
gcloud iam service-accounts keys list --iam-account=${SA_EMAIL}

# Test authentication
gcloud auth activate-service-account --key-file=sa-key.json
gcloud auth list
```

### Docker Authentication Failures

```bash
# Reconfigure Docker
gcloud auth activate-service-account --key-file=sa-key.json
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

# Verify Docker config
cat ~/.docker/config.json

# Test authentication
docker-credential-gcloud list
```

## Quick Reference

```bash
# Create service account
gcloud iam service-accounts create SA_NAME --display-name="Display Name"

# Create key
gcloud iam service-accounts keys create sa-key.json --iam-account=SA_EMAIL

# Grant project role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA_EMAIL" \
  --role="ROLE"

# Grant bucket role
gcloud storage buckets add-iam-policy-binding gs://BUCKET \
  --member="serviceAccount:SA_EMAIL" \
  --role="ROLE"

# Authenticate
gcloud auth activate-service-account --key-file=sa-key.json
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/sa-key.json"

# Configure Docker
gcloud auth configure-docker REGION-docker.pkg.dev

# List keys
gcloud iam service-accounts keys list --iam-account=SA_EMAIL

# Delete key
gcloud iam service-accounts keys delete KEY_ID --iam-account=SA_EMAIL
```

## Summary

Service accounts are essential for secure, automated GCP access. Key principles:

- **Use service accounts for applications, not humans**
- **Grant least privilege** - only necessary permissions
- **Rotate keys every 90 days**
- **Never commit keys** to version control
- **Use Secret Manager** for production keys
- **Monitor activity** with Cloud Logging
- **Prefer resource-level** over project-level permissions
- **Use Workload Identity** when possible (avoids keys entirely)

## Further Reading

- [Service Accounts Overview](https://cloud.google.com/iam/docs/service-accounts)
- [Best Practices for Service Accounts](https://cloud.google.com/iam/docs/best-practices-service-accounts)
- [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [Managing Service Account Keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
