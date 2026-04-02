# Puppet Enterprise Migration Strategy

![Puppet Enterprise Migration Strategy](./diagrams/puppet-migration-banner.webp)

> *Configuration drift is not a Puppet problem — it is a governance problem. Puppet makes it visible.*

---

## Executive Summary

This document defines a migration strategy for transitioning an environment from **Puppet Open Source** to **Puppet Enterprise**, while simultaneously restructuring the codebase to a production-grade **Roles & Profiles** architecture.

The migration is not a lift-and-shift. It is a deliberate redesign of:

- How Puppet code is structured (component modules → Roles & Profiles)
- How code is delivered to the Puppet master (manual → r10k / Code Manager with Git)
- How nodes are classified (flat manifest → structured Puppet Enterprise Console classification)
- How configuration is managed across environments (single branch → environment branches)

The objective is a Puppet estate where every node's configuration is defined in code, version-controlled in Git, and verifiable through catalog runs — with no manual changes required or tolerated.

---

## Current State Assessment

Before beginning migration, document the current estate:

| Assessment Area | Questions to Answer |
|---|---|
| Puppet version | What version of Puppet OSS is running? |
| Master infrastructure | Single master, or multiple compile masters? |
| Codebase structure | Roles & Profiles already? Flat manifests? Mixed? |
| Module sources | Puppet Forge only? Custom modules? Mixed? |
| Hiera configuration | Single hiera.yaml? Multiple backends? Encrypted data? |
| Node classification | site.pp? ENC? Console? |
| Environments | Single production environment? Multiple? |
| Certificate authority | Self-managed CA? External CA? |
| Git integration | Code in Git? Manual file copies? |

This assessment determines the migration scope and identifies the highest-risk areas.

---

## Target Architecture

### Roles & Profiles Pattern

The Roles & Profiles pattern is the standard for maintainable Puppet code at scale:

```
Node Classification
  └── Role (one per node — describes what the machine does)
        └── Profile (one per technology stack — manages a specific component)
              └── Component Module (manages a single resource type)
```

**Example:**

```puppet
# Role: a web application server
class role::webapp_server {
  include profile::base
  include profile::apache
  include profile::app_deploy
  include profile::monitoring
}

# Profile: Apache configuration
class profile::apache {
  include apache                          # Component module from Forge
  apache::vhost { 'app.example.com':
    port    => 443,
    docroot => '/var/www/app',
  }
  class { 'apache::mod::ssl': }
}
```

**Rules:**
- Every node has exactly **one Role**
- Roles contain **only `include` statements** — no resources directly
- Profiles manage one technology stack each
- Component modules (from Puppet Forge or internal) handle individual resources

### Hiera Data Hierarchy

```yaml
# hiera.yaml — production target hierarchy
version: 5
defaults:
  datadir: data
  data_hash: yaml_data

hierarchy:
  - name: "Node-specific"
    path: "nodes/%{trusted.certname}.yaml"

  - name: "Role"
    path: "roles/%{lookup('role')}.yaml"

  - name: "Environment"
    path: "environments/%{environment}.yaml"

  - name: "Operating system"
    path: "os/%{facts.os.family}.yaml"

  - name: "Common"
    path: "common.yaml"
```

Sensitive data (passwords, API keys, certificates) is managed with **Hiera-eyaml** — encrypted in Git, decrypted at catalog compilation time on the Puppet master.

---

## r10k / Code Manager Configuration

### Git Branch to Puppet Environment Mapping

r10k maps Git branches directly to Puppet environments. Each branch is a full, isolated Puppet environment:

| Git Branch | Puppet Environment | Purpose |
|---|---|---|
| `production` | production | Live infrastructure — all changes must pass staging first |
| `staging` | staging | Pre-production validation — code tested here before merge |
| `development` | development | Active development branch |
| `feature/*` | feature_* | Short-lived branches for individual changes |

**Workflow:**

```
Developer creates feature branch
  → Code reviewed via Pull Request
    → Merge to development → automatic deploy to dev environment
      → Promote to staging via PR → validated in staging environment
        → Promote to production via PR → applied to all production nodes
```

### r10k Configuration

```yaml
# /etc/puppetlabs/r10k/r10k.yaml
cachedir: /opt/puppetlabs/r10k/cache
sources:
  control-repo:
    remote: 'git@github.com:org/puppet-control-repo.git'
    basedir: /etc/puppetlabs/code/environments
    prefix: false
```

With Puppet Enterprise, Code Manager replaces manual r10k invocation and provides:
- Webhook-triggered deploys on Git push
- Deployment status reporting in the PE Console
- File sync to all compile masters

---

## Node Classification Governance

### Classification Strategy

In Puppet Enterprise, nodes are classified through the **PE Node Manager** (Console):

```
Node Groups (hierarchical)
  └── All Nodes
        └── PE Infrastructure (managed by PE itself)
        └── Production Servers
              └── Web Servers         → role::webapp_server
              └── Database Servers    → role::database_server
              └── Domain Controllers  → role::domain_controller
        └── Staging Servers
              └── (mirrors production structure)
```

**Rules for Node Classification:**

- Every production node belongs to exactly one role node group
- Node groups inherit from parent groups — common profiles (monitoring, base hardening) live at the parent level
- No `site.pp` node declarations — classification lives entirely in the Console
- Pinning individual nodes to node groups is acceptable for exceptions; document all pins

### Avoiding Classification Conflicts

Common source of duplicate resource declarations:

```
Node is in "All Nodes" group (includes profile::base)
AND is pinned to "Web Servers" group (also includes profile::base)
  → Duplicate resource: class profile::base
```

Resolve by ensuring profiles are included at only one level of the hierarchy. Use Hiera data for variation, not duplicate class includes.

---

## Certificate Migration

### Certificate Re-issuance Plan

When migrating from Puppet OSS to PE, all agent certificates must be re-issued against the PE Certificate Authority.

**Pre-migration:**
1. Export the current CA certificate and CRL from the OSS master
2. Document all agent certnames — these must remain consistent post-migration
3. Note any certificates with custom extensions or SANs

**Migration steps:**

```bash
# On each agent node — revoke old cert and request new one
puppet ssl clean
puppet agent -t --server=new-pe-master.domain.local

# On the PE master — sign pending cert requests
puppetserver ca list
puppetserver ca sign --certname agent.domain.local
```

**Risk:** Service accounts and applications that authenticate via Puppet certificates will break during the migration window. Identify all such consumers before starting.

---

## Hostname and Domain Realignment

If the migration involves renaming nodes or changing the domain suffix:

1. Update DNS — new FQDN must resolve before Puppet agent runs
2. Clean the old certificate on both master and agent
3. Update the `server` setting in `puppet.conf` on each agent
4. Run `puppet agent -t` to request a new certificate under the new certname
5. Update any Hiera node-specific data files referencing the old certname
6. Update node group pinning in the PE Console if nodes were pinned by certname

Certname changes are the highest-risk part of a hostname migration. Test the full sequence in a non-production environment first.

---

## Controlled Rollout Methodology

### Migration Phases

**Phase 1 — Foundation (Weeks 1–2)**
- Deploy PE master infrastructure (primary master + replica)
- Configure Code Manager with control repo
- Migrate base profiles: `profile::base`, `profile::monitoring`, `profile::ntp`
- Migrate 5–10 non-critical nodes to validate the process

**Phase 2 — Role Migration (Weeks 3–6)**
- Migrate roles in dependency order: identity → infrastructure → application
- For each role: define role class → define profiles → validate against staging nodes → promote to production
- Run `puppet agent --noop` on target nodes before committing changes
- Sign off each role with a successful catalog run on at least 3 nodes

**Phase 3 — Classification and Governance (Weeks 7–8)**
- Complete node group structure in PE Console
- Enforce that all nodes are classified (no unclassified nodes in production)
- Enable compliance reporting and drift detection
- Disable manual Puppet runs on production nodes — all changes via Code Manager

**Phase 4 — Decommission OSS Master (Week 9+)**
- Confirm zero agents pointing to the old master
- Archive the old manifests directory
- Decommission the OSS master VM
- Document the final estate state

---

## Puppet Code Quality Standards

### Pre-commit Checks

Every code change should pass the following before merging:

```bash
# Syntax validation
puppet parser validate manifests/

# Style linting
puppet-lint --no-80chars-check manifests/

# Unit tests
pdk test unit

# Catalog compilation test against a canary node
puppet agent --noop --server=pe-master.domain.local
```

### Module Structure Standard

```
profile/
├── manifests/
│   ├── base.pp
│   ├── apache.pp
│   └── monitoring.pp
├── data/                   # Module-level Hiera data
├── templates/              # ERB / EPP templates
├── files/                  # Static files
├── spec/                   # RSpec unit tests
└── metadata.json
```

---

## Architecture Context

Configuration management governs the state of the full infrastructure stack:

| Layer | Documentation |
|---|---|
| Identity | [Active Directory Multi-Site](../active-directory-multisite) |
| Backup | [Veeam On-Prem Deployment](../veeam-onprem-deployment) |
| Platform baseline | [Windows Infrastructure Stabilization](../windows-infrastructure-stabilization) |
| DR validation | [Disaster Recovery & Failure Testing](../disaster-recovery-testing) |

---

## Design Principles

1. **Every node's state is defined in code** — no undocumented manual changes
2. **Git is the source of truth** — the running Puppet environment reflects the Git branch
3. **Roles describe what a machine does; Profiles describe how** — never mix these concerns
4. **Hiera separates data from code** — no hardcoded values in manifests
5. **No-op first, enforce second** — validate catalog changes on canary nodes before production rollout

---

*Part of the [Enterprise Infrastructure Architecture](../README.md) portfolio.*
