# Azure Migration
**Ada Consulting | Infrastructure Modernization Plan**

---

## Overview

This document outlines a fictional company's plan to migrate its on-premises infrastructure to Microsoft Azure. The migration aims to improve operational efficiency, reduce infrastructure costs, and enhance scalability and resilience across all office locations.

The document begins with a case analysis that maps each area of the current infrastructure to its Azure solution and the expected business gain. It then presents the target architecture, followed by a detailed migration plan covering seven workstreams:

1. Migrate the Public-Facing Application to Azure
2. Protect All Application VMs with Azure Backup
3. Migrate File Servers to Azure Files with Azure File Sync
4. Implement Hybrid AD with Microsoft Entra ID
5. Migrate Internal DB Servers to Azure SQL
6. Replace Dedicated Lines with Azure Virtual WAN
7. Replace On-Premises Backup and DR Infrastructure

The document concludes with a recommended phased migration order designed to minimise risk and maximise early return on investment.

---

## Background

Ada is a consulting company with the following infrastructure:

| Location | Role | Infrastructure |
|---|---|---|
| Vancouver | Main Office (HQ) | File servers, Domain controllers, DB servers |
| Toronto | Branch Office | Connected to HQ via dedicated line |
| New York | Branch Office | Connected to HQ via dedicated line |

Ada also operates a public-facing application comprised of three tiers, each running on three virtual machines:

- SQL Database tier (3 VMs)
- Processing middle tier (3 VMs)
- Web front end tier (3 VMs)

Ada plans to implement an Azure migration of its current infrastructure to improve efficiency and reduce costs.

---

## Case Analysis

The Azure migration is expected to deliver the following improvements across all key areas of Ada's infrastructure. The table below maps each current pain point to its Azure solution and the resulting gain.

| Area | Current Pain Point | Azure Solution | Expected Gain |
|---|---|---|---|
| Public-facing app | 9 fixed VMs running 24/7 at fixed capacity | App Service + Azure SQL MI | Reduced compute cost; auto-scaling eliminates over-provisioning |
| VM Backup | No consistent backup policy across all 9 VMs; manual jobs | Azure Backup + Recovery Services Vault (GRS) | Automated protection for all VMs; guaranteed RPO/RTO; quarterly restore drills |
| File servers | On-prem hardware; branch offices hairpin file traffic through Vancouver HQ | Azure Files + Azure File Sync | Dedicated lines eliminated; branch offices read from local cache at LAN speed |
| WAN connectivity | Expensive dedicated lines between all three offices | Azure Virtual WAN + internet connections | Dedicated lines replaced entirely; each office needs only a broadband internet connection |
| Authentication & Identity | Multiple physical DCs across locations; branch offices depend on dedicated lines to reach Vancouver DC | Hybrid AD via Microsoft Entra Connect (Password Hash Sync) + Entra ID | Dedicated lines no longer needed for auth; branch DCs eliminated; SSO across all resources |
| DB servers | On-prem VMs with manual patching and HA | Azure SQL Managed Instance | Automated patching, built-in HA, reduced DBA overhead |
| Backup & DR | Physical backup infrastructure and secondary DR site | Azure Backup + Azure Site Recovery | No hardware refresh; pay-per-use DR; faster RTO/RPO |

Overall, the migration shifts Ada from a capital expenditure (CapEx) model to an operational expenditure (OpEx) model, aligning infrastructure costs directly with business usage. Notably, the combination of Azure File Sync, Entra ID, and Azure Virtual WAN eliminates the need for dedicated lines between offices entirely — each office requires only a reliable broadband internet connection to reach all Azure resources and communicate securely with other offices through the Azure Virtual WAN hub.

---

## Architecture Design

The diagram below illustrates Ada's target Azure architecture, showing how the three offices connect through Azure Virtual WAN, how the public-facing application tiers are hosted as managed PaaS services, and how Azure Backup protects all application VMs.

![Figure 1: Ada Consulting – Azure Target Architecture](Ada_Azure_Architecture.png)

*Figure 1: Ada Consulting – Azure Target Architecture*

---

## Migration Plan

The migration is structured into seven key areas, ordered by priority and ROI.

### 1. Migrate the Public-Facing Application to Azure

The public-facing application is the highest-priority migration target, as it offers the greatest cost savings and the least disruption risk. The 9 VMs across three tiers will be replaced with managed Azure PaaS services:

| Tier | Current State | Azure Target | Benefit |
|---|---|---|---|
| Web Front End | 3 VMs | Azure App Service | Auto-scaling, managed SSL, no OS patching |
| Middle Tier | 3 VMs | Azure App Service / Azure Functions | Scale to zero, pay-per-use model |
| SQL Database | 3 VMs | Azure SQL Managed Instance | Built-in HA, automated backups, no VM management |

An Azure Application Gateway with Web Application Firewall (WAF) will replace manual load balancing, providing built-in DDoS protection and SSL termination.

### 2. Protect All Application VMs with Azure Backup

All virtual machines supporting the public-facing application must be protected by Azure Backup before, during, and after migration. A dedicated Recovery Services Vault with geo-redundant storage (GRS) will be provisioned in the same Azure region as the protected resources, ensuring backup data survives even a full regional outage.

| VM Tier | VMs Protected | Backup Policy | Retention | Restore Target |
|---|---|---|---|---|
| Web Front End | 3 VMs (App Service) | Daily snapshot at 02:00 UTC | 30 days | Same region, RTO <4 hr |
| Processing Middle Tier | 3 VMs (App Service) | Daily snapshot at 02:30 UTC | 30 days | Same region, RTO <4 hr |
| SQL Database Tier | 3 VMs → Azure SQL MI | Automated full + log backups | 35 days (point-in-time) | Any point within retention |

Key backup configuration details:

- Recovery Services Vault uses geo-redundant storage (GRS): 3 copies in Canada Central + 3 automatically replicated to Canada East, at no extra configuration effort
- The vault must reside in the same Azure region as the resources it protects (Canada Central); GRS replication to the paired region is automatic
- Azure SQL Managed Instance provides automated full, differential, and transaction log backups with point-in-time restore up to 35 days
- Backup alerts configured in Azure Monitor notify the operations team of any failures within 15 minutes
- Quarterly restore drills validate backup integrity and confirm RTO targets are achievable
- Azure Site Recovery (ASR) replicates all IaaS VMs to a secondary Azure region, providing an RPO of <15 minutes

### 3. Migrate File Servers to Azure Files with Azure File Sync

On-premises file servers at HQ will be replaced with Azure Files. Azure File Sync agents will be deployed at all three offices, providing:

- A centralized Azure File Share as the single source of truth, synced across Vancouver, Toronto, and New York
- Local caching at each office so employees read and write files at LAN speed from a local server, with no dependency on the dedicated lines or HQ
- Cloud tiering to automatically move infrequently accessed files to Azure, freeing local disk space while keeping stubs accessible
- Elimination of the dedicated lines as a file-access dependency — branch offices no longer hairpin traffic through Vancouver HQ

### 4. Implement Hybrid AD with Microsoft Entra ID

Microsoft Entra Connect will be installed on a domain-joined server in Vancouver HQ and configured to sync on-premises Active Directory with Microsoft Entra ID using Password Hash Sync. Sync traffic travels outbound over HTTPS (port 443) to Microsoft's Azure endpoints — no inbound firewall ports are required. This enables:

- Single Sign-On (SSO) across on-premises and Azure resources using existing AD credentials
- Seamless authentication to Azure Files, Azure VMs, and other Azure resources without re-login
- Branch offices authenticating via Entra ID over the internet, eliminating their dependency on the dedicated lines to reach the Vancouver Domain Controller
- Reduction or elimination of physical Domain Controllers in Toronto and New York
- Incremental sync every 30 minutes; even if the on-prem connection drops, cloud authentication continues uninterrupted

### 5. Migrate Internal DB Servers to Azure SQL

Internal database servers will be evaluated for migration to Azure SQL Managed Instance, which provides near-full SQL Server compatibility with the benefits of a managed PaaS service. Where specific OS or SQL configurations cannot be accommodated, Azure VMs will be used as a lift-and-shift alternative.

### 6. Replace Dedicated Lines with Azure Virtual WAN

The dedicated lines between Vancouver, Toronto, and New York will be replaced entirely with Azure Virtual WAN. Each office establishes an encrypted Site-to-Site IPsec VPN tunnel over its own broadband internet connection into a centralized Azure Virtual WAN hub in Canada Central. This enables:

- Complete elimination of dedicated office-to-office circuits — each office only needs a broadband internet connection
- Office-to-office traffic routed securely through Azure's private backbone, never traversing the raw public internet between offices
- Direct access from all offices to Azure resources (Azure Files, App Service, SQL MI) without hairpinning through HQ
- Centralized network management and monitoring through the Azure portal
- Improved resiliency — a second ISP can be added at any office as a failover with no contract changes
- Optional ExpressRoute for Vancouver HQ if guaranteed bandwidth to Azure is required

### 7. Replace On-Premises Backup and DR Infrastructure

All on-premises backup infrastructure and any secondary DR site will be decommissioned and replaced with:

- Azure Backup for file shares, VMs, and databases — snapshot-based with easy point-in-time restore
- Azure Site Recovery to replicate on-premises VMs to Azure for disaster recovery
- Elimination of hardware refresh cycles for backup and DR infrastructure

---

## Recommended Migration Order

The following phased approach minimizes risk and maximizes early ROI:

| Phase | Area | Rationale |
|---|---|---|
| 1 | Public-facing application | Highest ROI, no end-user disruption, already internet-facing |
| 2 | VM Backup (Azure Backup + ASR) | Protect all application VMs immediately — no VM should run unprotected |
| 3 | File servers (Azure File Sync) | Phased migration with local cache; low risk, gradual cutover |
| 4 | Networking (Azure Virtual WAN) | Deploy alongside file migration; begin decommissioning dedicated lines |
| 5 | Internal DB servers | Assess and lift-and-shift; coordinate with app dependencies |
| 6 | Backup & DR (on-prem decommission) | Decommission on-prem backup infra once Azure Backup is validated |
| 7 | Domain Controllers (Hybrid AD) | Most sensitive; migrate last once all other services are stable on Azure |
