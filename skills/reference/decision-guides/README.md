# Decision Guides

Structured guides for common Ceph architecture and configuration decisions.
Each guide provides comparison tables, concrete recommendations, and exact commands.

## Index

| Guide | When to read it |
|---|---|
| [replication-vs-erasure.md](replication-vs-erasure.md) | Before creating a new pool; when weighing storage efficiency against performance; planning RBD, RGW, or CephFS data placement |
| [bluestore-tuning.md](bluestore-tuning.md) | Provisioning new OSDs; adding fast media (NVMe/SSD) to an HDD cluster; troubleshooting OSD memory or write-amplification issues |
| [network-topology.md](network-topology.md) | Initial cluster design; adding a second NIC or VLAN; separating client and replication traffic in a growing cluster |

## How to use these guides

1. Identify the decision you face from the index above.
2. Read the **context** section — it states constraints that cannot be changed after the fact.
3. Check the **recommendation by use case** section for your workload.
4. Run the exact commands shown; every command is ready to copy without placeholders.
5. If migrating an existing cluster, follow the **migration path** section in order.

<!-- source: Ceph official docs https://docs.ceph.com/en/latest/rados/ — reviewed 2026-03-29 -->
