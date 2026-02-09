# Kubernetes on Globular Substrate

**Complete documentation and package specs for running Kubernetes as a workload layer on Globular's infrastructure substrate.**

> "The goal is not to replace Kubernetes. The goal is to free it."
> — From the Globular Manifesto

## What Is This?

This repository contains the complete blueprint for running Kubernetes on Globular's substrate layer, demonstrating the separation of **infrastructure truth** (what exists, who is trusted) from **workload truth** (what runs, how it scales).

## The Core Insight

Kubernetes is a workload orchestrator. It was never designed to:
- Bootstrap itself
- Manage its own certificates
- Run its own storage infrastructure
- Establish root trust
- Survive its own failure modes

**Globular provides the missing substrate layer** — the infrastructure authority that exists before workloads, survives when workloads fail, and continuously enforces correctness.

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│  WORKLOAD LAYER (Kubernetes)                              │
│  • Container orchestration                                │
│  • Pod scheduling and scaling                             │
│  • Service routing and ingress                            │
│  • Application lifecycle management                       │
└───────────────────────────────────────────────────────────┘
                        ↓ consumes
┌───────────────────────────────────────────────────────────┐
│  SUBSTRATE LAYER (Globular)                               │
│  • etcd (service discovery + K8s state storage)           │
│  • DNS (port 53 authority)                                │
│  • TLS/CA (certificate authority, automatic rotation)     │
│  • MinIO (object storage for persistent volumes)          │
│  • Authentication & RBAC (identity root)                  │
│  • Service discovery and health monitoring                │
│  • Convergent installer (self-healing infrastructure)     │
└───────────────────────────────────────────────────────────┘
```

## What This Achieves

### Before (Traditional K8s)
- ❌ Circular dependencies (storage in K8s, K8s needs storage)
- ❌ Bootstrap complexity (kubeadm rituals, manual cert generation)
- ❌ Disaster recovery nightmares (if K8s dies, storage dies)
- ❌ Certificate expiry = 3am pages
- ❌ Weeks to production cluster

### After (K8s on Globular Substrate)
- ✅ No circular dependencies (infrastructure exists first)
- ✅ Simple bootstrap (K8s is just a package)
- ✅ Fast disaster recovery (substrate restores K8s)
- ✅ Automatic certificate rotation
- ✅ Hours to production cluster

## Documentation

### Quick Start
- **[Complete Guide](./globular-kubernetes-complete-guide.md)** — Day-0 through Day-N operations, includes full installation script

### Architecture Deep Dives
- **[Substrate Architecture](./architecture/kubernetes-substrate-architecture.md)** — How K8s integrates with Globular's infrastructure layer
- **[Storage Architecture](./architecture/kubernetes-storage-architecture.md)** — Persistent volumes backed by substrate MinIO

### Package Specs
- **[Control Plane Spec](./specs/kubernetes-control-plane-spec.yaml)** — kube-apiserver, controller-manager, scheduler
- **[Worker Node Spec](./specs/kubernetes-worker-node-spec.yaml)** — kubelet, kube-proxy
- **[Storage Integration Spec](./specs/kubernetes-storage-minio-spec.yaml)** — CSI driver, StorageClasses

## Installation

### Prerequisites
- Globular substrate installed (Day-0 complete)
- 3+ nodes (1 control plane, 2+ workers)

### Install K8s Cluster

```bash
# 1. Install control plane
globular-installer apply specs/kubernetes-control-plane-spec.yaml

# 2. Install worker nodes
for node in worker-{01..03}; do
  ssh $node "globular-installer apply specs/kubernetes-worker-node-spec.yaml"
done

# 3. Install storage integration
globular-installer apply specs/kubernetes-storage-minio-spec.yaml

# 4. Install CNI (pod networking)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Done! Deploy workloads
export KUBECONFIG=/etc/kubernetes/admin.kubeconfig
kubectl apply -f your-app.yaml
```

### Verify Substrate Integration

```bash
# Check K8s uses substrate etcd
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/globular/config/tls/ca.pem \
  get /registry/pods --prefix | grep -c pod

# Check K8s volumes in substrate MinIO
mc ls globular/ | grep k8s-pv

# Check K8s registered with substrate discovery
curl -sk https://127.0.0.1:10016/api/services | grep kubernetes
```

## Key Features

### 1. No Circular Dependencies

**Traditional:**
```
K8s needs etcd → etcd runs in K8s → Bootstrap deadlock
K8s needs DNS → CoreDNS runs in K8s → Bootstrap deadlock
K8s needs certs → cert-manager runs in K8s → Trust deadlock
```

**Substrate:**
```
Globular provides etcd → K8s uses it → No dependency loop
Globular provides DNS → K8s uses it → No dependency loop
Globular provides CA → K8s uses it → No dependency loop
```

### 2. Infrastructure as Packages

Kubernetes is installed as a convergent package:
```yaml
metadata:
  name: kubernetes-control-plane
  version: "1.29.2"
  dependencies:
    - etcd          # Substrate service
    - dns           # Substrate service
    - tls-infrastructure  # Substrate CA
```

**Benefits:**
- Idempotent installation (re-apply is safe)
- Automatic health checks
- Self-healing via convergent installer
- Version management (upgrade = update package version)

### 3. Automatic Certificate Rotation

**Traditional:** Manual cert renewal, hope you remember before expiry
```bash
# Every year, manually:
kubeadm certs renew all
systemctl restart kube-apiserver
# Pray nothing breaks
```

**Substrate:** Automatic rotation, zero downtime
```bash
# Globular rotates automatically:
# 1. Detects approaching expiry
# 2. Generates new cert from CA
# 3. Updates certificate files
# 4. Services pick up new certs
# No manual intervention
```

### 4. Disaster Recovery

**Traditional:** Days to weeks (if possible at all)
```
1. K8s cluster corrupted
2. Storage operator (Rook) also corrupted
3. Storage data inaccessible
4. Manual recovery procedures
5. Hope you have backups
6. Hope backups are correct
```

**Substrate:** Hours
```bash
# 1. Restore Globular substrate (includes all K8s data)
globular restore --snapshot=daily-backup

# 2. Reinstall K8s (idempotent package)
globular-installer apply kubernetes-control-plane-spec.yaml

# 3. Add worker nodes
for node in workers; do
  globular-installer apply kubernetes-worker-node-spec.yaml --node=$node
done

# Done - all K8s state and data intact
```

### 5. Storage Without Complexity

**Traditional:** Rook/Ceph operator managing storage inside K8s
- Complex installation (50+ CRDs)
- Resource overhead (storage pods consuming cluster resources)
- Circular dependency (storage needs K8s, K8s needs storage)
- Difficult disaster recovery

**Substrate:** MinIO storage layer, K8s consumes it via CSI
- Simple: Storage exists before K8s starts
- Efficient: Storage outside cluster resource pool
- Reliable: Storage survives K8s failures
- Easy DR: Restore substrate = restore all volumes

#### StorageClass Tiers

```yaml
minio-standard (default)
  • Use: Application data, configs, logs
  • Policy: Delete on PVC removal
  • Quota: 10Gi

minio-fast
  • Use: Build caches, CI/CD, ML training
  • Policy: Reduced redundancy
  • Quota: 100Gi

minio-replicated
  • Use: Databases, critical state
  • Policy: Retain (survives PVC deletion)
  • Feature: Cross-region replication
  • Quota: 50Gi

minio-archive
  • Use: Backups, compliance data
  • Policy: Lifecycle archival (30d → glacier)
  • Quota: 1Ti
```

## Operations

### Routine Tasks

```bash
# Scale application
kubectl scale deployment/app --replicas=10

# Update application
kubectl set image deployment/app app=v2.0

# Backup (includes all K8s state and volumes)
globular backup create --full

# Expand storage
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### Upgrade K8s

```bash
# Update package version
vim specs/kubernetes-control-plane-spec.yaml
# version: "1.29.2" → "1.30.0"

# Apply (convergent installer handles rolling update)
globular-installer apply specs/kubernetes-control-plane-spec.yaml

# Automatic rollback if health checks fail
```

### Add Worker Node

```bash
# On new node (auto-discovers cluster via substrate)
globular-installer apply specs/kubernetes-worker-node-spec.yaml

# Node automatically:
# - Gets certificate from Globular CA
# - Discovers API server via service registry
# - Joins cluster via TLS bootstrap
# - Registers with health monitoring
```

## Example Workloads

### Stateless Application

```yaml
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
          image: nginx:latest
```

### Stateful Application (Database)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: minio-replicated  # Substrate-backed storage
  accessModes: [ReadWriteOnce]
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
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
```

**Behind the scenes:**
- PVC → MinIO CSI driver
- CSI driver → creates MinIO bucket in substrate
- Bucket → backed up by Globular
- Data → survives K8s restarts, node failures, cluster rebuilds

## Monitoring

### Substrate Health

```bash
# Check substrate services
systemctl status globular-etcd
systemctl status globular-dns
systemctl status globular-minio

# Verify K8s using substrate
curl -sk https://127.0.0.1:10016/api/services | jq '.[] | select(.name | contains("kubernetes"))'
```

### K8s Health

```bash
# Standard K8s monitoring
kubectl get nodes
kubectl get pods --all-namespaces
kubectl top nodes
kubectl top pods

# Storage usage
kubectl get pvc --all-namespaces
mc ls globular/ | grep k8s-pv
```

## Philosophy

From the [Globular Manifesto](../manifesto.txt):

> **"Infrastructure dependencies were installed inside the system that depended on them."**

This was the core problem. Kubernetes was forced to:
- Bootstrap itself (kubeadm)
- Manage its own etcd
- Run its own DNS (CoreDNS pods)
- Generate its own certificates
- Manage its own storage (Rook/Ceph)

**The result:** Circular dependencies, fragility, heroics at 3am.

> **"A substrate cluster owns infrastructure truth. A workload cluster consumes it."**

This is the solution:
- **Globular (substrate)** owns: What exists, who is trusted, what must remain true
- **Kubernetes (workload)** owns: What runs, where it runs, how it scales

> **"The goal is that one day someone will say: 'Kubernetes isn't scary anymore.' And not remember why."**

**This is that day.** When you run K8s on Globular substrate:
- No more bootstrap rituals
- No more certificate expiry panic
- No more storage operator complexity
- No more disaster recovery nightmares

**Just boring, reliable Kubernetes.** And boring is beautiful.

## Contributing

This is a living document. As we build and refine the Globular substrate pattern, these specs and docs will evolve.

Areas for contribution:
- Real-world deployment experiences
- Performance benchmarks (substrate vs. traditional)
- Additional integrations (monitoring, logging, service mesh)
- Migration guides from existing K8s clusters

## Related Documentation

- [Globular Manifesto](../manifesto.txt) — The philosophical foundation
- [Day-0 Bootstrap](../globular-installer/README.md) — Installing substrate
- [Service Specs](../packages/specs/) — All Globular service packages
- [Architecture](../architecture/) — Overall Globular design

## License

[Add license information]

## Contact

[Add contact/community information]

---

**"Kubernetes isn't scary anymore."**

That's how you know the substrate is working.
