# StatefulSet Patterns for Stateful Applications

## Overview

StatefulSets are Kubernetes workloads designed for applications that require stable network identities, persistent storage, and ordered deployment/scaling. This guide covers patterns and best practices for deploying stateful applications like databases, message queues, and distributed systems.

## StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| **Pod Identity** | Random | Stable, ordered |
| **Hostname** | Random | Predictable (pod-0, pod-1) |
| **Storage** | Shared or ephemeral | Dedicated persistent volume per pod |
| **Scaling** | Parallel | Ordered (one at a time) |
| **Updates** | Rolling, random order | Ordered (reverse ordinal) |
| **Use Case** | Stateless apps | Databases, caches, queues |

## When to Use StatefulSets

Use StatefulSets for applications requiring:

1. **Stable network identity**: Predictable hostnames and DNS
2. **Stable storage**: Persistent volumes that follow pods
3. **Ordered operations**: Sequential deployment, scaling, updates
4. **Peer discovery**: Pods need to find and communicate with each other

### Examples

- **Databases**: PostgreSQL, MySQL, MongoDB, Cassandra
- **Message queues**: Kafka, RabbitMQ, NATS
- **Distributed storage**: Elasticsearch, Redis Cluster, etcd
- **Coordination services**: ZooKeeper, Consul

## Basic StatefulSet Pattern

### PostgreSQL Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Governs network identity
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: myapp_db
        - name: POSTGRES_USER
          value: myapp_user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### Key Components Explained

**1. Headless Service** (`clusterIP: None`):
- Provides stable DNS for pods
- DNS format: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
- Example: `postgres-0.postgres.default.svc.cluster.local`

**2. serviceName**:
- Links StatefulSet to headless service
- Governs pod network identity

**3. volumeClaimTemplates**:
- Creates PVC for each pod
- Naming: `<volume-name>-<statefulset-name>-<ordinal>`
- Example: `postgres-storage-postgres-0`
- Persists beyond pod lifecycle

## Headless Service Pattern

### Standard Service (Load Balancing)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-lb
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
  type: ClusterIP  # Has IP, load balances
```

Clients connect to: `postgres-lb:5432` → random pod

### Headless Service (Direct Pod Access)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - port: 5432
```

Clients can connect to specific pods:
- `postgres-0.postgres:5432` → pod 0
- `postgres-1.postgres:5432` → pod 1

### Combined Pattern

```yaml
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
# Regular service for read traffic
apiVersion: v1
kind: Service
metadata:
  name: postgres-read
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```

Usage:
- Write traffic: `postgres-0.postgres:5432` (master)
- Read traffic: `postgres-read:5432` (load balanced)

## Pod Identity and Ordering

### Stable Pod Names

Pods have predictable names with ordinal index:
```
postgres-0
postgres-1
postgres-2
```

### Ordered Operations

**Deployment**:
1. postgres-0 created and becomes Ready
2. postgres-1 created after postgres-0 is Ready
3. postgres-2 created after postgres-1 is Ready

**Scaling down**:
1. postgres-2 terminated first
2. postgres-1 terminated after postgres-2 is gone
3. postgres-0 terminated last

**Updates** (RollingUpdate):
1. postgres-2 updated first
2. postgres-1 updated after postgres-2 is Ready
3. postgres-0 updated last (reverse ordinal)

### DNS Resolution

```bash
# From inside cluster
nslookup postgres-0.postgres.default.svc.cluster.local

# Returns stable IP even after pod restart
```

## Storage Patterns

### 1. Dynamic Provisioning

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: fast-ssd  # Use appropriate storage class
    resources:
      requests:
        storage: 100Gi
```

### 2. Storage Classes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
```

### 3. Multiple Volumes

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 100Gi
- metadata:
    name: logs
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi
```

### 4. Pre-provisioned Volumes

```yaml
# Create PVCs manually before StatefulSet
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-0
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 100Gi
```

## Initialization Patterns

### 1. Init Containers

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: init-postgres
        image: busybox
        command:
        - sh
        - -c
        - |
          # Set proper permissions
          chown -R 999:999 /var/lib/postgresql/data
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      containers:
      - name: postgres
        # ... main container
```

### 2. Data Migration

```yaml
initContainers:
- name: migrate
  image: myapp/migrator:latest
  env:
  - name: DATABASE_URL
    value: postgresql://postgres:5432/mydb
  command: ["/app/migrate"]
```

### 3. Configuration Setup

```yaml
initContainers:
- name: config
  image: busybox
  command:
  - sh
  - -c
  - |
    cp /config-template/postgresql.conf /data/postgresql.conf
    sed -i "s/POD_NAME/$POD_NAME/g" /data/postgresql.conf
  env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  volumeMounts:
  - name: config-template
    mountPath: /config-template
  - name: data
    mountPath: /data
```

## High Availability Patterns

### PostgreSQL Primary-Replica

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3  # 1 primary, 2 replicas
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POSTGRES_REPLICATION_MODE
          value: "$([ \"$POD_NAME\" = \"postgres-0\" ] && echo \"master\" || echo \"slave\")"
        - name: POSTGRES_MASTER_SERVICE
          value: postgres-0.postgres
        # ... additional configuration
```

### Redis Cluster

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 6  # 3 masters, 3 replicas
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - sh
        - -c
        - |
          if [ "$(hostname | cut -d- -f2)" -lt "3" ]; then
            redis-server --cluster-enabled yes --cluster-node-timeout 5000
          else
            redis-server --cluster-enabled yes --cluster-node-timeout 5000 --slaveof redis-$(expr $(hostname | cut -d- -f2) - 3).redis 6379
          fi
```

### Kafka Cluster

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka
  replicas: 3
  template:
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: zookeeper:2181
        - name: KAFKA_ADVERTISED_LISTENERS
          value: PLAINTEXT://$(POD_NAME).kafka:9092
```

## Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods
```

Updates in reverse order: postgres-2 → postgres-1 → postgres-0

### Partition Updates (Canary)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods >= ordinal 2
```

1. Set partition: 2 → only postgres-2 updates
2. Verify postgres-2 works correctly
3. Set partition: 1 → postgres-2 and postgres-1 update
4. Set partition: 0 → all pods update

### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

Pods update only when manually deleted:
```bash
kubectl delete pod postgres-0
# Pod recreates with new version
```

## Pod Management Policies

### OrderedReady (Default)

```yaml
spec:
  podManagementPolicy: OrderedReady
```

- Sequential deployment: wait for previous pod to be Ready
- Sequential scaling: one at a time
- Use for: Databases requiring ordered setup

### Parallel

```yaml
spec:
  podManagementPolicy: Parallel
```

- All pods start simultaneously
- Faster deployment/scaling
- Use for: Peer-to-peer systems (Cassandra, Elasticsearch)

## Health Checks

### Liveness Probe

```yaml
livenessProbe:
  exec:
    command:
    - pg_isready
    - -U
    - postgres
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

### Readiness Probe

```yaml
readinessProbe:
  exec:
    command:
    - sh
    - -c
    - pg_isready -U postgres && psql -U postgres -c 'SELECT 1'
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 3
```

### Startup Probe (for slow starts)

```yaml
startupProbe:
  exec:
    command:
    - pg_isready
  failureThreshold: 30
  periodSeconds: 10
```

## Backup and Recovery

### Backup with CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            command:
            - sh
            - -c
            - |
              pg_dump -h postgres-0.postgres -U postgres myapp_db | \
              gzip > /backup/backup-$(date +%Y%m%d-%H%M%S).sql.gz
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

### Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: standard
  source:
    persistentVolumeClaimName: data-postgres-0
```

### Restore Procedure

```bash
# 1. Scale down StatefulSet
kubectl scale statefulset postgres --replicas=0

# 2. Delete PVC (if starting fresh)
kubectl delete pvc data-postgres-0

# 3. Create PVC from snapshot
kubectl apply -f pvc-from-snapshot.yaml

# 4. Scale up StatefulSet
kubectl scale statefulset postgres --replicas=1
```

## Disaster Recovery

### Cross-Region Replication

```yaml
# Primary cluster (region-a)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
  namespace: region-a
spec:
  replicas: 1
  # ... primary configuration

---
# Replica cluster (region-b)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-replica
  namespace: region-b
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        env:
        - name: POSTGRES_PRIMARY_HOST
          value: postgres-primary-0.postgres-primary.region-a.svc.cluster.local
```

### Velero Backups

```bash
# Install Velero
velero install --provider aws --bucket my-backup-bucket

# Create backup
velero backup create postgres-backup \
  --include-namespaces default \
  --selector app=postgres

# Restore
velero restore create --from-backup postgres-backup
```

## Monitoring Patterns

### Prometheus ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  endpoints:
  - port: metrics
    interval: 30s
```

### Pod Monitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  podMetricsEndpoints:
  - port: metrics
    interval: 30s
```

### Metrics Exporter Sidecar

```yaml
spec:
  template:
    spec:
      containers:
      - name: postgres
        # ... main container
      - name: metrics
        image: prometheuscommunity/postgres-exporter
        env:
        - name: DATA_SOURCE_NAME
          value: postgresql://postgres:5432/myapp_db?sslmode=disable
        ports:
        - containerPort: 9187
          name: metrics
```

## Common Operations

### Scaling

```bash
# Scale up (adds postgres-1, postgres-2, ...)
kubectl scale statefulset postgres --replicas=3

# Scale down (removes in reverse order)
kubectl scale statefulset postgres --replicas=1
```

### Rolling Restart

```bash
# Delete pods one by one (they recreate automatically)
kubectl delete pod postgres-0
# Wait for postgres-0 to be Ready
kubectl delete pod postgres-1
```

### Force Delete Stuck Pod

```bash
# If pod won't terminate
kubectl delete pod postgres-0 --force --grace-period=0

# May need to clean up PVC
kubectl patch pvc data-postgres-0 -p '{"metadata":{"finalizers":null}}'
```

### Update Configuration

```bash
# Update ConfigMap
kubectl edit configmap postgres-config

# Restart pods to pick up changes
kubectl rollout restart statefulset postgres
```

### Check PVC Status

```bash
# List PVCs
kubectl get pvc

# Describe specific PVC
kubectl describe pvc data-postgres-0

# Check PV
kubectl get pv
```

## Troubleshooting

### Pod Stuck in Pending

```bash
# Check events
kubectl describe pod postgres-0

# Common causes:
# - No storage class available
# - Insufficient storage quota
# - Node affinity not satisfied
```

### Pod Stuck in Terminating

```bash
# Check if PVC has finalizers
kubectl get pvc data-postgres-0 -o yaml | grep finalizers

# Force delete if needed
kubectl delete pod postgres-0 --force --grace-period=0
```

### Storage Issues

```bash
# Check PVC status
kubectl get pvc

# Check PV binding
kubectl get pv

# Resize PVC (if storage class allows)
kubectl patch pvc data-postgres-0 -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### Data Corruption

```bash
# Connect to pod
kubectl exec -it postgres-0 -- psql -U postgres

# Check database integrity
postgres=# SELECT * FROM pg_stat_database;

# Restore from backup if needed
```

## Best Practices

### 1. Use Appropriate Storage

- **Local SSD**: Lowest latency (use for performance-critical databases)
- **Network SSD**: Balance of performance and durability
- **Network HDD**: Cost-effective for large, less frequently accessed data

### 2. Set Resource Limits

```yaml
resources:
  requests:
    cpu: 500m
    memory: 1Gi
    ephemeral-storage: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
    ephemeral-storage: 2Gi
```

### 3. Enable Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2  # For 3-replica setup
  selector:
    matchLabels:
      app: postgres
```

### 4. Use Anti-Affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: postgres
      topologyKey: kubernetes.io/hostname
```

### 5. Implement Proper Backups

- Regular automated backups
- Test restore procedures
- Store backups off-cluster
- Document recovery steps

### 6. Monitor Everything

- Pod health (liveness/readiness)
- Resource usage (CPU, memory, disk)
- Database metrics (connections, queries, replication lag)
- Backup success/failures

### 7. Version Control Configuration

```yaml
# Use ConfigMaps for database configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
```

### 8. Document Your Setup

Include in your documentation:
- Architecture diagram
- Scaling procedures
- Backup/restore procedures
- Disaster recovery plan
- Runbook for common issues

## Advanced Patterns

### Operator Pattern

Use operators for complex stateful applications:
- [PostgreSQL Operator](https://github.com/zalando/postgres-operator)
- [MongoDB Operator](https://github.com/mongodb/mongodb-kubernetes-operator)
- [Kafka Operator](https://strimzi.io/)
- [Elasticsearch Operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)

Benefits:
- Automated operations (backup, restore, scaling)
- Best practices built-in
- Simplified management

### Sharded Databases

```yaml
# Shard 0
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-shard0
spec:
  serviceName: mongodb-shard0
  replicas: 3
  # ... configuration

---
# Shard 1
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-shard1
spec:
  serviceName: mongodb-shard1
  replicas: 3
  # ... configuration
```

## Conclusion

StatefulSets provide the foundation for running stateful applications on Kubernetes. Key takeaways:

- Use for applications requiring stable identity and storage
- Leverage headless services for direct pod access
- Implement proper backup and recovery procedures
- Monitor health and performance metrics
- Consider operators for complex setups
- Test disaster recovery procedures regularly

Start with single-instance deployments and evolve to high-availability configurations as requirements grow.

## Additional Resources

- [Kubernetes StatefulSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [Database Operators Awesome List](https://github.com/operator-framework/awesome-operators)
- [Cloud Native PostgreSQL Operator](https://cloudnative-pg.io/)
