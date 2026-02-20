# Puppet Enterprise Migration & Configuration Governance Playbook

---

## Executive Summary

This playbook defines a structured migration from Puppet Open Source to Puppet Enterprise (PE) with an emphasis on governance, drift control, certificate hygiene, and controlled rollout sequencing.

The objective is to:

- Eliminate manual configuration drift
- Standardize Roles & Profiles architecture
- Enforce Git-based change governance
- Align hostnames and domain structure during migration
- Ensure observability of configuration state

Configuration management must be predictable, measurable, and recoverable.

---

## Migration Context & Drivers

This migration typically occurs when:

- Manual updates have created configuration drift
- Hostname/domain structures are being realigned
- Change governance requires RBAC and auditing
- Reporting and compliance visibility is needed
- Puppet infrastructure must scale reliably

The migration must avoid configuration regressions during transition.

---

## Architecture Decisions

### Puppet Enterprise Adoption

**Decision:** Standardize on Puppet Enterprise with Code Manager and PuppetDB enabled.

**Rationale:**

- Centralized RBAC
- Built-in reporting & node state visibility
- Integrated orchestration capability
- Structured environment promotion (dev → prod)
- Better compliance reporting

Open Source Puppet is suitable for small environments but lacks governance depth required at scale.

---

### Roles & Profiles Model

**Decision:** Enforce strict Roles & Profiles architecture.

**Structure:**

- `role::` classes define node purpose
- `profile::` classes define reusable components
- No direct module declarations in node definitions

**Rationale:**

- Separation of intent vs implementation
- Cleaner Hiera data management
- Reduced duplication
- Predictable catalog compilation

---

## Environment Structure (r10k / Code Manager)

Recommended structure:
