
---

### SCCM Site Recovery: Restoring Site Database Backup to Passive Site Server

#### Purpose
This document provides a detailed procedure to recover a System Center Configuration Manager (SCCM) primary site when the active site server (`SCCM-Active`), which hosts both SCCM and its SQL Server database, becomes unavailable due to failure. The recovery leverages an existing SCCM site database backup (created via the site maintenance task) by restoring it to a SQL Server instance installed on the passive site server (`SCCM-Passive`) and reconfiguring `SCCM-Passive` to operate as the active site server using the restored database.

This is a **manual disaster recovery (DR)** process, not a high-availability (HA) solution like SQL Server Always On Availability Groups (AG). It ensures operational continuity using existing infrastructure when `SCCM-Active` is offline.

#### Scope
- **Environment**: Single SCCM primary site with an active site server (`SCCM-Active`) and a passive site server (`SCCM-Passive`).
- **Scenario**: `SCCM-Active`, hosting both SCCM and SQL Server, is down and unrecoverable.
- **Objective**: Restore the SCCM site database to `SCCM-Passive`, install SQL Server if needed, and configure `SCCM-Passive` to become the active site server using the restored database.

#### Background
- **Current Configuration**: SCCM and SQL Server are co-located on `SCCM-Active`. A failure of this server results in the loss of both the site server and the site database, halting SCCM operations.
- **Backup Source**: The SCCM site maintenance task regularly backs up the site database (e.g., `CM_ABC`), providing a recoverable snapshot.
- **Passive Site Role**: `SCCM-Passive` is pre-configured as a passive site server in the same site, ready to assume the active role with proper configuration.
- **Approach**: This process restores the backup to a new SQL Server instance on `SCCM-Passive`, updates the SCCM configuration to point to this database, and promotes `SCCM-Passive` to active mode.

#### Key Considerations
- **Data Integrity**: The restored database reflects the state at the time of the last backup. Any changes (e.g., new clients, deployments) made after the backup are lost unless transaction log backups are also restored (not covered here).
- **Content Library**: The database contains configuration data, not content (e.g., packages, applications). The content library must be accessible to `SCCM-Passive` via a shared location or restored separately.
- **Manual Effort**: This is a recovery process requiring manual intervention, not an automated failover. For real-time HA, SQL Server Always On AG is recommended (not in scope here).
- **Resource Impact**: Installing SQL Server on `SCCM-Passive` means it will host both SCCM and SQL, potentially straining resources (CPU, memory, disk) during operation.
- **Licensing**: Ensure SQL Server licensing covers installation on `SCCM-Passive` (e.g., using an existing enterprise license).

#### Assumptions
- **SCCM Version**: Current Branch (e.g., 2309 or later).
- **Site Database Name**: `CM_ABC` (replace with your actual site database name).
- **Backup Location**: `\\NetworkShare\SCCMBackup` (a shared folder accessible to both servers).
- **Content Library**: Already relocated to `\\NetworkShare\ContentLibrary` prior to failure, or recoverable from a separate backup.
- **SCCM-Active Status**: Offline and unrecoverable in this scenario.
- **SCCM-Passive Status**: Configured as a passive site server in the same site as `SCCM-Active`.
- **SQL Version**: Matches the version on `SCCM-Active` (e.g., SQL Server 2019 Standard or Enterprise).

#### Prerequisites
1. **Backup Availability**:
   - A recent full backup of the site database from SCCM’s site maintenance task (e.g., `CM_ABC.bak`) stored at `\\NetworkShare\SCCMBackup`.
   - Backup verified as valid and complete prior to `SCCM-Active` failure.
2. **SQL Server Media**:
   - Installation media for SQL Server (e.g., SQL Server 2019 ISO) available for installation on `SCCM-Passive`.
3. **Permissions**:
   - Recovery administrator account:
     - Local administrator on `SCCM-Passive`.
     - Will be granted `sysadmin` rights on the SQL instance post-installation.
   - `SCCM-Passive` computer account (`DOMAIN\SCCM-Passive$`) requires SQL access post-restoration.
4. **SCCM Installation Files**:
   - Access to SCCM installation media (e.g., `D:\SMSSETUP\BIN\X64\setup.exe`) on a network share or local drive on `SCCM-Passive`.
5. **Network Connectivity**:
   - `SCCM-Passive` can access `\\NetworkShare\SCCMBackup` and `\\NetworkShare\ContentLibrary`.
   - Firewall ports open for SQL (TCP 1433) and SCCM communication.
6. **Hardware**:
   - `SCCM-Passive` has sufficient resources (CPU, RAM, disk space) to run both SCCM and SQL Server post-recovery.

---

### Detailed Step-by-Step Procedure

#### Step 1: Install SQL Server on SCCM-Passive
- **Objective**: Deploy a SQL Server instance on `SCCM-Passive` to host the restored site database.
- **Procedure**:
  1. Log in to `SCCM-Passive` using an account with local administrator privileges.
  2. Obtain the SQL Server installation media (e.g., SQL Server 2019 ISO) and mount it or copy it to a local directory (e.g., `D:\SQLSetup`).
  3. Launch the SQL Server setup executable:
     - Run `setup.exe` from the installation media.
     - Select **New SQL Server stand-alone installation or add features to an existing installation**.
  4. Configure installation options:
     - **Feature Selection**: Check **Database Engine Services** (minimum requirement; add other features like SSMS if desired).
     - **Instance Configuration**: Use the default instance (`MSSQLSERVER`) or a named instance (e.g., `SCCM`), matching `SCCM-Active` if known.
     - **Server Configuration**:
       - Service Accounts: Use domain accounts (e.g., `DOMAIN\SQLService`) with local admin rights or accept defaults.
       - Startup Type: Set to **Automatic** for SQL Server and SQL Server Agent.
     - **Database Engine Configuration**:
       - Authentication Mode: Select **Mixed Mode** and set a strong SA password (e.g., match `SCCM-Active` if known).
       - Add the recovery admin account (e.g., `DOMAIN\AdminUser`) as a SQL Server administrator.
       - Collation: Set to `SQL_Latin1_General_CP1_CI_AS` (required for SCCM compatibility).
  5. Complete the installation:
     - Review settings, click **Install**, and wait for completion.
     - Verify SQL Server is running via SQL Server Configuration Manager (services: `MSSQLSERVER` or named instance).
  6. Install SQL Server Management Studio (SSMS) if not included:
     - Download SSMS from Microsoft’s official site and install it on `SCCM-Passive`.

#### Step 2: Restore the Site Database Backup to SCCM-Passive
- **Objective**: Restore the SCCM site database from the backup file to the SQL instance on `SCCM-Passive`.
- **Procedure**:
  1. Copy the latest backup file:
     - Locate `CM_ABC.bak` in `\\NetworkShare\SCCMBackup`.
     - Copy it to a local directory on `SCCM-Passive` (e.g., `C:\SCCMBackup`).
  2. Open SQL Server Management Studio (SSMS):
     - Launch SSMS on `SCCM-Passive`.
     - Connect to the local SQL instance (e.g., `SCCM-Passive` or `SCCM-Passive\SCCM`).
  3. Restore the database:
     - Right-click **Databases** in Object Explorer, select **Restore Database**.
     - In the **Source** section, choose **Device**, click the ellipsis (...), then **Add**.
     - Browse to `C:\SCCMBackup\CM_ABC.bak` and select it.
     - In the **Destination** section, ensure the database name is `CM_ABC`.
     - Go to the **Options** tab:
       - Check **Overwrite the existing database (WITH REPLACE)** if prompted.
       - Ensure recovery state is **RESTORE WITH RECOVERY**.
     - Click **OK** to start the restore process.
  4. Verify the restoration:
     - Confirm `CM_ABC` appears under **Databases** in SSMS.
     - Expand the database and check for key tables (e.g., `CI_ConfigurationItems`) to ensure data integrity.
  5. Configure the recovery model:
     - Right-click `CM_ABC` > **Properties** > **Options**.
     - Set **Recovery model** to **Full** (required for SCCM).
     - Click **OK**.

#### Step 3: Configure SQL Permissions for SCCM-Passive
- **Objective**: Grant the `SCCM-Passive` computer account necessary permissions to access the SQL instance and database.
- **Procedure**:
  1. Open SSMS on `SCCM-Passive` and connect to the local SQL instance.
  2. Create a SQL login for the computer account:
     - Open a new query window and execute:
       ```sql
       USE [master]
       GO
       CREATE LOGIN [DOMAIN\SCCM-Passive$] FROM WINDOWS WITH DEFAULT_DATABASE=[master]
       GO
       ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\SCCM-Passive$]
       GO
       ```
     - Replace `DOMAIN` with your Active Directory domain name.
  3. Grant database access:
     - Execute:
       ```sql
       USE [CM_ABC]
       GO
       CREATE USER [DOMAIN\SCCM-Passive$] FOR LOGIN [DOMAIN\SCCM-Passive$]
       GO
       ALTER ROLE [db_owner] ADD MEMBER [DOMAIN\SCCM-Passive$]
       GO
       ```
  4. Verify permissions:
     - In Object Explorer, expand **Security** > **Logins** and confirm `DOMAIN\SCCM-Passive$` is listed.
     - Expand `CM_ABC` > **Security** > **Users** and verify the account appears with appropriate roles.

#### Step 4: Reconfigure SCCM-Passive to Use the Restored Database
- **Objective**: Update `SCCM-Passive`’s configuration to point to the restored database on its local SQL instance.
- **Procedure**:
  1. Stop SCCM services:
     - Open PowerShell as Administrator on `SCCM-Passive`.
     - Run:
       ```powershell
       Stop-Service -Name "SMS_EXECUTIVE" -Force
       ```
     - Verify the service is stopped via `Get-Service SMS_EXECUTIVE`.
  2. Launch SCCM setup:
     - Locate the SCCM installation media (e.g., `D:\SMSSETUP\BIN\X64\setup.exe`).
     - Run `setup.exe` with administrative privileges.
     - Select **Perform site maintenance or reset this site**.
  3. Update database configuration:
     - Proceed through the wizard until the **Database Information** page.
     - Set **SQL Server name** to `SCCM-Passive` (or its FQDN, e.g., `SCCM-Passive.domain.com`).
     - Verify **Database name** is `CM_ABC`.
     - Ensure the SQL Server port (default: 1433) is correct.
     - Click **Next**, then **Test Connection** to confirm connectivity.
  4. Complete the setup:
     - Proceed through remaining steps, accepting defaults unless specific changes are needed.
     - Click **Begin Install** and monitor progress.
     - Check `C:\ConfigMgrSetup.log` for successful completion (look for “Setup completed successfully”).
  5. Restart SCCM services:
     - Run:
       ```powershell
       Start-Service -Name "SMS_EXECUTIVE"
       ```
     - Verify the service is running via `Get-Service SMS_EXECUTIVE`.

#### Step 5: Promote SCCM-Passive to Active Site Server
- **Objective**: Officially designate `SCCM-Passive` as the active site server since `SCCM-Active` is unavailable.
- **Procedure**:
  1. Open the SCCM console:
     - Launch the console from `SCCM-Passive` or an administrative workstation.
     - Connect to the site using `SCCM-Passive` as the server name (e.g., `SCCM-Passive.domain.com`).
  2. Promote the passive server:
     - Navigate to **Administration** > **Site Configuration** > **Servers and Site System Roles**.
     - Locate `SCCM-Passive` (listed as “Site server in passive mode”).
     - Right-click and select **Switch site server to active mode**.
     - Confirm the action and wait for the switch to complete.
     - Note: If the console cannot connect due to `SCCM-Active` being offline, the setup in Step 4 may have already made `SCCM-Passive` active by default.
  3. Verify the switch:
     - Go to **Monitoring** > **Site Server Status**.
     - Confirm `SCCM-Passive` is listed as “Active” and `SCCM-Active` is no longer present or marked as offline.

#### Step 6: Validate SCCM Functionality
- **Objective**: Ensure the SCCM site operates correctly on `SCCM-Passive` with the restored database.
- **Procedure**:
  1. Check console functionality:
     - In the SCCM console:
       - **Administration** > **Site Configuration** > **Sites**: Verify the site properties list `SCCM-Passive` as the SQL Server name.
       - **Assets and Compliance** > **Devices**: Confirm client data loads (may take time for clients to reconnect).
       - **Software Library** > **Applications**: Ensure applications and packages are listed (assuming content library access).
  2. Test client operations:
     - Trigger a hardware inventory or application deployment on a test client.
     - Monitor **Monitoring** > **Client Status** for activity.
  3. Review key logs:
     - `C:\Program Files\Microsoft Configuration Manager\Logs\sitestat.log`: Confirms site server status.
     - `C:\Program Files\Microsoft Configuration Manager\Logs\smsdbmon.log`: Verifies database connectivity.
     - `C:\Program Files\Microsoft Configuration Manager\Logs\sitecomp.log`: Checks component status post-recovery.
  4. Address any errors:
     - If issues arise (e.g., clients not reporting), ensure network connectivity and DNS resolution for `SCCM-Passive`.

#### Step 7: Configure Content Library Access
- **Objective**: Ensure `SCCM-Passive` can access the SCCM content library for software distribution.
- **Procedure**:
  1. Verify existing content library:
     - If previously moved to `\\NetworkShare\ContentLibrary`:
       - In the SCCM console, go to **Administration** > **Site Configuration** > **Sites** > **Properties** > **Content Locations**.
       - Confirm the path is set to `\\NetworkShare\ContentLibrary`.
       - Test access by browsing the share from `SCCM-Passive`.
  2. Recover content library if inaccessible:
     - If the content library was local to `SCCM-Active` and lost:
       - Restore it from a separate backup to `\\NetworkShare\ContentLibrary` using the **Content Library Transfer** tool (`ContentLibraryTransfer.exe`) if available.
       - Alternatively, recreate packages and applications manually (time-intensive).
     - Update the content location in the console as above.
  3. Validate content:
     - Deploy a test application to a client and confirm it downloads from the content library.

#### Step 8: Post-Recovery Configuration
- **Objective**: Ensure ongoing stability and preparedness for future incidents.
- **Procedure**:
  1. Enable site backup task:
     - In the SCCM console, go to **Administration** > **Site Configuration** > **Sites** > **Site Maintenance**.
     - Enable the **Backup Site Server** task.
     - Set the backup path to `\\NetworkShare\SCCMBackup` and schedule it (e.g., daily at 2:00 AM).
  2. Update documentation:
     - Record `SCCM-Passive` as the new active site server in your environment documentation.
  3. Notify stakeholders:
     - Inform SCCM administrators and end-users of the recovery and any temporary disruptions (e.g., client reconnection delays).

---

### Post-Recovery Recommendations
- **Monitoring**: Regularly review the SCCM console (**Monitoring** workspace) and logs for stability issues, as `SCCM-Passive` now runs both SCCM and SQL.
- **High Availability**: Consider implementing SQL Server Always On AG with a separate SQL server (e.g., `SQL-Passive`) for real-time failover in the future to avoid reliance on manual recovery.
- **Resource Assessment**: Monitor `SCCM-Passive`’s performance (CPU, memory, disk I/O) and scale hardware if needed to handle both SCCM and SQL workloads.
- **Client Reconnection**: Clients may need time to rediscover `SCCM-Passive` as the active site server; ensure DNS and network configurations are updated if necessary.

---

### Troubleshooting
- **Database Restore Fails**: Verify the backup file is not corrupted (test restore on another server) and ensure the SQL instance has sufficient disk space.
- **SCCM Setup Errors**: Check `C:\ConfigMgrSetup.log` for detailed error codes and ensure the `SCCM-Passive$` account has `sysadmin` rights.
- **Clients Not Reporting**: Confirm firewall ports (e.g., TCP 80, 443, 10123) are open and clients can resolve `SCCM-Passive`’s FQDN.
- **Content Library Issues**: If content is missing, validate the share path and permissions (`Everyone` or `SCCM-Passive$` should have read access).

---
