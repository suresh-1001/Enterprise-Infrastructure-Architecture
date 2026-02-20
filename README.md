# Enterprise Infrastructure Architecture Portfolio

This repository documents enterprise-grade infrastructure design patterns used in regulated, high-availability environments.

Focus Areas:

- Active Directory multi-site resiliency
- On-prem VMware + NetApp + Veeam backup architecture
- Puppet Enterprise migration strategy
- Windows infrastructure stabilization
- Disaster recovery testing methodology

The designs here prioritize stability, operational clarity, and failure resilience.

---

## 🏗 Architecture Domains Covered

### 🔐 Active Directory Multi-Site Design
Location: `/active-directory-multisite`

Covers:
- Single forest consolidation strategy
- Site-aware Domain Controller placement
- RODC vs Writable DC tradeoffs
- WAN failure survivability
- Badge system authentication resilience
- Migration playbook for domain consolidation

---

### 💾 Veeam On-Prem Deployment Blueprint
Location: `/veeam-onprem-deployment`

Covers:
- VMware transport modes (HotAdd, NBD, Direct SAN)
- Proxy and repository sizing strategy
- Application-aware processing
- NetApp snapshot alignment
- Restore validation methodology
- DR testing approach

---

### ⚙ Puppet Enterprise Migration Strategy
Location: `/puppet-enterprise-migration`

Covers:
- Open Source → PE migration considerations
- r10k / Code Manager structure
- Roles and Profiles model
- Certificate re-issuance
- Domain/hostname realignment planning

---

### 🖥 Windows Infrastructure Stabilization Plan
Location: `/windows-infrastructure-stabilization`

Covers:
- 30-60-90 day onboarding framework
- AD health validation (dcdiag / repadmin)
- DNS and GPO audit
- Patch governance
- Monitoring integration

---

### 🧪 Disaster Recovery Testing Framework
Location: `/disaster-recovery-testing`

Covers:
- AD WAN failure simulation
- VMware HA validation
- NetApp failover test methodology
- Backup restore drills
- RTO measurement

---

## 🎯 Design Philosophy

The infrastructure patterns documented here follow five principles:

1. Detect issues before users do
2. Design for failure, not perfection
3. Keep architecture simple and governable
4. Validate restores — backups alone are not enough
5. Prefer boring, stable systems over trendy complexity

---

## 📌 Intended Audience

Infrastructure engineers, architects, and technical leaders operating:

- On-prem VMware environments
- Hybrid AD deployments
- Regulated workloads (PCI / SOC)
- Multi-site enterprise networks

---

This portfolio represents structured thinking applied to real-world infrastructure constraints.
