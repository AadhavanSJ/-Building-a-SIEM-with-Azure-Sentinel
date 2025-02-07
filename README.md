# SIEM & Honeypot | Microsoft Azure Sentinel Attack Map

## Summary
SIEM (Security Information and Event Management) helps organizations detect, analyze, and respond to security threats before they cause harm. It collects event log data from sources like Firewalls, IDS/IPS, and Identity solutions, enabling security teams to monitor and mitigate threats in real-time.

A **honeypot** is a security mechanism that simulates a vulnerable system to attract and analyze cyberattacks in a controlled environment. This lab demonstrates how to collect honeypot attack log data, query it in a **SIEM (Microsoft Azure Sentinel)**, and display it on a world map based on event count and geolocation.

---

## Learning Objectives
- Configure and deploy **Azure** resources: Virtual Machines, Log Analytics Workspaces, and Azure Sentinel
- Gain hands-on experience with **Microsoft Azure Sentinel (SIEM)**
- Understand **Windows Security Event logs**
- Use **Kusto Query Language (KQL)** to query logs
- Display attack data on a **world map dashboard** using Workbooks

---

## Tools & Requirements
- **Microsoft Azure Subscription** (Free $200 Credit for 30 days)
- **Azure Sentinel**
- **Kusto Query Language (KQL)**
- **Network Security Groups (NSG)**
- **Remote Desktop Protocol (RDP)**
- **3rd Party API:** [ipgeolocation.io](https://ipgeolocation.io)
- **Custom PowerShell Script by Josh Madakor**

---

## Lab Steps

### Step 1: Create a Microsoft Azure Subscription  
[Sign up for Azure](https://azure.microsoft.com/free) (Free $200 Credit for 30 days).

---

### Step 2: Create a Honeypot Virtual Machine  
1. Go to [Azure Portal](https://portal.azure.com)
2. Search for **"Virtual Machines"**  
3. Click **Create > Azure virtual machine**  
4. Configure:
   - **Resource Group:** `honeypotlab`
   - **VM Name:** `honeypot-vm`
   - **Region:** `(US) East US 2`
   - **OS Image:** Windows 10 Pro, version 21H2  
   - **Size:** `Standard_D2s_v3 (2 vCPUs, 8 GB RAM)`
   - **Public Inbound Ports:** Allow **RDP (3389)**
5. **Networking:**
   - **NIC Security Group:** Create new **NSG**
   - **Inbound Rule:** `ALLOW_ALL_INBOUND` (Allow all traffic)
6. Click **Review + Create**

⚠️ *This configuration makes the VM discoverable by attackers.*

---

### Step 3: Create a Log Analytics Workspace  
1. Search **"Log Analytics Workspaces"**  
2. Click **Create Log Analytics Workspace**  
3. Configure:
   - **Resource Group:** `honeypotlab`
   - **Workspace Name:** `honeypot-log`
   - **Region:** `East US 2`
4. Click **Review + Create**

---

### Step 4: Configure Microsoft Defender for Cloud  
1. Search **"Microsoft Defender for Cloud"**  
2. Go to **Environment settings > Subscription > Log Analytics Workspace (honeypot-log)**  
3. Enable:
   - **Cloud Security Posture Management**
   - **Servers**
4. Under **Data Collection**, select **"All Events"** and **Save**.

---

### Step 5: Connect Log Analytics Workspace to Virtual Machine  
1. Go to **Log Analytics Workspaces > honeypot-log > Virtual Machines**  
2. Select **honeypot-vm** and **Connect**.

---

### Step 6: Configure Microsoft Sentinel  
1. Search **"Microsoft Sentinel"**  
2. Click **Create Microsoft Sentinel**  
3. Select **Log Analytics Workspace:** `honeypot-log`  
4. Click **Add**.

---

### Step 7: Disable Windows Firewall in Virtual Machine  
1. Copy **honeypot-vm IP Address** from Azure  
2. **RDP into the VM** using credentials from Step 2  
3. Open **Windows Defender Firewall** (`wf.msc`)  
4. Turn **OFF** firewall for:
   - Domain Profile
   - Private Profile
   - Public Profile  
5. Click **Apply & OK**  
6. From **Host Machine**, verify connectivity:  
   ```sh
   ping -t <VM-IP>
## Step 8: Setup Security Log Exporter Script  
1. In VM, open **PowerShell ISE**  
2. Save the powershell scripit as `Log_Exporter.ps1`  
3. Create a free account at [ipgeolocation.io](https://ipgeolocation.io)  
4. Copy API Key and paste it into script:  
   ```powershell
   $API_KEY = "<YOUR_API_KEY>"
   ```
5. Run the script (Green Play Button in PowerShell ISE).  

🔹 **Script Purpose:**  
- Exports Windows Event Viewer logs  
- Uses IP Geolocation API to map attack source  
- Saves data to `C:\ProgramData\failed_rdp.log`  

## Step 9: Create Custom Log in Log Analytics Workspace  
1. Search **Run** in VM  
2. Open `C:\ProgramData\failed_rdp.log`  
3. Copy all data and save it on Host PC as `failed_rdp.log`  
4. In Azure, go to:  
   - **Log Analytics Workspaces > honeypot-log > Custom Logs**  
   - Click **Add Custom Log**  
   - Upload `failed_rdp.log`  
   - Configure:  
     - **Log Type:** Windows  
     - **Collection Path:** `C:\ProgramData\failed_rdp.log`  
     - **Log Name:** `FAILED_RDP_WITH_GEO`  
   - Click **Create**.  

## Step 10: Query the Custom Log  
1. In **Log Analytics Workspaces**, select `honeypot-log > Logs`  
2. Run the query:  
   ```kql
   FAILED_RDP_WITH_GEO_CL
   ```
3. Wait for data ingestion.  

## Step 11: Extract Fields from Custom Log  
1. Right-click `FAILED_RDP_WITH_GEO_CL` results  
2. Select **Extract Fields**  
3. Highlight values (e.g., `latitude`, `longitude`, `country`, `sourcehost`)  
4. Name and assign the appropriate **Field Type**  
5. Click **Save Extraction**.  

## Step 12: Display Data on a World Map in Microsoft Sentinel  
1. Go to **Microsoft Sentinel > Workbooks > Add Workbook**  
2. Click **Edit**  
3. Add a **Query Widget** and paste:  
   ```kql
   FAILED_RDP_WITH_GEO_CL
   | summarize event_count=count() by sourcehost_CF, latitude_CF, longitude_CF, country_CF, label_CF, destinationhost_CF
   | where destinationhost_CF != "samplehost"
   | where sourcehost_CF != ""
   ```
4. Select **Visualization: Map**  
5. Configure:  
   - **Location Info:** Latitude/Longitude  
   - **Color Type:** Heatmap  
   - **Metric Label:** `label_CF`  
   - **Metric Value:** `event_count`  
6. Click **Save as "Failed RDP World Map"**.  

## Step 13: Deprovision Resources 🚨  
1. Go to **Azure Portal > Resource Groups**  
2. Select `honeypotlab`  
3. Click **Delete Resource Group**  
4. Confirm by typing `honeypotlab`  
5. Check **"Apply force delete for VM"**  
6. Click **Delete**.  

⚠️ **Important Notes**  
- Failure to delete resources may incur **Azure charges**!  
- **DO NOT** use honeypots in production environments.  

🚀 **Happy Threat Hunting!**
