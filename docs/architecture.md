# XADSync Architecture Document

## Document Status
**Status:** In Progress
**Last Updated:** 2025-12-25

---

## 1. Project Overview

### Purpose
Cross-domain Active Directory synchronisation tool for syncing user accounts between separate AD domains.

### Scope
One-way synchronization from corporate domain to production domain.

### Domain Architecture
- **Source Domain:** corp.local (Corporate)
- **Target Domain:** prod.local (Production)
- **Relationship:** Two isolated domains, no trust relationship
- **Network:** Same organization, separate network segments
- **Deployment:** Sync server runs in production network segment

---

## 2. Requirements

### 2.1 Business Requirements
*To be documented*

### 2.2 Technical Requirements

#### 2.2.1 Authentication & Permissions
- **Corp Domain Service Account:** Read-only access to source domain (corp.local)
- **Prod Domain Service Account:** Write access to target domain (prod.local)
- **Server Location:** Production network with connectivity to both domains

---

## 3. Architecture Design

### 3.1 Technology Stack
*To be decided*

### 3.2 Components
*To be designed*

### 3.3 Data Flow
*To be mapped*

---

## 4. Implementation Roadmap

### 4.1 Phases
*To be planned*

---

## Questions & Answers

### Q1: What AD domains are you synchronizing between?
**A:** Two domains - corp.local (corporate/source) and prod.local (production/target). They are isolated with no trust relationship, in the same organization but on separate network segments. The sync server will run in the production network with service accounts for both domains (read on corp, write on prod).
