# Active Directory Multi-Site Architecture

![Active Directory Multi-Site Architecture](./diagrams/active-directory-banner.png)

> *Identity infrastructure must survive WAN failure. Design it to operate independently at every site.*

---

## Executive Summary

This document defines an enterprise Active Directory architecture for organisations operating across multiple physical sites connected by WAN links of varying reliability.

The design goals are:

- Consolidate fragmented identity domains into a single, governable forest
- Preserve site-level authentication autonomy during WAN outages
- Support badge-based and workstation authentication without dependency on a remote DC
- Define a clear placement strategy for Writable DCs vs. Read-Only DCs (RODCs)
- Provide a sequenced migration playbook for domain consolidation

This is not a lab design. It represents production-grade thinking for regulated environments where WAN failure is a realistic operational condition — not an edge case.

---

## Forest and Domain Strategy

### Single Forest, Multiple Sites

The recommended model for most enterprise environments is a **single Active Directory forest** with domain controllers distributed across all sites.

| Topology Component | Decision |
|---|---|
| Number of forests | 1 (unless regulatory boundaries require separation) |
| Domain structure | Single domain preferred; child domains only if administrative autonomy is required |
| Forest functional level | Windows Server 2016 minimum |
| Domain functional level | Windows Server 2016 minimum |
| Global Catalog placement | At least one GC per site |
| DNS integration | AD-integrated DNS zones on all DCs |

A single domain with properly scoped OUs and delegated administration is more maintainable and auditable than a multi-domain forest. Additional domains are justified only by genuine administrative, regulatory, or security boundaries.

---

## DC Placement Strategy

### Writable DCs vs. RODCs

| Factor | Writable DC | Read-Only DC (RODC) |
|---|---|---|
| Holds full copy of NTDS.dit | Yes | Yes (filtered) |
| Accepts write operations | Yes | No |
| Suitable for untrusted physical locations | No | Yes |
| Can process Kerberos locally | Yes | Yes (with password caching) |
| Requires WAN to authenticate | No | Only for uncached accounts |
| Risk if stolen or compromised | High | Low (limited credential exposure) |

**Rule:** Deploy writable DCs only at physically secured, primary datacentre sites. Deploy RODCs at branch offices and remote sites where physical security cannot be guaranteed to the same standard.

### Minimum DC Count per Site

| Site Type | Minimum DCs | Recommended |
|---|---|---|
| Primary datacentre | 2 writable DCs | 2+ writable, 1 GC per DC |
| Secondary datacentre | 2 writable DCs | 2+ writable |
| Branch office (trusted) | 1 RODC | 1 RODC + monitoring |
| Remote site (untrusted) | 1 RODC | 1 RODC, careful password cache policy |

Never run a site with a single DC. A failed DC at a site with no redundancy means users are dependent on WAN authentication.

---

## AD Sites & Services Topology

### Site Design Principles

- One AD Site per physical network subnet or geographic location
- Site links reflect actual WAN capacity and latency — do not use defaults
- Site link costs should be configured to reflect relative bandwidth and reliability
- Schedule replication during off-peak hours for large or slow WAN links
- Enable Change Notification on high-bandwidth links for near-real-time replication

### Site Link Configuration

| Parameter | Recommended Value |
|---|---|
| Default site link cost | 100 |
| Fast WAN link (100 Mbps+) | 50–75 |
| Slow WAN link (<10 Mbps) | 150–200 |
| Unreliable or metered link | 250+ |
| Replication interval | 15 minutes minimum (fast links), 60–180 minutes (slow links) |
| Change Notification | Enabled on links ≥10 Mbps |

### FSMO Role Placement

| FSMO Role | Recommended Placement |
|---|---|
| Schema Master | Primary DC, primary site |
| Domain Naming Master | Primary DC, primary site |
| PDC Emulator | Most reliable DC, primary site — this is the most operationally critical role |
| RID Master | Same site as PDC Emulator |
| Infrastructure Master | DC that is NOT a Global Catalog (or any DC if all DCs are GCs) |

The PDC Emulator processes password changes, handles account lockouts, and is the authoritative time source for the domain. Place it on the most reliable, best-monitored DC.

---

## WAN Outage Survivability Model

When the WAN link between a branch site and the primary datacentre fails, users at that branch must be able to:

1. Authenticate to workstations and servers using domain credentials
2. Apply Group Policy from local cache
3. Resolve internal DNS names from the local RODC
4. Access locally hosted resources without interruption

**Authentication flow during WAN outage (RODC site):**

```
User logs in
  → Workstation contacts local RODC
    → RODC checks local password cache
      → Cached: authentication succeeds locally (no WAN needed)
      → Not cached: authentication fails (WAN required — this is the gap to manage)
```

**Password caching policy decisions:**

| Account Type | Cache on RODC? | Rationale |
|---|---|---|
| Regular user accounts (branch staff) | Yes | Enables offline authentication |
| Privileged accounts (Domain Admins) | No | Limits credential exposure if RODC is compromised |
| Service accounts (local services) | Yes, selectively | Only cache accounts whose services run locally |
| Administrator accounts | No | Never cache on RODC |

Managing the password replication policy (PRP) on each RODC is the single most important operational task for ensuring WAN outage survivability.

---

## Replication Health Monitoring Framework

### Key Commands

```powershell
# Check replication summary across all DCs
repadmin /replsummary

# Show replication failures
repadmin /showrepl * /csv > repl-report.csv

# Check DC health at a specific DC
dcdiag /s:DC01 /test:replications /test:connectivity /test:dns

# Verify SYSVOL replication (DFS-R)
dfsrdiag ReplicationState /member:DC01

# Test DC locator from a workstation
nltest /dsgetdc:domain.local /site:BranchSiteName /force
```

### Replication Health Thresholds

| Metric | Acceptable | Investigate |
|---|---|---|
| Replication lag | < 15 minutes (intrasite) | > 60 minutes |
| Replication failures | 0 | Any persistent failure |
| USN rollback events | 0 | Any occurrence |
| SYSVOL sync state | Synchronized | Any mismatch |
| DNS SRV record registration | All DCs registered | Any missing record |

---

## Failure Mode Analysis

| Failure Scenario | Impact | Mitigation |
|---|---|---|
| WAN link failure to branch | Branch users can't authenticate (if no RODC) | Deploy RODC with password caching |
| Single DC failure at primary site | Degraded but functional (2nd DC handles requests) | Maintain minimum 2 DCs per critical site |
| PDC Emulator failure | Password changes and lockout processing stop | Monitor and seize role promptly |
| RODC compromise | Limited credentials exposed | RODC password caching policy limits blast radius |
| SYSVOL unsynchronised | Group Policy application inconsistent | Monitor DFS-R health, resolve promptly |
| AD database corruption | Partial or full domain outage | Authoritative restore from Veeam backup |

---

## Domain Consolidation Migration Sequence

When consolidating multiple existing domains into a single forest:

**Phase 1 — Discovery and Baseline**
- Inventory all existing domains, DCs, trust relationships
- Identify FSMO role holders in each domain
- Audit users, groups, computers, and service accounts
- Document all applications with AD dependencies (LDAP, Kerberos, NTLM)

**Phase 2 — Target Forest Design**
- Define OU structure, delegation model, and GPO architecture
- Configure AD Sites & Services for all physical locations
- Deploy new writable DCs and RODCs per the placement strategy above
- Establish DNS delegation and conditional forwarding

**Phase 3 — Trust Establishment**
- Create forest trusts between source and target domains
- Validate cross-forest authentication and resource access
- Test application compatibility across trust boundaries

**Phase 4 — Migration Execution**
- Migrate users and groups using ADMT (Active Directory Migration Tool)
- Re-join workstations and servers to the target domain in controlled waves
- Migrate service accounts last — these carry the most application dependency risk

**Phase 5 — Decommission**
- Validate no remaining traffic to source domain DCs
- Lower source domain functional level to allow FSMO seizure
- Decommission source domain DCs in reverse order of dependency

---

## Architecture Context

This identity layer underpins the full infrastructure stack:

| Layer | Documentation |
|---|---|
| Backup & recovery | [Veeam On-Prem Deployment](../veeam-onprem-deployment) |
| Configuration management | [Puppet Enterprise Migration](../puppet-enterprise-migration) |
| Platform baseline | [Windows Infrastructure Stabilization](../windows-infrastructure-stabilization) |
| DR validation | [Disaster Recovery & Failure Testing](../disaster-recovery-testing) |

---

## Design Principles

1. **Every site must be able to authenticate locally** — WAN dependency is a single point of failure
2. **RODC placement reduces blast radius** — a compromised branch DC should not compromise the forest
3. **Monitor replication continuously** — silent replication failures accumulate into major incidents
4. **Fewer domains, more OUs** — complexity in domain structure is rarely justified
5. **FSMO roles are not redundant** — they require active monitoring and documented seizure procedures

---

*Part of the [Enterprise Infrastructure Architecture](../README.md) portfolio.*
