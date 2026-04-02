# Windows Infrastructure Stabilization Plan

![Windows Infrastructure Stabilization](./diagrams/windows-stabilization-banner.webp)

> *You cannot govern what you have not measured. Stabilization starts with an honest baseline.*

---

## Executive Summary

This document defines a structured stabilization plan for Windows-based enterprise infrastructure — covering Active Directory health, DNS integrity, Group Policy governance, patch compliance, and monitoring integration.

This plan is designed for environments that have been inherited, grown organically, or suffered from undocumented manual changes over time. The objective is to move from an unknown, potentially unstable state to a **documented, measurable, and governable baseline** within a defined timeframe.

The framework is structured around a **30-60-90 day operational onboarding model** — each phase builds on the previous, moving from visibility to remediation to governance.

---

## 30-60-90 Day Framework

### Day 0–30 — Visibility

The first month is about gaining an honest picture of the environment. No changes are made to production until the assessment is complete.

**Goals:**
- Know exactly what is running and in what state
- Identify all critical gaps and risks
- Establish a remediation priority list

### Day 31–60 — Remediation

Address the gaps identified in month one, in priority order. Focus on stability before optimisation.

**Goals:**
- Resolve all critical AD health issues
- Establish DNS integrity
- Validate and document Group Policy
- Implement patch compliance baseline

### Day 61–90 — Governance

Build the operational processes that prevent the environment from returning to a degraded state.

**Goals:**
- Monitoring and alerting fully operational
- Change management process in place
- Documentation current and accurate
- Ongoing health checks automated

---

## Active Directory Health Assessment

### Core Diagnostic Commands

Run these against every domain controller in the environment:

```powershell
# Full DC health check
dcdiag /s:DC01 /v /test:all > dcdiag-DC01.txt

# Replication status across all DCs
repadmin /replsummary
repadmin /showrepl * /csv > replication-report.csv

# SYSVOL and DFS-R replication state
dfsrdiag ReplicationState /member:DC01
dfsrdiag BacklogStatus /rgName:"Domain System Volume" /rfName:SYSVOL /sendingmember:DC01 /receivingmember:DC02

# FSMO role holders
netdom query fsmo

# Domain and forest functional levels
(Get-ADDomain).DomainMode
(Get-ADForest).ForestMode

# Stale computer objects (not seen in 90+ days)
Get-ADComputer -Filter * -Properties LastLogonDate |
  Where-Object { $_.LastLogonDate -lt (Get-Date).AddDays(-90) } |
  Select-Object Name, LastLogonDate | Sort-Object LastLogonDate
```

### AD Health Checklist

| Check | Acceptable State | Action if Failed |
|---|---|---|
| `dcdiag /test:all` passes | 0 failures | Investigate and resolve each failure |
| Replication latency intrasite | < 15 minutes | Check network, replmon, event logs |
| Replication latency intersite | Within scheduled interval | Verify site link config |
| SYSVOL synchronized | All DCs synchronized | Run DFS-R diagnostic, authoritative restore if needed |
| FSMO roles responding | All 5 roles accessible | Seize unavailable roles per documented procedure |
| DNS SRV records registered | All DCs registered | Run `dcdiag /test:registerindns` |
| Stale computer objects | < 5% of total | Clean up or disable stale objects |
| Stale user accounts | 0 active accounts for departed staff | Disable immediately, delete after retention period |

---

## DNS Audit Model

DNS failures are the root cause of a disproportionate share of AD authentication and replication issues. A clean DNS environment is a prerequisite for a stable AD.

### DNS Health Checks

```powershell
# Verify DNS zones on each DC
Get-DnsServerZone -ComputerName DC01

# Check for scavenging configuration
Get-DnsServerZone -ComputerName DC01 |
  Select-Object ZoneName, IsScavengingEnabled, RefreshInterval, NoRefreshInterval

# Find records with no refresh (stale candidates)
Get-DnsServerResourceRecord -ZoneName "domain.local" -ComputerName DC01 |
  Where-Object { $_.TimeStamp -lt (Get-Date).AddDays(-90) -and $_.TimeStamp -ne $null } |
  Select-Object HostName, TimeStamp, RecordType

# Check _msdcs zone delegation
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.domain.local" -Type SRV

# Validate all DCs are registered in DNS
Resolve-DnsName -Name "DC01.domain.local"
```

### DNS Configuration Standards

| Setting | Recommended Value |
|---|---|
| Zone type | AD-integrated, stored in domain partition |
| Dynamic updates | Secure only |
| Scavenging enabled | Yes |
| No-refresh interval | 7 days |
| Refresh interval | 7 days |
| Scavenge interval | 7 days |
| Forwarders | Internal DNS only — no public forwarders for internal zones |
| Root hints | Disabled if using forwarders |

Conditional forwarders should be configured for any partner domains, Azure AD Connect, or cloud-connected services. Never configure external root hints as the default for internal zone resolution.

---

## SYSVOL and DFS-R Validation

SYSVOL replication failures are often silent — Group Policy applies from a stale copy without any obvious error to end users.

```powershell
# Check DFS-R service state on all DCs
Invoke-Command -ComputerName (Get-ADDomainController -Filter *).Name {
  Get-Service -Name DFSR | Select-Object MachineName, Status
}

# Check for DFS-R backlog between DC pairs
dfsrdiag BacklogStatus /rgName:"Domain System Volume" `
  /rfName:SYSVOL `
  /sendingmember:DC01 `
  /receivingmember:DC02

# Verify SYSVOL share is accessible on all DCs
Invoke-Command -ComputerName (Get-ADDomainController -Filter *).Name {
  Test-Path "\\$env:COMPUTERNAME\SYSVOL"
}
```

**If SYSVOL is out of sync:** Perform a non-authoritative DFS-R sync on the affected DC first. Only proceed to an authoritative restore if non-authoritative sync fails to resolve the inconsistency within 48 hours.

---

## Group Policy Audit and Governance

### GPO Inventory

```powershell
# Export all GPOs with their link status and scope
Get-GPO -All | Select-Object DisplayName, Id, GpoStatus,
  @{N='Links';E={(Get-GPOReport -Guid $_.Id -ReportType Xml | 
    Select-Xml -XPath "//LinksTo/SOMPath").Node.InnerText -join ', '}} |
  Export-Csv -Path gpo-inventory.csv -NoTypeInformation

# Find GPOs with no links (orphaned)
Get-GPO -All | Where-Object {
  $links = (Get-GPOReport -Guid $_.Id -ReportType Xml |
    Select-Xml -XPath "//LinksTo").Node
  $links.Count -eq 0
} | Select-Object DisplayName, Id
```

### GPO Governance Standards

| Standard | Requirement |
|---|---|
| GPO naming convention | `[Scope]-[Function]-[Version]` e.g. `CORP-SecurityBaseline-v2` |
| Orphaned GPOs | None — delete or link all GPOs |
| GPO owner | Each GPO has a documented owner in the description field |
| Testing before production | All GPOs validated in staging OU before production linkage |
| WMI filters | Documented and tested — untested WMI filters cause silent non-application |
| Loopback processing | Documented where used — can cause unexpected policy application |
| Block inheritance | Minimise use — document every instance |

---

## Patch Compliance Baseline

### Patch Assessment

```powershell
# Check last Windows Update time on all servers
Invoke-Command -ComputerName (Get-ADComputer -Filter {OperatingSystem -like "*Server*"}).Name {
  $session = New-Object -ComObject Microsoft.Update.Session
  $searcher = $session.CreateUpdateSearcher()
  $history = $searcher.QueryHistory(0, 1)
  [PSCustomObject]@{
    Computer    = $env:COMPUTERNAME
    LastUpdate  = $history[0].Date
    Title       = $history[0].Title
  }
} | Sort-Object LastUpdate
```

### Patch Compliance Thresholds

| Category | Standard |
|---|---|
| Critical / Security updates | Applied within 30 days of release |
| Important updates | Applied within 60 days of release |
| Optional updates | Reviewed quarterly, applied selectively |
| Emergency patches (zero-day) | Applied within 72 hours per documented emergency patch procedure |
| Patch testing | All patches tested in non-production before production deployment |
| Reboot compliance | Pending reboots resolved within scheduled maintenance window |

### Patch Deployment Rings

```
Ring 1 — Canary (5% of servers)
  → Patches applied on Patch Tuesday + 2 days
  → Monitored for 48 hours

Ring 2 — Early (20% of servers)
  → Applied on Patch Tuesday + 5 days
  → Only if Ring 1 shows no issues

Ring 3 — Standard (75% of servers)
  → Applied on Patch Tuesday + 14 days
  → Only if Ring 2 shows no issues
```

Domain controllers should be patched last within Ring 3, during a defined maintenance window with rollback capability confirmed before starting.

---

## Monitoring Integration Checkpoints

### Minimum Monitoring Baseline

By Day 90, the following must be monitored with alerting:

| Component | Metric | Alert Threshold |
|---|---|---|
| Domain Controllers | CPU, Memory, Disk | > 85% utilisation for > 10 minutes |
| AD Replication | Replication failures | Any failure |
| AD Replication | Replication latency | > 60 minutes |
| DFSR | Backlog size | > 100 objects |
| DNS | Query failure rate | > 5% failure rate |
| DNS | Zone transfer failures | Any failure |
| Patch compliance | Unpatched critical CVEs | Any server > 30 days unpatched |
| Event Logs | Error rate (System, Application) | > 50 errors/hour per server |
| DC availability | Ping/RPC response | Any DC unavailable > 2 minutes |
| FSMO roles | Role accessibility | Any FSMO role unreachable > 5 minutes |

### Event IDs to Alert On

| Event ID | Source | Description | Priority |
|---|---|---|---|
| 4625 | Security | Failed logon (repeated) | High |
| 4740 | Security | Account lockout | High |
| 1925 | NETLOGON | Failed to establish secure channel | High |
| 2087, 2088 | DFSR | SYSVOL replication errors | High |
| 5774–5778 | NETLOGON | DC registration failures | High |
| 13559 | DFSR | USN rollback detected | Critical |
| 4226 | NTDS | AD database corruption | Critical |

---

## Documentation Requirements

By the end of the 90-day stabilization, the following must be documented and version-controlled:

- [ ] Network diagram showing all DCs, sites, and site links
- [ ] FSMO role holder assignments with contact/runbook for each role
- [ ] DC inventory: name, IP, site, OS version, roles, last patched
- [ ] DNS zone documentation: zones, scavenging settings, forwarder config
- [ ] GPO inventory with owner, scope, and purpose for each GPO
- [ ] Patch management schedule and ring assignments
- [ ] Monitoring alert runbooks — what to do when each alert fires
- [ ] AD backup and recovery procedure (tested and validated)
- [ ] Emergency contact list for infrastructure team

---

## Architecture Context

The Windows infrastructure baseline supports every other layer of the stack:

| Layer | Documentation |
|---|---|
| Identity | [Active Directory Multi-Site](../active-directory-multisite) |
| Backup | [Veeam On-Prem Deployment](../veeam-onprem-deployment) |
| Configuration management | [Puppet Enterprise Migration](../puppet-enterprise-migration) |
| DR validation | [Disaster Recovery & Failure Testing](../disaster-recovery-testing) |

---

## Design Principles

1. **Measure before you change** — an honest baseline prevents remediation that makes things worse
2. **DNS health is AD health** — almost every AD issue has a DNS component
3. **GPO governance requires ownership** — unowned GPOs accumulate and cause unpredictable behaviour
4. **Patch rings exist to protect production** — never skip straight to Ring 3
5. **Monitoring is not optional** — infrastructure without alerting degrades silently

---

*Part of the [Enterprise Infrastructure Architecture](../README.md) portfolio.*
