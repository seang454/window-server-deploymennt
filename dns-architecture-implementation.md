# DNS Architecture Implementation Guide (with Technical Rationale)

**Target Environment:** Windows Server 2019 / 2022 / 2025 on VMware Workstation  
**Complementary Document:** [dns-architecture.md](file:///D:/RUPPClass/year4/window-server/project-note/dns-architecture.md) (Theory, Diagrams, and Threat Model)  
**Deliverable:** Working Two-Tier DNS with Ad-Blocking (AdGuard Docker) and Local Resolution (Native Windows DNS)

---

### Pre-Implementation Checklist

| Requirement | Value / Target (NAT Mode - Recommended) | Value / Target (Bridged Mode) |
| :--- | :--- | :--- |
| **Physical Host OS** | Windows 10 / 11 with VMware Workstation | Windows 10 / 11 with VMware Workstation |
| **Guest Virtual Machine** | Windows Server (2019 / 2022 / 2025) | Windows Server (2019 / 2022 / 2025) |
| **VMware Network Mode** | **NAT (VMnet8)** [Works anywhere, immune to Wi-Fi changes] | **Bridged (VMnet0)** [Direct L2 connection to physical LAN] |
| **Static IP for Windows Server**| **`192.168.1.10`** | **`192.168.100.50`** (or matching router subnet) |
| **Default Gateway** | **`192.168.1.1`** (VMware Virtual NAT Gateway) | **`192.168.100.1`** (Physical Home Router) |
| **Client Devices** | Virtual Machine clients (`pro-win-client`, `pro-win-client2`) | Physical hardware (Smartphones, Smart TVs, external PCs) |
| **Docker Engine on Server** | Docker Desktop with WSL2 backend | Docker Desktop with WSL2 backend |

---

## Phase 0: VMware Workstation & Physical Host Pre-Configuration

To run Docker (WSL2) inside a Windows Server VM running on VMware Workstation, you are setting up **Nested Virtualization** ("a VM inside a VM"). 

Because modern Windows 11 hosts run **Virtualization-Based Security (VBS)** and **Hyper-V** by default, Windows locks the CPU's hardware virtualization extensions (VT-x). If you try to enable VT-x in VMware without unlocking the host first, VMware will fail to boot with:
`Feature 'hv.capable' was 0, but must be at least 0x1. Module 'FeatureCompatLate' power on failed.`

Follow these sub-phases carefully.

---

### Phase 0.1: Unlock Hardware Virtualization on Physical Host (Laptop)

> [!WARNING]
> **Host Linux / Docker Impact:**  
> When you disable the host hypervisor lock, WSL2 (Ubuntu) and Docker Desktop on your **physical laptop** will temporarily be unable to start. They will resume working normally once you restore the settings (see **Phase 0.4** below).

1. **Turn Off Memory Integrity (Core Isolation):**
   * On your physical host PC, open the Start Menu and search for **Core isolation** (or open **Windows Security** ➔ **Device Security** ➔ **Core isolation details**).
   * Toggle **Memory integrity** to **OFF**.
2. **Disable Host Hyper-V Boot Lock:**
   * Right-click the Start Menu on your physical PC and open **PowerShell as Administrator** (or Windows Terminal Admin).
   * Run the following command:
     ```powershell
     bcdedit /set hypervisorlaunchtype off
     ```
   * Ensure it returns: *"The operation completed successfully."*
3. **Boot into Windows 11.** VMware Workstation now has direct hardware access to Intel VT-x.

---

### Phase 0.2: Configure VMware Workstation VM Settings

1. Ensure the Windows Server VM is **Powered Off**.
2. Right-click the VM ➔ Select **Settings**.
3. **Hardware ➔ Processors:**
   * Check: ✅ **Virtualize Intel VT-x/EPT or AMD-V/RVI**.
   * *(Do not check CPU performance counters or IOMMU unless specifically required).*
4. **Hardware ➔ Network Adapter (Choose Deployment Mode):**
   * **Choice A: NAT Mode (`VMnet8`) [HIGHLY RECOMMENDED FOR LABS / LAPTOPS]:**
     * Select **NAT: Used to share the host's IP address**.
     * *(Benefits: Never breaks when switching between Home, School/Campus, or Cafe Wi-Fi. Bypasses 802.1X and captive portals).*
   * **Choice B: Bridged Mode (`VMnet0`) [FOR PHYSICAL HOME DEVICES]:**
     * Select **Bridged: Connected directly to the physical network**.
     * Check **Replicate physical network connection state**.
     * *Important:* In VMware main window, open **Edit ➔ Virtual Network Editor ➔ Change Settings (Admin)** ➔ Select **VMnet0** ➔ Change **Bridged to:** from *Automatic* to your specific physical Wi-Fi card (e.g., `Intel(R) Wi-Fi 6E AX211 160MHz`).
5. Power on the Windows Server VM. It will now boot up cleanly with nested virtualization active.

---

### Phase 0.3: Alternative - Run AdGuard Home Natively (No Host Changes Needed)

> [!TIP]
> If you do not want to disable Hyper-V on your physical laptop because you actively use Docker/Ubuntu on your host, you can run **AdGuard Home as a native Windows service** directly on Windows Server instead of inside Docker.  
> 1. Download `AdGuardHome_windows_amd64.zip` inside the VM.  
> 2. Extract to `C:\AdGuardHome`.  
> 3. Run `.\AdGuardHome.exe -s install` in PowerShell.  
> This requires **no nested virtualization (VT-x can stay OFF)** and uses only ~30 MB RAM while achieving the exact same ad-blocking and DoH results.

---

### Phase 0.4: How to Revert & Restore Linux (WSL2 / Docker) on Your Physical Laptop

When you have finished your Windows Server lab and want your physical host's WSL2, Ubuntu, and Docker Desktop to work again:

1. **Power off the Windows Server VM** in VMware.
2. In VMware VM Settings ➔ **Processors** ➔ **UNCHECK** `Virtualize Intel VT-x/EPT or AMD-V/RVI`.  
   *(If you leave this checked while the host hypervisor is re-enabled, VMware will show the `hv.capable` error again).*
3. Open **PowerShell as Administrator** on your physical laptop and run:
   ```powershell
   bcdedit /set hypervisorlaunchtype auto
   ```
4. (Optional) In **Windows Security ➔ Device Security ➔ Core isolation details**, turn **Memory integrity** back to **ON**.
5. **Restart your physical laptop.**
6. Once rebooted, launch **Ubuntu (WSL2)** or **Docker Desktop** on your physical laptop. They will start normally!

---

### 💡 Why we do this (Technical Rationale):
* **Why enable "Virtualize Intel VT-x/EPT"?**  
  AdGuard Home is packaged as a Linux-based container. Running Docker Desktop on Windows Server requires a Linux VM backend (WSL2 or Hyper-V). Because your Windows Server is *already* a virtual machine inside VMware, running Docker creates a "VM inside a VM" (**Nested Virtualization**). If you don't pass VT-x into VMware, Windows Server cannot run the WSL2 Linux kernel.
* **Why `bcdedit /set hypervisorlaunchtype off` is necessary on Windows 11?**  
  Windows 11 utilizes the Microsoft Hyper-V hypervisor for system security (VBS) and WSL2. Hyper-V monopolizes CPU VT-x instructions at Ring -1. Setting `hypervisorlaunchtype off` releases this lock so VMware Workstation can access VT-x directly.
* **Why Bridged Mode must be mapped manually?**  
  VMware's "Automatic" bridge detection often binds to secondary virtual adapters (such as VPN tunnels, Tailscale, or disconnected Ethernet ports) rather than the active Wi-Fi card. Manually pinning VMnet0 ensures stable layer-2 connectivity to your home/lab router.

---

## Phase 1: Set Static IP on Windows Server

A DNS server must always maintain a fixed IP address. Choose the configuration matching your VMware network adapter setting from Phase 0.2.

---

### Choice A: For NAT Mode (`VMnet8`) **[Recommended for Labs & Laptops]**

Use this mode if your VMware network adapter is set to **NAT**. It provides a fixed, reliable subnet that never breaks when moving between Home, School/Campus, or Cafe Wi-Fi.

#### 1. Via PowerShell
```powershell
# Get active network adapter interface alias (usually "Ethernet0")
Get-NetAdapter

# Set Static IP (192.168.1.10) and VMware NAT Gateway (192.168.1.1)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.1.10 -PrefixLength 24 -DefaultGateway 192.168.1.1

# Set temporary public DNS (1.1.1.1 / 8.8.8.8) so VM has internet for downloading Docker
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("1.1.1.1", "8.8.8.8")
```

#### 2. Via GUI (`ncpa.cpl`)
1. Open **Network Connections** (`ncpa.cpl`).
2. Right-click your Ethernet adapter ➔ **Properties** ➔ Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
3. Configure:
   * **IP address:** `192.168.1.10`
   * **Subnet mask:** `255.255.255.0`
   * **Default gateway:** `192.168.1.1` (VMware Virtual NAT Gateway)
   * **Preferred DNS server:** `1.1.1.1` *(switch to `127.0.0.1` after Phase 4)*
   * **Alternate DNS server:** `8.8.8.8`
4. Click **OK** ➔ **OK**.

---

### Choice B: For Bridged Mode (`VMnet0`) **[For Fixed Home Wi-Fi]**

Use this mode ONLY if your VMware network adapter is set to **Bridged** and your laptop is on your fixed home Wi-Fi network.

#### 1. Via PowerShell
```powershell
# Set Static IP on home subnet and Home Router Gateway
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.50 -PrefixLength 24 -DefaultGateway 192.168.100.1

# Set temporary public DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("1.1.1.1", "8.8.8.8")
```

#### 2. Via GUI (`ncpa.cpl`)
* **IP address:** `192.168.100.50`
* **Subnet mask:** `255.255.255.0`
* **Default gateway:** `192.168.100.1` (Home Router IP)
* **Preferred DNS server:** `1.1.1.1` *(switch to `127.0.0.1` after Phase 4)*
* **Alternate DNS server:** `8.8.8.8`

---

### 💡 Why we do this (Technical Rationale):
* **Why a Static IP is Mandatory:**  
  If the server used dynamic DHCP, its IP address could change after reboot. Any client VM (`pro-win-client`) pointing to that DNS IP would immediately lose internet and domain name resolution.
* **Why use temporary public DNS (`1.1.1.1`) before switching to `127.0.0.1`?**  
  In Phase 1, the Windows DNS Server role and AdGuard Home container are not yet running. If you point DNS to `127.0.0.1` immediately, the server queries itself, receives no reply, and loses internet access (blocking you from downloading Docker Desktop and WSL packages). Once Phase 4 is completed, DNS is switched to `127.0.0.1`.

---

## Phase 2: Install Native Windows DNS Server Role

### 1. Execution Steps

#### Option A: Via PowerShell (Recommended)
```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
```

#### Option B: Via Server Manager GUI
1. Open **Server Manager** ➔ Click **Add roles and features**.
2. Advance through wizard until **Server Roles**.
3. Check the box for **DNS Server** (Click **Add Features** when prompted).
4. Click **Next** ➔ **Next** ➔ **Install**.
5. Once complete, verify the service is running:
   ```powershell
   Get-Service -Name DNS
   ```

### 💡 Why we do this (Technical Rationale):
* **Why Native Windows DNS instead of just using Docker?**  
  Native Windows DNS is fully integrated into the Windows kernel and Active Directory. It supports:
  * **Dynamic DNS (DDNS):** Computers register their names automatically upon boot.
  * **SRV & Kerberos Records:** Windows clients query `_ldap._tcp.dc._msdcs.<domain>` to find Domain Controllers for login authentication. Third-party DNS servers cannot manage these records reliably without breaking domain logins.

---

## Phase 2.5: Install Docker on Windows Server

Because AdGuard Home is packaged as a Linux container, Docker needs a lightweight Linux engine to execute it.

### 1. Execution Steps (Docker Desktop with WSL2 Backend)

#### Step 1: Enable Virtual Machine Platform & Containers Features

You can enable these built-in features using either the Graphical User Interface (GUI) or PowerShell:

##### Option 1: Via Server Manager GUI (Visual Method)
1. Open **Server Manager** (from the Start Menu or Taskbar).
2. In the top-right corner, click **Manage** ➔ select **Add Roles and Features**.
3. Click **Next** through:
   * **Before You Begin** ➔ Click **Next**
   * **Installation Type** ➔ Select *Role-based or feature-based installation* ➔ Click **Next**
   * **Server Selection** ➔ Select your local server ➔ Click **Next**
4. **Server Roles:** Click **Next** (no changes needed here).
5. **Features (Important):**
   * Scroll down the list and check the box for **`Containers`**.
   * When prompted, click **Add Features** to include management tools.
6. **Confirmation:**
   * Check the box: **"Restart the destination server automatically if required"**.
   * Click **Install**.
7. Once finished, restart the server.

##### Option 2: Via Classic Control Panel GUI (`appwiz.cpl`)
1. Press `Win + R` on your keyboard, type **`appwiz.cpl`**, and press **Enter**.
2. In the left sidebar, click **"Turn Windows features on or off"**.
3. In the feature tree, check:
   * ✅ **Containers**
   * ✅ **Virtual Machine Platform** (or **Hyper-V Platform**)
4. Click **OK**, let Windows apply changes, and click **Restart Now**.

##### Option 3: Via PowerShell (Fastest - One Command)
Open **PowerShell as Administrator**:
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All -NoRestart
Enable-WindowsOptionalFeature -Online -FeatureName Containers -All -NoRestart

# Restart server to initialize hypervisor features
Restart-Computer
```

#### Step 2: Install WSL2 Linux Kernel
After the VM restarts, open PowerShell as Administrator:
```powershell
wsl --install --no-distribution
wsl --update
```

#### Step 3: Download & Install Docker Desktop

##### Option A: Via Microsoft Edge Browser (Standard GUI Method)
1. Inside the Windows Server VM, open **Microsoft Edge**.
2. Navigate to: `https://www.docker.com/products/docker-desktop/`
3. Click the blue button: **Download for Windows** (downloads `Docker Desktop Installer.exe`, ~606 MB).
4. Once downloaded, open your **Downloads** folder and double-click `Docker Desktop Installer.exe`.

##### Option B: Via PowerShell (Automated Download)
Open PowerShell as Administrator:
```powershell
Invoke-WebRequest -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" -OutFile "$env:TEMP\DockerDesktopInstaller.exe"
Start-Process "$env:TEMP\DockerDesktopInstaller.exe" -Wait
```

##### Installer Configuration Screen (What to choose):
When the **"Configuration"** window appears:
* **Option 1 (Default & Recommended):** Keep **`Per-user installation (Recommended)`** selected. This automatically uses the **WSL 2 backend**.
* **Option 2 (All Users):** If you select **`All-users installation`**, make sure the checkbox **"Use WSL 2 instead of Hyper-V"** is **CHECKED** (do NOT check "Allow Windows Containers").
* Keep **"Add shortcut to desktop"** checked.
* Click **OK**.

##### Post-Installation:
1. Wait 2–3 minutes while packages unpack and install.
2. When the installation completes, click **Close and restart** (or sign out and log back in).
3. Once back on your desktop, double-click the **Docker Desktop** shortcut.
4. Accept the Docker Service Agreement. Docker Desktop will start up with its green engine indicator!

#### Step 4: Verify Docker Engine
Open PowerShell and verify:
```powershell
docker --version
docker compose version
docker info
```

### 💡 Why we do this (Technical Rationale):
* **Why Docker for AdGuard Home?**  
  Running AdGuard in Docker isolates it from the Windows host OS. If you want to update AdGuard, you simply pull the new image—no registry edits, no system file corruption, and no complicated Linux installation.
* **Why WSL2 instead of Hyper-V containers?**  
  WSL2 provides a real, optimized Linux kernel with direct memory management and near-native speed, consuming fewer system resources (RAM/CPU) than running a full secondary virtual machine.

---

## Phase 3: Deploy AdGuard Home in Docker

### 1. Execution Steps

#### Step 1: Create Project Directory
```powershell
mkdir C:\adguard
cd C:\adguard
```

#### Step 2: Create `docker-compose.yml`
Create `C:\adguard\docker-compose.yml`:

```yaml
version: '3.8'

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      # Map host port 5353 to container port 53 (Avoids Windows DNS conflict)
      - "5353:53/tcp"
      - "5353:53/udp"
      # Initial setup wizard port
      - "3000:3000/tcp"
      # Web Admin dashboard
      - "8080:80/tcp"
    volumes:
      - ./workdir:/opt/adguardhome/work
      - ./confdir:/opt/adguardhome/conf
```

#### Step 3: Launch the Container
```powershell
cd C:\adguard
docker compose up -d
```

Verify status:
```powershell
docker ps
```

### 💡 Why we do this (Technical Rationale):
* **Why map host port `5353` to container port `53` (`- "5353:53"`)?**  
  Native Windows DNS is already listening on port `53`. If Docker attempts to bind `-p 53:53`, the Windows networking stack throws `bind: address already in use` error. Mapping to `5353` allows both services to run simultaneously on the same host without conflict.
* **Why use persistent volumes (`./workdir` and `./confdir`)?**  
  Containers are ephemeral by design (if a container restarts or updates, everything inside is wiped). Volume mappings store AdGuard's blocklists, query logs, and admin passwords in folders on the Windows Server hard drive (`C:\adguard\confdir`), ensuring data survives updates.
* **Why `restart: unless-stopped`?**  
  Ensures AdGuard automatically boots up when Windows Server starts or after a power outage.

---

## Phase 4: Configure AdGuard Home & Cloudflare DoH

### 1. Execution Steps

#### Step 1: Complete Initial Setup Wizard
1. Open browser: `http://localhost:3000` (or `http://192.168.100.50:3000`).
2. Click **Get Started**.
3. **Admin Web Interface:** Set listen port to `80` (mapped to `8080` outside).
4. **DNS Server:** Set listen port to `53` (mapped to `5353` outside).
5. Set **Admin Username** and **Password** ➔ Click **Next** ➔ **Finish**.

#### Step 2: Configure Encrypted Upstream DNS
1. Open AdGuard dashboard at `http://localhost:8080` and log in.
2. Go to **Settings** ➔ **DNS Settings**.
3. Under **Upstream DNS servers**, clear default entries and paste:
   ```text
   # Cloudflare DNS-over-HTTPS (Encrypted, fast in Cambodia)
   https://dns.cloudflare.com/dns-query

   # Google Public DNS (Fallback)
   8.8.8.8
   ```
4. Scroll down and ensure **DHCP is disabled** (AdGuard DHCP is OFF by default).
5. Click **Apply** and verify with **Test upstreams**.

### 💡 Why we do this (Technical Rationale):
* **Why Cloudflare DoH (`https://dns.cloudflare.com/dns-query`)?**  
  Standard DNS sends domain queries in plain text over UDP 53. Ezecom ISP can inspect and log every domain you visit. By using DNS-over-HTTPS, queries are encrypted with TLS 1.3 over TCP port 443. Ezecom only sees encrypted traffic to Cloudflare's IP (`1.1.1.1`), keeping your browsing private.
* **Why Google (`8.8.8.8`) as secondary fallback?**  
  If Cloudflare experiences an outage or fiber routing issue, AdGuard automatically falls back to Google DNS, ensuring high availability.
* **Why keep AdGuard DHCP disabled?**  
  Having two DHCP servers on the same network causes a **DHCP Race Condition** where devices receive conflicting network configurations.

---

## Phase 5: Connect Windows DNS to AdGuard (Forwarder)

### 1. Execution Steps

#### Option A: Via PowerShell
```powershell
Set-DnsServerForwarder -IPAddress 127.0.0.1 -PassThru
```

#### Option B: Via DNS Manager GUI
1. Open **DNS Manager** (`dnsmgmt.msc`).
2. Right-click your server name (e.g., `WIN-SERVER`) ➔ Select **Properties**.
3. Click on the **Forwarders** tab ➔ Click **Edit...**
4. Type `127.0.0.1` and press Enter.
5. Click **OK** ➔ **Apply** ➔ **OK**.

### 💡 Why we do this (Technical Rationale):
* **Why configure a Forwarder?**  
  Windows DNS knows all local records inside your domain (`*.itp.local`). However, when a client asks for `google.com` or `facebook.com`, Windows DNS says: *"I don't host that domain. Let me ask my Forwarder."*
* **Why point the forwarder to `127.0.0.1` (AdGuard)?**  
  This completes the hybrid chain. Unresolved external queries flow directly from Windows DNS into AdGuard Home, where ads and malware are stripped away before being securely encrypted to Cloudflare.

---

## Phase 6: Create Local Authoritative Zone (For Testing)

### 1. Execution Steps
1. In **DNS Manager**, expand server name.
2. Right-click **Forward Lookup Zones** ➔ Select **New Zone...**
3. Select **Primary zone** ➔ Click **Next**.
4. Zone Name: Type `itp.local` (or `rupp.local`) ➔ Click **Next** ➔ **Finish**.
5. Right-click inside your new `itp.local` zone ➔ Select **New Host (A or AAAA)...**:
   * **Name:** `fileserver`
   * **IP address:** `192.168.100.20`
   * Click **Add Host**.

### 💡 Why we do this (Technical Rationale):
* **Why an Authoritative Zone?**  
  This demonstrates the core power of Windows DNS: any query ending in `.itp.local` is answered immediately from the server's local database. It never leaves your network and never hits AdGuard or Ezecom, guaranteeing instant response times for internal servers.

---

## Phase 7: Verification & Testing Suite

Run these tests in PowerShell to prove that each layer functions as designed:

### Test 1: Verify Local Authoritative Resolution (Windows DNS)
```powershell
nslookup fileserver.itp.local 192.168.100.50
```
* **Expected Result:** Returns `192.168.100.20`.
* **Rationale:** Proves local DNS resolves immediately without going to the internet.

### Test 2: Verify Internet Resolution (Cloudflare DoH via AdGuard)
```powershell
nslookup google.com 192.168.100.50
```
* **Expected Result:** Returns Google's public IP address.
* **Rationale:** Proves Windows DNS successfully forwarded the request to AdGuard, and AdGuard retrieved the answer from Cloudflare DoH.

### Test 3: Verify Ad-Blocking & Threat Sinkhole
```powershell
nslookup doubleclick.net 192.168.100.50
```
* **Expected Result:** Returns `0.0.0.0` or `Name does not exist`.
* **Rationale:** Proves AdGuard intercepted the known advertising domain and blocked it before it could load.

### Test 4: Inspect AdGuard Dashboard Query Log
1. Go to `http://192.168.100.50:8080` and click **Query Log**.
2. Notice `google.com` is marked as **Processed** (encrypted) and `doubleclick.net` is marked in **RED as Blocked**.

---

## Phase 8: Troubleshooting & Diagnostic Commands

### 1. Check for Port Conflicts
If Windows DNS or Docker fails to start:
```powershell
netstat -ano | findstr :53
```
* **Rationale:** Shows which PID (Process ID) is using port 53. Windows DNS should be listening on port 53; Docker AdGuard should be on port 5353.

### 2. Test AdGuard Port 5353 Directly
```powershell
Resolve-DnsName -Name google.com -Server 127.0.0.1 -Port 5353
```
* **Rationale:** Tests AdGuard in isolation to ensure the container is healthy independent of Windows DNS.

### 3. Open Windows Firewall for Client Access
If other computers cannot reach your DNS:
```powershell
New-NetFirewallRule -DisplayName "Inbound DNS (UDP 53)" -Direction Inbound -LocalPort 53 -Protocol UDP -Action Allow
New-NetFirewallRule -DisplayName "Inbound DNS (TCP 53)" -Direction Inbound -LocalPort 53 -Protocol TCP -Action Allow
```
* **Rationale:** By default, Windows Server Firewall blocks inbound UDP port 53 traffic from external subnet clients.
