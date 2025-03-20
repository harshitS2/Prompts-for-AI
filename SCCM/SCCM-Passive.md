
---

# SCCM Site Recovery: Restoring Database Backup to Passive Site Server

## Overview

This guide explains how to recover a System Center Configuration Manager (SCCM) primary site when the active site server (`SCCM-Active`), which hosts both SCCM and its SQL Server database, becomes unavailable. The process involves restoring a site database backup (created via SCCM’s site maintenance task) to a SQL Server instance on the passive site server (`SCCM-Passive`) and reconfiguring `SCCM-Passive` to take over as the active site server.

This is a **manual disaster recovery (DR)** approach, not an automatic high-availability solution like SQL Server Always On Availability Groups (AG). It ensures business continuity using existing infrastructure when `SCCM-Active` fails.

---

## Why This Approach?

- **Current Setup**: SCCM and SQL Server are installed on `SCCM-Active`. If this server goes down, both the site server and database are lost, halting SCCM operations.
- **Backup Utilization**: SCCM’s site maintenance task generates a full database backup (e.g., `CM_ABC`), which can be restored to any compatible SQL Server instance.
- **Passive Site Leverage**: `SCCM-Passive` is already configured as a passive site server, making it an ideal candidate to become the active site server after restoring the database locally.
- **Outcome**: This method resumes SCCM operations with minimal additional hardware, though it requires manual steps and may involve data loss since the last backup.

---

## Key Considerations

- **Data Loss**: The restored database reflects the state at the time of the last backup. Changes after that (e.g., new deployments) are lost unless transaction log backups are also used (not covered here).
- **Content Library**: The database holds configuration data, not content (e.g., packages). Ensure the content library is accessible to `SCCM-Passive`, ideally on a shared network location.
- **Manual Process**: This is a recovery method, not real-time failover. For automatic HA, consider SQL Server Always On AG with a separate SQL server.
- **SQL on SCCM-Passive**: SQL Server will be installed on `SCCM-Passive` to host the restored database, which may impact performance due to resource sharing.

---

## Assumptions

- SCCM version: Current Branch (e.g., 2309 or later).
- Site database name: `CM_ABC` (adjust as needed).
- Backup location: `\\NetworkShare\SCCMBackup` (shared folder accessible to both servers).
- Content library: Moved to `\\NetworkShare\ContentLibrary` (or recoverable separately).
- `SCCM-Active` is offline and unrecoverable.
- `SCCM-Passive` is a configured passive site server in the same site.

---

## Prerequisites

Before starting:
1. **Backup Availability**: A recent SCCM site database backup from the site maintenance task on `SCCM-Active`.
2. **SQL Server Installation**: SQL Server (same version as `SCCM-Active`, e.g., SQL Server 2019) installed or installable on `SCCM-Passive`.
3. **Permissions**:
   - Your account is a local admin on `SCCM-Passive` and will have `sysadmin` rights on SQL after installation.
   - `SCCM-Passive` computer account (`DOMAIN\SCCM-Passive$`) needs SQL access.
4. **SCCM Installation Files**: Access to SCCM installation media (e.g., network share or local drive).
5. **Network**: `SCCM-Passive` can reach the backup and content library shares.

---

## Step-by-Step Guide

### Step 1: Prepare SCCM-Passive for SQL Server
- **Goal**: Install SQL Server on `SCCM-Passive` to host the restored database.
- **Steps**:
  1. Log in to `SCCM-Passive` as a local administrator.
  2. Mount or copy the SQL Server installation media (e.g., SQL Server 2019 ISO).
  3. Run SQL Server setup:
     - Select **New SQL Server standalone installation**.
     - Features: **Database Engine Services** (minimum).
     - Instance name: Default (`MSSQLSERVER`) or named (e.g., `SCCM`).
     - Service accounts: Use domain accounts or defaults with local admin rights.
     - Authentication: Enable **Mixed Mode**, set a strong SA password.
     - Collation: Use `SQL_Latin1_General_CP1_CI_AS` (SCCM requirement).
  4. Complete installation and confirm SQL Server is running via SQL Server Configuration Manager.

### Step 2: Restore the Site Database Backup
- **Goal**: Restore the SCCM database from backup to `SCCM-Passive`’s SQL instance.
- **Steps**:
  1. Copy the latest backup (e.g., `CM_ABC.bak`) from `\\NetworkShare\SCCMBackup` to `SCCM-Passive` (e.g., `C:\SCCMBackup`).
  2. Open SQL Server Management Studio (SSMS) on `SCCM-Passive`:
     - Connect to the local SQL instance (e.g., `SCCM-Passive`).
     - Right-click **Databases** > **Restore Database**.
     - Select **Device**, add `C:\SCCMBackup\CM_ABC.bak`.
     - Set destination database name to `CM_ABC`.
     - In **Options**, check **Overwrite the existing database** if needed, then click **OK**.
  3. Confirm the database appears under **Databases**.
  4. Set recovery model to **Full**:
     - Right-click `CM_ABC` > **Properties** > **Options** > **Recovery model** > **Full** > **OK**.

### Step 3: Grant Permissions to SCCM-Passive
- **Goal**: Allow `SCCM-Passive`’s computer account to access SQL and the database.
- **Steps**:
  1. In SSMS, run:
     ```sql
     USE [master]
     GO
     CREATE LOGIN [DOMAIN\SCCM-Passive$] FROM WINDOWS
     GO
     ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\SCCM-Passive$]
     GO
     ```
  2. Grant database access:
     ```sql
     USE [CM_ABC]
     GO
     CREATE USER [DOMAIN\SCCM-Passive$] FOR LOGIN [DOMAIN\SCCM-Passive$]
     GO
     ```
  3. Verify under **Security** > **Logins** and `CM_ABC` > **Users**.

### Step 4: Update SCCM-Passive to Use the Restored Database
- **Goal**: Point `SCCM-Passive` to the local restored database.
- **Steps**:
  1. Stop SCCM services:
     ```powershell
     Stop-Service -Name "SMS_EXECUTIVE"
     ```
  2. Run SCCM setup from installation media (e.g., `D:\SMSSETUP\BIN\X64\setup.exe`):
     - Choose **Perform site maintenance or reset this site**.
     - On **Database Information**:
       - Set **SQL Server name** to `SCCM-Passive` (or FQDN, e.g., `SCCM-Passive.domain.com`).
       - Verify **Database name** as `CM_ABC`.
     - Complete the wizard (check `C:\ConfigMgrSetup.log`).
  3. Start SCCM services:
     ```powershell
     Start-Service -Name "SMS_EXECUTIVE"
     ```

### Step 5: Promote SCCM-Passive to Active Site Server
- **Goal**: Switch `SCCM-Passive` to active mode since `SCCM-Active` is down.
- **Steps**:
  1. Open SCCM console (from `SCCM-Passive` or an admin workstation):
     - Connect using `SCCM-Passive` as the server name.
  2. Go to **Administration** > **Site Configuration** > **Servers and Site System Roles**.
  3. Right-click `SCCM-Passive` > **Switch site server to active mode**.
     - If console can’t connect due to `SCCM-Active` being offline, Step 4 may have already made it active.
  4. Verify in **Monitoring** > **Site Server Status**.

### Step 6: Validate the Environment
- **Goal**: Ensure SCCM runs correctly on ` Flowers with the restored database.
- **Steps**:
  1. In SCCM console:
     - **Administration** > **Sites**: Confirm SQL Server name is `SCCM-Passive`.
     - **Assets and Compliance** > **Devices**: Check client data.
     - **Software Library**: Verify applications/packages (if content library is accessible).
  2. Test a client action (e.g., hardware inventory).
  3. Review logs:
     - `sitestat.log`
     - `smsdbmon.log`

### Step 7: Ensure Content Library Access
- **Goal**: Confirm or set up access to the content library.
- **Steps**:
  1. If on `\\NetworkShare\ContentLibrary`:
     - **Administration** > **Sites** > **Properties** > **Content Locations** > Verify or update path.
  2. If local to `SCCM-Active` and inaccessible:
     - Restore from backup or rebuild manually.

---

## Post-Recovery Notes

- **Monitoring**: Regularly check console and logs for stability.
- **Future HA**: Consider SQL Server Always On AG for real-time failover.
- **Backups**: Enable **Backup Site Server** task on `SCCM-Passive` under **Site Maintenance**.

---
