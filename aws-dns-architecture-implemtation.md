# AWS Windows Active Directory & DNS Implementation Guide

This step-by-step implementation guide walks you through deploying a Windows Server 2022/2025 Domain Controller with Active Directory and DNS in an AWS VPC, and using AWS VPC DHCP Option Sets to manage client IP and DNS settings.

---

## Prerequisites
* An AWS Account with permissions to create VPCs, Subnets, Security Groups, and EC2 instances.
* A Key Pair (`.pem` or `.ppk`) for Windows Administrator password retrieval.

---

## Phase 1: Set Up the VPC and Networking

### 1. Create the VPC
1. Navigate to **VPC Console** > **Your VPCs** > **Create VPC**.
2. Settings:
   * **Name tag:** `lab-ad-vpc`
   * **IPv4 CIDR block:** `10.0.0.0/16`
3. Click **Create VPC**.

### 2. Create the Subnets
Create at least one private/application subnet for your Domain Controller:
* **Subnet Name:** `lab-private-subnet-a`
* **VPC:** `lab-ad-vpc`
* **Availability Zone:** Select any AZ (e.g., `us-east-1a` or `ap-southeast-1a`)
* **IPv4 CIDR block:** `10.0.1.0/24`

*(Optional)* Create a public subnet (`10.0.0.0/24`) attached to an Internet Gateway if you need direct RDP or internet access for lab updates.

---

## Phase 2: Create Security Groups

Create a Security Group named `sg-domain-controller`:
1. In the **EC2 Console**, go to **Network & Security** > **Security Groups** > **Create security group**.
2. Add Inbound Rules for VPC traffic (Source: `10.0.0.0/16`):
   * **DNS (UDP/TCP):** Port `53`
   * **Kerberos (UDP/TCP):** Port `88`
   * **LDAP (TCP/UDP):** Port `389`
   * **SMB (TCP):** Port `445`
   * **RPC Endpoint Mapper (TCP):** Port `135`
   * **Dynamic RPC Ports (TCP):** Ports `49152 - 65535`
3. Add **RDP (TCP 3389)** strictly from your own trusted IP or Bastion host.

---

## Phase 3: Launch and Configure the Domain Controller EC2 Instance

> [!IMPORTANT]
> In AWS, never change your primary IP address to static *inside the Windows Network Adapter settings* unless it matches the ENI private IP, or you may lock yourself out. Assign the static private IP via the AWS Console.

### 1. Launch the EC2 Instance
1. Go to **EC2 Console** > **Launch Instances**.
2. **Name:** `DC-Server-01`
3. **AMI:** Microsoft Windows Server 2022 Base (or 2025 Base).
4. **Instance Type:** `t3.medium` (minimum 2 vCPU, 4GB RAM recommended for AD DS).
5. **Network Settings** (Click *Edit*):
   * **VPC:** `lab-ad-vpc`
   * **Subnet:** `lab-private-subnet-a` (10.0.1.0/24)
   * **Auto-assign public IP:** Enable (if public testing) or use Bastion/Session Manager.
   * **Primary IP:** Set to **Custom IP** and enter `10.0.1.10`.
   * **Security Group:** Select `sg-domain-controller`.
6. Launch instance and decrypt the Administrator password using your Key Pair.

---

## Phase 4: Install AD DS & DNS on Windows Server

Connect to `DC-Server-01` via RDP and open **PowerShell as Administrator**:

### Step 1: Install AD DS and DNS roles
```powershell
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
```

### Step 2: Promote to Domain Controller
Run the following PowerShell script to create a new forest (replace domain name and passwords as desired):

```powershell
$domainName = "cambodia.local"
$safeModePassword = ConvertTo-SecureString "P@ssw0rdLab2026!" -AsPlainText -Force

Install-ADDSForest `
    -DomainName $domainName `
    -DomainNetbiosName "CAMBODIA" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword $safeModePassword `
    -Force:$true
```
*The server will automatically reboot upon completion.*

---

## Phase 5: Configure DNS Forwarders on Windows Server

Log back into `DC-Server-01` after reboot. Configure Windows DNS to forward external queries to the AWS VPC resolver (`10.0.0.2` or base VPC network + 2):

```powershell
# Set DNS forwarder to AWS AmazonProvidedDNS (VPC CIDR 10.0.0.0/16 -> 10.0.0.2)
Set-DnsServerForwarder -IPAddress 10.0.0.2
```

---

## Phase 6: Configure AWS DHCP Option Set

Now, instruct AWS to automatically distribute your Windows Server (`10.0.1.10`) as the DNS server for any EC2 instance in the VPC.

1. Open the **AWS VPC Console**.
2. In the navigation menu, select **DHCP option sets** > **Create DHCP option set**.
3. Fill in the values:
   * **Name tag:** `dopt-cambodia-local`
   * **Domain name:** `cambodia.local`
   * **Domain name servers:** `10.0.1.10, AmazonProvidedDNS`
4. Click **Create DHCP option set**.

### Attach to the VPC:
1. Go to **Your VPCs**.
2. Select `lab-ad-vpc`.
3. Click **Actions** > **Edit VPC settings** (or **Edit DHCP option set**).
4. Change the DHCP option set from default to `dopt-cambodia-local`.
5. Click **Save**.

---

## Phase 7: Verification & Testing

### 1. Launch a Member Client Instance
1. Launch a second EC2 instance (`Win-Client-01`) running Windows Server or Windows 10/11 AMI in `lab-private-subnet-a`.
2. Do **not** touch its network adapter; keep it set to default DHCP.

### 2. Verify IP and DNS Assignment on Client
Log into `Win-Client-01` and run:

```powershell
ipconfig /all
```
**Expected Output:**
* **IPv4 Address:** `10.0.1.x` (Leased automatically from AWS VPC)
* **Primary Connection-Specific DNS Suffix:** `cambodia.local`
* **DNS Servers:** `10.0.1.10`

### 3. Test DNS Resolution
```powershell
nslookup dc-server-01.cambodia.local
nslookup amazon.com
```

Both internal domain names and public internet addresses should resolve successfully.

### 4. Join the Domain
```powershell
Add-Computer -DomainName "cambodia.local" -Credential (Get-Credential) -Restart
```
Provide the `cambodia\Administrator` credentials. The machine will reboot and will be a fully functional domain member inside AWS!
