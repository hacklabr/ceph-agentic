# Ceph — Agentic Stack

An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that
teaches AI agents how to deploy and operate production Ceph storage clusters
using cephadm on bare metal or Rook on Kubernetes.

## What This Stack Covers

- **Foundation** — Ceph architecture, hardware planning, host preparation, Kubernetes prerequisites
- **Deployment** — cephadm bootstrap, Rook deployment, network configuration, RBD/CephFS/RGW services
- **Operations** — health checks, scaling, upgrades, backups, pool management, TLS, Rook operations
- **Diagnostics** — symptom-based troubleshooting, performance analysis, Rook troubleshooting
- **Reference** — known issues, compatibility matrices, decision guides

**Target versions:** Ceph Reef (18.2.x), Squid (19.2.x), and Tentacle (20.2.x)
**Deployment methods:** cephadm (container-based) and Rook on Kubernetes
**Infrastructure:** Bare metal and Kubernetes clusters

## Usage

    agentic-stacks init agentic-stacks/ceph my-ceph-cluster
    cd my-ceph-cluster

Then ask your agent to help plan, deploy, or operate your Ceph cluster.

## Composability

This stack focuses on Ceph storage. It pairs well with:
- A Kubernetes stack (for Rook deployment, RBD/CephFS CSI integration, and workload orchestration)
- A hardware/IPMI stack (for bare metal provisioning)

## Authoring

See the [authoring guide](https://github.com/agentic-stacks/agentic-stacks/blob/main/docs/guides/authoring-a-stack.md).
