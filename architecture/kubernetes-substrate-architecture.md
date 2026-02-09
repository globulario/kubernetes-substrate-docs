# Kubernetes on Globular Substrate: Architecture

This document explains how Kubernetes control plane runs as a package on Globular's substrate layer, demonstrating the separation of infrastructure truth from workload truth.

## The Two-Layer Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    WORKLOAD TRUTH (Kubernetes)                 │
│                                                                │
│  What Runs:                                                    │
│  • Containers, Pods, Deployments                              │
│  • Application services and ingress                            │
│  • Horizontal pod autoscaling                                  │
│  • Rolling updates and rollbacks                               │
│                                                                │
│  Responsibilities:                                             │
│  ✓ Schedule pods onto nodes                                   │
│  ✓ Manage pod lifecycle and health                            │
│  ✓ Scale workloads based on demand                            │
│  ✓ Route traffic to services                                  │
│                                                                │
│  What K8s Does NOT Do:                                        │
│  ✗ Establish root trust (Globular's CA)                       │
│  ✗ Manage its own certificates (Globular rotates them)        │
│  ✗ Bootstrap itself (Globular installed it)                   │
│  ✗ Define which nodes exist (Globular manages nodes)          │
└───────────────────────────────────────────────────────────────┘
                              ↓
                     Consumes Infrastructure
                              ↓
┌───────────────────────────────────────────────────────────────┐
│               INFRASTRUCTURE TRUTH (Globular Substrate)        │
│                                                                │
│  What Exists:                                                  │
│  • etcd cluster (service discovery + K8s state)               │
│  • DNS authority (port 53)                                     │
│  • Certificate authority (TLS root of trust)                   │
│  • Service discovery registry                                  │
│  • Storage foundation (MinIO)                                  │
│  • Authentication root (RBAC)                                  │
│                                                                │
│  Responsibilities:                                             │
│  ✓ Establish identity and trust                               │
│  ✓ Continuously enforce correctness (convergent installer)    │
│  ✓ Survive workload failures                                  │
│  ✓ Manage node membership                                      │
│  ✓ Rotate certificates automatically                           │
│  ✓ Monitor infrastructure health                               │
│                                                                │
│  What Globular Does NOT Do:                                   │
│  ✗ Schedule application containers (that's K8s)               │
│  ✗ Manage pod networking (that's CNI)                         │
│  ✗ Scale workloads (that's HPA)                               │
└───────────────────────────────────────────────────────────────┘
```

## Installation Sequence

### Day 0: Bootstrap Substrate (Globular)

```bash
# Step 1: Install Globular substrate layer
./install-day0.sh

# What gets installed:
# ✓ Certificate authority (Globular's root trust)
# ✓ etcd cluster (distributed key-value store)
# ✓ DNS service (port 53 authority)
# ✓ Service discovery
# ✓ MinIO (object storage)
# ✓ Authentication & RBAC services
# ✓ Gateway & Envoy (networking)

# Infrastructure truth is now established
```

### Day 1: Install Kubernetes (as a Package)

```bash
# Step 2: Install K8s control plane
globular-installer apply kubernetes-control-plane.yaml

# What happens:
# ✓ Generates K8s certificates from Globular's CA
# ✓ Configures kube-apiserver to use Globular's etcd
# ✓ Starts kube-apiserver with Globular's TLS
# ✓ Starts kube-controller-manager and kube-scheduler
# ✓ Registers K8s with Globular's service discovery
# ✓ Health checks validate all components running

# Workload orchestration is now available
```

### Day 2+: Operate

```bash
# Infrastructure operations (Globular)
globular node add worker-03              # Add node to substrate
globular cert rotate --all               # Rotate all certs (including K8s)
globular backup restore --snapshot=pre-upgrade

# Workload operations (Kubernetes)
kubectl apply -f application.yaml        # Deploy app
kubectl scale deployment/app --replicas=10
kubectl rollout restart deployment/app
```

## Integration Points

### 1. etcd Storage

**Before (Traditional K8s):**
```bash
# K8s must bootstrap its own etcd cluster
kubeadm init --control-plane-endpoint=... --upload-certs
# Manually manage etcd backups, scaling, disaster recovery
```

**With Globular Substrate:**
```yaml
# kube-apiserver config
etcd:
  endpoints:
    - https://127.0.0.1:2379      # Globular's etcd
  cafile: /var/lib/globular/config/tls/ca.pem
  certfile: /var/lib/globular/config/tls/server.crt
  keyfile: /var/lib/globular/config/tls/server.key
```

**Benefits:**
- ✅ etcd already running before K8s starts
- ✅ Backups handled by Globular's persistence layer
- ✅ Health monitoring by Globular's health service
- ✅ No circular dependency (etcd doesn't depend on K8s)

### 2. Certificate Management

**Before (Traditional K8s):**
```bash
# Generate certs with kubeadm
kubeadm init phase certs all

# Manually rotate certs before expiry
kubeadm certs renew all
systemctl restart kube-apiserver  # Hope nothing breaks
```

**With Globular Substrate:**
```bash
# Certificate generation script (in package spec)
/usr/lib/globular/scripts/k8s-generate-certs.sh

# Uses Globular's CA to sign K8s certificates
openssl x509 -req -in apiserver.csr \
  -CA /var/lib/globular/pki/ca.crt \
  -CAkey /var/lib/globular/pki/ca.key \
  -out /var/lib/kubernetes/pki/apiserver.crt
```

**Benefits:**
- ✅ Single root of trust (Globular's CA)
- ✅ Automatic rotation via Globular's TLS system
- ✅ No manual cert renewal
- ✅ K8s services trust Globular services (same CA)

### 3. DNS Resolution

**Before (Traditional K8s):**
```bash
# CoreDNS runs as a pod INSIDE the cluster
kubectl apply -f coredns.yaml

# Circular dependency: DNS pod needs DNS to schedule
# Bootstrap problem: How does the first pod get DNS?
```

**With Globular Substrate:**
```
┌─────────────────────────────────────────┐
│  K8s CoreDNS (optional, for .svc.local) │  ← Workload DNS
├─────────────────────────────────────────┤
│  Globular DNS (port 53, authoritative)  │  ← Infrastructure DNS
└─────────────────────────────────────────┘
```

**Benefits:**
- ✅ DNS exists before K8s starts (no bootstrap problem)
- ✅ Infrastructure services discoverable immediately
- ✅ K8s CoreDNS can forward to Globular DNS
- ✅ No circular dependency

### 4. Service Discovery

**Before (Traditional K8s):**
```bash
# K8s services only discoverable within cluster
# External tools need kubeconfig, API access, etc.
```

**With Globular Substrate:**
```bash
# K8s API server registers with Globular's discovery
curl -X POST https://127.0.0.1:10016/api/register -d '{
  "name": "kubernetes.KubernetesService",
  "address": "127.0.0.1:6443",
  "protocol": "https"
}'

# Now discoverable by all substrate services
globular service list | grep kubernetes
# kubernetes.KubernetesService  127.0.0.1:6443  running
```

**Benefits:**
- ✅ K8s is a first-class service in the substrate
- ✅ Health monitoring by Globular
- ✅ Consistent service discovery across layers
- ✅ No special tooling needed to find K8s

## Failure Scenarios & Recovery

### Scenario 1: Certificate Expires

**Traditional K8s:**
```
1. API server fails to start (cert expired)
2. kubectl doesn't work (can't reach API)
3. Manual intervention required:
   - SSH into control plane node
   - Run kubeadm certs renew
   - Manually restart kube-apiserver
   - Hope you did it before the cluster is dead
```

**With Globular Substrate:**
```
1. Globular's cert rotation detects upcoming expiry
2. Generates new cert from CA (automated)
3. Updates /var/lib/kubernetes/pki/apiserver.crt
4. systemd notices file change, restarts kube-apiserver
5. No human intervention needed
```

### Scenario 2: etcd Corruption

**Traditional K8s:**
```
1. etcd pod fails to start (data corruption)
2. API server can't reach etcd, becomes unhealthy
3. Cluster is down, workloads unreachable
4. Manual recovery:
   - Find backup (do you have one?)
   - Restore etcd data
   - Restart etcd pods
   - Restart API server
   - Validate cluster state
```

**With Globular Substrate:**
```
1. Globular's health monitor detects etcd failure
2. Convergent installer kicks in:
   - Checks expected state (etcd running)
   - Detects deviation (etcd stopped)
   - Applies corrective action (restart/restore)
3. etcd comes back up (using Globular's persistence backup)
4. API server reconnects automatically
5. K8s resumes normal operation
```

### Scenario 3: Control Plane Node Failure

**Traditional K8s:**
```
1. Control plane node hardware fails
2. etcd quorum lost (if single-node)
3. API server unreachable
4. Manual recovery:
   - Provision new node
   - Rejoin to control plane (if multi-node)
   - Or full cluster rebuild (if single-node)
```

**With Globular Substrate:**
```
1. Globular detects control plane node offline
2. Substrate services (etcd, DNS, discovery) failover to healthy nodes
3. Provision new node with Globular bootstrap
4. Run: globular-installer apply kubernetes-control-plane.yaml
5. New control plane rejoins substrate automatically
6. K8s control plane reconstituted from substrate state
```

## Operational Workflows

### Adding a Worker Node

**Traditional K8s:**
```bash
# 1. Manually provision node
# 2. Install container runtime (containerd/docker)
# 3. Install kubelet, kubeadm, kubectl
# 4. Generate join token on control plane
kubeadm token create --print-join-command

# 5. SSH to worker node, run join command
kubeadm join 192.168.1.10:6443 --token abc123...

# 6. Hope it works
```

**With Globular Substrate:**
```bash
# 1. Bootstrap node with Globular (gets identity, certs, DNS)
./install-day0.sh --role=worker

# 2. Install K8s worker package (convergent, idempotent)
globular-installer apply kubernetes-worker-node.yaml

# 3. Node automatically:
#    - Gets TLS certs from Globular CA
#    - Discovers API server via Globular discovery
#    - Joins cluster using substrate identity
#    - Registers with Globular health monitoring

# 4. Done
kubectl get nodes
# worker-03   Ready   <none>   1m   v1.29.2
```

### Upgrading Kubernetes

**Traditional K8s:**
```bash
# 1. Read 50-page upgrade guide
# 2. Backup everything manually
# 3. Upgrade control plane first
kubeadm upgrade plan
kubeadm upgrade apply v1.30.0
# 4. Restart all control plane components
# 5. Upgrade kubelets on all nodes
# 6. Drain nodes one by one
# 7. Pray nothing breaks
```

**With Globular Substrate:**
```bash
# 1. Update package version
sed -i 's/version: "1.29.2"/version: "1.30.0"/' \
  kubernetes-control-plane.yaml

# 2. Apply updated package (convergent installer handles it)
globular-installer apply kubernetes-control-plane.yaml

# What happens automatically:
# - Downloads new K8s binaries (1.30.0)
# - Compares with running version (1.29.2)
# - Performs rolling update:
#   - Stop kube-scheduler
#   - Replace binary
#   - Start kube-scheduler (validate health)
#   - Repeat for controller-manager
#   - Repeat for apiserver (graceful)
# - Validates all health checks pass
# - Rollback automatically if health checks fail

# 3. Done
kubectl version
# Server Version: v1.30.0
```

### Disaster Recovery

**Traditional K8s:**
```bash
# 1. Entire cluster is gone (datacenter fire?)
# 2. Restore from backups (do you have them?)
# 3. Manually rebuild control plane
# 4. Manually rejoin worker nodes
# 5. Manually redeploy all applications
# 6. Days of work
```

**With Globular Substrate:**
```bash
# 1. Provision new infrastructure (bare metal or VMs)

# 2. Bootstrap Globular substrate from backup
globular restore --snapshot=daily-backup-2024-02-08

# What gets restored:
# - etcd data (includes ALL K8s state)
# - Service discovery registry
# - Certificate authority and certs
# - DNS zones
# - Configuration

# 3. Install K8s control plane (idempotent package)
globular-installer apply kubernetes-control-plane.yaml

# 4. K8s comes up with all state intact (from etcd restore)
kubectl get pods --all-namespaces
# All pods recreate themselves automatically

# 5. Add worker nodes
for node in worker-{01..05}; do
  globular node provision $node
  globular-installer apply kubernetes-worker-node.yaml --node=$node
done

# 6. Cluster fully operational
# Hours instead of days
```

## Why This Architecture Matters

### The Traditional Problem

```
┌─────────────────────────────────────────────┐
│         Kubernetes (trying to do it all)     │
│                                              │
│  • Scheduling workloads                      │
│  • Managing its own etcd                     │
│  • Bootstrapping itself (kubeadm)           │
│  • Generating its own certificates           │
│  • Running its own DNS (CoreDNS pod)        │
│  • Managing node identity                    │
│  • Surviving its own failures (???)         │
│                                              │
│  Result: Circular dependencies, fragility   │
└─────────────────────────────────────────────┘
```

**Quote from the Manifesto:**
> "Infrastructure dependencies were installed inside the system that depended on them."

### The Substrate Solution

```
┌───────────────────────────────────────────────┐
│  Kubernetes (focused on workloads)            │
│  • Schedule pods                              │
│  • Scale deployments                          │
│  • Route traffic                              │
│  • Manage application lifecycle              │
└───────────────────────────────────────────────┘
          ↓ consumes stable infrastructure
┌───────────────────────────────────────────────┐
│  Globular Substrate (infrastructure truth)    │
│  • etcd (exists before K8s)                   │
│  • TLS/CA (rotates K8s certs)                 │
│  • DNS (resolves before K8s starts)           │
│  • Service discovery (tracks K8s health)      │
│  • Convergent installer (heals K8s)          │
└───────────────────────────────────────────────┘
```

**Quote from the Manifesto:**
> "A substrate cluster owns infrastructure truth. A workload cluster consumes it."

## The End Result

When someone asks: **"How do I run Kubernetes?"**

**Before (Traditional):**
```
1. Read Kubernetes The Hard Way
2. Understand etcd, TLS, networking
3. Master kubeadm, kubelet, kubectl
4. Learn disaster recovery procedures
5. Build runbooks for every failure scenario
6. Hire SREs to keep it running
7. Still get paged at 3am
```

**With Globular Substrate:**
```bash
# Install substrate (once)
./install-day0.sh

# Install Kubernetes (package)
globular-installer apply kubernetes-control-plane.yaml

# Deploy applications
kubectl apply -f my-app.yaml

# That's it
```

**Quote from the Manifesto:**
> "The goal is that one day someone will say: 'Kubernetes isn't scary anymore.' And not remember why."

---

**This is how you know the layer is correct.**
