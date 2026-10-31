# Kubernetes Secrets Management Strategies

## Overview

Managing sensitive data like passwords, API keys, certificates, and tokens is critical for application security. This guide covers various strategies for secrets management in Kubernetes, from simple approaches to enterprise-grade solutions.

## The Problem

Applications need sensitive data, but:
- **Version control risks**: Committing secrets to Git exposes them
- **Access control**: Not everyone should see all secrets
- **Rotation**: Secrets need periodic updates
- **Audit trails**: Track who accessed what and when
- **Compliance**: Meet regulatory requirements (GDPR, SOC2, HIPAA)

## Secrets Management Hierarchy

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph TD
    A[Level 1: Manual Creation<br/>Development]:::red --> B[Level 2: Kubernetes Secrets<br/>Basic]:::orange
    B --> C[Level 3: Sealed Secrets<br/>GitOps]:::cyan
    C --> D[Level 4: External Secrets Operator<br/>Cloud]:::blue
    D --> E[Level 5: HashiCorp Vault<br/>Enterprise]:::green

    classDef red fill:#E74C3C,stroke:#A93226,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef cyan fill:#3498DB,stroke:#2471A3,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
```

Choose based on your requirements, team size, and infrastructure maturity.

## Level 1: Manual Secret Creation

### Overview

Create secrets manually in each environment. Simplest approach for development and small teams.

### Implementation

**Create from literals**:
```bash
kubectl create secret generic app-secrets \
  --from-literal=database-password='mySecureP@ss123' \
  --from-literal=api-key='abc-def-ghi-jkl' \
  --namespace myapp
```

**Create from files**:
```bash
# Create secret files locally (never commit!)
echo -n 'admin' > ./username.txt
echo -n 'SuperSecret123!' > ./password.txt

# Create secret from files
kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt \
  --namespace myapp

# Clean up local files
rm -f ./username.txt ./password.txt
```

**Create from env file**:
```bash
# secrets.env (add to .gitignore!)
DATABASE_PASSWORD=mySecureP@ss123
API_KEY=abc-def-ghi-jkl
ENCRYPTION_KEY=xyz-123-456-789

# Create secret
kubectl create secret generic app-secrets \
  --from-env-file=./secrets.env \
  --namespace myapp
```

### Usage in Pods

**Environment variables**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
```

**Volume mounts**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
```

### Pros and Cons

**Pros**:
- Simple to understand
- No additional tools required
- Quick setup
- Good for development

**Cons**:
- Manual process per environment
- No version control
- No audit trail
- Difficult to rotate
- Doesn't scale

### Best For

- Development environments
- Small teams
- Proof of concepts
- Learning Kubernetes

## Level 2: Kubernetes Native Secrets

### Overview

Use Kubernetes Secrets with proper RBAC and practices.

### ConfigMaps vs Secrets

**Use ConfigMaps for**:
- Non-sensitive configuration
- Public settings
- Environment variables (non-secret)

**Use Secrets for**:
- Passwords
- API keys
- Certificates
- Private keys
- Tokens

### Secret Types

**1. Opaque (generic)**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  username: YWRtaW4=        # base64 encoded
  password: cGFzc3dvcmQxMjM= # base64 encoded
```

**2. Docker registry**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

**3. TLS certificates**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**4. Service account token**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sa-token
type: kubernetes.io/service-account-token
```

### Security Best Practices

**1. Enable encryption at rest**:
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <32-byte base64 encoded string>
  - identity: {}
```

**2. RBAC restrictions**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: myapp
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["app-secrets"]  # Specific secrets only

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: myapp
subjects:
- kind: ServiceAccount
  name: myapp-sa
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**3. Network policies**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
```

### Pros and Cons

**Pros**:
- Native Kubernetes feature
- RBAC integration
- Encryption at rest (if enabled)
- Better than ConfigMaps for sensitive data

**Cons**:
- Base64 encoding (not encryption)
- Visible to cluster admins
- No built-in rotation
- Not GitOps-friendly
- Limited audit capabilities

### Best For

- Small to medium deployments
- Basic security requirements
- Internal applications
- When external tools aren't available

## Level 3: Sealed Secrets (GitOps)

### Overview

Bitnami Sealed Secrets encrypts secrets so they can be safely stored in Git. The controller decrypts them in-cluster.

### Architecture

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Developer]:::purple --> B[Seal Secret]:::orange
    B --> C[Git Repository]:::blue
    C --> D[ArgoCD/Flux]:::cyan
    D --> E[Cluster]:::green

    B -.Public Key.-> F[Key Store]:::gray
    E --> G[Sealed Secrets Controller]:::teal
    G -.Decrypts with Private Key.-> H[Kubernetes Secret]:::green

    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef cyan fill:#3498DB,stroke:#2471A3,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
    classDef gray fill:#95A5A6,stroke:#7F8C8D,color:#333
```

### Installation

```bash
# Install controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Or download binary
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
chmod +x kubeseal-linux-amd64
sudo mv kubeseal-linux-amd64 /usr/local/bin/kubeseal
```

### Usage

**1. Create secret (locally)**:
```bash
kubectl create secret generic app-secrets \
  --from-literal=database-password='mySecureP@ss123' \
  --from-literal=api-key='abc-def-ghi-jkl' \
  --dry-run=client -o yaml > secret.yaml
```

**2. Seal the secret**:
```bash
kubeseal -f secret.yaml -w sealed-secret.yaml

# Or pipe directly
kubectl create secret generic app-secrets \
  --from-literal=database-password='mySecureP@ss123' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml
```

**3. Commit to Git** (now safe!):
```bash
git add sealed-secret.yaml
git commit -m "Add application secrets"
git push
```

**4. Apply to cluster**:
```bash
kubectl apply -f sealed-secret.yaml

# Controller automatically creates the Secret
kubectl get secrets -n myapp
```

### Sealed Secret Example

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
  namespace: myapp
spec:
  encryptedData:
    database-password: AgB7Tn5m3O3K...encrypted...
    api-key: AgCX8nPqM9fL...encrypted...
  template:
    metadata:
      name: app-secrets
      namespace: myapp
    type: Opaque
```

### Scopes

**Strict** (default): Namespace and name specific
```bash
kubeseal --scope strict -f secret.yaml -w sealed-secret.yaml
```

**Namespace-wide**: Any name in the namespace
```bash
kubeseal --scope namespace-wide -f secret.yaml -w sealed-secret.yaml
```

**Cluster-wide**: Any namespace, any name
```bash
kubeseal --scope cluster-wide -f secret.yaml -w sealed-secret.yaml
```

### Secret Rotation

```bash
# Create new sealed secret
kubeseal -f new-secret.yaml -w sealed-secret-v2.yaml

# Apply
kubectl apply -f sealed-secret-v2.yaml

# Restart pods to pick up new secrets
kubectl rollout restart deployment/myapp
```

### Pros and Cons

**Pros**:
- GitOps-friendly (commit to Git safely)
- Encryption with public/private keys
- Automatic decryption in-cluster
- Works with ArgoCD/Flux
- Free and open source

**Cons**:
- Requires controller installation
- Cluster-specific (must reseal for different clusters)
- No built-in rotation mechanism
- Manual key backup required

### Best For

- GitOps workflows
- Multiple environments
- Teams using ArgoCD/Flux
- Mid-sized deployments
- Declarative infrastructure

## Level 4: External Secrets Operator

### Overview

Syncs secrets from external providers (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault) into Kubernetes.

### Architecture

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
sequenceDiagram
    participant ESM as External Secret Manager<br/>(AWS/GCP/Azure/Vault)
    participant ESO as External Secrets<br/>Operator
    participant K8S as Kubernetes<br/>Secrets

    ESO->>ESM: 1. Poll for updates
    ESM->>ESO: 2. Retrieve secrets
    ESO->>K8S: 3. Create/Update

    Note over ESM: Cloud Provider
    Note over ESO: Sync Controller
    Note over K8S: K8s Native
```

### Installation

```bash
# Add Helm repo
helm repo add external-secrets https://charts.external-secrets.io

# Install operator
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace
```

### AWS Secrets Manager Example

**1. Create IAM role for IRSA**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret"
    ],
    "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/*"
  }]
}
```

**2. Create SecretStore**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretstore
  namespace: myapp
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

**3. Create ExternalSecret**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: myapp
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretstore
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: myapp/database
      property: password
  - secretKey: api-key
    remoteRef:
      key: myapp/api
      property: key
```

**4. Operator creates Kubernetes Secret**:
```bash
kubectl get secrets app-secrets -n myapp
```

### GCP Secret Manager Example

**SecretStore**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secretstore
  namespace: myapp
spec:
  provider:
    gcpsm:
      projectID: "my-project-123"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: my-cluster
          serviceAccountRef:
            name: external-secrets-sa
```

### Azure Key Vault Example

**SecretStore**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-secretstore
  namespace: myapp
spec:
  provider:
    azurekv:
      vaultUrl: "https://my-vault.vault.azure.net"
      tenantId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      authSecretRef:
        clientId:
          name: azure-secret
          key: client-id
        clientSecret:
          name: azure-secret
          key: client-secret
```

### Pros and Cons

**Pros**:
- Centralized secret management
- Automatic synchronization
- Built-in rotation support
- Audit logging (provider-dependent)
- Multi-cluster support
- Cloud-native integration

**Cons**:
- Requires external service
- Cloud vendor dependency (or Vault setup)
- Additional cost (cloud services)
- Network dependency
- Complexity increases

### Best For

- Cloud-native applications
- Multi-cluster deployments
- Enterprise environments
- Compliance requirements
- Automatic rotation needs

## Level 5: HashiCorp Vault

### Overview

Enterprise-grade secrets management with dynamic secrets, encryption as a service, and comprehensive audit logging.

### Architecture

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Application<br/>Injected Secrets]:::purple --> B[Vault Agent<br/>Sidecar Container]:::orange
    B --> C[HashiCorp Vault<br/>Secrets Engine<br/>Dynamic]:::blue
    C --> D[Storage Backend<br/>Encrypted Storage<br/>Consul/etcd/S3]:::green

    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
```

### Installation

**Helm installation**:
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace
```

### Kubernetes Auth Setup

**1. Enable Kubernetes auth**:
```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

**2. Create policy**:
```hcl
# app-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
```

```bash
vault policy write myapp-policy app-policy.hcl
```

**3. Create role**:
```bash
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=myapp \
  policies=myapp-policy \
  ttl=24h
```

### Vault Agent Injector

**Deployment with annotations**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/myapp/database"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/myapp/database" -}}
          export DATABASE_PASSWORD="{{ .Data.data.password }}"
          export DATABASE_USER="{{ .Data.data.username }}"
          {{- end -}}
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: app
        image: myapp:latest
        command: ["/bin/sh"]
        args:
        - -c
        - |
          source /vault/secrets/database
          exec /app/start.sh
```

### External Secrets with Vault

**SecretStore**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secretstore
  namespace: myapp
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: myapp-sa
```

### Dynamic Database Credentials

**1. Configure database engine**:
```bash
vault secrets enable database

vault write database/config/mypostgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/myapp_db" \
  username="vault" \
  password="vault-password"
```

**2. Create role**:
```bash
vault write database/roles/myapp-role \
  db_name=mypostgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

**3. Application retrieves credentials**:
```bash
vault read database/creds/myapp-role

# Output:
# Key                Value
# ---                -----
# lease_id           database/creds/myapp-role/abc123
# lease_duration     1h
# username           v-kubernetes-myapp-abc123
# password           generated-secure-password
```

### Pros and Cons

**Pros**:
- Enterprise-grade security
- Dynamic secrets (auto-rotation)
- Encryption as a service
- Comprehensive audit logs
- Secret versioning
- Multi-cloud support
- High availability
- Fine-grained access control

**Cons**:
- Complex setup and operation
- Requires Vault infrastructure
- Learning curve
- Operational overhead
- Cost (Enterprise features)

### Best For

- Large enterprises
- Regulated industries
- Multi-cloud environments
- High security requirements
- Dynamic credential needs
- Compliance-driven organizations

## Comparison Matrix

| Feature | Manual | K8s Secrets | Sealed Secrets | External Secrets | Vault |
|---------|--------|-------------|----------------|------------------|-------|
| **Setup Complexity** | Low | Low | Medium | Medium | High |
| **GitOps Support** | No | No | Yes | Yes | Yes |
| **Encryption** | No | Optional | Yes | Yes | Yes |
| **Rotation** | Manual | Manual | Manual | Auto | Auto |
| **Audit Logs** | No | Basic | No | Provider | Yes |
| **Multi-Cluster** | Manual | Manual | Manual | Yes | Yes |
| **Cost** | Free | Free | Free | Cloud costs | License |
| **Dynamic Secrets** | No | No | No | Limited | Yes |
| **Best For** | Dev | Small teams | GitOps | Cloud-native | Enterprise |

## Security Best Practices

### 1. Never Commit Secrets to Git

**.gitignore**:
```gitignore
# Secrets
*.env
*secret*.yaml
secrets/
*.key
*.pem
*.p12
*.pfx
```

**Git hooks to prevent commits**:
```bash
# .git/hooks/pre-commit
#!/bin/bash
if git diff --cached | grep -E "(password|secret|key|token)" | grep -v "SecretStore\|ExternalSecret\|SealedSecret"; then
    echo "ERROR: Potential secret in commit!"
    echo "Please review staged files."
    exit 1
fi
```

### 2. Use RBAC

```yaml
# Deny secret access by default
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["configmaps", "pods", "services"]
  verbs: ["get", "list", "watch"]
# Note: secrets NOT included
```

### 3. Enable Audit Logging

```yaml
# Audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
```

### 4. Rotate Regularly

```bash
# Set expiration reminder
kubectl annotate secret app-secrets \
  expires="2024-12-31" \
  owner="platform-team"
```

### 5. Principle of Least Privilege

```yaml
# Only mount secrets actually needed
volumes:
- name: db-secret
  secret:
    secretName: database-credentials
    items:  # Only include needed keys
    - key: password
      path: db-password
```

## Migration Strategies

### From Manual to Sealed Secrets

```bash
# 1. Export existing secrets
kubectl get secret app-secrets -o yaml > temp-secret.yaml

# 2. Seal them
kubeseal -f temp-secret.yaml -w sealed-secret.yaml

# 3. Commit sealed version
git add sealed-secret.yaml
git commit -m "Migrate to sealed secrets"

# 4. Update deployment process
# Use GitOps instead of manual kubectl create
```

### From K8s Secrets to External Secrets

```bash
# 1. Create secrets in external provider
aws secretsmanager create-secret \
  --name myapp/database/password \
  --secret-string "mySecureP@ss123"

# 2. Create ExternalSecret resource
kubectl apply -f external-secret.yaml

# 3. Verify sync
kubectl get externalsecret -n myapp

# 4. Delete old Secret (after verification)
kubectl delete secret app-secrets -n myapp
```

## Troubleshooting

### Common Issues

**Secret not found**:
```bash
kubectl get secrets -n myapp
kubectl describe secret app-secrets -n myapp
```

**Permission denied**:
```bash
kubectl auth can-i get secrets --as=system:serviceaccount:myapp:myapp-sa
```

**Sealed Secret not decrypting**:
```bash
kubectl get sealedsecret -n myapp
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

**External Secret not syncing**:
```bash
kubectl get externalsecret app-secrets -n myapp
kubectl describe externalsecret app-secrets -n myapp
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets
```

## Emergency Procedures

### Suspected Secret Exposure

1. **Immediately rotate affected secrets**
2. **Revoke access** to exposed credentials
3. **Audit access logs** for unauthorized usage
4. **Update all affected applications**
5. **Review and improve** secret handling procedures

### Remove Secret from Git History

```bash
# Use BFG Repo Cleaner
bfg --delete-files secret.yaml
bfg --replace-text passwords.txt
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force

# Or use git filter-branch
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret.yaml" \
  --prune-empty --tag-name-filter cat -- --all
```

## Conclusion

Choose the right secrets management strategy based on your needs:

- **Development**: Manual or K8s Secrets
- **GitOps**: Sealed Secrets
- **Cloud-native**: External Secrets Operator
- **Enterprise**: HashiCorp Vault

Start simple and evolve as requirements grow. Security is a journey, not a destination.

## Additional Resources

- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
- [HashiCorp Vault](https://www.vaultproject.io/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
