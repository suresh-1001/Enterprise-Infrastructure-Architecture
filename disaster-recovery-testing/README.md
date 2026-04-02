# Disaster Recovery & Failure Testing Playbook

![Disaster Recovery & Failure Testing](./diagrams/disaster-recovery-banner.webp)

> *Disaster recovery must be tested, measured, and documented — not assumed.*

---

## Executive Summary

This playbook defines a structured **Disaster Recovery (DR) validation methodology** for enterprise on-premises infrastructure environments built on Active Directory, VMware, NetApp, and Veeam.

The objective is to:

- Validate recoverability of identity, compute, and storage layers independently
- Measure actual Recovery Time Objectives (RTO) — not estimates
- Confirm backup and replication systems function correctly under real failure conditions
- Eliminate silent dependency assumptions before they become production incidents

---

## DR Design Philosophy

Enterprise DR strategy separates four recovery tiers, each requiring independent validation:

| Tier | Description | Validated By |
|---|---|---|
| **Operational Recovery** | Fast restores from primary site without failover | Veeam restore drills |
| **Disaster Survivability** | Secondary site failover for full workload continuity | NetApp + VMware HA tests |
| **Identity Continuity** | Active Directory resilience during WAN failure | AD RODC simulation |
| **Configuration Continuity** | Puppet state consistency on freshly provisioned nodes | Puppet rebuild validation |

A successful backup job does not confirm restore success. Each tier must be proven independently.

---

## Failure Scenarios

### 🌐 Scenario 1 — Active Directory WAN Failure

**Objective:** Confirm branch sites authenticate locally when the WAN link to the primary DC is severed.

**Test Steps:**
1. Disconnect or firewall the WAN path from the branch site to the primary AD site
2. Attempt workstation and server authentication using domain credentials
3. Validate RODC at branch processes Kerberos ticket requests
4. Confirm Group Policy application continues from cached policy
5. Verify DNS resolution operates from the local RODC

**Pass Criteria:**

| Check | Target |
|---|---|
| User authentication during WAN outage | Under 30 seconds |
| RODC responds to Kerberos requests | Confirmed |
| No fallback to NTLM required | Confirmed |
| DNS resolution functional from local cache | Confirmed |

**Tools:** `nltest /dsgetdc`, `repadmin /replsummary`, `dcdiag`, Wireshark (Kerberos trace)

---

### 💾 Scenario 2 — Veeam VM Restore Validation

**Objective:** Confirm backup jobs produce restorable artifacts — not just successful job completion logs.

**Test Steps:**
1. Select representative VMs: domain controller, application server, file server
2. Execute Instant VM Recovery to an isolated test network segment
3. Boot each VM and validate application-layer services respond
4. Measure elapsed time from recovery initiation to confirmed operational state
5. Document actual RTO achieved vs. target

**Pass Criteria:**

| Check | Target |
|---|---|
| VM boots successfully from backup artifact | Confirmed |
| Application services respond post-boot | Confirmed |
| No corruption errors during restore | Confirmed |
| RTO target met | Document actual vs. target |

**Tools:** Veeam Backup & Replication Console, isolated VLAN or test portgroup, application health checks

---

### 🖥️ Scenario 3 — VMware HA Host Failure Simulation

**Objective:** Validate VMware High Availability restarts critical VMs on surviving hosts within defined tolerance.

**Test Steps:**
1. Document all critical VMs and their current host placement
2. Confirm admission control policy has sufficient reserved capacity
3. Simulate host failure (controlled power-off of ESXi host)
4. Monitor HA restart events in vCenter
5. Validate critical VMs are accessible and datastores remain mounted

**Pass Criteria:**

| Check | Target |
|---|---|
| Critical VMs restart on surviving hosts | Within 5 minutes |
| No datastore isolation events | Confirmed |
| vCenter alert fires on host loss | Within 60 seconds |
| Admission control confirms capacity pre-test | Confirmed |

**Tools:** vSphere Client, vCenter event log, ESXi host power simulation

---

### 🗄️ Scenario 4 — NetApp SnapMirror Failover

**Objective:** Confirm secondary storage can be promoted and workloads resumed when primary storage fails.

**Test Steps:**
1. Break SnapMirror relationship and promote secondary volume to read-write
2. Remount promoted datastores on ESXi hosts
3. Power on VMs from secondary datastore
4. Confirm Veeam backup jobs resume successfully against secondary storage
5. Document and test the full failback sequence

**Pass Criteria:**

| Check | Target |
|---|---|
| Secondary volume promotes to read-write | Within 10 minutes |
| VMs power on from secondary datastore | No corruption errors |
| Veeam jobs resume against secondary storage | Within RPO window |
| Failback sequence tested and documented | Confirmed |

**Tools:** NetApp ONTAP System Manager, `snapmirror` CLI, ESXi storage remount procedure

---

### ⚙️ Scenario 5 — Puppet Node Rebuild Validation

**Objective:** Confirm a freshly provisioned node reaches its correct defined state within one catalog run.

**Test Steps:**
1. Provision a clean VM with base OS only (no manual configuration)
2. Point the node at the Puppet Enterprise master and sign the certificate
3. Execute first catalog run: `puppet agent -t --verbose`
4. Validate all resources applied correctly: packages, services, files, configurations
5. Confirm no manual intervention was required post-run

**Pass Criteria:**

| Check | Target |
|---|---|
| Node classifies correctly in Puppet Console | Confirmed |
| Catalog applies without errors on first run | Confirmed |
| All required services running post-run | Confirmed |
| No manual post-run intervention required | Confirmed |

**Tools:** Puppet Enterprise Console, `puppet agent`, `puppet resource`, node classification audit

---

## RTO / RPO Measurement Template

All tests produce a documented record. Fill one in for every test run.

```
Test ID       : DR-[YYYY]-[NNN]
Scenario      : [Scenario name]
Date          : [YYYY-MM-DD]
Test Window   : [Start time] → [End time]

Failure Point : [What was simulated or triggered]
Detection Time: [Time from failure to first alert]
Response Time : [Time from alert to recovery action started]
Recovery Time : [Time from action to confirmed operational state]

Actual RTO    : [X minutes]
Target RTO    : [X minutes]
Variance      : [+/- X minutes]
Status        : PASS / FAIL / EXCEPTION

Deviations    : [Any unexpected behaviour, workarounds used]
Validated By  : [Engineer name]
Reviewed By   : [Lead or manager]
```

---

## Pre-Test Checklist

- [ ] Change request approved and communicated to stakeholders
- [ ] Test network or isolated environment confirmed ready
- [ ] Monitoring suppression window opened (prevents alert storm)
- [ ] Rollback procedure documented and reviewed before test starts
- [ ] Secondary engineer available for escalation
- [ ] Most recent backup job confirmed successful before test window
- [ ] vCenter, NetApp, and Puppet Console access verified

---

## Post-Test Documentation Requirements

- [ ] Actual RTO recorded and compared against target
- [ ] Pass / Fail determination documented with justification
- [ ] Exceptions and unexpected behaviours captured
- [ ] Root cause analysis completed for any failures
- [ ] Remediation tasks raised as tracked items
- [ ] Results shared with infrastructure leadership

---

## DR Test Schedule

| Scenario | Frequency | Last Tested | Next Scheduled |
|---|---|---|---|
| AD WAN Failure Simulation | Quarterly | — | — |
| Veeam VM Restore Validation | Monthly | — | — |
| VMware HA Host Failure | Semi-Annual | — | — |
| NetApp SnapMirror Failover | Semi-Annual | — | — |
| Puppet Node Rebuild | Quarterly | — | — |

---

## Architecture Context

This playbook validates the recovery capability of the full infrastructure stack:

| Layer | Documentation |
|---|---|
| Identity | [Active Directory Multi-Site](../active-directory-multisite) |
| Backup | [Veeam On-Prem Deployment](../veeam-onprem-deployment) |
| Configuration | [Puppet Enterprise Migration](../puppet-enterprise-migration) |
| Platform baseline | [Windows Infrastructure Stabilization](../windows-infrastructure-stabilization) |

---

## Design Principles

1. **Restores are the only proof** — a backup job log is not a restore confirmation
2. **Measure actual RTO** — estimates are not operational data
3. **Test under realistic conditions** — controlled lab results diverge from production
4. **Document failures as thoroughly as successes** — failures drive the remediation backlog
5. **Each tier is validated independently** — combined failure modes compound differently

---

*Part of the [Enterprise Infrastructure Architecture](../README.md) portfolio.*
