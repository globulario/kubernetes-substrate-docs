# Kubernetes on Globular Substrate — Documentation Index

**Quick reference guide to all documentation and specs.**

## 📋 Overview

This repository contains **4,829 lines** of complete documentation for running Kubernetes on Globular's substrate layer, including:
- Package specs (ready to deploy)
- Architecture documentation (how it works)
- Complete guides (Day-0 through Day-N)
- Working examples (WordPress stack)

## 🚀 Quick Start

**Want to get started immediately?**

1. Read: [Complete Guide](./globular-kubernetes-complete-guide.md) (864 lines)
2. Run: Install script from the guide
3. Deploy: [WordPress Example](./examples/wordpress-stack.yaml)

**Time to working cluster:** ~2 hours

## 📚 Documentation

### Entry Points

| Document | Lines | Purpose |
|----------|-------|---------|
| [README.md](./README.md) | 435 | Project overview, philosophy, quick start |
| [Complete Guide](./globular-kubernetes-complete-guide.md) | 864 | Day-0 to Day-N operations, full walkthrough |

### Architecture Deep Dives

| Document | Lines | Covers |
|----------|-------|--------|
| [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md) | 497 | How K8s integrates with Globular infrastructure |
| [Storage Architecture](./architecture/kubernetes-storage-architecture.md) | 907 | Persistent volumes backed by substrate MinIO |

**Total architecture docs:** 1,404 lines

### Package Specs (Ready to Deploy)

| Spec | Lines | Installs |
|------|-------|----------|
| [Control Plane](./specs/kubernetes-control-plane-spec.yaml) | 589 | kube-apiserver, controller-manager, scheduler |
| [Worker Node](./specs/kubernetes-worker-node-spec.yaml) | 458 | kubelet, kube-proxy |
| [Storage Integration](./specs/kubernetes-storage-minio-spec.yaml) | 744 | CSI driver, StorageClasses (4 tiers) |

**Total specs:** 1,791 lines

### Examples

| Example | Lines | Demonstrates |
|---------|-------|--------------|
| [WordPress Stack](./examples/wordpress-stack.yaml) | 335 | Complete app with DB, storage tiers, backups |

## 🎯 Use Cases

**Find what you need:**

### "I want to understand the architecture"
→ Start with [README.md](./README.md)
→ Deep dive: [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md)

### "I want to understand storage"
→ Read: [Storage Architecture](./architecture/kubernetes-storage-architecture.md)
→ See: StorageClass tiers (standard, fast, replicated, archive)

### "I want to install K8s on Globular"
→ Follow: [Complete Guide](./globular-kubernetes-complete-guide.md)
→ Use: Package specs in [specs/](./specs/)

### "I want to see a working example"
→ Deploy: [WordPress Stack](./examples/wordpress-stack.yaml)
→ Examine: How workload layer consumes substrate

### "I want to understand the philosophy"
→ Read: [README - Philosophy section](./README.md#philosophy)
→ See: [Manifesto](../manifesto.txt) (referenced throughout)

## 🔑 Key Concepts

### The Two Layers

```
┌─────────────────────────────────────┐
│  WORKLOAD (Kubernetes)              │
│  • What runs, where, how it scales  │
└─────────────────────────────────────┘
              ↓ consumes
┌─────────────────────────────────────┐
│  SUBSTRATE (Globular)               │
│  • What exists, who is trusted      │
└─────────────────────────────────────┘
```

**Defined in:** [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md#the-two-layer-architecture)

### Storage Tiers

| Tier | Use Case | Policy | Quota |
|------|----------|--------|-------|
| **standard** | App data, configs | Delete | 10Gi |
| **fast** | Caches, CI/CD | Delete (reduced redundancy) | 100Gi |
| **replicated** | Databases, critical data | Retain (replicated) | 50Gi |
| **archive** | Backups, compliance | Retain (lifecycle) | 1Ti |

**Defined in:** [Storage Architecture](./architecture/kubernetes-storage-architecture.md#storage-tiers--use-cases)

### Integration Points

| Component | Substrate Provides | K8s Consumes |
|-----------|-------------------|--------------|
| **State Storage** | etcd | API server stores all K8s resources |
| **Certificates** | CA, auto-rotation | All K8s components use substrate certs |
| **DNS** | Port 53 authority | Cluster DNS, service discovery |
| **Storage** | MinIO buckets | Persistent volumes via CSI |
| **Discovery** | Service registry | API server registration, health |

**Defined in:** [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md#integration-points)

## 📖 Reading Paths

### For Operators

1. [README](./README.md) — Understand the problem being solved
2. [Complete Guide - Day-0](./globular-kubernetes-complete-guide.md#day-0-bootstrap-substrate) — Bootstrap substrate
3. [Complete Guide - Day-1](./globular-kubernetes-complete-guide.md#day-1-install-kubernetes) — Install K8s
4. [Complete Guide - Day-N](./globular-kubernetes-complete-guide.md#day-n-operations) — Daily operations

### For Architects

1. [README - Architecture](./README.md#architecture) — High-level overview
2. [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md) — Deep dive
3. [Storage Architecture](./architecture/kubernetes-storage-architecture.md) — Storage patterns
4. Package specs — Implementation details

### For Developers

1. [README - Example Workloads](./README.md#example-workloads) — How to use it
2. [WordPress Example](./examples/wordpress-stack.yaml) — Complete app
3. [Storage Architecture - Use Cases](./architecture/kubernetes-storage-architecture.md#storage-tiers--use-cases) — Which StorageClass to use

## 📊 Statistics

```
Total Lines:    4,829
Total Size:     188 KB

Documentation:  2,703 lines (56%)
Specs:          1,791 lines (37%)
Examples:         335 lines (7%)

Largest file:   Storage Architecture (907 lines)
Ready to deploy: 3 package specs
Working examples: 1 complete stack
```

## 🎓 Learning Path

### Beginner → Expert

**Level 1: Understand the Problem**
- Read: [README - What This Achieves](./README.md#what-this-achieves)
- Time: 15 minutes

**Level 2: See It Working**
- Deploy: [WordPress Example](./examples/wordpress-stack.yaml)
- Verify: Substrate integration
- Time: 30 minutes

**Level 3: Understand Architecture**
- Read: [Substrate Architecture](./architecture/kubernetes-substrate-architecture.md)
- Understand: Integration points
- Time: 1 hour

**Level 4: Deep Dive Storage**
- Read: [Storage Architecture](./architecture/kubernetes-storage-architecture.md)
- Understand: All 4 storage tiers
- Time: 1 hour

**Level 5: Deploy Your Own**
- Follow: [Complete Guide](./globular-kubernetes-complete-guide.md)
- Deploy: Full cluster
- Time: 2-3 hours

**Level 6: Operations**
- Practice: Upgrades, backups, DR
- Reference: [Complete Guide - Day-N](./globular-kubernetes-complete-guide.md#day-n-operations)
- Time: Ongoing

## 🔗 Cross-References

**This documentation references:**
- [Globular Manifesto](../manifesto.txt) — The philosophical foundation
- [Service Specs](../packages/specs/) — All Globular service packages
- [Day-0 Installer](../globular-installer/) — Substrate bootstrap

**This documentation is referenced by:**
- _(To be added as other projects link here)_

## ✅ Verification

**How to verify documentation is working:**

```bash
# 1. All files exist
cd /home/dave/Documents/github.com/globulario/kubernetes-substrate-docs
find . -type f -name "*.md" -o -name "*.yaml" | wc -l
# Expected: 8 files

# 2. Specs are valid YAML
yamllint specs/*.yaml

# 3. Examples deploy cleanly
kubectl apply -f examples/wordpress-stack.yaml --dry-run=client

# 4. Documentation builds (if using docs generator)
# (Add your doc build command here)
```

## 🚧 Future Additions

**Planned documentation:**
- [ ] DNS integration deep dive
- [ ] Authentication flow documentation
- [ ] Monitoring setup guide (Prometheus + Grafana)
- [ ] Logging integration (substrate logs + K8s logs)
- [ ] Service mesh integration (Istio on substrate)
- [ ] Multi-cluster federation
- [ ] Migration guides (from existing K8s)
- [ ] Performance benchmarks

## 📝 Contributing

**To add documentation:**

1. Place in appropriate directory:
   - `architecture/` — How things work
   - `specs/` — Deployable package specs
   - `examples/` — Working examples

2. Update this INDEX.md with:
   - File name and line count
   - Purpose and reading path
   - Cross-references

3. Link from README.md if it's a major addition

## 🎯 Quick Reference

**Most Important Files:**

```
kubernetes-substrate-docs/
├── README.md                    ← Start here
├── globular-kubernetes-complete-guide.md  ← Full walkthrough
├── architecture/
│   ├── kubernetes-substrate-architecture.md  ← How it works
│   └── kubernetes-storage-architecture.md    ← Storage deep dive
├── specs/
│   ├── kubernetes-control-plane-spec.yaml    ← Deploy K8s
│   ├── kubernetes-worker-node-spec.yaml      ← Add nodes
│   └── kubernetes-storage-minio-spec.yaml    ← Add storage
└── examples/
    └── wordpress-stack.yaml     ← Working example
```

**Most Important Concepts:**

1. **Substrate Layer** — Infrastructure that exists before workloads
2. **Workload Layer** — Kubernetes consuming substrate services
3. **No Circular Dependencies** — Storage, DNS, certs exist first
4. **Convergent Installation** — K8s as a self-healing package
5. **Automatic Recovery** — Substrate restores K8s, not vice versa

**Most Important Commands:**

```bash
# Install K8s control plane
globular-installer apply specs/kubernetes-control-plane-spec.yaml

# Add worker node
globular-installer apply specs/kubernetes-worker-node-spec.yaml

# Install storage
globular-installer apply specs/kubernetes-storage-minio-spec.yaml

# Deploy workload
kubectl apply -f examples/wordpress-stack.yaml

# Verify substrate integration
mc ls globular/ | grep k8s-pv
```

---

**Last Updated:** February 8, 2024
**Total Documentation:** 4,829 lines
**Status:** Complete and ready for use

**"Kubernetes isn't scary anymore."**
