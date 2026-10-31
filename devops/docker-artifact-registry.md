# GCP Artifact Registry: Docker Image Management Guide

## Overview

Artifact Registry is Google Cloud's next-generation container registry service that securely stores, manages, and scans Docker container images. It's the successor to Container Registry (GCR) with enhanced features for security, access control, and multi-format support.

## Why Artifact Registry?

**Benefits over public Docker Hub:**
- **Private storage**: Images are private by default
- **IAM integration**: Fine-grained access control
- **Regional repositories**: Reduced latency and costs
- **Vulnerability scanning**: Automated security analysis
- **Integration**: Native integration with GCP services (Cloud Run, GKE, Cloud Build)
- **Versioning**: Automatic image tagging and version management

## Prerequisites

- GCP project with billing enabled
- `gcloud` CLI installed and configured
- Docker installed locally
- Appropriate IAM permissions

## Setup Artifact Registry

### Enable APIs

```bash
# Enable required APIs
gcloud services enable \
  artifactregistry.googleapis.com \
  cloudresourcemanager.googleapis.com

# Verify APIs are enabled
gcloud services list --enabled | grep artifact
```

### Create Repository (Manual)

```bash
# Set variables
PROJECT_ID="my-project"
REGION="us-central1"
REPO_NAME="docker-repo"
DESCRIPTION="Docker images for my application"

# Create Docker repository
gcloud artifacts repositories create ${REPO_NAME} \
  --repository-format=docker \
  --location=${REGION} \
  --description="${DESCRIPTION}" \
  --project=${PROJECT_ID}

# Verify creation
gcloud artifacts repositories list --project=${PROJECT_ID}
```

### Create Repository (Terraform)

```hcl
# variables.tf
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "GCP region"
}

variable "repository_id" {
  type        = string
  default     = "docker-repo"
  description = "Artifact Registry repository ID"
}

# main.tf
resource "google_project_service" "artifact_registry" {
  project = var.project_id
  service = "artifactregistry.googleapis.com"
}

resource "google_artifact_registry_repository" "docker_repo" {
  location      = var.region
  repository_id = var.repository_id
  description   = "Docker images repository"
  format        = "DOCKER"
  project       = var.project_id

  depends_on = [google_project_service.artifact_registry]
}

# outputs.tf
output "repository_url" {
  value       = "${var.region}-docker.pkg.dev/${var.project_id}/${var.repository_id}"
  description = "Full repository URL for Docker images"
}

output "repository_id" {
  value       = google_artifact_registry_repository.docker_repo.repository_id
  description = "Repository ID"
}
```

Apply Terraform:

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Get repository URL
terraform output repository_url
# Output: us-central1-docker.pkg.dev/my-project/docker-repo
```

## Authentication

### Option 1: User Account (Local Development)

```bash
# Authenticate with your user account
gcloud auth login

# Configure Docker to use gcloud credentials
gcloud auth configure-docker ${REGION}-docker.pkg.dev

# Verify configuration
cat ~/.docker/config.json
```

The `configure-docker` command adds a credential helper to Docker config:

```json
{
  "credHelpers": {
    "us-central1-docker.pkg.dev": "gcloud"
  }
}
```

### Option 2: Service Account (Automation/CI/CD)

```bash
# Create service account
gcloud iam service-accounts create docker-pusher \
  --display-name="Docker Image Pusher"

SA_EMAIL="docker-pusher@${PROJECT_ID}.iam.gserviceaccount.com"

# Grant Artifact Registry permissions
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/artifactregistry.writer"

# Create and download key
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=${SA_EMAIL}

# Authenticate using service account
gcloud auth activate-service-account --key-file=sa-key.json

# Configure Docker
gcloud auth configure-docker ${REGION}-docker.pkg.dev

# Secure key file
chmod 600 sa-key.json
```

### Option 3: Application Default Credentials

```bash
# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/sa-key.json"

# Docker will automatically use these credentials via gcloud
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

## Building and Pushing Images

### Basic Workflow

```bash
# Set variables
REGION="us-central1"
PROJECT_ID="my-project"
REPO_NAME="docker-repo"
IMAGE_NAME="my-app"
IMAGE_TAG="v1.0.0"

# Construct full image path
IMAGE_PATH="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:${IMAGE_TAG}"

# Build Docker image
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

# Tag image for Artifact Registry
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_PATH}

# Push to Artifact Registry
docker push ${IMAGE_PATH}

# Verify upload
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}
```

### Example Dockerfile

```dockerfile
# Multi-stage build for smaller images
FROM python:3.11-slim as builder

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Final stage
FROM python:3.11-slim

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY . .

# Set PATH
ENV PATH=/root/.local/bin:$PATH

# Expose port
EXPOSE 8080

# Run application
CMD ["python", "app.py"]
```

### Build Script Example

Create `build-and-push.sh`:

```bash
#!/bin/bash
set -e

# Configuration
REGION="us-central1"
PROJECT_ID="my-project"
REPO_NAME="docker-repo"
IMAGE_NAME="my-app"

# Use git commit SHA as tag
IMAGE_TAG=$(git rev-parse --short HEAD)

# Full image path
IMAGE_PATH="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}"

echo "Building ${IMAGE_NAME}:${IMAGE_TAG}..."
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

echo "Tagging for Artifact Registry..."
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_PATH}:${IMAGE_TAG}
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_PATH}:latest

echo "Pushing to Artifact Registry..."
docker push ${IMAGE_PATH}:${IMAGE_TAG}
docker push ${IMAGE_PATH}:latest

echo "Image pushed successfully:"
echo "  ${IMAGE_PATH}:${IMAGE_TAG}"
echo "  ${IMAGE_PATH}:latest"
```

Make it executable:

```bash
chmod +x build-and-push.sh
./build-and-push.sh
```

## Pulling Images

### Pull from Artifact Registry

```bash
# Pull specific version
docker pull ${IMAGE_PATH}:v1.0.0

# Pull latest
docker pull ${IMAGE_PATH}:latest

# Verify image
docker images | grep my-app
```

### Run Container

```bash
# Run container from pulled image
docker run -d -p 8080:8080 --name my-app ${IMAGE_PATH}:latest

# View logs
docker logs my-app

# Stop container
docker stop my-app
docker rm my-app
```

## Image Management

### List Images

```bash
# List all images in repository
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}

# List with tags
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME} \
  --include-tags

# Filter by tag
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME} \
  --filter="tags:v1.*"
```

### Delete Images

```bash
# Delete specific version
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:v1.0.0 \
  --delete-tags

# Delete all versions with tag
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest \
  --delete-tags

# Delete entire image (all versions)
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME} \
  --delete-tags
```

### Image Scanning

```bash
# Enable vulnerability scanning
gcloud artifacts repositories update ${REPO_NAME} \
  --location=${REGION} \
  --enable-vulnerability-scanning

# View scan results
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME} \
  --show-occurrences

# Get detailed vulnerability report
gcloud artifacts docker images describe \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:v1.0.0 \
  --show-all-metadata
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PROJECT_ID: my-project
  REGION: us-central1
  REPO_NAME: docker-repo
  IMAGE_NAME: my-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -t ${{ env.IMAGE_NAME }}:latest \
            .

      - name: Tag images for Artifact Registry
        run: |
          IMAGE_PATH="${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}"
          docker tag ${{ env.IMAGE_NAME }}:${{ github.sha }} ${IMAGE_PATH}:${{ github.sha }}
          docker tag ${{ env.IMAGE_NAME }}:latest ${IMAGE_PATH}:latest

      - name: Push images to Artifact Registry
        run: |
          IMAGE_PATH="${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}"
          docker push ${IMAGE_PATH}:${{ github.sha }}
          docker push ${IMAGE_PATH}:latest

      - name: Output image URL
        run: |
          echo "Image pushed to:"
          echo "${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}"
```

**Setup:**
1. Create service account with `roles/artifactregistry.writer`
2. Create key: `gcloud iam service-accounts keys create sa-key.json --iam-account=SA_EMAIL`
3. Add to GitHub Secrets: `GCP_SA_KEY` = contents of `sa-key.json`

### GitLab CI/CD

```yaml
variables:
  GCP_PROJECT: my-project
  GCP_REGION: us-central1
  REPO_NAME: docker-repo
  IMAGE_NAME: my-app
  DOCKER_DRIVER: overlay2

stages:
  - build
  - push

before_script:
  - echo $GCP_SA_KEY | base64 -d > sa-key.json
  - gcloud auth activate-service-account --key-file=sa-key.json
  - gcloud config set project $GCP_PROJECT
  - gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:latest
  only:
    - main
    - develop

push:
  stage: push
  image: google/cloud-sdk:alpine
  services:
    - docker:24-dind
  script:
    - IMAGE_PATH="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${REPO_NAME}/${IMAGE_NAME}"
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA ${IMAGE_PATH}:$CI_COMMIT_SHA
    - docker tag $IMAGE_NAME:latest ${IMAGE_PATH}:latest
    - docker push ${IMAGE_PATH}:$CI_COMMIT_SHA
    - docker push ${IMAGE_PATH}:latest
  dependencies:
    - build
  only:
    - main
```

### Cloud Build

```yaml
# cloudbuild.yaml
steps:
  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}:${SHORT_SHA}'
      - '-t'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}:latest'
      - '.'

  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - '--all-tags'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}'

  # Optional: Deploy to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - '${_IMAGE_NAME}'
      - '--image=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}:${SHORT_SHA}'
      - '--region=${_REGION}'
      - '--platform=managed'
      - '--allow-unauthenticated'

substitutions:
  _REGION: us-central1
  _REPO_NAME: docker-repo
  _IMAGE_NAME: my-app

images:
  - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}:${SHORT_SHA}'
  - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_IMAGE_NAME}:latest'

options:
  logging: CLOUD_LOGGING_ONLY
```

Submit build:

```bash
# Manual submission
gcloud builds submit --config=cloudbuild.yaml .

# Create GitHub trigger
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

## Access Control

### Grant Repository Access

```bash
# Writer access (push images)
gcloud artifacts repositories add-iam-policy-binding ${REPO_NAME} \
  --location=${REGION} \
  --member="serviceAccount:sa@my-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Reader access (pull images)
gcloud artifacts repositories add-iam-policy-binding ${REPO_NAME} \
  --location=${REGION} \
  --member="serviceAccount:sa@my-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# Admin access (manage repository)
gcloud artifacts repositories add-iam-policy-binding ${REPO_NAME} \
  --location=${REGION} \
  --member="user:developer@example.com" \
  --role="roles/artifactregistry.admin"
```

### View Repository IAM Policy

```bash
gcloud artifacts repositories get-iam-policy ${REPO_NAME} \
  --location=${REGION}
```

## Best Practices

### 1. Use Semantic Versioning

```bash
# Tag with semantic version
docker tag my-app:latest ${IMAGE_PATH}:v1.2.3
docker tag my-app:latest ${IMAGE_PATH}:v1.2
docker tag my-app:latest ${IMAGE_PATH}:v1
docker tag my-app:latest ${IMAGE_PATH}:latest

# Push all tags
docker push --all-tags ${IMAGE_PATH}
```

### 2. Tag with Git Commit SHA

```bash
# Use git SHA for traceability
GIT_SHA=$(git rev-parse --short HEAD)
docker tag my-app:latest ${IMAGE_PATH}:${GIT_SHA}
docker push ${IMAGE_PATH}:${GIT_SHA}
```

### 3. Multi-Stage Builds

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 4. Image Cleanup Policy

Create lifecycle policy to auto-delete old images:

```bash
# Create policy file
cat > lifecycle-policy.json <<EOF
{
  "rules": [
    {
      "name": "keep-recent-versions",
      "action": {
        "type": "Delete"
      },
      "condition": {
        "olderThan": "30d",
        "tagState": "any"
      }
    },
    {
      "name": "keep-tagged-releases",
      "action": {
        "type": "Keep"
      },
      "condition": {
        "tagPrefixes": ["v"]
      }
    }
  ]
}
EOF

# Apply policy (not yet supported in Artifact Registry, use manual cleanup)
```

### 5. Security Scanning

Enable and monitor vulnerability scanning:

```bash
# Enable scanning
gcloud artifacts repositories update ${REPO_NAME} \
  --location=${REGION} \
  --enable-vulnerability-scanning

# View vulnerabilities
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME} \
  --show-occurrences \
  --occurrence-filter="kind=VULNERABILITY"
```

## Troubleshooting

### Authentication Issues

```bash
# Verify authentication
gcloud auth list

# Reconfigure Docker
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

# Test access
gcloud artifacts repositories list --location=${REGION}
```

### Permission Denied

```bash
# Check IAM permissions
gcloud artifacts repositories get-iam-policy ${REPO_NAME} \
  --location=${REGION}

# Verify service account roles
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:SA_EMAIL"
```

### Push/Pull Failures

```bash
# Clear Docker credentials
rm ~/.docker/config.json

# Reconfigure authentication
gcloud auth login
gcloud auth configure-docker ${REGION}-docker.pkg.dev

# Test connection
docker pull ${IMAGE_PATH}:latest
```

## Cost Optimization

### Storage Costs

```bash
# Check repository storage usage
gcloud artifacts repositories describe ${REPO_NAME} \
  --location=${REGION} \
  --format="value(sizeBytes)"

# Delete old images
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:old-tag
```

### Regional Placement

- Use regional repositories (e.g., `us-central1`) instead of multi-regional
- Place repository in same region as compute resources (Cloud Run, GKE)
- Reduces egress costs and latency

## Quick Reference

```bash
# Enable API
gcloud services enable artifactregistry.googleapis.com

# Create repository
gcloud artifacts repositories create REPO_NAME \
  --repository-format=docker \
  --location=REGION

# Configure Docker
gcloud auth configure-docker REGION-docker.pkg.dev

# Build and push
docker build -t IMAGE_NAME:TAG .
docker tag IMAGE_NAME:TAG REGION-docker.pkg.dev/PROJECT/REPO/IMAGE_NAME:TAG
docker push REGION-docker.pkg.dev/PROJECT/REPO/IMAGE_NAME:TAG

# Pull
docker pull REGION-docker.pkg.dev/PROJECT/REPO/IMAGE_NAME:TAG

# List images
gcloud artifacts docker images list REGION-docker.pkg.dev/PROJECT/REPO

# Delete image
gcloud artifacts docker images delete REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG
```

## Summary

Artifact Registry provides secure, scalable Docker image storage with seamless GCP integration. Key points:

- **Regional repositories** for cost and performance optimization
- **IAM-based access control** for fine-grained permissions
- **Automated vulnerability scanning** for security
- **CI/CD integration** with GitHub Actions, GitLab, Cloud Build
- **Semantic versioning** and git SHA tagging for traceability
- **Multi-stage builds** for smaller, more secure images

## Further Reading

- [Artifact Registry Documentation](https://cloud.google.com/artifact-registry/docs)
- [Docker Image Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Container Security Best Practices](https://cloud.google.com/architecture/best-practices-for-operating-containers)
