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

#### 2.1.1 Sync Scope
- **Object Type:** User accounts only (no groups, OUs, contacts, etc.)
- **Primary Filter:** Users with an EmployeeID attribute populated
- **Exclusion List:** Support for excluding specific users (e.g., service accounts, test users)

#### 2.1.2 Attributes to Synchronize
- **Basic Attributes:** Name, email, phone, etc.
- **Extended Attributes:**
  - Manager
  - Department
  - EmployeeID
- **Excluded:**
  - Passwords (accounts created without passwords)
  - Group memberships

#### 2.1.3 Matching Logic
- **Unique Identifier:** EmployeeID attribute
- Users are matched between domains based on EmployeeID value

#### 2.1.4 Synchronization Behavior

**New Accounts** (exists in corp.local, not in prod.local):
- Create user account in prod.local
- Set account as **disabled**
- Create without password
- Place in designated "new users" OU

**Existing Accounts** (exists in both domains):
- Update prod.local account attributes to match corp.local
- Ensure all specified attributes are synchronized

**Orphaned Accounts** (exists in prod.local, not in corp.local):
- Disable the account immediately
- Move to quarantine OU
- Set quarantine date attribute
- After 90 days from quarantine date, permanently delete the account

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

### Q2: What Active Directory objects do you want to synchronize?
**A:** User accounts only. Filter by EmployeeID attribute (must be populated), with support for exclusion lists. Sync basic attributes plus manager, department, and employeeid. No passwords or group memberships. EmployeeID is the matching attribute. New accounts created as disabled without passwords in a specific OU. Existing accounts updated to match corp. Orphaned accounts (in prod but not corp) are disabled, moved to quarantine OU, and deleted after 90 days.
