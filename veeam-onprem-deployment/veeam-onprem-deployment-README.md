# Veeam On-Premises Enterprise Deployment

![Veeam On-Premises Enterprise Deployment](./diagrams/veeam-banner.png)

> *A backup job that has never been tested for restore is not a backup strategy — it is a liability.*

---

## Executive Summary

This document defines an enterprise Veeam Backup & Replication deployment architecture for on-premises environments built on VMware vSphere and NetApp storage.

The design goals are:

- Implement a measurable, testable backup strategy — not just a scheduled job
- Integrate VMware, NetApp snapshots, and Veeam into a coherent data protection stack
- Enforce restore validation discipline: backups are only considered valid after a restore test confirms integrity
- Align RPO and RTO objectives with operational reality, measured through regular testing

---

## Architecture Overview

### Core Components

| Component | Role |
|---|---|
| **Veeam Backup Server** | Central management, job scheduling, configuration database |
| **Proxy Servers** | Data movers — handle backup and restore I/O between datastores and repositories |
| **Backup Repositories** | Storage targets for backup files (.vbk, .vib, .vrb) |
| **Scale-Out Backup Repository (SOBR)** | Abstraction layer across multiple repositories with tiering |
| **NetApp SnapVault / SnapMirror** | Storage-layer snapshot coordination and replication |
| **Enterprise Manager** | Centralised reporting, self-service restore portal, licensing |

---

## Proxy Server Sizing and Placement

### Transport Mode Selection

| Transport Mode | When to Use | Notes |
|---|---|---|
| **Direct SAN** | Proxies with direct fibre channel or iSCSI access to datastores | Highest throughput, no ESXi overhead |
| **HotAdd (Virtual)** | Virtual proxy VMs on the same datastore | Good throughput, requires snapshot-capable storage |
| **NBD (Network Block Device)** | Fallback — no SAN access, no HotAdd | Lower throughput, suitable for small environments or off-hours only |

**Rule:** Deploy at least one physical proxy with Direct SAN transport for production backup jobs. Use HotAdd virtual proxies as secondary or for DR site replication. Never rely on NBD as the primary transport mode for large datastores.

### Proxy Sizing Guidelines

| Concurrent Tasks | vCPU | RAM | Network |
|---|---|---|---|
| 1–4 concurrent VMs | 4 vCPU | 8 GB | 1 GbE minimum |
| 5–10 concurrent VMs | 8 vCPU | 16 GB | 10 GbE recommended |
| 10+ concurrent VMs | 16+ vCPU | 32 GB | 10 GbE required |

Each proxy task processes one VM disk at a time. Size proxy count to allow your desired backup window to complete without queuing.

---

## Repository Architecture

### Repository Tiers

```
Primary Repository (Fast Disk — NAS or SAN)
  └── Holds recent restore points (last 7–14 days)
  └── Target for synthetic full jobs

Scale-Out Repository (SOBR)
  └── Extends across multiple extents
  └── Moves older restore points to capacity tier automatically

Capacity Tier (Object Storage or Tape)
  └── Long-term retention (30, 90, 365+ days)
  └── Immutable object storage recommended for ransomware resilience

Offsite / DR Copy
  └── Veeam Backup Copy Job to secondary site
  └── Separate physical failure domain from primary
```

### Repository Sizing Formula

```
Usable capacity = (Average VM size × VM count × Change rate per day × Retention days)
                  ÷ Deduplication ratio × Compression ratio

Recommended overhead: add 20% to calculated size for metadata and overhead
```

### 3-2-1 Rule Compliance Check

| Rule | Requirement | How Addressed |
|---|---|---|
| **3** copies of data | 3 restore points | Production backup + backup copy + offsite |
| **2** different media | 2 media types | Primary repo (disk) + offsite (separate disk or object) |
| **1** offsite copy | 1 off-site location | Backup Copy Job to DR site or cloud |

---

## VMware Integration

### Application-Aware Processing

Enable application-aware processing for all VMs running:

- Microsoft SQL Server
- Microsoft Exchange
- Microsoft SharePoint
- Active Directory Domain Controllers
- Any VSS-aware application

Application-aware processing invokes VSS inside the guest to quiesce the application before snapshot, producing a crash-consistent backup that can be restored to a clean application state.

| Setting | Recommended Value |
|---|---|
| Guest interaction proxy | Dedicated proxy — avoid using production DCs |
| Guest credentials | Dedicated service account per application type |
| Log truncation (SQL/Exchange) | Enable only if log backups are managed by Veeam |
| Indexing | Enable for file-level restore capability |

### VMware Tag-Based Policies

Use VMware tags to drive backup job inclusion rather than manual VM selection:

```
Tag: backup-policy=daily-7day     → Daily job, 7 restore points
Tag: backup-policy=daily-30day    → Daily job, 30 restore points
Tag: backup-policy=critical       → 4-hourly job, application-aware, 14 restore points
Tag: backup-policy=exclude        → Excluded from all backup jobs
```

Newly provisioned VMs inherit backup coverage automatically when tagged at deployment — no manual job editing required.

---

## NetApp Snapshot Coordination

### Storage Snapshot Integration

Veeam integrates with NetApp ONTAP to coordinate storage-layer snapshots with backup jobs:

1. Veeam triggers a NetApp snapshot via the Storage Integration API
2. The snapshot is mounted to the proxy server as a datastore
3. Veeam reads VM data directly from the snapshot — zero I/O impact on production storage
4. Snapshot is released after backup job completes

This approach eliminates VMware CBT (Changed Block Tracking) snapshot stun on production VMs during backup windows, which is the leading cause of backup-induced performance degradation.

### SnapVault / SnapMirror Alignment

| Veeam Job Type | NetApp Coordination |
|---|---|
| Primary backup | Storage snapshot + Veeam job |
| Backup Copy to DR | SnapMirror replication to DR volume + Veeam copy job |
| Long-term retention | SnapVault to secondary storage |

Align Veeam job schedules with NetApp snapshot schedules to avoid competing I/O.

---

## RPO / RTO Objective Framework

### Defining Objectives by Workload Tier

| Tier | Example Workloads | RPO Target | RTO Target |
|---|---|---|---|
| **Tier 1 — Critical** | Domain controllers, primary SQL, core apps | 4 hours | 1 hour |
| **Tier 2 — Important** | Secondary app servers, file servers | 24 hours | 4 hours |
| **Tier 3 — Standard** | Dev/test, non-critical workloads | 48 hours | 8 hours |

These are targets — not guarantees. They must be validated through regular restore testing. See [Disaster Recovery & Failure Testing](../disaster-recovery-testing) for test procedures.

---

## Restore Validation Methodology

Backups are only considered valid after restore testing confirms integrity. The following restore scenarios are tested on a defined schedule:

### Restore Types and Test Schedule

| Restore Type | Frequency | Method |
|---|---|---|
| Instant VM Recovery | Monthly | Restore to isolated test network, validate application services |
| Full VM Restore | Quarterly | Full restore to production or test host, validate boot and services |
| Guest File Restore | Monthly | Restore individual files, validate integrity |
| Application Item Restore | Quarterly | SQL database restore, Exchange mailbox restore |
| Entire Datastore Restore | Semi-annual | Full datastore failover via NetApp integration |

### Restore Test Record Template

```
Test ID       : VBR-[YYYY]-[NNN]
Restore Type  : [Instant VM / Full VM / File / Application Item]
VM Restored   : [VM name]
Backup Date   : [Date of restore point used]
Test Date     : [Date of test]

Restore Start : [HH:MM]
Restore End   : [HH:MM]
Elapsed Time  : [Minutes]

Application Validated : [Yes / No — services responding post-restore]
Data Integrity        : [No corruption errors]
RTO Achieved          : [Minutes]
RTO Target            : [Minutes]
Status                : PASS / FAIL

Notes         : [Deviations, errors, observations]
Validated By  : [Engineer name]
```

---

## Backup Job Configuration Standards

### Job Settings Checklist

- [ ] Application-aware processing enabled for all VSS-capable VMs
- [ ] Guest file indexing enabled for file-level restore capability
- [ ] Changed Block Tracking (CBT) enabled on all VMware VMs
- [ ] Active full backup scheduled weekly (not relying solely on increments)
- [ ] Encryption enabled on all jobs (especially backup copy and offsite)
- [ ] Email notifications configured to alert on job failures within 15 minutes
- [ ] Retry on failure: 3 retries, 10-minute wait between attempts
- [ ] Backup window defined and enforced — jobs must not run during peak hours

### Retention Policy Standards

| Restore Point Type | Minimum Retention |
|---|---|
| Daily incremental | 14 restore points |
| Weekly synthetic full | 4 restore points (4 weeks) |
| Monthly full | 12 restore points (12 months) |
| Annual full (compliance) | 7 years (adjust to regulatory requirement) |

---

## Monitoring and Alerting

### Metrics to Monitor

| Metric | Alert Threshold |
|---|---|
| Job success rate | < 100% — any failure triggers alert |
| Backup window overrun | Job running >20% past scheduled end |
| Repository capacity | > 80% full |
| Restore point age | Oldest restore point > 2× RPO target |
| Proxy task queue | Queue depth > 10 for > 30 minutes |
| NetApp snapshot accumulation | > 50 snapshots on any volume |

Veeam ONE or a SIEM integration should monitor these metrics and page the on-call engineer for any critical threshold breach.

---

## Architecture Context

The backup layer protects the full infrastructure stack:

| Layer | Documentation |
|---|---|
| Identity | [Active Directory Multi-Site](../active-directory-multisite) |
| Configuration management | [Puppet Enterprise Migration](../puppet-enterprise-migration) |
| Platform baseline | [Windows Infrastructure Stabilization](../windows-infrastructure-stabilization) |
| DR validation | [Disaster Recovery & Failure Testing](../disaster-recovery-testing) |

---

## Design Principles

1. **A backup is not valid until a restore confirms it** — job success is not restore success
2. **Transport mode selection drives performance** — NBD is a fallback, not a design choice
3. **Application consistency requires VSS** — crash-consistent is not sufficient for SQL or Exchange
4. **Align with storage-layer snapshots** — reduce VMware stun, improve backup window reliability
5. **3-2-1 is the minimum** — offsite and immutable copies are non-negotiable for ransomware resilience

---

*Part of the [Enterprise Infrastructure Architecture](../README.md) portfolio.*
