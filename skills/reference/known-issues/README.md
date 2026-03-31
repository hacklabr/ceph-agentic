# Known Issues — Reference

This directory documents real, confirmed known issues extracted from official Ceph release notes and tracker entries. Each issue is written in a consistent format so that operators can quickly locate symptoms, understand the root cause, apply a workaround, and determine whether they are affected.

## Version Index

| File | Ceph Series | Latest Covered Release | Source |
|------|-------------|------------------------|--------|
| [reef-18.2.md](./reef-18.2.md) | Reef (18.2.x) | v18.2.8 | [Official release notes](https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst) |
| [squid-19.2.md](./squid-19.2.md) | Squid (19.2.x) | v19.2.3 | [Official release notes](https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst) |

## Issue Entry Format

Every issue entry follows this template:

```
### <Short title — component: one-line description>

| Field | Value |
|-------|-------|
| **Affected versions** | e.g. 18.2.0 – 18.2.3 (fixed in 18.2.4) |
| **Component** | e.g. RGW, CephFS/MDS, RADOS, RBD, BlueStore |
| **Severity** | Critical / High / Medium / Low |
| **Status** | Fixed in vX.Y.Z / Open / Mitigated |
| **Tracker** | https://tracker.ceph.com/issues/NNNNN |
| **PR** | https://github.com/ceph/ceph/pull/NNNNN |

**Symptom**
What the operator observes — error messages, log output, degraded health states.

**Cause**
Root cause as described in the release notes or tracker.

**Workaround**
Step-by-step commands or config changes that mitigate the issue before upgrading.

**Fix**
Which release contains the fix and the relevant PR or commit reference.
```

## Usage Notes

- Issues are ordered newest-first within each file (highest patch version at the top).
- The "Affected versions" range is inclusive: *fixed in* means the issue does not appear in that version or later.
- Workaround commands assume a standard cephadm-managed cluster. Adjust for non-cephadm deployments.
- Where a tracker link is available, it supersedes this document for the latest status.

---
*Sources: official Ceph release notes at https://github.com/ceph/ceph/tree/main/doc/releases and https://tracker.ceph.com*
