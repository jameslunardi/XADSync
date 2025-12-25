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

#### 2.2.2 Sync Schedule & Execution
- **Schedule Type:** Configurable scheduled task
- **Default Frequency:** Every 2 hours
- **Manual Trigger:** Support for on-demand manual execution
- **Execution Windows:** No specific time restrictions (runs 24/7)

#### 2.2.3 Sync Methodology
- **Approach:** Full synchronization (not incremental/delta)
- **Process Flow:**
  1. Read all user accounts from corp.local (with EmployeeID filter)
  2. Read all user accounts from prod.local (previously synced accounts)
  3. Compare both sets and categorize users into:
     - **Additions:** Users in corp but not in prod (create new accounts)
     - **Modifications:** Users in both domains (update attributes)
     - **Removals:** Users in prod but not in corp (quarantine/delete)
  4. Execute the identified actions in order

#### 2.2.4 Logging & Auditing
- **Log Storage:** File-based logging to local disk
- **Log Rotation:** Automatic deletion of log files older than configurable threshold (e.g., 90 days)
- **Log Detail:** Record all sync actions (create, update, disable, delete operations per user)
- **Log Location:** Configurable log directory path

#### 2.2.5 Error Handling & Fail-Safes

**Change Volume Fail-Safe:**
- Monitor total number of changes (additions + modifications + removals)
- If changes exceed configurable threshold, halt execution before making changes
- Send email alert to support team for manual review
- Support override mechanism to allow manual continuation after verification

**Per-User Error Handling:**
- Single user error should NOT stop the sync process
- Log the error and continue processing other users
- Track errors per user across multiple sync runs
- If the same user fails repeatedly (configurable threshold), generate error report

**Critical Error Handling:**
- If multiple errors occur in a single sync run (configurable threshold), halt execution
- Send alert to support team with error details

**Error Tracking:**
- Maintain persistent error history for users
- Track error count and last error date per user
- Reset error count after successful sync of previously failed user

#### 2.2.6 Reporting & Notifications
- **Summary Reports:** After each sync, generate summary report with:
  - Total users processed
  - Additions count
  - Modifications count
  - Quarantines/removals count
  - Errors encountered
  - Sync duration
- **Email Notifications:**
  - Send summary report after each successful sync
  - Send alert email for fail-safe triggers
  - Send alert email for critical errors
  - Configurable recipient list

---

## 3. Architecture Design

### 3.1 Technology Stack

**Primary Language:** PowerShell

**Core Dependencies:**
- ActiveDirectory PowerShell Module (for AD operations)
- Built-in cmdlets: Get-ADUser, Set-ADUser, New-ADUser, Move-ADObject, Remove-ADObject
- Send-MailMessage (for email notifications)

**Advantages for this use case:**
- Native AD cmdlets designed for domain operations
- Simple credential management with PSCredential objects
- Easy to debug and maintain
- Standard choice for Windows AD automation
- Scheduled Task integration on Windows Server

### 3.2 Configuration Management

**Configuration File Format:** JSON (config.json)

**Configuration File Contents:**
- Domain settings (domain names, domain controllers)
- OU paths (new users OU, quarantine OU in prod.local)
- Credential file paths (references to encrypted credential files)
- Thresholds:
  - Change volume limit
  - Error count thresholds (per-user, per-run)
  - Log retention days
  - Quarantine retention days (90 days default)
- Email settings:
  - SMTP server address
  - From address
  - Recipient list (support team)
- Exclusion list (usernames or EmployeeIDs to skip)
- Attribute mappings (which AD attributes to sync)
- Logging settings (log directory path, verbosity level)

**Credential Storage (Secure Local Method):**
- Use PowerShell's `Export-Clixml` and `Import-Clixml` cmdlets
- Create encrypted credential files using `Get-Credential | Export-Clixml`
- Credentials are encrypted using Windows Data Protection API (DPAPI)
- Encryption is user-specific and machine-specific
- Only the service account on the designated server can decrypt
- Store credential files:
  - `corp-domain-credential.xml` - Corp domain read account
  - `prod-domain-credential.xml` - Prod domain write account

**Setup Process:**
1. Run setup script as the service account that will execute scheduled task
2. Script prompts for credentials and saves encrypted files
3. Config file references the credential file paths
4. Main sync script imports credentials at runtime using `Import-Clixml`

**Security Model:**
- Config file contains non-sensitive settings (can be in version control with redacted emails)
- Credential files are machine/user-specific, cannot be copied to other systems
- Credential files excluded from version control (.gitignore)
- Only the service account can read the encrypted credentials

### 3.3 Components
*To be designed*

### 3.4 Data Flow
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

### Q3: How often should the synchronization run?
**A:** Configurable scheduled task running every 2 hours by default, with support for manual triggering. Use full sync approach: read all users from both domains, compare them, categorize into additions/modifications/removals, then execute actions.

### Q4: What are your logging and error handling requirements?
**A:** File-based logging with automatic cleanup (older than X days). Change volume fail-safe: if more than X changes detected, stop and email support for manual verification with override capability. Per-user error handling: single errors continue processing, but same user erroring repeatedly triggers alert. Multiple errors in one run may halt execution. Email summary reports after each sync with counts of additions/modifications/removals/errors.

### Q5: What technology stack do you want to use?
**A:** PowerShell with the ActiveDirectory module. This provides native AD cmdlets, simple credential management, and is the standard choice for Windows AD automation.

### Q6: How should configuration be managed?
**A:** Use a JSON config file for all settings (domains, OUs, thresholds, email, exclusions). For credentials, use PowerShell's Export-Clixml/Import-Clixml which encrypts credentials using Windows DPAPI. Credentials are user and machine-specific, ensuring only the service account on the sync server can decrypt them. A setup script will create the encrypted credential files.
