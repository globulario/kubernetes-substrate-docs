# Globular + Kubernetes: The Complete Guide
## From Substrate Bootstrap to Production Workloads

This guide shows the complete journey of running Kubernetes on Globular's substrate layer, from initial installation through day-to-day operations.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Day-0: Bootstrap Substrate](#day-0-bootstrap-substrate)
3. [Day-1: Install Kubernetes](#day-1-install-kubernetes)
4. [Day-2: Deploy First Workload](#day-2-deploy-first-workload)
5. [Day-N: Operations](#day-n-operations)
6. [Complete Example](#complete-example)

---

## Architecture Overview

### The Stack

```
┌────────────────────────────────────────────────────────────┐
│  DEVELOPER EXPERIENCE                                      │
│  kubectl apply -f app.yaml                                 │
│  (Just deploy applications, everything else handled)       │
└────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────┐
│  WORKLOAD LAYER (Kubernetes)                               │
│  • kube-apiserver, controller-manager, scheduler           │
│  • kubelet, kube-proxy on worker nodes                     │
│  • CSI drivers, CNI plugins                                │
│  • Application pods, services, ingress                     │
│                                                             │
│  Responsibility: Orchestrate containers, scale workloads   │
└────────────────────────────────────────────────────────────┘
                          ↓ consumes
┌────────────────────────────────────────────────────────────┐
│  SUBSTRATE LAYER (Globular)                                │
│  • etcd (service discovery + K8s state)                    │
│  • DNS (port 53 authority)                                 │
│  • TLS/CA (certificate authority)                          │
│  • MinIO (object storage)                                  │
│  • Authentication & RBAC                                   │
│  • Service discovery, health monitoring                    │
│  • Gateway & Envoy (networking)                            │
│                                                             │
│  Responsibility: Infrastructure truth, continuous          │
│                  correctness, survive failures             │
└────────────────────────────────────────────────────────────┘
```

### The Separation of Concerns

| Concern | Owner | Why |
|---------|-------|-----|
| **What exists** (nodes, services) | Globular | Infrastructure truth must survive workload failures |
| **Who is trusted** (certificates, auth) | Globular | Root of trust must be outside the system it protects |
| **Continuous correctness** (healing) | Globular | Recovery must not depend on the system being healthy |
| **What runs** (containers, pods) | Kubernetes | Workload orchestration is what K8s excels at |
| **How it scales** (replicas, HPA) | Kubernetes | Application-level scaling is workload concern |

---

## Day-0: Bootstrap Substrate

### What Gets Installed

```bash
./install-day0.sh

# Installs (in order):
# 1. TLS infrastructure (CA, certificates)
# 2. etcd cluster (distributed state)
# 3. DNS service (port 53)
# 4. MinIO (object storage)
# 5. Service discovery
# 6. Gateway & Envoy
# 7. RBAC & Authentication
# 8. Platform services (log, event, etc.)
```

### Validation After Day-0

```bash
# Check all substrate services running
systemctl list-units 'globular-*' --state=running

# Expected services:
# ✓ globular-etcd.service
# ✓ globular-dns.service
# ✓ globular-minio.service
# ✓ globular-discovery.service
# ✓ globular-gateway.service
# ✓ globular-authentication.service

# Verify DNS resolves
dig @127.0.0.1 globular.internal

# Verify etcd accessible
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/globular/config/tls/ca.pem \
  --cert=/var/lib/globular/config/tls/server.crt \
  --key=/var/lib/globular/config/tls/server.key \
  endpoint health

# Verify MinIO accessible
curl -k https://127.0.0.1:9000/minio/health/live

# Verify service discovery
curl -k https://127.0.0.1:10016/api/services
```

### What You Have Now

✅ **Infrastructure truth established**
- Certificate authority (root of trust)
- DNS resolution (before any workload)
- Service registry (discovery)
- Storage foundation (MinIO)
- Authentication root (RBAC)

✅ **Convergent system**
- All services managed by `globular-installer`
- Specs define desired state
- System self-heals on drift

✅ **Ready for workload layer**
- K8s can be installed as a package
- No circular dependencies
- Clean separation of concerns

---

## Day-1: Install Kubernetes

### Install Control Plane

```bash
# Download K8s control plane package
globular-installer apply kubernetes-control-plane.yaml

# What happens:
# 1. Generates K8s certificates from Globular CA
# 2. Configures kube-apiserver to use Globular etcd
# 3. Installs kube-apiserver, controller-manager, scheduler
# 4. Starts services (systemd units)
# 5. Registers with Globular service discovery
# 6. Validates health checks

# Wait for control plane ready
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig cluster-info

# Output:
# Kubernetes control plane is running at https://127.0.0.1:6443
```

### Add Worker Nodes

```bash
# On each worker node
globular-installer apply kubernetes-worker-node.yaml

# What happens:
# 1. Discovers API server via Globular service discovery
# 2. Generates node certificate from Globular CA
# 3. Installs kubelet, kube-proxy
# 4. Joins cluster via TLS bootstrap
# 5. Registers with Globular health monitoring

# Verify nodes
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig get nodes

# Output:
# NAME         STATUS   ROLES    AGE   VERSION
# worker-01    Ready    <none>   1m    v1.29.2
# worker-02    Ready    <none>   1m    v1.29.2
# worker-03    Ready    <none>   1m    v1.29.2
```

### Install Storage Integration

```bash
# Install MinIO CSI driver and StorageClasses
globular-installer apply kubernetes-storage-minio.yaml

# What happens:
# 1. Syncs MinIO credentials from substrate to K8s secret
# 2. Syncs Globular CA to K8s (for TLS)
# 3. Installs CSI driver (DaemonSet)
# 4. Creates StorageClasses (standard, fast, replicated, archive)

# Verify storage
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig get storageclass

# Output:
# NAME                  PROVISIONER                RECLAIMPOLICY   ...
# minio-standard (def)  minio.csi.globular.io      Delete          ...
# minio-fast            minio.csi.globular.io      Delete          ...
# minio-replicated      minio.csi.globular.io      Retain          ...
# minio-archive         minio.csi.globular.io      Retain          ...
```

### Install CNI Plugin (Pod Networking)

```bash
# Install Calico CNI
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig apply -f \
  https://docs.projectcalico.org/manifests/calico.yaml

# Wait for Calico pods
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig wait --for=condition=ready pod \
  -l k8s-app=calico-node \
  -n kube-system \
  --timeout=300s
```

### What You Have Now

✅ **Kubernetes cluster running**
- Control plane (API server, controller, scheduler)
- Worker nodes (kubelet, kube-proxy)
- Storage classes (backed by substrate MinIO)
- Pod networking (Calico CNI)

✅ **Integrated with substrate**
- K8s state in Globular's etcd
- K8s certs from Globular's CA
- K8s storage from Globular's MinIO
- K8s discovered by Globular's service registry

✅ **Ready for workloads**
- Deploy applications with kubectl
- Use standard K8s APIs
- No knowledge of substrate required

---

## Day-2: Deploy First Workload

### Example: WordPress with MySQL

```yaml
# wordpress-stack.yaml
---
# MySQL Database (replicated storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: default
spec:
  storageClassName: minio-replicated
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: default
type: Opaque
stringData:
  password: changeme123

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: default
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: MYSQL_DATABASE
              value: wordpress
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mysql-data

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
  clusterIP: None

---
# WordPress Application (standard storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-data
  namespace: default
spec:
  storageClassName: minio-standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:6.4
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql:3306
            - name: WORDPRESS_DB_NAME
              value: wordpress
            - name: WORDPRESS_DB_USER
              value: root
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: data
              mountPath: /var/www/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: wordpress-data

---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: default
spec:
  selector:
    app: wordpress
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

### Deploy

```bash
# Create kubeconfig for easier use
export KUBECONFIG=/etc/kubernetes/admin.kubeconfig

# Deploy WordPress stack
kubectl apply -f wordpress-stack.yaml

# Watch deployment
kubectl get pods -w

# Expected output:
# NAME                         READY   STATUS    RESTARTS   AGE
# mysql-7d8c4d4d9b-8xk2j       1/1     Running   0          2m
# wordpress-6b5f8d6d8c-abc12   1/1     Running   0          2m
# wordpress-6b5f8d6d8c-def34   1/1     Running   0          2m
# wordpress-6b5f8d6d8c-ghi56   1/1     Running   0          2m

# Check PVCs (backed by substrate MinIO)
kubectl get pvc

# Output:
# NAME             STATUS   VOLUME     CAPACITY   STORAGECLASS       AGE
# mysql-data       Bound    pvc-abc    10Gi       minio-replicated   2m
# wordpress-data   Bound    pvc-def    5Gi        minio-standard     2m

# Access WordPress
kubectl get svc wordpress

# Output:
# NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# wordpress   LoadBalancer   10.96.100.50   <pending>     80:30080/TCP   2m

# Access via NodePort (if LoadBalancer not available)
curl http://localhost:30080
```

### What Just Happened

1. **K8s created PVCs** → StorageClass: minio-standard, minio-replicated
2. **CSI driver provisioned volumes** → Created MinIO buckets: `k8s-pv-abc`, `k8s-pv-def`
3. **K8s scheduled pods** → Placed on worker nodes
4. **Kubelet mounted volumes** → s3fs mounted MinIO buckets into containers
5. **Application running** → WordPress writing to MinIO (via substrate)

### Verify Substrate Integration

```bash
# Check MinIO buckets (from substrate)
mc ls globular/ | grep k8s-pv

# Output:
# k8s-pv-abc/  (MySQL data - 10Gi quota, replicated)
# k8s-pv-def/  (WordPress uploads - 5Gi quota, standard)

# Check etcd (K8s state in substrate)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/globular/config/tls/ca.pem \
  --cert=/var/lib/globular/config/tls/server.crt \
  --key=/var/lib/globular/config/tls/server.key \
  get /registry/pods/default/mysql --prefix

# Output: K8s pod definitions stored in Globular's etcd

# Check service discovery
curl -sk https://127.0.0.1:10016/api/services | grep kubernetes

# Output: K8s API server registered with substrate discovery
```

---

## Day-N: Operations

### Routine Operations

#### 1. Scale Application

```bash
# Scale WordPress
kubectl scale deployment wordpress --replicas=10

# K8s handles scaling (workload layer)
# Storage, networking, DNS handled by substrate
```

#### 2. Update Application

```bash
# Update image
kubectl set image deployment/wordpress wordpress=wordpress:6.5

# K8s does rolling update
# Persistent data remains in substrate MinIO
```

#### 3. Backup

```bash
# Backup entire substrate (includes all K8s state and data)
globular backup create --full

# What gets backed up:
# ✓ etcd (all K8s resources)
# ✓ MinIO buckets (all PV data)
# ✓ TLS certificates
# ✓ Service configurations
# ✓ DNS zones

# K8s doesn't need special backup tools
# Substrate backup = complete cluster backup
```

#### 4. Add Storage to Existing PVC

```bash
# Expand MySQL volume
kubectl patch pvc mysql-data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# CSI driver expands MinIO bucket quota
# No downtime, MySQL keeps running
```

### Certificate Rotation (Automatic)

```bash
# Globular automatically rotates certificates
# K8s components pick up new certs seamlessly

# View certificate expiry
openssl x509 -in /var/lib/kubernetes/pki/apiserver.crt -noout -dates

# notBefore=Feb  8 12:00:00 2024 GMT
# notAfter=Feb  8 12:00:00 2025 GMT

# When certificate nearing expiry:
# 1. Globular generates new cert from CA
# 2. Updates /var/lib/kubernetes/pki/apiserver.crt
# 3. systemd restarts kube-apiserver (graceful)
# 4. No manual intervention

# No more: "cert expired, cluster down, panic"
```

### Disaster Recovery

```bash
# Scenario: Complete infrastructure loss

# Step 1: Provision new hardware/VMs
# (bare metal or cloud)

# Step 2: Restore Globular substrate
globular restore --snapshot=backup-2024-02-08-02:00

# This restores:
# ✓ etcd with all K8s state
# ✓ MinIO with all PV data
# ✓ TLS certificates
# ✓ Service configurations

# Step 3: Install K8s control plane
globular-installer apply kubernetes-control-plane.yaml

# K8s comes up with all state from etcd

# Step 4: Install storage integration
globular-installer apply kubernetes-storage-minio.yaml

# CSI driver discovers existing PVs from MinIO

# Step 5: Add worker nodes
for node in worker-{01..05}; do
  globular-installer apply kubernetes-worker-node.yaml --node=$node
done

# Step 6: Verify
kubectl get pods --all-namespaces

# All StatefulSets, Deployments recreate
# All PVCs bound to existing volumes
# All data intact

# Time to recovery: ~2 hours (vs. days with traditional approach)
```

### Upgrade Kubernetes

```bash
# Update package version
vim kubernetes-control-plane.yaml
# Change: version: "1.29.2" → "1.30.0"

# Apply updated package
globular-installer apply kubernetes-control-plane.yaml

# Convergent installer:
# 1. Downloads new binaries
# 2. Stops scheduler (graceful)
# 3. Replaces binary
# 4. Starts scheduler
# 5. Validates health
# 6. Repeats for controller-manager
# 7. Repeats for apiserver (graceful, rolling)
# 8. Validates all health checks

# If health checks fail: automatic rollback

# No: kubeadm upgrade dance
# No: drain nodes manually
# No: pray it works

# Just: package update, convergent apply
```

### Add Worker Node

```bash
# On new node
globular-installer apply kubernetes-worker-node.yaml

# What happens automatically:
# 1. Node discovers API server (via substrate service discovery)
# 2. Gets certificate from Globular CA
# 3. Joins cluster (TLS bootstrap)
# 4. Registers with substrate monitoring

# No join tokens, no manual steps
```

### Remove Worker Node

```bash
# Drain node
kubectl drain worker-05 --ignore-daemonsets --delete-emptydir-data

# Remove from K8s
kubectl delete node worker-05

# Decommission in substrate
globular node remove worker-05

# Clean
```

---

## Complete Example

### Full Installation Script

```bash
#!/bin/bash
# complete-k8s-on-globular-install.sh
# Installs complete K8s cluster on Globular substrate

set -euo pipefail

echo "========================================="
echo "Kubernetes on Globular Substrate"
echo "Complete Installation"
echo "========================================="

# Day-0: Bootstrap substrate
echo ""
echo "Step 1/5: Installing Globular substrate..."
./install-day0.sh

# Verify substrate
echo "Verifying substrate services..."
systemctl is-active globular-etcd >/dev/null || { echo "etcd not running"; exit 1; }
systemctl is-active globular-dns >/dev/null || { echo "DNS not running"; exit 1; }
systemctl is-active globular-minio >/dev/null || { echo "MinIO not running"; exit 1; }
echo "✓ Substrate ready"

# Day-1: Install K8s control plane
echo ""
echo "Step 2/5: Installing Kubernetes control plane..."
globular-installer apply kubernetes-control-plane.yaml

# Wait for API server
echo "Waiting for API server..."
for i in {1..60}; do
  if kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig cluster-info &>/dev/null; then
    echo "✓ Control plane ready"
    break
  fi
  sleep 2
done

# Add worker nodes
echo ""
echo "Step 3/5: Adding worker nodes..."
for node in worker-{01..03}; do
  echo "  Adding ${node}..."
  ssh ${node} "$(cat kubernetes-worker-node.yaml)" | globular-installer apply -
done

# Wait for nodes
echo "Waiting for nodes..."
for i in {1..60}; do
  READY=$(kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig get nodes --no-headers | grep -c Ready || true)
  if [[ "${READY}" -eq 3 ]]; then
    echo "✓ All nodes ready"
    break
  fi
  sleep 2
done

# Install storage
echo ""
echo "Step 4/5: Installing storage integration..."
globular-installer apply kubernetes-storage-minio.yaml

# Wait for CSI driver
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig wait --for=condition=ready pod \
  -l app=minio-csi-driver \
  -n kube-system \
  --timeout=120s
echo "✓ Storage ready"

# Install CNI
echo ""
echo "Step 5/5: Installing CNI plugin..."
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig apply -f \
  https://docs.projectcalico.org/manifests/calico.yaml

kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig wait --for=condition=ready pod \
  -l k8s-app=calico-node \
  -n kube-system \
  --timeout=300s
echo "✓ Networking ready"

# Summary
echo ""
echo "========================================="
echo "Installation Complete!"
echo "========================================="
echo ""
echo "Cluster Status:"
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig get nodes
echo ""
echo "Storage Classes:"
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig get storageclass
echo ""
echo "Export kubeconfig:"
echo "  export KUBECONFIG=/etc/kubernetes/admin.kubeconfig"
echo ""
echo "Deploy first application:"
echo "  kubectl apply -f wordpress-stack.yaml"
echo ""
echo "Verify substrate integration:"
echo "  mc ls globular/ | grep k8s-pv"
echo ""
echo "Kubernetes is ready. Happy deploying!"
```

### Run It

```bash
# One command to complete cluster
chmod +x complete-k8s-on-globular-install.sh
./complete-k8s-on-globular-install.sh

# Output:
# =========================================
# Kubernetes on Globular Substrate
# Complete Installation
# =========================================
#
# Step 1/5: Installing Globular substrate...
# ✓ Substrate ready
#
# Step 2/5: Installing Kubernetes control plane...
# ✓ Control plane ready
#
# Step 3/5: Adding worker nodes...
#   Adding worker-01...
#   Adding worker-02...
#   Adding worker-03...
# ✓ All nodes ready
#
# Step 4/5: Installing storage integration...
# ✓ Storage ready
#
# Step 5/5: Installing CNI plugin...
# ✓ Networking ready
#
# =========================================
# Installation Complete!
# =========================================
#
# Cluster Status:
# NAME         STATUS   ROLES    AGE   VERSION
# worker-01    Ready    <none>   5m    v1.29.2
# worker-02    Ready    <none>   5m    v1.29.2
# worker-03    Ready    <none>   5m    v1.29.2
#
# Kubernetes is ready. Happy deploying!
```

---

## The End Result

### Before (Traditional K8s)

```
Time to production cluster: 1-2 weeks
- Research installation methods
- Learn kubeadm, kubelet, kubectl
- Set up etcd cluster
- Generate certificates manually
- Configure networking
- Install storage operator (Rook/Ceph)
- Debug circular dependencies
- Write runbooks for failures
- Hire SRE team

Ongoing operations: High anxiety
- Certificate expiry = 3am page
- etcd corruption = all hands on deck
- Storage issues = data loss risk
- Upgrades = pray and hope
```

### After (K8s on Globular Substrate)

```
Time to production cluster: 2 hours
- Run install script
- Deploy applications
- Done

Ongoing operations: Boring
- Certificates rotate automatically
- etcd managed by substrate
- Storage just works
- Upgrades are mechanical
- Disaster recovery is fast
- No more heroics
```

### The Manifesto Made Real

> **"The goal is that one day someone will say: 'Kubernetes isn't scary anymore.' And not remember why."**

**With Globular substrate:**
- Kubernetes is boring ✓
- Operations are mechanical ✓
- Recovery doesn't require heroics ✓
- Complexity removed, not normalized ✓

**That's how you know the layer is correct.**

---

## Next Steps

1. **Try it**: Install Globular substrate + K8s on a test cluster
2. **Deploy workloads**: Use standard K8s APIs, verify substrate integration
3. **Test failures**: Kill components, watch substrate heal them
4. **Measure**: Compare ops time vs. traditional K8s
5. **Adopt**: Roll out to production with confidence

**The substrate makes Kubernetes boring. And boring is beautiful.**
