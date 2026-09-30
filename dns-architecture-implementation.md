# DNS Architecture Implementation Guide

**Target Environment:** Windows Server 2019 / 2022 / 2025 on VMware Workstation  
**Complementary Document:** [dns-architecture.md](file:///D:/RUPPClass/year4/window-server/project-note/dns-architecture.md) (Read first for concepts & diagrams)  
**Deliverable:** Working Two-Tier DNS with Ad-Blocking (AdGuard Docker) and Local Resolution (Native Windows DNS)

---

## Pre-Implementation Checklist

| Requirement | Value / Target | Verified? |
| :--- | :--- | :---: |
| **Physical Host OS** | Windows 10 / 11 with VMware Workstation Pro or Player | [ ] |
| **Guest Virtual Machine** | Windows Server (2019 / 2022 / 2025) | [ ] |
| **VMware Network Mode** | **Bridged (VMnet0)** (Connects to home Ezecom LAN) | [ ] |
| **Static IP for Windows Server** | `192.168.100.50` (or appropriate IP in your router's subnet) | [ ] |
| **Ezecom Router Gateway** | `192.168.100.1` | [ ] |
| **Docker Engine on Windows Server** | Docker Desktop or Mirantis Container Runtime | [ ] |

---

## Phase 0: VMware Workstation Pre-Configuration

Before powering on the Windows Server VM, ensure nested virtualization and bridged networking are configured:

1. In VMware Workstation, ensure the Windows Server VM is **Powered Off**.
2. Right-click the VM ➔ Select **Settings**.
3. **Hardware ➔ Processors:**
   * Check the box: **Virtualize Intel VT-x/EPT or AMD-V/RVI** *(Crucial: allows Docker to run inside Windows Server)*.
4. **Hardware ➔ Network Adapter:**
   * Select **Bridged: Connected directly to the physical network**.
   * Check **Replicate physical network connection state**.
5. Click **OK** and power on the VM.

---

## Phase 1: Set Static IP on Windows Server

A DNS server must always have a permanent static IP address.

### Option A: Via PowerShell (Fastest)
Run PowerShell as Administrator on Windows Server:

```powershell
# Get your active network adapter interface alias
Get-NetAdapter

# Set Static IP, Subnet Mask (/24), and Gateway (Ezecom Router)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.50 -PrefixLength 24 -DefaultGateway 192.168.100.1

# Set loopback as the preferred DNS (points to itself)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("127.0.0.1")
```

### Option B: Via GUI
1. Open **Network Connections** (`ncpa.cpl`).
2. Right-click your Ethernet adapter ➔ **Properties** ➔ Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
3. Configure:
   * **IP address:** `192.168.100.50`
   * **Subnet mask:** `255.255.255.0`
   * **Default gateway:** `192.168.100.1` (Ezecom router IP)
   * **Preferred DNS server:** `127.0.0.1`
4. Click **OK** ➔ **OK**.

---

## Phase 2: Install Native Windows DNS Server Role

### Option A: Via PowerShell (Recommended)
```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
```

### Option B: Via Server Manager GUI
1. Open **Server Manager** ➔ Click **Add roles and features**.
2. Click **Next** until you reach **Server Roles**.
3. Check the box for **DNS Server** (Click **Add Features** when prompted).
4. Click **Next** ➔ **Next** ➔ **Install**.
5. Once completed, verify the service is running:
   ```powershell
   Get-Service -Name DNS
   ```

---

## Phase 2.5: Install Docker on Windows Server

Because **AdGuard Home is a Linux-based container**, Docker on Windows Server requires a Linux backend. In modern Windows Server (2022 / 2025), this is powered by **WSL2** or **Hyper-V / Containers**.

> [!IMPORTANT]
> Ensure you completed **Phase 0** (checking "Virtualize Intel VT-x/EPT" in VMware settings). Without this, Docker cannot launch its Linux engine inside the VM.

### Option A: Install Docker Desktop (Recommended - Includes GUI & Compose)

#### Step 1: Enable Virtual Machine Platform & Containers Features
Open **PowerShell as Administrator** on your Windows Server VM and run:

```powershell
# Enable Virtual Machine Platform and Containers features
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All -NoRestart
Enable-WindowsOptionalFeature -Online -FeatureName Containers -All -NoRestart

# Restart the Windows Server VM to apply changes
Restart-Computer
```

#### Step 2: Install WSL2 Linux Kernel
After the VM restarts, open PowerShell as Administrator:

```powershell
# Install/Update the WSL2 core engine
wsl --install --no-distribution
wsl --update
```

#### Step 3: Download and Install Docker Desktop
1. Download the Docker Desktop installer using PowerShell:
   ```powershell
   Invoke-WebRequest -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" -OutFile "$env:TEMP\DockerDesktopInstaller.exe"
   ```
2. Run the installer:
   ```powershell
   Start-Process "$env:TEMP\DockerDesktopInstaller.exe" -Wait
   ```
   * During installation, ensure the box **"Use WSL 2 instead of Hyper-V"** is **checked**.
3. Once completed, restart your VM or log out and log back in.
4. Launch **Docker Desktop** from the Start Menu.
5. In Docker Desktop Settings ➔ **General** ➔ Verify **"Use the WSL 2 based engine"** is enabled.

#### Step 4: Verify Docker Installation
Open PowerShell and check:
```powershell
docker --version
docker compose version
```
*(You should see Docker version 24.x+ and Docker Compose v2.x+).*

---

### Option B: Install Docker via PowerShell (CLI-only / Server Core)

If you prefer installing Docker via Microsoft's official PowerShell package:

```powershell
# Install the Docker-Microsoft package provider
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force

# Install Docker Engine
Install-Package -Name docker -ProviderName DockerMsftProvider -Force

# Start Docker Service
Start-Service docker

# Install Docker Compose CLI
Invoke-WebRequest -Uri "https://github.com/docker/compose/releases/latest/download/docker-compose-windows-x86_64.exe" -OutFile "$env:ProgramFiles\Docker\docker-compose.exe"
```

---

## Phase 3: Deploy AdGuard Home in Docker

AdGuard Home will run in a Docker container on port `5353` to prevent conflicting with Windows DNS on port `53`.

### 1. Create Project Directory
In PowerShell:
```powershell
mkdir C:\adguard
cd C:\adguard
```

### 2. Create `docker-compose.yml`
Create a file named `C:\adguard\docker-compose.yml` with the following content:

```yaml
version: '3.8'

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      # Map host port 5353 to container port 53 (Avoids Windows DNS Port 53 conflict)
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

### 3. Launch Container
```powershell
cd C:\adguard
docker compose up -d
```

Verify that the container is healthy and running:
```powershell
docker ps
```

---

## Phase 4: Configure AdGuard Home & Cloudflare DoH

### 1. Complete Setup Wizard
1. Open your browser and navigate to: `http://localhost:3000` (or `http://192.168.100.50:3000`).
2. Click **Get Started**.
3. **Admin Web Interface:** Set listen port to `80` (inside container, which maps to `8080` outside).
4. **DNS Server:** Set listen port to `53` (inside container, which maps to `5353` outside).
5. Set your **Admin Username** and **Password** ➔ Click **Next** ➔ **Finish**.

### 2. Configure Encrypted Upstream DNS
1. Open the AdGuard dashboard: `http://localhost:8080` and log in.
2. Go to **Settings** ➔ **DNS Settings**.
3. In the **Upstream DNS servers** text box, delete everything and paste:

```text
# Cloudflare DNS-over-HTTPS (Encrypted, fast in Cambodia)
https://dns.cloudflare.com/dns-query

# Google DNS (Reliable fallback)
8.8.8.8
```

4. Scroll down to **DNS server configuration**:
   * **Rate limit:** `0` (or default `20`)
   * **EDNS Client Subnet:** Enabled
5. Click **Apply** and then click **Test upstreams**. You should see a green confirmation: `"All upstream servers are working correctly"`.

---

## Phase 5: Connect Windows DNS to AdGuard (Forwarder)

Now configure Windows DNS so that whenever a client requests an external website (e.g. `google.com`), Windows DNS forwards the query to AdGuard on `127.0.0.1:5353`.

### Option A: Via PowerShell
```powershell
# Set Windows DNS forwarder to AdGuard
Set-DnsServerForwarder -IPAddress 127.0.0.1 -PassThru
```

### Option B: Via DNS Manager GUI
1. Open **DNS Manager** (`dnsmgmt.msc` or via **Server Manager ➔ Tools ➔ DNS**).
2. Right-click your server name (e.g., `WIN-SERVER`) ➔ Select **Properties**.
3. Click on the **Forwarders** tab.
4. Click **Edit...**
5. Type `127.0.0.1` and press Enter.
6. Click **OK** ➔ **Apply** ➔ **OK**.

---

## Phase 6: Create Local Authoritative Zone (For Lab Testing)

To verify that local domain resolution works independently of AdGuard:

1. In **DNS Manager**, expand your server name.
2. Right-click **Forward Lookup Zones** ➔ Select **New Zone...**
3. Click **Next** ➔ Choose **Primary zone** ➔ Click **Next**.
4. Zone Name: Type `itp.local` (or `rupp.local`) ➔ Click **Next** ➔ **Next** ➔ **Finish**.
5. Right-click inside your new `itp.local` zone ➔ Select **New Host (A or AAAA)...**
   * **Name:** `fileserver`
   * **IP address:** `192.168.100.20`
   * Click **Add Host**.

---

## Phase 7: Verification & Testing Suite

Execute the following verification tests from PowerShell on the server or any client machine configured to use `192.168.100.50` as DNS:

### Test 1: Verify Local Domain Resolution (Windows DNS)
```powershell
nslookup fileserver.itp.local 192.168.100.50
```
* **Expected Result:** Resolves immediately to `192.168.100.20` directly from Windows DNS.

### Test 2: Verify Internet Resolution (Cloudflare DoH via AdGuard)
```powershell
nslookup google.com 192.168.100.50
```
* **Expected Result:** Returns Google's public IP address.

### Test 3: Verify Ad-Blocking & Threat Sinkhole
```powershell
nslookup doubleclick.net 192.168.100.50
```
* **Expected Result:** Returns `0.0.0.0` or `Name does not exist` (Blocked by AdGuard!).

### Test 4: Inspect AdGuard Query Log
1. Go to `http://192.168.100.50:8080`.
2. Click **Query Log** in the top navigation.
3. You will see `google.com` marked as **Processed** (encrypted to Cloudflare) and `doubleclick.net` marked in **RED** as **Blocked**.

---

## Phase 8: Troubleshooting & Diagnostics

### Issue 1: "Ports are not available: exposing port 53: bind forbidden"
* **Cause:** Docker tried to bind to host port `53`, which is owned by Native Windows DNS.
* **Fix:** Ensure `docker-compose.yml` has `"5353:53/udp"` and `"5353:53/tcp"` on the host side.

### Issue 2: External websites fail to resolve
* Check if AdGuard container is running:
  ```powershell
  docker ps
  ```
* Test AdGuard directly from PowerShell:
  ```powershell
  Resolve-DnsName -Name google.com -Server 127.0.0.1 -Port 5353
  ```

### Issue 3: Windows Firewall blocks client queries
If other computers or VMs cannot query your DNS:
```powershell
# Open DNS Port 53 UDP and TCP on Windows Server Firewall
New-NetFirewallRule -DisplayName "Inbound DNS (UDP 53)" -Direction Inbound -LocalPort 53 -Protocol UDP -Action Allow
New-NetFirewallRule -DisplayName "Inbound DNS (TCP 53)" -Direction Inbound -LocalPort 53 -Protocol TCP -Action Allow
```
