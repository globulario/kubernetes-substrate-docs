# Kubernetes Storage on Globular Substrate: Architecture Deep Dive

This document explains how Kubernetes persistent storage integrates with Globular's MinIO substrate, demonstrating the separation of storage infrastructure from workload consumption.

## The Storage Layers

```
┌───────────────────────────────────────────────────────────────┐
│  APPLICATION LAYER (Workload Code)                            │
│                                                                │
│  const data = fs.readFile('/data/config.json')                │
│  // Application doesn't know about K8s or MinIO               │
└───────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────┐
│  KUBERNETES CONSUMPTION LAYER (Workload API)                  │
│                                                                │
│  apiVersion: v1                                                │
│  kind: PersistentVolumeClaim                                  │
│  spec:                                                         │
│    storageClassName: minio-standard                           │
│    resources:                                                  │
│      requests:                                                 │
│        storage: 5Gi                                           │
│                                                                │
│  • Developer uses K8s PVC API (standard, portable)            │
│  • Doesn't know or care about underlying storage              │
│  • Can switch providers by changing storageClassName          │
└───────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────┐
│  KUBERNETES INTEGRATION LAYER (CSI Driver)                    │
│                                                                │
│  • CSI Driver runs as DaemonSet in K8s                        │
│  • Translates PVC requests to MinIO operations                │
│  • Creates buckets, manages credentials, mounts volumes       │
│  • Uses Globular's MinIO credentials (from substrate)         │
│                                                                │
│  What CSI Driver Does:                                        │
│  1. Receives CreateVolume request from K8s                    │
│  2. Authenticates to MinIO using substrate credentials        │
│  3. Creates bucket: k8s-pv-<uuid>                             │
│  4. Returns PV reference to K8s                               │
│  5. On pod start: mounts bucket via s3fs/FUSE                 │
└───────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────┐
│  GLOBULAR SUBSTRATE LAYER (Infrastructure Truth)              │
│                                                                │
│  • MinIO running at https://127.0.0.1:9000                    │
│  • Started before K8s (Day-0 bootstrap)                       │
│  • Uses Globular's TLS certificates                           │
│  • Credentials managed by Globular                            │
│  • Backups handled by Globular persistence                    │
│  • Survives K8s failures                                      │
│                                                                │
│  What Globular Manages:                                       │
│  • MinIO service lifecycle (start, stop, restart)            │
│  • TLS certificate rotation                                   │
│  • Credential management and rotation                         │
│  • Backup scheduling and retention                            │
│  • Replication configuration                                  │
│  • Health monitoring and alerting                             │
└───────────────────────────────────────────────────────────────┘
```

## Why This Separation Matters

### The Traditional Problem

**Kubernetes-native storage (like Rook/Ceph):**

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                          │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Application Pods                                   │    │
│  │  (need storage)                                     │    │
│  └────────────────────────────────────────────────────┘    │
│                     ↓                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Storage Operator (Rook)                           │    │
│  │  (runs INSIDE K8s, manages storage)                │    │
│  └────────────────────────────────────────────────────┘    │
│                     ↓                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Ceph/MinIO Pods                                    │    │
│  │  (storage system running IN K8s)                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Problems:**
1. **Circular dependency**: Storage system needs K8s to run, but K8s scheduler needs storage
2. **Bootstrap deadlock**: How do you start storage pods if storage isn't available yet?
3. **Disaster recovery**: If K8s is corrupted, your storage management is also corrupted
4. **Resource overhead**: Storage management consumes cluster resources meant for workloads
5. **Upgrade complexity**: Upgrading K8s risks breaking storage, upgrading storage risks breaking K8s

### The Substrate Solution

**Storage as Infrastructure Truth (Globular pattern):**

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (Workload Layer)                        │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Application Pods                                   │    │
│  │  (consume storage via PVC)                         │    │
│  └────────────────────────────────────────────────────┘    │
│                     ↓                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  CSI Driver (bridge to substrate)                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       ↓ consumes (no management)
┌─────────────────────────────────────────────────────────────┐
│  Globular Substrate (Infrastructure Layer)                  │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  MinIO Service                                      │    │
│  │  (native systemd service, not a K8s pod)           │    │
│  │  (exists before K8s starts)                         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Benefits:**
1. ✅ **No circular dependency**: MinIO runs before K8s starts
2. ✅ **Simple bootstrap**: MinIO is part of Day-0 substrate setup
3. ✅ **Disaster recovery**: MinIO survives K8s failures, can restore K8s state
4. ✅ **Resource efficiency**: Storage management outside cluster resources
5. ✅ **Independent upgrades**: Upgrade K8s or MinIO independently

## Storage Tiers & Use Cases

### 1. Standard Storage (Default)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: minio-standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: minio.csi.globular.io
parameters:
  minio.csi.globular.io/bucket-prefix: "k8s-pv"
  minio.csi.globular.io/versioning: "false"
  minio.csi.globular.io/quota: "10Gi"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Characteristics:**
- General-purpose storage for most workloads
- Deleted when PVC is deleted (ephemeral workload data)
- 10Gi max per volume (can be expanded)
- No versioning (single copy)

**Use Cases:**
- Application configuration files
- Application logs (before shipping to log aggregator)
- Cache directories
- Build artifacts
- User uploads (non-critical)

**Example:**
```yaml
# Web application with config storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-config
spec:
  storageClassName: minio-standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          volumeMounts:
            - name: config
              mountPath: /app/config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: webapp-config
```

### 2. Fast Storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: minio-fast
provisioner: minio.csi.globular.io
parameters:
  minio.csi.globular.io/bucket-prefix: "k8s-pv-fast"
  minio.csi.globular.io/storage-class: "REDUCED_REDUNDANCY"
  minio.csi.globular.io/quota: "100Gi"
```

**Characteristics:**
- Reduced redundancy for speed
- Larger quota (100Gi)
- Optimized for high-throughput workloads

**Use Cases:**
- Build caches (Maven, npm, Docker layers)
- CI/CD temporary workspaces
- Video transcoding scratch space
- Machine learning training data (ephemeral)

**Example:**
```yaml
# Jenkins build cache
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-cache
spec:
  storageClassName: minio-fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  template:
    spec:
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          volumeMounts:
            - name: cache
              mountPath: /var/jenkins_home/.m2
            - name: cache
              mountPath: /var/jenkins_home/.npm
      volumes:
        - name: cache
          persistentVolumeClaim:
            claimName: jenkins-cache
```

### 3. Replicated Storage (Critical Data)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: minio-replicated
provisioner: minio.csi.globular.io
parameters:
  minio.csi.globular.io/bucket-prefix: "k8s-pv-replicated"
  minio.csi.globular.io/versioning: "true"
  minio.csi.globular.io/replication: "enabled"
  minio.csi.globular.io/replication-target: "secondary-minio"
reclaimPolicy: Retain  # Data preserved even if PVC deleted
```

**Characteristics:**
- Cross-region/cross-cluster replication
- Versioning enabled (can restore previous versions)
- Retain policy (data survives PVC deletion)
- Higher durability guarantees

**Use Cases:**
- Database storage (PostgreSQL, MySQL, MongoDB)
- Critical application state
- Payment/transaction data
- User authentication data
- Audit logs

**Example:**
```yaml
# PostgreSQL with replicated storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: minio-replicated
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          ports:
            - containerPort: 5432
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: minio-replicated
        resources:
          requests:
            storage: 20Gi
```

### 4. Archive Storage (Long-term Retention)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: minio-archive
provisioner: minio.csi.globular.io
parameters:
  minio.csi.globular.io/bucket-prefix: "k8s-pv-archive"
  minio.csi.globular.io/versioning: "true"
  minio.csi.globular.io/lifecycle-policy: "transition-to-glacier-30d"
  minio.csi.globular.io/quota: "1Ti"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

**Characteristics:**
- Lifecycle policies (auto-archive after 30 days)
- Very large quota (1TB)
- Versioning enabled
- Cost-optimized for infrequent access

**Use Cases:**
- Database backups
- Compliance data retention
- Historical logs
- Audit trails
- Disaster recovery snapshots

**Example:**
```yaml
# Automated backup job
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
              image: postgres:16
              command:
                - /bin/bash
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  pg_dump -h postgres -U postgres -d mydb > /backups/backup_${TIMESTAMP}.sql
                  # Compress
                  gzip /backups/backup_${TIMESTAMP}.sql
                  # Cleanup backups older than 90 days (lifecycle handles archiving)
                  find /backups -name "*.sql.gz" -mtime +90 -delete
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
              volumeMounts:
                - name: backup-storage
                  mountPath: /backups
          restartPolicy: OnFailure
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-archive-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-archive-pvc
spec:
  storageClassName: minio-archive
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
```

## Data Flow: From Application to MinIO

### Write Path

```
1. Application writes to /data/file.txt
   └─ Container filesystem (mounted PVC)

2. Kernel VFS layer
   └─ Routes write to mounted filesystem

3. s3fs/FUSE driver (in CSI driver)
   └─ Intercepts write, converts to S3 API call

4. CSI Driver
   └─ Authenticates using Globular credentials
   └─ Sends PutObject API request

5. Network Layer
   └─ HTTPS to https://127.0.0.1:9000
   └─ TLS using Globular's CA certificates

6. MinIO Server (Substrate)
   └─ Validates credentials (Globular-managed)
   └─ Writes object to bucket: k8s-pv-<uuid>/file.txt
   └─ Returns success

7. CSI Driver
   └─ Confirms write to application
   └─ Application continues
```

### Read Path

```
1. Application reads from /data/file.txt
   └─ Container filesystem (mounted PVC)

2. s3fs checks local cache
   ├─ Cache hit? Return immediately
   └─ Cache miss? Fetch from MinIO

3. CSI Driver
   └─ Sends GetObject API request
   └─ Using Globular credentials + TLS

4. MinIO Server (Substrate)
   └─ Retrieves object from bucket
   └─ Returns data stream

5. s3fs/FUSE
   └─ Caches data locally
   └─ Returns to application

6. Application receives data
```

## Failure Scenarios & Recovery

### Scenario 1: Pod Restart

**What Happens:**
```
1. Pod crashes or is rescheduled
2. K8s scheduler places pod on (possibly different) node
3. Kubelet calls CSI driver to mount volume
4. CSI driver connects to MinIO (substrate)
5. Bucket k8s-pv-<uuid> still exists
6. Volume mounted, application sees same data
```

**Key Point:** Data survived because it's in substrate layer, not tied to pod/node lifecycle.

### Scenario 2: Kubernetes Cluster Failure

**Traditional (Storage in K8s):**
```
1. K8s cluster becomes corrupted
2. Storage operator (Rook) also corrupted
3. Ceph/MinIO pods won't start
4. Storage system is down
5. Cannot restore K8s because K8s needs storage to start
6. Circular dependency deadlock
```

**With Substrate:**
```
1. K8s cluster becomes corrupted
2. MinIO continues running (substrate layer, systemd service)
3. All K8s persistent data is safe in MinIO
4. Rebuild K8s cluster from scratch:
   - Install K8s control plane (from substrate)
   - Install CSI driver
   - Install StorageClasses
5. Existing PVs re-discovered from MinIO
6. Applications resume with all data intact
```

### Scenario 3: Storage Credential Rotation

**Traditional (Manual):**
```
1. Rotate MinIO credentials manually
2. Update K8s secret
3. Restart all CSI driver pods
4. Hope all volumes remount correctly
5. Debug mount failures
6. Update application secrets if they use MinIO directly
```

**With Substrate:**
```
1. Globular rotates MinIO credentials (automatic, scheduled)
2. Globular updates K8s secret (automatic sync)
3. CSI driver pods detect secret change, reload credentials
4. Active mounts use new credentials on next API call
5. No manual intervention
6. Zero downtime
```

### Scenario 4: Disaster Recovery

**Full Cluster Loss:**

```bash
# Day 0: Backup (automated by Globular)
globular backup create --full --include-minio

# Backup includes:
# - MinIO buckets (all K8s PVs)
# - MinIO configuration
# - MinIO credentials
# - K8s state in etcd

# Day 30: Disaster strikes (datacenter destroyed)

# Step 1: Provision new infrastructure
# Bare metal or VMs

# Step 2: Restore Globular substrate
globular restore --snapshot=daily-backup-2024-02-08

# What gets restored:
# ✓ MinIO service with all buckets
# ✓ All K8s persistent volume data
# ✓ etcd with K8s state
# ✓ TLS certificates
# ✓ Service discovery
# ✓ DNS zones

# Step 3: Install K8s control plane
globular-installer apply kubernetes-control-plane.yaml

# Step 4: Install storage integration
globular-installer apply kubernetes-storage-minio.yaml

# Step 5: Add worker nodes
for node in worker-{01..05}; do
  globular-installer apply kubernetes-worker-node.yaml --node=$node
done

# Result: Cluster fully operational
kubectl get pv
# All PVs present (discovered from MinIO buckets)

kubectl get pods --all-namespaces
# All stateful pods recreate themselves
# All data intact from MinIO substrate
```

**Time to Recovery:**
- Traditional (Rook/Ceph in K8s): Days (if possible at all)
- Substrate pattern: Hours

## Performance Considerations

### Comparison: Native vs. S3-backed Storage

**Native Block Storage (Rook/Ceph):**
- ✅ Lower latency (direct block access)
- ✅ Better for databases with heavy I/O
- ❌ More complex to operate
- ❌ Harder to backup/replicate
- ❌ Resource-intensive

**S3-backed Storage (MinIO Substrate):**
- ✅ Simpler operations (HTTP API)
- ✅ Easy backup/replication
- ✅ Separation from K8s lifecycle
- ❌ Higher latency (HTTP overhead)
- ❌ FUSE layer adds overhead

### Optimization Strategies

**1. Local Caching (s3fs)**
```yaml
# CSI driver configuration
parameters:
  minio.csi.globular.io/cache-size: "10Gi"
  minio.csi.globular.io/cache-location: "/var/lib/kubelet/cache"
  minio.csi.globular.io/cache-ttl: "300s"
```

**2. Read-mostly Workloads**
- Best use case for S3-backed storage
- Cache hit rate > 90% for static content
- Examples: Configuration, ML models, static assets

**3. Write-heavy Workloads**
- Consider using minio-fast StorageClass
- Or use native block storage for databases (not via substrate)
- Hybrid approach: Critical data in substrate, temp data in local

**4. Multipart Uploads**
```yaml
parameters:
  minio.csi.globular.io/multipart-threshold: "100Mi"
  minio.csi.globular.io/multipart-chunksize: "16Mi"
```

## Monitoring & Observability

### What to Monitor

**1. MinIO Substrate Metrics:**
```bash
# MinIO provides Prometheus metrics
curl -sk https://127.0.0.1:9000/minio/v2/metrics/cluster

# Key metrics:
# - minio_bucket_usage_total_bytes
# - minio_node_disk_used_bytes
# - minio_s3_requests_total
# - minio_s3_errors_total
```

**2. CSI Driver Metrics:**
```bash
# CSI driver pod logs
kubectl logs -n kube-system -l app=minio-csi-driver

# Key events:
# - Volume provisioning time
# - Mount/unmount success rate
# - Connection errors to MinIO
```

**3. K8s Storage Metrics:**
```bash
# PVC usage
kubectl get pvc --all-namespaces -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.status.capacity.storage,\
STORAGECLASS:.spec.storageClassName

# PV status
kubectl get pv -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name
```

### Grafana Dashboard Example

```yaml
# Key panels for Globular storage monitoring

1. MinIO Storage Capacity
   - Total capacity
   - Used space by bucket (including k8s-pv-* buckets)
   - Growth rate

2. K8s Storage Consumption
   - PVCs by StorageClass
   - Top 10 largest volumes
   - Volume provisioning rate

3. Performance
   - S3 request latency (p50, p95, p99)
   - CSI driver mount time
   - Error rate

4. Substrate Health
   - MinIO service uptime
   - TLS certificate expiry
   - Credential rotation status
```

## Cost Optimization

### Storage Tier Selection

```yaml
# Cost comparison (relative costs)

minio-standard:     1x   (baseline)
minio-fast:         0.8x (reduced redundancy)
minio-replicated:   2x   (cross-region replication)
minio-archive:      0.3x (lifecycle archival)
```

### Lifecycle Policies

```yaml
# Automatic cost optimization via MinIO lifecycle

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: minio-tiered
parameters:
  minio.csi.globular.io/lifecycle-policy: |
    {
      "Rules": [
        {
          "ID": "transition-old-data",
          "Status": "Enabled",
          "Filter": {},
          "Transitions": [
            {
              "Days": 30,
              "StorageClass": "STANDARD_IA"
            },
            {
              "Days": 90,
              "StorageClass": "GLACIER"
            }
          ]
        },
        {
          "ID": "expire-old-logs",
          "Status": "Enabled",
          "Filter": {"Prefix": "logs/"},
          "Expiration": {
            "Days": 365
          }
        }
      ]
    }
```

## Migration from Traditional Storage

### From Rook/Ceph to Substrate MinIO

**Step 1: Install MinIO substrate storage alongside existing:**
```bash
# Install storage integration (doesn't affect existing PVs)
globular-installer apply kubernetes-storage-minio.yaml

# Verify both StorageClasses exist
kubectl get storageclass
# rook-ceph-block       (existing)
# minio-standard        (new, substrate-backed)
```

**Step 2: Migrate data volume-by-volume:**
```yaml
# Create new PVC on substrate storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data-minio
spec:
  storageClassName: minio-standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# Migration job
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-myapp-data
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: busybox
          command:
            - /bin/sh
            - -c
            - |
              # Copy data from old to new volume
              cp -av /old-data/* /new-data/
              # Verify
              diff -r /old-data /new-data
          volumeMounts:
            - name: old-data
              mountPath: /old-data
            - name: new-data
              mountPath: /new-data
      restartPolicy: OnFailure
      volumes:
        - name: old-data
          persistentVolumeClaim:
            claimName: myapp-data-ceph  # Old
        - name: new-data
          persistentVolumeClaim:
            claimName: myapp-data-minio # New
---
# Update application to use new PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: myapp-data-minio  # Changed
```

**Step 3: Decommission Rook after full migration:**
```bash
# Once all workloads migrated
kubectl delete -f rook-operator.yaml
kubectl delete storageclass rook-ceph-block

# Rook pods stop, but substrate storage keeps working
```

## Summary: Why This Pattern Works

### The Key Insight

**Quote from the Manifesto:**
> "Every reliable system separates: what keeps the world correct from what uses the world to do work"

**Applied to Storage:**
- **Substrate (Globular)** keeps storage correct: MinIO running, credentials valid, backups happening, replication working
- **Workload (K8s)** uses storage to do work: Applications read/write data via PVCs

### What This Achieves

1. **Simplicity**: K8s developers just use PVCs (standard API), don't manage storage infrastructure
2. **Reliability**: Storage survives K8s failures, can restore K8s from storage
3. **Operations**: Manage storage separately from workloads (independent upgrades, backups)
4. **Performance**: Right tool for the job (substrate for storage, K8s for orchestration)
5. **Cost**: Shared infrastructure (same MinIO for substrate + K8s workloads)

### The End Result

```bash
# Developer experience
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  storageClassName: minio-standard
  resources:
    requests:
      storage: 5Gi
EOF

# That's it. Storage "just works"
# - No storage operator to install
# - No complex Ceph/Rook configuration
# - No circular dependencies
# - No bootstrap problems
# - Just boring, reliable storage
```

**"Storage isn't scary anymore."**

And developers won't remember why - because the substrate made it invisible.
