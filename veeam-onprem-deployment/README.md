# Veeam On-Prem Enterprise Backup Architecture Playbook

---

## Executive Summary

This playbook defines a production-grade Veeam Backup & Replication architecture for on-prem VMware environments with NetApp storage.

The design prioritizes:

- Measurable RPO/RTO outcomes
- Restore validation discipline (not “backup-only” confidence)
- Predictable operational workflows
- Safe integration with storage snapshots and replication

Backups are considered valid only after restore testing confirms integrity.

---

## Scope & Assumptions

This architecture assumes:

- VMware vCenter-managed virtual infrastructure
- Mixed workload tiers (AD, SQL, app servers, file servers, VDI)
- NetApp-backed datastores (NFS and/or iSCSI)
- Requirement for offsite survivability (secondary site and/or immutable repository)
- Change control and monitoring are in place

---

## Architecture Overview

Core components:

- **Veeam Backup Server** (management and job orchestration)
- **Backup Proxy** (data mover; scale horizontally as needed)
- **Backup Repository** (primary storage target)
- **Offsite/DR Repository** (secondary copy; ideally immutable)
- **NetApp Snapshot Integration** (optional; used intentionally, not blindly)

---

## Architecture Decisions

### Repository Strategy

**Decision:** Use a primary repository for fast operational restores + a secondary copy target for disaster scenarios.

**Rationale:**
- Fast local restores reduce MTTR for common incidents
- Secondary copies protect against site loss and ransomware events
- Retention policies and immutability are easier to enforce at the copy tier

**Recommended structure:**
- Tier 1: Local repo for recent restore points
- Tier 2: Copy job to offsite or immutable storage

---

### Proxy Strategy

**Decision:** Scale proxies based on backup window requirements and datastore layout.

**Rationale:**
- Proxy bottlenecks are the most common reason backups miss RPO
- Horizontal scaling is safer than oversizing a single proxy
- Proxy placement should align with network and storage topology

**Operational note:** Proxies should be treated as disposable infrastructure (rebuildable with documented config).

---

### Transport Mode Selection (VMware)

**Preferred:** HotAdd (where supported and stable)

**Fallback options:**
- NBD (Network) when HotAdd is not viable
- Direct SAN when design and operational access permit it

**Guidance:**
- Standardize transport mode where possible
- Validate performance during pilot (throughput, job duration, proxy CPU)
- Avoid mixing modes without a reason (harder troubleshooting)

---

## Workload Classification & Job Design

### Tiering Model

| Tier | Workload Examples | Backup Frequency | Retention (example) | Notes |
|------|-------------------|------------------|---------------------|------|
| Tier 1 | AD, SQL, critical auth/services | Hourly incremental | 30 days | Requires app-aware processing |
| Tier 2 | Core app servers, file services | Daily incremental | 14 days | Restore time measured quarterly |
| Tier 3 | Dev/Test, non-critical | Daily | 7 days | Prefer cost-efficient retention |

Adjust based on business RPO/RTO requirements.

---

## Application-Aware Processing

Enable application-aware processing for:

- **Domain Controllers** (VSS-consistent)
- **SQL Servers** (transaction log integrity)
- **Exchange** (if present)
- Any workload where crash-consistent restores are unacceptable

Minimum expectation:
- Validate successful application-aware restore at least quarterly for Tier 1 systems.

---

## NetApp Snapshot + SnapMirror Integration

### Decision Framework

Storage snapshots are used for **speed** and **operational recovery**, but they do **not** replace Veeam backup chains.

Use storage snapshot integration when:

- You need very fast RPO for critical systems
- You want snapshot offload to reduce VMware impact
- You can align retention policies cleanly to prevent snapshot churn

Avoid snapshot overuse when:

- SnapMirror + Veeam schedules overlap and amplify snapshot creation
- Snapshot retention grows uncontrolled (capacity risk)
- Operational teams treat “snapshots” as “backups”

### Coordination Rules

- Align snapshot schedules to avoid peak workloads
- Avoid overlapping Veeam fulls with heavy SnapMirror replication windows
- Monitor write amplification indicators (latency / queue depth) during backup windows

---

## Restore Validation Strategy (Non-Negotiable)

### Restore Scenarios Tested

Minimum restore verification must include:

1. **Domain Controller full VM restore** (isolated network)
2. **SQL point-in-time recovery** (where applicable)
3. **File-level restore** (random sampling)
4. **Boot validation** in isolated network (NIC mapping and DNS sanity)
5. **Instant VM Recovery** test (performance + usability)

### Cadence

- Monthly: file-level restore sampling
- Quarterly: Tier 1 full restore simulation (isolated network)
- Semi-annually: DR restore test to alternate site target

---

## Monitoring & Alerting

Monitoring must cover:

- Job failures and repeated retries
- Backup duration drift (e.g., >20% increase week-over-week)
- Repository capacity thresholds
- Proxy CPU/memory saturation
- Network throughput saturation during backup windows

Alerting should trigger before missed RPO becomes a business incident.

---

## Failure Mode Analysis

| Scenario | Expected Behavior | Mitigation |
|----------|------------------|------------|
| Proxy overload | Jobs exceed backup window | Add proxies, rebalance jobs, validate transport mode |
| Repository capacity growth | Retention fails or jobs stop | Capacity alerts + retention policy review |
| CBT/VMware snapshot issues | Jobs slow or fail | Pilot validation + periodic health checks + controlled fulls |
| Ransomware event | Restore points at risk | Secondary immutable repo + access controls |
| Site failure | Primary restores unavailable | Copy jobs + DR restore testing |

---

## Operational Runbook (High-Level)

1. Confirm RPO/RTO requirements per workload tier
2. Pilot a representative set (Tier 1 + Tier 2)
3. Validate transport mode performance and proxy sizing
4. Establish job templates per tier (standardized settings)
5. Configure copy jobs to offsite/immutable target
6. Implement restore verification schedule and document results
7. Integrate monitoring into existing alerting workflow
8. Review quarterly and adjust as environment grows

---

## Design Principle

Backup architecture must prioritize:

- Restore confidence through testing
- Measurable recovery time (RTO)
- Predictable operational ownership
- Controlled retention and capacity growth
- Separation of “fast recovery” from “disaster survivability”
