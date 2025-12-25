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

**Main Scripts:**
1. **Sync-XADUsers.ps1**
   - Main synchronization orchestrator
   - Entry point for scheduled tasks or manual execution
   - Coordinates all modules and workflow steps
   - Implements fail-safe checks and error thresholds

2. **Setup-XADSync.ps1**
   - Initial setup and configuration utility
   - Creates encrypted credential files
   - Validates configuration file
   - Tests domain connectivity
   - Creates required directory structure

**PowerShell Modules:**
1. **XADSync.Config.psm1**
   - Load and parse config.json
   - Import encrypted credentials
   - Validate configuration settings
   - Provide configuration access to other modules

2. **XADSync.Logging.psm1**
   - Write log entries to file
   - Implement log rotation (delete old logs)
   - Format log messages with timestamps
   - Support different log levels (INFO, WARNING, ERROR)

3. **XADSync.ADOperations.psm1**
   - Get-XADUsers: Retrieve users from domain with filters
   - New-XADUser: Create new user in target domain
   - Update-XADUser: Update user attributes
   - Move-XADUserToQuarantine: Move user to quarantine OU and set date
   - Remove-XADQuarantinedUser: Delete users past retention period
   - Test-XADConnection: Validate domain connectivity

4. **XADSync.Compare.psm1**
   - Compare-XADUserSets: Compare source and target users
   - Categorize into additions, modifications, removals
   - Detect attribute changes requiring updates
   - Apply exclusion list filtering

5. **XADSync.ErrorTracking.psm1**
   - Load/save error-history.json
   - Track per-user error counts
   - Update error timestamps
   - Reset error count on success
   - Check if user exceeds error threshold

6. **XADSync.Notifications.psm1**
   - Send-XADSyncReport: Email summary reports
   - Send-XADAlert: Email alert messages
   - Format-XADSyncSummary: Generate report content
   - Format-XADFailSafeAlert: Generate fail-safe alerts

**Directory Structure:**
```
XADSync/
├── Sync-XADUsers.ps1           # Main sync script
├── Setup-XADSync.ps1           # Setup utility
├── Modules/
│   ├── XADSync.Config.psm1
│   ├── XADSync.Logging.psm1
│   ├── XADSync.ADOperations.psm1
│   ├── XADSync.Compare.psm1
│   ├── XADSync.ErrorTracking.psm1
│   └── XADSync.Notifications.psm1
├── Config/
│   ├── config.json             # Main configuration
│   ├── config.example.json     # Example config (for repo)
│   └── Credentials/            # Encrypted credential files (gitignored)
│       ├── corp-domain-credential.xml
│       └── prod-domain-credential.xml
├── Data/
│   └── error-history.json      # Persistent error tracking
├── Logs/                       # Log files (gitignored)
│   └── sync-YYYY-MM-DD.log
└── docs/
    └── architecture.md
```

### 3.4 Data Flow

**Sync Execution Flow:**

1. **Initialization**
   - Load configuration from config.json
   - Import encrypted credentials
   - Initialize logging
   - Load error history

2. **Data Collection**
   - Connect to corp.local with read credential
   - Retrieve all users with EmployeeID populated
   - Connect to prod.local with write credential
   - Retrieve all previously synced users

3. **Comparison & Planning**
   - Apply exclusion list to source users
   - Compare source vs target users by EmployeeID
   - Categorize into:
     - Additions (in corp, not in prod)
     - Modifications (in both, attributes differ)
     - Removals (in prod, not in corp)

4. **Fail-Safe Check**
   - Count total changes (additions + modifications + removals)
   - If exceeds threshold:
     - Send alert email to support team
     - Log details and halt execution
     - Require manual override to continue

5. **Execute Changes** (if fail-safe passes)
   - Process Additions:
     - Create disabled users in target OU
     - Track successes and errors
   - Process Modifications:
     - Update user attributes
     - Track successes and errors
   - Process Removals:
     - Disable and move to quarantine
     - Set quarantine date
     - Track successes and errors
   - Process Quarantine Cleanup:
     - Find users quarantined > 90 days
     - Delete permanently

6. **Error Handling**
   - Update error-history.json with results
   - Check for repeated user failures
   - Check for excessive errors in single run
   - Send alerts if thresholds exceeded

7. **Reporting**
   - Generate summary report
   - Clean up old log files
   - Send email report to support team

---

## 4. Implementation Roadmap

### 4.1 Phases

**Phase 1: Foundation & Setup**
- Create directory structure (Config/, Modules/, Data/, Logs/)
- Create .gitignore (exclude Credentials/, Logs/, Data/)
- Implement XADSync.Config.psm1
  - JSON config loading
  - Credential import from XML files
  - Configuration validation
- Implement XADSync.Logging.psm1
  - Basic logging functions
  - Log rotation
- Create config.example.json template
- Create Setup-XADSync.ps1
  - Directory creation
  - Credential file generation
  - Configuration validation
  - Domain connectivity tests

**Phase 2: Core AD Operations**
- Implement XADSync.ADOperations.psm1
  - Get-XADUsers (with EmployeeID filter)
  - Test-XADConnection
- Create basic Sync-XADUsers.ps1 skeleton
  - Load config
  - Initialize logging
  - Retrieve users from both domains
  - Display user counts (no modifications yet)
- Test against corp.local and prod.local test domains

**Phase 3: Comparison Logic**
- Implement XADSync.Compare.psm1
  - Compare-XADUserSets function
  - Categorize additions, modifications, removals
  - Detect attribute differences
  - Apply exclusion list filtering
- Update Sync-XADUsers.ps1
  - Integrate comparison logic
  - Log planned changes (don't execute yet)
- Test comparison accuracy with sample data

**Phase 4: Write Operations**
- Extend XADSync.ADOperations.psm1
  - New-XADUser (create disabled users)
  - Update-XADUser (update attributes)
  - Move-XADUserToQuarantine
  - Remove-XADQuarantinedUser
- Update Sync-XADUsers.ps1
  - Execute additions, modifications, removals
  - Add per-user error handling (continue on failure)
- Test each operation type in isolation

**Phase 5: Error Tracking & Fail-Safes**
- Implement XADSync.ErrorTracking.psm1
  - Load/save error-history.json
  - Track per-user errors
  - Check error thresholds
- Implement XADSync.Notifications.psm1 (basic version)
  - Send-XADAlert function
  - Format-XADFailSafeAlert function
- Update Sync-XADUsers.ps1
  - Implement change volume fail-safe
  - Implement per-run error threshold
  - Implement repeated user error detection
  - Add override mechanism for fail-safe
- Test fail-safe triggers and alerts

**Phase 6: Reporting & Notifications**
- Complete XADSync.Notifications.psm1
  - Send-XADSyncReport function
  - Format-XADSyncSummary function
- Update Sync-XADUsers.ps1
  - Generate summary statistics
  - Send email reports
  - Include sync duration
- Test email delivery and report formatting

**Phase 7: Quarantine Cleanup**
- Update XADSync.ADOperations.psm1
  - Implement 90-day quarantine cleanup logic
- Update Sync-XADUsers.ps1
  - Check quarantined users
  - Delete users past retention period
- Test quarantine date logic and deletion

**Phase 8: Production Readiness**
- Create comprehensive documentation
  - Installation guide
  - Configuration guide
  - Troubleshooting guide
- Create Windows Scheduled Task setup guide
- End-to-end testing with realistic dataset
- Security audit (credential handling, permissions)
- Performance testing (large user sets)
- Create example config files for different scenarios

### 4.2 Testing Strategy

**Unit Testing:**
- Test each module function independently
- Mock AD operations where possible
- Validate configuration parsing

**Integration Testing:**
- Test against corp.local and prod.local test domains
- Use dedicated test OUs
- Create test users with known attributes
- Verify each sync operation type

**End-to-End Testing:**
- Full sync cycles with various scenarios:
  - New user creation
  - Attribute updates
  - User removal and quarantine
  - Fail-safe triggering
  - Error handling
  - Report generation

**Security Testing:**
- Verify credential encryption
- Test with minimal permissions
- Validate exclusion list functionality

### 4.3 Deployment Checklist

- [ ] Install ActiveDirectory PowerShell module on sync server
- [ ] Create service account in corp.local (read-only)
- [ ] Create service account in prod.local (write access)
- [ ] Configure OUs in prod.local (new users, quarantine)
- [ ] Copy XADSync files to sync server
- [ ] Run Setup-XADSync.ps1 as service account
- [ ] Configure config.json with production settings
- [ ] Test manual sync execution
- [ ] Create Windows Scheduled Task (every 2 hours)
- [ ] Configure SMTP for email notifications
- [ ] Monitor first few sync cycles
- [ ] Document any environment-specific configurations

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

### Q7: Component/Module Structure?
**A:** Modular design with 2 main scripts (Sync-XADUsers.ps1, Setup-XADSync.ps1) and 6 PowerShell modules (Config, Logging, ADOperations, Compare, ErrorTracking, Notifications). Organized directory structure with Config/, Modules/, Data/, and Logs/ folders.
