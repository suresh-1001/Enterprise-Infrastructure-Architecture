# Active Directory Multi-Site Architecture Playbook

---

## Executive Summary

This architecture consolidates multiple Active Directory domains into a unified, multi-site forest while preserving local authentication autonomy.

The design assumes:

- Offices may lose WAN connectivity
- Badge systems require local authentication
- Central governance must remain intact
- Replication health must be measurable

The objective is resilient identity without cross-site dependency during outages.

---

## Architecture Decisions

### Single Forest, Single Domain

**Decision:** Deploy a single forest with a single domain.

**Rationale:**

- Simplifies trust relationships
- Reduces replication complexity
- Centralizes GPO governance
- Minimizes cross-domain SID history issues
- Eases long-term operational management

Multi-forest models are rejected unless legal or regulatory isolation explicitly requires separation.

---

### Domain Controller Placement Strategy

| Office Type        | Deployment Model |
|-------------------|------------------|
| Large / Critical  | 2 Writable DCs   |
| Medium Office     | 1 Writable DC    |
| Small / High Risk | 1 RODC           |

**Decision Criteria:**

- Does the site require local object creation?
- Does badge provisioning require writable directory?
- Is physical security limited?

RODCs are deployed where attack surface reduction is preferred and write capability is not required.

---

## AD Sites & Subnet Design

- Each office mapped to a dedicated AD Site
- Subnets strictly associated to correct site
- Replication schedules defined via Site Links
- Site link costs tuned to WAN topology

Improper site mapping is one of the most common causes of cross-site authentication latency and replication drift.

---

## Replication Strategy

- Intra-site replication: Change notification
- Inter-site replication: Scheduled and compressed
- DFS-R for SYSVOL
- Site link costs explicitly defined

Replication topology changes must follow formal change control and post-change validation to prevent silent divergence between sites.

---

## Operational Considerations

- AD Sites and Subnets strictly maintained
- Time synchronization hierarchy enforced (PDC Emulator authoritative)
- SYSVOL replication via DFS-R only
- System State backups for all DCs
- FSMO role placement reviewed annually
- LDAP and Kerberos authentication path validated

---

## Failure Mode Analysis

| Scenario | Expected Behavior | Mitigation |
|----------|------------------|------------|
| WAN outage | Local DC authenticates users | Per-site DC deployment |
| Central DC failure | Sites operate independently | Writable DC per critical site |
| Replication failure | Password updates delayed | Monitoring + manual validation |
| DNS misconfiguration | Authentication latency | AD-integrated DNS per site |
| Time drift | Kerberos failures | Strict NTP hierarchy |

The architecture is designed so identity services degrade in a controlled manner rather than fail unpredictably.

---

## Monitoring Strategy

Minimum monitoring must include:

- `repadmin /replsummary` scheduled validation
- `dcdiag` health checks
- DFS-R SYSVOL status
- Event ID monitoring (1311, 1865, 2042)
- NTP drift monitoring
- LDAP bind failure rate tracking

Alerting must trigger before authentication failures reach end users.

---

## Migration Phases (High-Level)

1. Inventory current forests and domain dependencies
2. Establish trust relationships where required
3. Pilot migration with a non-critical office
4. Migrate users and computers via ADMT
5. Validate replication and authentication flows
6. Decommission legacy domains in controlled phases

Migration is executed incrementally to prevent cross-domain authentication disruption. Rollback checkpoints must be defined before each phase to ensure identity continuity.

---

## Design Principle

Identity architecture must prioritize:

- Predictability
- Observability
- Resilience under WAN failure
- Minimal cross-site dependency
- Controlled change management
