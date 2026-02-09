# Kubernetes DNS Integration with Globular Substrate

This document explains how DNS resolution works across the substrate and workload layers, solving the circular dependency problem that plagues traditional Kubernetes DNS.

## The DNS Problem in Traditional Kubernetes

### The Circular Dependency

```
┌─────────────────────────────────────────────────────┐
│  Traditional K8s (DNS Bootstrap Problem)            │
│                                                      │
│  1. K8s scheduler needs to place CoreDNS pods       │
│     └─ But scheduler needs DNS to function          │
│                                                      │
│  2. CoreDNS pods need to be scheduled               │
│     └─ But DNS doesn't exist yet                    │
│                                                      │
│  3. Bootstrap workaround:                           │
│     • Use /etc/hosts temporarily                    │
│     • Hard-code IP addresses                        │
│     • Hope CoreDNS starts                           │
│     • Manual intervention if it fails               │
│                                                      │
│  Result: Fragile, requires heroics                  │
└─────────────────────────────────────────────────────┘
```

**Quote from the Manifesto:**
> "Infrastructure dependencies were installed inside the system that depended on them."

CoreDNS running **inside** Kubernetes is the perfect example of this anti-pattern.

## The Substrate Solution

### Two-Layer DNS Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  APPLICATION LAYER                                             │
│                                                                 │
│  Pod queries: postgres.database.svc.cluster.local             │
│  App doesn't know about DNS layers                             │
└────────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────────┐
│  KUBERNETES DNS (Workload Layer)                               │
│  CoreDNS pods at 10.96.0.10                                    │
│                                                                 │
│  Handles:                                                       │
│  • *.cluster.local (K8s services)                             │
│    Example: kubernetes.default.svc.cluster.local → 10.96.0.1  │
│                                                                 │
│  Forwards:                                                      │
│  • *.globular.internal → Globular DNS (substrate)             │
│  • Everything else → Upstream DNS (1.1.1.1)                   │
└────────────────────────────────────────────────────────────────┘
                           ↓ forwards substrate queries
┌────────────────────────────────────────────────────────────────┐
│  GLOBULAR DNS (Substrate Layer)                                │
│  Native systemd service on host port 53                        │
│                                                                 │
│  Authoritative for: globular.internal                          │
│  Records:                                                       │
│  • etcd.globular.internal → 127.0.0.1:2379                    │
│  • minio.globular.internal → 127.0.0.1:9000                   │
│  • gateway.globular.internal → 127.0.0.1:8443                 │
│  • All substrate services                                      │
│                                                                 │
│  Exists: Before K8s starts                                     │
│  Survives: K8s failures                                        │
└────────────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **DNS Infrastructure Exists First**
   - Globular DNS starts as part of Day-0 substrate bootstrap
   - Running before any workload layer (K8s) exists
   - No bootstrap problem: DNS is already there

2. **Split-Horizon DNS**
   - `*.cluster.local` → Handled by CoreDNS (K8s workload services)
   - `*.globular.internal` → Handled by Globular DNS (infrastructure services)
   - Everything else → Upstream DNS (public internet)

3. **No Circular Dependencies**
   - Globular DNS doesn't depend on K8s
   - K8s CoreDNS can start because substrate DNS already exists
   - Clean separation: Infrastructure DNS vs. Workload DNS

## DNS Resolution Flow

### Flow 1: K8s Service Resolution

```
┌─────────────────────────────────────────────────────────────┐
│  Pod queries: postgres.database.svc.cluster.local          │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Pod's /etc/resolv.conf                                     │
│  nameserver 10.96.0.10  (CoreDNS ClusterIP)                │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  CoreDNS (running in K8s)                                   │
│  Checks: Does query match "cluster.local"?                  │
│  ✓ Yes → Use Kubernetes plugin                             │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Plugin                                          │
│  1. Query K8s API: "Get service postgres in namespace      │
│     database"                                               │
│  2. API returns: ClusterIP 10.96.50.20                     │
│  3. Return A record: postgres.database.svc.cluster.local   │
│     → 10.96.50.20                                          │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Pod receives: 10.96.50.20                                  │
│  Connects to PostgreSQL via K8s Service                     │
└─────────────────────────────────────────────────────────────┘
```

### Flow 2: Substrate Service Resolution

```
┌─────────────────────────────────────────────────────────────┐
│  Pod queries: minio.globular.internal                       │
│  (Needs to access substrate storage)                        │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  CoreDNS (running in K8s)                                   │
│  Checks: Does query match "cluster.local"?                  │
│  ✗ No → Check if matches "globular.internal"               │
│  ✓ Yes → Forward to Globular DNS                           │
└─────────────────────────────────────────────────────────────┘
                       ↓ forwards
┌─────────────────────────────────────────────────────────────┐
│  Globular DNS (substrate, host port 53)                     │
│  1. Check zone: globular.internal (authoritative)           │
│  2. Lookup: minio.globular.internal                         │
│  3. Return A record: 127.0.0.1                              │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Pod receives: 127.0.0.1 (via host networking)              │
│  Connects to substrate MinIO service                        │
└─────────────────────────────────────────────────────────────┘
```

### Flow 3: External Resolution

```
┌─────────────────────────────────────────────────────────────┐
│  Pod queries: api.github.com                                │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  CoreDNS (running in K8s)                                   │
│  Checks: cluster.local? No. globular.internal? No.          │
│  → Use catch-all forward rule                               │
└─────────────────────────────────────────────────────────────┘
                       ↓ forwards
┌─────────────────────────────────────────────────────────────┐
│  Upstream DNS (1.1.1.1 Cloudflare)                          │
│  Returns: api.github.com → 140.82.121.6                     │
└─────────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Pod receives: 140.82.121.6                                 │
│  Connects to GitHub API                                     │
└─────────────────────────────────────────────────────────────┘
```

## CoreDNS Configuration

### Corefile (CoreDNS Config)

```
# Zone 1: K8s Services (cluster.local)
cluster.local:53 {
    errors
    health
    ready

    # Kubernetes plugin: query K8s API for service IPs
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }

    prometheus :9153
    cache 30
    loop
    reload
    loadbalance
}

# Zone 2: Substrate Services (globular.internal)
globular.internal:53 {
    errors
    log

    # Forward to Globular DNS (substrate layer)
    # THIS IS THE KEY INTEGRATION POINT!
    forward . 127.0.0.1:53 {
        max_concurrent 1000
        policy sequential
        health_check 2s
    }

    cache 30
    reload
}

# Zone 3: External (catch-all)
.:53 {
    errors
    health
    ready

    # Forward to public DNS
    forward . 1.1.1.1 1.0.0.1 {
        max_concurrent 1000
        policy random
        health_check 5s
    }

    cache 30
    reload
}
```

### What Each Section Does

**cluster.local zone:**
- Handles all K8s service DNS queries
- Uses Kubernetes plugin to query API server
- Returns ClusterIPs for services
- Enables K8s-native service discovery

**globular.internal zone:**
- Handles substrate service DNS queries
- **Forwards to Globular DNS at 127.0.0.1:53**
- This is where workload layer consumes infrastructure layer
- Enables pods to discover substrate services

**catch-all zone:**
- Handles everything else (internet domains)
- Forwards to upstream DNS (Cloudflare 1.1.1.1)
- Standard external DNS resolution

## Comparison: Traditional vs. Substrate

### Traditional K8s DNS Bootstrap

```
Time 0:00 - K8s cluster starting
├─ Problem: No DNS service exists
├─ Problem: CoreDNS pods need to be scheduled
├─ Problem: Scheduler may need DNS to function
├─ Workaround: Use /etc/hosts with hard-coded IPs
├─ Workaround: Hope CoreDNS pods schedule
├─ Time 0:05 - CoreDNS pods scheduled
├─ Time 0:06 - CoreDNS pods starting
├─ Time 0:08 - CoreDNS ready
└─ Result: DNS working after ~8 minutes of fragility

What breaks:
• If CoreDNS pods fail to schedule (no resources)
• If CoreDNS crashes (no DNS until restart)
• If CoreDNS ConfigMap is corrupt
• If K8s API is slow (plugin times out)
```

### Substrate K8s DNS Bootstrap

```
Time 0:00 - Globular substrate already running
├─ Globular DNS on port 53 (substrate layer)
├─ etcd.globular.internal → works
├─ minio.globular.internal → works
├─ All infrastructure DNS → works
│
Time 0:01 - K8s cluster starting
├─ K8s uses substrate DNS (no bootstrap problem)
├─ CoreDNS pods scheduled (can resolve substrate)
├─ CoreDNS starts, adds cluster.local zone
├─ Time 0:02 - K8s DNS ready
└─ Result: DNS working immediately, no fragility

What doesn't break:
✓ If CoreDNS crashes: substrate DNS still works
✓ If K8s is slow: substrate DNS unaffected
✓ If no resources: CoreDNS schedules eventually, substrate DNS works meanwhile
```

## DNS Namespace Organization

### Recommended Structure

```
globular.internal (substrate zone)
├─ etcd.globular.internal → 127.0.0.1:2379
├─ minio.globular.internal → 127.0.0.1:9000
├─ gateway.globular.internal → 127.0.0.1:8443
├─ dns.globular.internal → 127.0.0.1:10006
├─ discovery.globular.internal → 127.0.0.1:10016
└─ auth.globular.internal → 127.0.0.1:10012

cluster.local (K8s zone)
├─ kubernetes.default.svc.cluster.local → 10.96.0.1
├─ kube-dns.kube-system.svc.cluster.local → 10.96.0.10
├─ postgres.database.svc.cluster.local → 10.96.50.20
└─ myapp.production.svc.cluster.local → 10.96.100.50

# Optional: Register K8s services in substrate for external access
myapp.apps.globular.internal → <LoadBalancer IP>
```

### Naming Conventions

**Substrate services:**
- Pattern: `<service>.globular.internal`
- Examples: `etcd.globular.internal`, `minio.globular.internal`
- Managed by: Globular DNS service
- Authoritative: Yes (substrate owns this zone)

**K8s services (internal):**
- Pattern: `<service>.<namespace>.svc.cluster.local`
- Examples: `postgres.database.svc.cluster.local`
- Managed by: CoreDNS (Kubernetes plugin)
- Authoritative: Yes (K8s owns this zone)

**K8s services (external):**
- Pattern: `<service>.apps.globular.internal`
- Examples: `myapp.apps.globular.internal`
- Managed by: ExternalDNS (registers with Globular DNS)
- Authoritative: Globular DNS (with records from K8s)

## ExternalDNS Integration (Optional)

### What It Does

ExternalDNS watches K8s Services and Ingresses, then registers them with Globular DNS, making K8s services discoverable outside the cluster.

```
┌─────────────────────────────────────────────────────────┐
│  K8s Service with LoadBalancer                          │
│                                                          │
│  apiVersion: v1                                         │
│  kind: Service                                          │
│  metadata:                                              │
│    name: myapp                                          │
│    annotations:                                         │
│      external-dns.alpha.kubernetes.io/hostname:         │
│        myapp.apps.globular.internal                     │
│  spec:                                                  │
│    type: LoadBalancer                                   │
│    loadBalancerIP: 192.168.1.100                       │
└─────────────────────────────────────────────────────────┘
                       ↓ watches
┌─────────────────────────────────────────────────────────┐
│  ExternalDNS Controller (in K8s)                        │
│  1. Detects new service with annotation                │
│  2. Extracts hostname: myapp.apps.globular.internal    │
│  3. Extracts IP: 192.168.1.100                         │
└─────────────────────────────────────────────────────────┘
                       ↓ registers
┌─────────────────────────────────────────────────────────┐
│  Globular DNS (substrate)                               │
│  Creates record:                                        │
│  myapp.apps.globular.internal → 192.168.1.100          │
└─────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  External Client                                        │
│  curl https://myapp.apps.globular.internal             │
│  → Resolves to 192.168.1.100                           │
│  → Connects to K8s LoadBalancer                        │
│  → Traffic routes to pods                              │
└─────────────────────────────────────────────────────────┘
```

### Use Cases

1. **Expose K8s services to external clients**
   - Web applications with LoadBalancer
   - APIs that need stable DNS names
   - Services accessed from outside K8s

2. **Unified DNS namespace**
   - `minio.globular.internal` (substrate service)
   - `myapp.apps.globular.internal` (K8s service)
   - Both resolve via substrate DNS

3. **Service migration**
   - Move service from substrate to K8s (or vice versa)
   - DNS record updates automatically
   - Clients unaffected (same DNS name)

## Failure Scenarios & Recovery

### Scenario 1: CoreDNS Crashes

**Traditional K8s:**
```
1. CoreDNS pods crash
2. All DNS in cluster stops working
3. Existing connections survive
4. New connections fail (can't resolve DNS)
5. Services become unreachable
6. Manual intervention: restart CoreDNS
7. Cluster degraded until DNS restored
```

**With Substrate:**
```
1. CoreDNS pods crash
2. K8s service DNS (cluster.local) stops
3. BUT substrate DNS keeps working
4. Pods can still reach:
   - minio.globular.internal ✓
   - etcd.globular.internal ✓
   - All infrastructure services ✓
5. K8s self-heals (restarts CoreDNS)
6. cluster.local DNS restored
7. Full functionality back
```

**Impact:** Substrate services remain accessible even during K8s DNS outage.

### Scenario 2: Globular DNS Crashes

**Impact:**
```
1. Globular DNS service stops
2. K8s service DNS (cluster.local) unaffected ✓
3. Substrate service DNS fails:
   - minio.globular.internal ✗
   - etcd.globular.internal ✗
4. Globular's convergent installer detects failure
5. Restarts DNS service (self-healing)
6. DNS restored within seconds
```

**Why this is better:**
- Globular DNS managed by systemd + convergent installer
- Automatic restart on failure
- Health monitoring by substrate layer
- No manual intervention needed

### Scenario 3: Split-Brain DNS

**Problem:** CoreDNS forwards to Globular DNS, but network partition occurs.

**Detection:**
```
CoreDNS Corefile health_check:
forward . 127.0.0.1:53 {
    health_check 2s  # Check every 2 seconds
}

If Globular DNS unreachable:
- CoreDNS logs error
- Queries for *.globular.internal fail
- But cluster.local queries still work
```

**Recovery:**
- Network partition heals
- Health check passes
- Forwarding resumes
- No manual intervention

### Scenario 4: DNS Cache Poisoning

**Protection:**
```
CoreDNS configuration:
cache 30  # 30 second TTL

Benefits:
- Short TTL limits poisoning window
- Cache invalidates quickly
- Changes propagate fast
- Balance between performance and freshness
```

## Performance Considerations

### Latency Analysis

**Query Path Lengths:**

| Query Type | Hops | Latency |
|------------|------|---------|
| K8s service (cluster.local) | Pod → CoreDNS → K8s API | ~5ms |
| Substrate service (globular.internal) | Pod → CoreDNS → Globular DNS | ~3ms |
| External domain | Pod → CoreDNS → Upstream | ~20ms |

**Caching Impact:**

```
First query: Full path (5ms)
Cached query: In-memory (<1ms)

Cache hit rate for typical workload: >90%
Effective average latency: ~1ms
```

### Throughput

**CoreDNS capacity:**
- 2 replicas (recommended)
- ~10,000 queries/sec per replica
- Total: ~20,000 queries/sec

**Globular DNS capacity:**
- Single systemd service
- ~50,000 queries/sec
- Substrate services queried less frequently
- Cache hit rate reduces load

### Optimization Strategies

1. **Increase CoreDNS replicas** for high query load
   ```yaml
   replicas: 3  # or more
   ```

2. **Tune cache TTL** based on change frequency
   ```
   cache 60  # Longer for stable infrastructure
   cache 10  # Shorter for dynamic services
   ```

3. **Use pod anti-affinity** to spread CoreDNS pods
   ```yaml
   podAntiAffinity:
     preferredDuringSchedulingIgnoredDuringExecution:
       topologyKey: kubernetes.io/hostname
   ```

4. **Monitor DNS metrics** (Prometheus)
   ```
   coredns_dns_request_duration_seconds
   coredns_dns_requests_total
   coredns_cache_hits_total
   ```

## Monitoring & Troubleshooting

### Key Metrics

**CoreDNS (K8s):**
```bash
# Prometheus metrics
curl http://10.96.0.10:9153/metrics

Key metrics:
- coredns_dns_requests_total
- coredns_dns_responses_total
- coredns_cache_hits_total
- coredns_cache_misses_total
- coredns_forward_requests_total
```

**Globular DNS (Substrate):**
```bash
# Service health
systemctl status globular-dns

# Query logs
journalctl -u globular-dns -f

# Zone info
curl -sk https://127.0.0.1:10006/api/dns/zones
```

### Troubleshooting Commands

**Test from pod:**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Test K8s service
nslookup kubernetes.default.svc.cluster.local
# Should return: 10.96.0.1

# Test substrate service
nslookup minio.globular.internal
# Should return: 127.0.0.1

# Test external
nslookup google.com
# Should return: public IP

# Check pod's DNS config
cat /etc/resolv.conf
# Should show: nameserver 10.96.0.10
```

**Check CoreDNS logs:**
```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

**Check Globular DNS logs:**
```bash
journalctl -u globular-dns --since "5 minutes ago"
```

**Verify forwarding:**
```bash
# From CoreDNS pod
kubectl exec -n kube-system -it <coredns-pod> -- sh

# Test forward to Globular DNS
dig @127.0.0.1 minio.globular.internal
```

## Migration from Traditional CoreDNS

### If You Have Existing K8s Cluster

**Step 1: Install Globular substrate** (on same hosts)
```bash
./install-day0.sh
# Globular DNS starts on port 53 (host network)
```

**Step 2: Update CoreDNS ConfigMap**
```bash
kubectl edit configmap coredns -n kube-system

# Add globular.internal zone:
globular.internal:53 {
    forward . 127.0.0.1:53
    cache 30
}
```

**Step 3: Restart CoreDNS**
```bash
kubectl rollout restart deployment/coredns -n kube-system
```

**Step 4: Test**
```bash
kubectl run test --image=busybox --rm -it -- \
  nslookup minio.globular.internal
```

**Step 5: Migrate services gradually**
- Move services from K8s to substrate (or vice versa)
- Update DNS records in Globular DNS
- No client changes needed (same DNS names)

## Summary: Why This Pattern Works

### The Key Insights

1. **DNS is infrastructure truth** (what exists, what is named)
   - Belongs in substrate layer
   - Must exist before workloads start
   - Must survive workload failures

2. **K8s needs DNS, not to manage DNS**
   - CoreDNS handles K8s-specific names (cluster.local)
   - Forwards infrastructure queries to substrate
   - Clean separation of concerns

3. **No circular dependencies**
   - Substrate DNS exists first
   - K8s CoreDNS adds workload layer on top
   - Both layers work independently
   - Failure in one doesn't break the other

### The End Result

```
Before (Traditional):
- DNS bootstrap problem (circular dependency)
- CoreDNS crash = cluster-wide DNS outage
- Manual intervention required
- Fragile, requires heroics

After (Substrate):
- No bootstrap problem (DNS exists first)
- CoreDNS crash = substrate DNS still works
- Self-healing (convergent installer)
- Boring, reliable DNS
```

**"DNS isn't scary anymore."**

And now you know why - because DNS lives where it belongs: in the substrate layer, as infrastructure truth that workloads consume.
