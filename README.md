# Platform

This repository manages Kubernetes platform infrastructure and applications using GitOps principles with Flux CD.

## Repository Structure

```
.
├── apps/               # Application-layer components
├── infrastructure/     # Infrastructure-layer components
└── clusters/          # Cluster-specific configurations
```

## Apps vs Infrastructure

### Infrastructure (`infrastructure/`)

The `infrastructure/` directory contains **foundational cluster components** that must be:

1. **Bootstrapped with Terraform** - These components are deployed to the cluster during initial provisioning via Terraform
2. **Adopted by Flux** - After Terraform creates them, Flux takes over management for ongoing updates and configuration drift detection

**Current infrastructure components:**
- **Cilium** - CNI (Container Network Interface) plugin providing networking and network policies

**Why infrastructure is separate:**
- These components are prerequisites for cluster operation
- They must exist before Flux can manage the cluster
- Terraform handles the initial deployment because the cluster networking/core services need to be functional first
- Flux adoption ensures GitOps principles apply after bootstrap

### Apps (`apps/`)

The `apps/` directory contains **all other platform components** that are:

1. **Managed entirely by Flux** - No Terraform involvement
2. **Deployed via GitOps** - Changes are applied automatically when this repository is updated

**Examples of apps:**
- cert-manager - Certificate management
- external-dns - DNS record management
- external-secrets - Secret synchronization
- envoy-gateway - API gateway
- argo-cd - GitOps continuous delivery
- tailscale - VPN connectivity
- grafana - Observability dashboards
- and more...

**Why these are apps:**
- They run on top of the established infrastructure
- They don't require pre-existing cluster state
- They can be deployed after Flux is operational

## Deployment Flow

```
1. Terraform provisions cluster
   └─> Deploys infrastructure/ components (e.g., Cilium)
       └─> Cluster networking is operational

2. Terraform installs Flux
   └─> Flux adopts infrastructure/ components
   └─> Flux deploys apps/ components

3. All changes managed via GitOps
   └─> infrastructure/ managed by Flux (post-adoption)
   └─> apps/ managed by Flux (from inception)
```

## Key Distinction

| Aspect             | Infrastructure        | Apps                                 |
| ------------------ | --------------------- | ------------------------------------ |
| Initial deployment | Terraform             | Flux                                 |
| Ongoing management | Flux (adopted)        | Flux                                 |
| Purpose            | Cluster prerequisites | Platform capabilities                |
| Examples           | CNI, CSI drivers      | Applications, operators, controllers |
| Required for       | Cluster to function   | Platform features                    |
