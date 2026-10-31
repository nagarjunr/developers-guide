# Terraform Remote State Management with GCS

## Overview

Managing Terraform state remotely is essential for team collaboration and infrastructure safety. This guide covers setting up and working with remote state stored in Google Cloud Storage (GCS), though the concepts apply to other backends like S3 or Azure Blob Storage.

## Why Remote State?

**Benefits:**
- **Team Collaboration**: Multiple team members can work on the same infrastructure
- **State Locking**: Prevents concurrent modifications that could corrupt state
- **Versioning**: Automatic backup of state history
- **Security**: Centralized access control and encryption
- **Disaster Recovery**: State is not lost if a developer's laptop fails

## Prerequisites

- Terraform >= 1.5.0
- Cloud provider CLI (gcloud, aws, az)
- Appropriate permissions to create storage buckets
- Existing cloud project

## Step 1: Bootstrap Remote State Storage

Before using remote state, you need to create the storage bucket. This is typically a one-time setup.

### Option A: Manual Bucket Creation (GCP)

```bash
# Set variables
PROJECT_ID="my-project"
BUCKET_NAME="${PROJECT_ID}-terraform-state"
REGION="us-central1"

# Create bucket with versioning
gcloud storage buckets create gs://${BUCKET_NAME} \
  --project=${PROJECT_ID} \
  --location=${REGION} \
  --uniform-bucket-level-access

# Enable versioning for state history
gcloud storage buckets update gs://${BUCKET_NAME} --versioning

# Prevent public access
gcloud storage buckets update gs://${BUCKET_NAME} --public-access-prevention
```

### Option B: Bootstrap Script

Create a `bootstrap_state.sh` script:

```bash
#!/bin/bash
set -e

PROJECT_ID="my-project"
BUCKET_NAME="${PROJECT_ID}-terraform-state"
REGION="us-central1"

echo "Creating Terraform state bucket: ${BUCKET_NAME}"

# Check if bucket exists
if gcloud storage buckets describe gs://${BUCKET_NAME} &>/dev/null; then
  echo "Bucket already exists: ${BUCKET_NAME}"
  exit 0
fi

# Create bucket
gcloud storage buckets create gs://${BUCKET_NAME} \
  --project=${PROJECT_ID} \
  --location=${REGION} \
  --uniform-bucket-level-access

# Enable versioning
gcloud storage buckets update gs://${BUCKET_NAME} --versioning

# Block public access
gcloud storage buckets update gs://${BUCKET_NAME} --public-access-prevention

echo "State bucket created successfully"
```

Make it executable:
```bash
chmod +x bootstrap_state.sh
./bootstrap_state.sh
```

## Step 2: Configure Terraform Backend

Create a `backend.tf` file in your Terraform project:

```hcl
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "terraform/state"
  }
}
```

**Key Parameters:**
- `bucket`: Name of the GCS bucket
- `prefix`: Path within the bucket (allows multiple projects in one bucket)

### AWS S3 Example

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "terraform/state/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Azure Blob Storage Example

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "terraformstate"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}
```

## Step 3: Initialize Remote State

### First-Time Setup (New Project)

```bash
cd terraform/

# Initialize with remote backend
terraform init

# Verify backend configuration
terraform state list
```

### Migrating from Local State

If you have existing local state (`terraform.tfstate`):

```bash
# 1. Add backend configuration to backend.tf

# 2. Initialize and migrate
terraform init -migrate-state

# 3. Confirm migration when prompted
# Type: yes

# 4. Verify state was migrated
terraform state pull > migrated-state.json
cat migrated-state.json | jq '.resources | length'

# 5. Backup and remove local state
mv terraform.tfstate terraform.tfstate.backup
mv terraform.tfstate.backup ../.
```

## Step 4: Team Onboarding

When a new team member joins, they don't need to run the bootstrap script:

```bash
# 1. Authenticate with cloud provider
gcloud auth login
gcloud auth application-default login
gcloud config set project my-project

# 2. Navigate to Terraform directory
cd terraform/

# 3. Initialize (connects to existing remote state)
terraform init

# 4. Verify access to remote state
terraform state list
terraform output

# 5. You're ready to work!
```

**What Happens:**
- Terraform downloads the shared state from the bucket
- State is cached locally in `.terraform/` (gitignored)
- All operations read/write to remote state
- Local cache is refreshed on each `terraform plan/apply`

## State Locking

State locking prevents multiple users from modifying infrastructure simultaneously.

### How It Works (GCS)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "terraform/state"
    # State locking is automatic with GCS
  }
}
```

GCS automatically handles locking - no additional configuration needed.

### Manual Unlock (If Needed)

If a Terraform operation is interrupted, the state might remain locked:

```bash
# List current locks
terraform force-unlock -help

# Unlock with lock ID from error message
terraform force-unlock <LOCK_ID>

# Confirm unlock
# Type: yes
```

**⚠️ Warning**: Only force-unlock if you're certain no other operation is running.

## State Management Commands

### View State

```bash
# List all resources
terraform state list

# Show specific resource
terraform state show google_storage_bucket.my_bucket

# Pull full state
terraform state pull > current-state.json
```

### Modify State

```bash
# Move resource to new address
terraform state mv aws_instance.old aws_instance.new

# Remove resource from state (doesn't delete actual resource)
terraform state rm aws_instance.example

# Import existing resource
terraform import aws_instance.example i-1234567890abcdef
```

### Refresh State

```bash
# Sync state with actual infrastructure
terraform refresh

# Or use newer command
terraform apply -refresh-only
```

## Backend Configuration Best Practices

### Use Variables for Backend Config

Create `backend-config.hcl`:

```hcl
bucket = "my-project-terraform-state"
prefix = "terraform/state"
```

Initialize with config file:

```bash
terraform init -backend-config=backend-config.hcl
```

This allows different environments:

```bash
# Development
terraform init -backend-config=backend-dev.hcl

# Production
terraform init -backend-config=backend-prod.hcl
```

### Separate State Per Environment

```
project/
├── terraform/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── backend.tf   # prefix = "dev/state"
│   │   │   └── main.tf
│   │   ├── staging/
│   │   │   ├── backend.tf   # prefix = "staging/state"
│   │   │   └── main.tf
│   │   └── prod/
│   │       ├── backend.tf   # prefix = "prod/state"
│   │       └── main.tf
```

Each environment has its own state in the same bucket but different prefixes.

## Troubleshooting

### Backend Initialization Errors

```bash
# Reconfigure backend (useful after changing backend config)
terraform init -reconfigure

# Migrate state to new backend
terraform init -migrate-state
```

### State Lock Timeout

```bash
# Increase timeout
terraform plan -lock-timeout=10m
```

### Access Denied

```bash
# Verify authentication
gcloud auth list
gcloud auth application-default login

# Check bucket permissions
gcloud storage buckets get-iam-policy gs://my-terraform-state
```

### State Drift Detection

```bash
# Detect changes between state and actual infrastructure
terraform plan -refresh-only

# Show detailed diff
terraform show
```

## State Versioning and Rollback

### View State History (GCS)

```bash
# List state versions
gcloud storage objects list gs://my-project-terraform-state/terraform/state/ \
  --all-versions

# Download specific version
gcloud storage cp \
  gs://my-project-terraform-state/terraform/state/default.tfstate#<generation> \
  terraform.tfstate.backup
```

### Rollback State

```bash
# 1. Download desired state version
gcloud storage cp \
  gs://my-project-terraform-state/terraform/state/default.tfstate#<generation> \
  rollback-state.tfstate

# 2. Push as current state
terraform state push rollback-state.tfstate

# 3. Verify rollback
terraform state list
```

**⚠️ Caution**: Only rollback if you understand the implications.

## Security Best Practices

### 1. Encrypt State at Rest

```hcl
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "terraform/state"
    # GCS encrypts by default, but you can specify a key
    encryption_key = "projects/my-project/locations/global/keyRings/my-keyring/cryptoKeys/my-key"
  }
}
```

### 2. Restrict Access

```bash
# Grant access only to specific users/service accounts
gcloud storage buckets add-iam-policy-binding gs://my-terraform-state \
  --member="user:developer@example.com" \
  --role="roles/storage.objectAdmin"

# For CI/CD
gcloud storage buckets add-iam-policy-binding gs://my-terraform-state \
  --member="serviceAccount:terraform-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

### 3. Enable Audit Logging

```bash
# Monitor state access
gcloud logging read "resource.type=gcs_bucket AND resource.labels.bucket_name=my-terraform-state" \
  --limit=50 \
  --format=json
```

### 4. Never Commit State Files

Add to `.gitignore`:

```
# Terraform
.terraform/
*.tfstate
*.tfstate.*
.terraform.lock.hcl
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Terraform Apply

on:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_CREDENTIALS }}

      - name: Terraform Init
        run: terraform init
        working-directory: ./terraform

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: ./terraform

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
        working-directory: ./terraform
```

### GitLab CI Example

```yaml
variables:
  TF_ROOT: terraform
  TF_STATE_BUCKET: my-project-terraform-state

before_script:
  - echo $GCP_CREDENTIALS | base64 -d > gcp-key.json
  - export GOOGLE_APPLICATION_CREDENTIALS=gcp-key.json

terraform:plan:
  stage: plan
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - $TF_ROOT/tfplan

terraform:apply:
  stage: apply
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform apply -auto-approve tfplan
  dependencies:
    - terraform:plan
  only:
    - main
```

## Common Workflow

### Daily Development

```bash
# 1. Pull latest code
git pull origin main

# 2. Initialize (if backend config changed)
terraform init

# 3. See what would change
terraform plan

# 4. Apply changes
terraform apply

# 5. Commit code changes (not state!)
git add *.tf
git commit -m "Add new resources"
git push origin main
```

### Adding New Resources

```bash
# 1. Edit Terraform files
nano main.tf

# 2. Format and validate
terraform fmt
terraform validate

# 3. Plan changes
terraform plan -out=tfplan

# 4. Review plan output carefully

# 5. Apply
terraform apply tfplan

# 6. Verify in cloud console
```

### Removing Resources

```bash
# Option 1: Remove from code and apply
# - Delete resource from .tf files
terraform plan  # Shows resources to be destroyed
terraform apply

# Option 2: Targeted destroy
terraform destroy -target=google_storage_bucket.example

# Option 3: Remove from state only (resource remains)
terraform state rm google_storage_bucket.example
```

## Summary

Remote state management is critical for team collaboration and infrastructure reliability. Key takeaways:

- **Always use remote state** for team projects
- **Bootstrap once** - team members connect to existing state
- **State locking** prevents concurrent modifications
- **Versioning** enables rollback in emergencies
- **Secure access** with IAM and encryption
- **Never commit** state files to version control
- **Use separate states** for different environments

## Further Reading

- [Terraform Backend Configuration](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)
- [GCS Backend Documentation](https://developer.hashicorp.com/terraform/language/settings/backends/gcs)
- [State Management Best Practices](https://developer.hashicorp.com/terraform/tutorials/state)
