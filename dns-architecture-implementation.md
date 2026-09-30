# DNS Architecture Implementation Guide (with Technical Rationale)

**Target Environment:** Windows Server 2019 / 2022 / 2025 on VMware Workstation  
**Complementary Document:** [dns-architecture.md](file:///D:/RUPPClass/year4/window-server/project-note/dns-architecture.md) (Theory, Diagrams, and Threat Model)  
**Deliverable:** Working Two-Tier DNS with Ad-Blocking (AdGuard Docker) and Local Resolution (Native Windows DNS)

---

## Pre-Implementation Checklist

| Requirement | Value / Target | Verified? |
| :--- | :--- | :---: |
| **Physical Host OS** | Windows 10 / 11 with VMware Workstation Pro or Player | [ ] |
| **Guest Virtual Machine** | Windows Server (2019 / 2022 / 2025) | [ ] |
| **VMware Network Mode** | **Bridged (VMnet0)** (Direct L2 connection to Ezecom LAN) | [ ] |
| **Static IP for Windows Server** | `192.168.100.50` (in Ezecom router's subnet) | [ ] |
| **Ezecom Router Gateway** | `192.168.100.1` | [ ] |
| **Docker Engine on Windows Server** | Docker Desktop with WSL2 backend | [ ] |

---

## Phase 0: VMware Workstation Pre-Configuration

Before powering on the Windows Server VM, two critical hypervisor settings must be configured.

### 1. Execution Steps
1. In VMware Workstation, ensure the Windows Server VM is **Powered Off**.
2. Right-click the VM ➔ Select **Settings**.
3. **Hardware ➔ Processors:**
   * Check the box: **Virtualize Intel VT-x/EPT or AMD-V/RVI**.
4. **Hardware ➔ Network Adapter:**
   * Select **Bridged: Connected directly to the physical network**.
   * Check **Replicate physical network connection state**.
5. Click **OK** and power on the VM.

### 💡 Why we do this (Technical Rationale):
* **Why enable "Virtualize Intel VT-x/EPT"?**  
  AdGuard Home is a Linux-based container. Running Docker on Windows Server requires a lightweight Linux VM in the background (WSL2 or Hyper-V). Because your Windows Server is *already* a virtual machine inside VMware, running Docker creates a "VM inside a VM" (**Nested Virtualization**). If you don't enable VT-x in VMware, Windows Server cannot access hardware CPU virtualization instructions, and Docker will crash with: `Hardware assisted virtualization is not enabled`.
* **Why Bridged Mode instead of NAT?**  
  In NAT mode (`VMnet8`), VMware hides the VM behind a private virtual router that only the host PC can reach; physical devices on your Wi-Fi (phones, test PCs) cannot send DNS packets to the VM. In **Bridged mode**, VMware attaches the virtual network card directly to your physical network interface, giving the VM its own real IP on your Ezecom Wi-Fi (`192.168.100.50`).

---

## Phase 1: Set Static IP on Windows Server

A DNS server must always maintain a fixed IP address.

### 1. Execution Steps

#### Option A: Via PowerShell (Fastest)
Run PowerShell as Administrator on Windows Server:

```powershell
# Get your active network adapter interface alias (usually "Ethernet0")
Get-NetAdapter

# Set Static IP, Subnet Mask (/24), and Gateway (Ezecom Router)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.50 -PrefixLength 24 -DefaultGateway 192.168.100.1

# Set loopback (127.0.0.1) as the preferred DNS resolver
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("127.0.0.1")
```

#### Option B: Via GUI
1. Open **Network Connections** (`ncpa.cpl`).
2. Right-click your Ethernet adapter ➔ **Properties** ➔ Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
3. Configure:
   * **IP address:** `192.168.100.50`
   * **Subnet mask:** `255.255.255.0`
   * **Default gateway:** `192.168.100.1` (Ezecom router IP)
   * **Preferred DNS server:** `127.0.0.1`
4. Click **OK** ➔ **OK**.

### 💡 Why we do this (Technical Rationale):
* **Why a Static IP is Mandatory:**  
  If the server used DHCP, its IP address could change after a reboot or lease expiration. If the DNS server's IP changes from `.50` to `.89`, every computer, phone, and VM pointing to `.50` will immediately lose internet and domain name resolution.
* **Why `192.168.100.50`?**  
  Most home routers hand out dynamic IPs starting from `.100` to `.200`. Choosing `.50` places the server in the safe static pool below the router's dynamic range, avoiding IP collision with other devices.
* **Why set Preferred DNS to `127.0.0.1` (Loopback)?**  
  The Windows Server itself must use its own native DNS service to resolve domain lookups and locate its own Active Directory services, rather than querying an outside DNS server.

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
Open PowerShell as Administrator:
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
```powershell
Invoke-WebRequest -Uri "https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe" -OutFile "$env:TEMP\DockerDesktopInstaller.exe"
Start-Process "$env:TEMP\DockerDesktopInstaller.exe" -Wait
```
* During setup, ensure **"Use WSL 2 instead of Hyper-V"** is checked.
* Log out and log back in, then launch Docker Desktop from the Start Menu.

#### Step 4: Verify Docker Engine
```powershell
docker --version
docker compose version
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
