### Structured Notes on Config Manager High Availability (HA) Feature

---

#### **Introduction**
- **Presenter**: Justin Shelton, Founder and Engineering Lead at Patch My PC.
- **Topic**: High Availability (HA) feature of the Site Server in Config Manager (introduced in version 1806).
- **Objective**: Understand the role of the Site Server, the impact of its failure, and how to implement HA for improved reliability.

---

### **Key Concepts**

#### **1. Role of the Site Server**
- **Primary Functions**:
  - Processes client data (e.g., hardware inventory, software inventory, update compliance).
  - Manages the **Content Library** (source for distributing content to Distribution Points).
  - Runs **Site Component Manager** (installs remote site systems).
- **Impact of Site Server Failure**:
  - Client-facing roles (e.g., Management Point, Software Update Point) continue to communicate with clients, but data processing stops.
  - **State messages** (e.g., software update compliance) are not processed into the database.
  - Content distribution and remote site system installations halt.

#### **2. High Availability (HA) in Config Manager**
- **Versions**:
  - **1806**: Supported only standalone primary sites.
  - **1810**: Extended support to primary sites within a hierarchy (CAS or child primary sites).
- **Prerequisites**:
  - **Remote Content Library**: Required for HA to allow both active and passive servers to access the same content.
  - **SQL Server High Availability**: 
    - Use a remote SQL Server with clustering or SQL AlwaysOn.
    - **SQL AlwaysOn** (introduced in 1810): Allows clustering of the site database on both active and passive servers.
- **Key Improvements in 1810**:
  - SQL AlwaysOn support for site database HA.
  - Simplified setup by allowing the site database to reside on the site servers.

#### **3. Lab Setup**
- **Scenario**:
  - Remote SQL Server (not using SQL AlwaysOn for simplicity).
  - Two client-facing site systems (Management Point, Distribution Point, Software Update Point).
  - Active Site Server (Windows Server 2016).
  - Passive Site Server (Windows Server 2019).
- **Key Points**:
  - **Distribution Point (DP) Role**: Cannot be installed on a site server configured for HA. Requires remote site systems for DP roles.
  - **Future Enhancements**: Potential support for multiple active site servers to offload processing.

---

### **Step-by-Step Implementation**

#### **1. Prepare the New Passive Site Server**
- **Step 1: Add Computer Account to AD Security Group**
  - Add the new passive site server’s computer account to the **Site Servers** security group in Active Directory.
  - This grants permissions to the **System Management** container and simplifies local admin rights management.
  - **Action**: 
    - Open **Active Directory Users and Computers**.
    - Navigate to the **Site Servers** security group.
    - Add the new server’s computer account (e.g., `SCCM-Server2$`).
  - **Reboot**: Reboot the new server to apply group membership changes.

- **Step 2: Grant SQL Server Permissions**
  - **Local Admin Rights**: Ensure the new server’s computer account has local admin rights on the SQL Server.
    - **Action**: 
      - Open **Computer Management** on the SQL Server.
      - Add the **Site Servers** security group or the specific computer account to the **Local Administrators** group.
  - **SQL SysAdmin Rights**: Grant sysadmin rights to the SQL instance.
    - **Action**:
      - Open **SQL Server Management Studio**.
      - Navigate to **Security > Logins**.
      - Add the new server’s computer account (e.g., `DOMAIN\SCCM-Server2$`) and assign **sysadmin** role.

- **Step 3: Install Prerequisites**
  - **.NET Framework 3.5**:
    - **Action**:
      - Open **Server Manager**.
      - Go to **Add Roles and Features**.
      - Install **.NET Framework 3.5** (provide source path if offline, e.g., `D:\sources\sxs`).
  - **WSUS APIs**:
    - **Action**:
      - Run PowerShell command: 
        ```powershell
        Install-WindowsFeature -Name UpdateServices-API
        ```
  - **Remote Differential Compression**:
    - **Action**:
      - Run PowerShell command:
        ```powershell
        Install-WindowsFeature -Name RDC
        ```
  - **Windows 10 ADK**:
    - Install **Windows 10 ADK (1809)**:
      - **Components**: Deployment Tools, User State Migration Tool (USMT), Windows PE.
      - **Action**:
        - Download and install ADK from Microsoft.
        - Install **Windows PE** separately (required for boot images).

- **Step 4: Grant Local Admin Rights**
  - Ensure the **Site Servers** security group has local admin rights on the new passive site server.
    - **Action**:
      - Open **Computer Management** on the new server.
      - Add the **Site Servers** group to the **Local Administrators** group.

- **Step 5: Reboot**
  - Reboot the new server to apply pending changes (e.g., .NET installation).

---

#### **2. Move Content Library to Remote UNC Path**
- **Step 1: Create and Share Content Library Folder**
  - **Action**:
    - On a remote server, create a folder (e.g., `ContentLibrary`).
    - Share the folder with **Full Control** permissions.
    - Grant **NTFS permissions** to the **Site Servers** security group or the specific computer accounts.

- **Step 2: Move Content Library**
  - **Action**:
    - Open **Config Manager Console**.
    - Navigate to **Administration > Site Configuration > Sites**.
    - Select the site and click **Manage Content Library**.
    - Specify the remote UNC path (e.g., `\\RemoteServer\ContentLibrary\SitePR3`).
    - Monitor progress in the **distmgr.log** file.

---

#### **3. Add Passive Site Server**
- **Step 1: Create New Site System Server**
  - **Action**:
    - Open **Config Manager Console**.
    - Navigate to **Administration > Site Configuration > Servers and Site System Roles**.
    - Click **Create Site System Server**.
    - Enter the new server’s name (e.g., `SCCM-Server2`).
    - Select the site and move it to the **Selected** column.
    - Click **Next**.

- **Step 2: Install Site Server in Passive Mode**
  - **Action**:
    - Check the option **Install the site server in passive mode**.
    - Specify the installation directory (e.g., `D:\Microsoft Configuration Manager`).
    - Choose to copy installation files from the active site server.
    - Click **Next** and complete the wizard.

- **Step 3: Monitor Installation**
  - **Action**:
    - Navigate to **Monitoring > Site Server Status**.
    - Select the new server and click **Show Status**.
    - Monitor prerequisite checks and installation progress.
    - Review logs:
      - **distmgr.log**: Content library movement.
      - **fellovermgr.log**: Failover process.
      - **ConfigMgrSetup.log**: Core installation.
      - **sitecomp.log**: Component installation.

---

#### **4. Install SMS Provider on Passive Site Server**
- **Step 1: Verify SMS Provider Installation**
  - **Action**:
    - Open **Config Manager Console**.
    - Navigate to **Administration > Site Configuration > Servers and Site System Roles**.
    - Verify the SMS Provider is installed on the passive site server.
    - If not registered correctly, reinstall the provider.

- **Step 2: Reinstall SMS Provider**
  - **Action**:
    - Open **Config Manager Setup** on the active site server.
    - Choose **Modify SMS Provider Configuration**.
    - Add the passive site server and reinstall the provider.
    - Monitor progress in the **SMSProvider.log**.

---

#### **5. Promote Passive Site Server to Active**
- **Step 1: Manual Promotion**
  - **Action**:
    - Open **Config Manager Console**.
    - Navigate to **Administration > Site Configuration > Sites**.
    - Select the site and click **Nodes**.
    - Right-click the passive server and choose **Promote to Active**.
    - Monitor progress in **fellovermgr.log**.

- **Step 2: Unplanned Failover**
  - If the active server is offline for 30 minutes, the passive server automatically promotes itself to active.

---

### **Quick Reference**

| **Step**                          | **Action**                                                                 |
|------------------------------------|----------------------------------------------------------------------------|
| Add Computer Account to AD Group   | Add to **Site Servers** security group in AD.                             |
| Grant SQL Permissions              | Local admin and sysadmin rights on SQL Server.                            |
| Install Prerequisites              | .NET 3.5, WSUS APIs, Remote Differential Compression, ADK.                |
| Move Content Library               | Use **Manage Content Library** in the console.                            |
| Add Passive Site Server            | Use **Create Site System Server** and select **Passive Mode**.             |
| Install SMS Provider               | Reinstall if not registered correctly.                                     |
| Promote to Active                  | Use **Promote to Active** in the console or wait for automatic failover.   |

---

### **Best Practices**
- **Test Failover**: Regularly test manual and unplanned failover to ensure reliability.
- **Monitor Logs**: Use logs (`distmgr.log`, `fellovermgr.log`, `sitecomp.log`) for troubleshooting.
- **Future Enhancements**: Keep an eye on updates for multiple active site server support.

---

These structured notes provide a comprehensive guide to implementing Config Manager High Availability, ensuring a reliable and resilient site server setup.
