# Ceph — Agentic Stack

An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that
teaches AI agents how to deploy and operate production Ceph storage clusters
using cephadm on bare metal.

## What This Stack Covers

- **Foundation** — Ceph architecture, hardware planning, host preparation
- **Deployment** — cephadm bootstrap, network configuration, RBD/CephFS/RGW services
- **Operations** — health checks, scaling, upgrades, backups, pool management, TLS
- **Diagnostics** — symptom-based troubleshooting, performance analysis
- **Reference** — known issues, compatibility matrices, decision guides

**Target versions:** Ceph Reef (18.2.x) and Squid (19.2.x)
**Deployment method:** cephadm (container-based)
**Infrastructure:** Bare metal

## Usage

    agentic-stacks init agentic-stacks/ceph my-ceph-cluster
    cd my-ceph-cluster

Then ask your agent to help plan, deploy, or operate your Ceph cluster.

## Composability

This stack focuses on Ceph storage. It pairs well with:
- A Kubernetes stack (for RBD/CephFS CSI integration)
- A hardware/IPMI stack (for bare metal provisioning)

## Authoring

See the [authoring guide](https://github.com/agentic-stacks/agentic-stacks/blob/main/docs/guides/authoring-a-stack.md).
