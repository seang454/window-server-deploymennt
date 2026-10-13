# AWS Windows Active Directory, DNS & AdGuard Home (Docker) Architecture Guide

## 1. Executive Summary & Design Principle

When architecting a modern, enterprise-grade hybrid environment on **Amazon Web Services (AWS)** that integrates **Windows Active Directory Domain Services (AD DS)**, **Windows DNS**, and **AdGuard Home (Docker)** for network-wide ad and tracker filtering:

* **Layer 2 Broadcasts & DHCP:** AWS VPC natively manages IP allocation and network settings via **VPC DHCP Option Sets**. In-guest Windows DHCP server roles are not supported natively on AWS VPC.
* **Internal Resolution & Active Directory:** Windows Server hosts the primary authoritative DNS zones (`cambodia.local`, `_msdcs`). Clients query Windows Server directly to preserve domain logon, Dynamic DNS (RFC 2136), and Kerberos authentication.
* **External Resolution & Security Filtering (Architecture A - Recommended):** Windows DNS does not resolve public traffic directly; instead, it uses **DNS Forwarders** pointing to the **AdGuard Home container running in Docker**. AdGuard Home strips malware, ads, and telemetry, then queries upstream public resolvers (e.g., Cloudflare `1.1.1.1` or AWS Route 53 Resolver `10.0.0.2`).

---

## 2. Architecture Diagram

```mermaid
flowchart LR
    %% Styling Definitions
    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#FFFFFF,font-weight:bold;
    classDef vpc fill:#F0F7FD,stroke:#0284C7,stroke-width:2px,stroke-dasharray: 4 4,color:#0B3C5D;
    classDef subnet fill:#FFFFFF,stroke:#3B82F6,stroke-width:1.5px,color:#1E3A8A;
    classDef server fill:#1E293B,stroke:#0EA5E9,stroke-width:2px,color:#F8FAFC;
    classDef docker fill:#0284C7,stroke:#0369A1,stroke-width:2px,color:#FFFFFF;
    classDef client fill:#334155,stroke:#10B981,stroke-width:2px,color:#F8FAFC;
    classDef awsService fill:#FFFBEB,stroke:#F59E0B,stroke-width:1.5px,color:#78350F;

    subgraph AWSCloud ["☁️ AWS Cloud (Region: ap-southeast-1)"]

        subgraph VPC ["🌐 Amazon VPC (10.0.0.0/16)"]

            subgraph Provisioning ["⚙️ AWS VPC Infrastructure"]
                direction TB
                DHCPOpt["<b>VPC DHCP Option Set</b><br/>domain-name: e6.local<br/>DNS: 10.0.1.10"]:::awsService
                Route53Res["<b>Route 53 Resolver</b><br/>VPC DNS: 10.0.0.2<br/>Native IP Allocator"]:::awsService
            end

            subgraph SubnetPriv ["🔒 Private Subnet (10.0.1.0/24)"]
                direction LR
                Client["<b>💻 Windows Member Instance</b><br/>Role: Workstation / App Server<br/>Private IP: <b>10.0.1.50</b> (AWS DHCP)<br/>Primary DNS: 10.0.1.10"]:::client
                DC["<b>🖥️ Windows Domain Controller</b><br/>Roles: AD DS + DNS Server<br/>Private IP: <b>10.0.1.10</b> (Static ENI)<br/>Authoritative: e6.local"]:::server
                DockerHost["<b>🐳 Docker Host Instance</b><br/>Role: AdGuard Home Filter<br/>Private IP: <b>10.0.1.20</b><br/>Port 53 (DNS) & 3000 (Web UI)"]:::docker
            end

        end

        Internet["🌍 Public Internet & Upstream<br/>(Cloudflare 1.1.1.1 / Google 8.8.8.8)"]:::aws
    end

    %% Flow connections
    DHCPOpt -.->|Injected at boot via DHCP| Client
    Route53Res -.->|Assigns IP 10.0.1.50| Client
    Client -->|"1. All DNS Queries (Port 53)"| DC
    DC -.->|"2a. Internal zone (e6.local): Auth reply"| Client
    DC -->|"2b. External zone forwarder"| DockerHost
    DockerHost -->|"3. Filters ads, trackers & malware"| DockerHost
    DockerHost -->|"4. Clean upstream queries"| Internet

    class VPC vpc;
    class SubnetPriv subnet;
    class Provisioning awsService;
```

---

## 3. Query Flow Breakdown (Architecture A)

| Step | Initiator | Target | Protocol / Port | Description |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **Client (10.0.1.50)** | **DC (10.0.1.10)** | UDP/TCP 53 | Client always queries Windows DC directly (configured by VPC DHCP Option Set). |
| **2A** | **DC (10.0.1.10)** | **Client (10.0.1.50)** | UDP/TCP 53 | If query matches `e6.local` or `_msdcs`, DC resolves it internally immediately. |
| **2B** | **DC (10.0.1.10)** | **AdGuard (10.0.1.20)** | UDP/TCP 53 | If query is external (e.g. `youtube.com`), DC forwards it to AdGuard Home. |
| **3** | **AdGuard (10.0.1.20)** | **Internal Filter Engine** | Internal | AdGuard checks query against blocklists. If blocked, returns `0.0.0.0`. |
| **4** | **AdGuard (10.0.1.20)** | **Upstream (1.1.1.1 / 10.0.0.2)** | UDP 53 / DoH / DoT | Allowed requests are forwarded upstream to public resolvers or Route 53. |
| **5** | **AdGuard** $\rightarrow$ **DC** $\rightarrow$ **Client** | — | UDP/TCP 53 | Clean resolved IP address is returned back to the client machine. |

---

## 4. Key Advantages of Architecture A

1. **Zero Domain Join Failures:** Active Directory relies extensively on hidden SRV records (`_ldap._tcp`, `_kerberos._udp`). Windows DNS handles these directly without third-party interference.
2. **Native Dynamic DNS (DDNS):** When new Windows instances boot, their computer names and IPs automatically register into Windows DNS Manager via RFC 2136.
3. **High Reliability / Fault Tolerance:** If the AdGuard Docker container is stopped or restarted, local domain logins, file shares, and Kerberos continue operating without interruption.
4. **Centralized Ad-Blocking:** Every device on the network automatically receives filtered external DNS without needing local browser extensions or agent software.

---

## 5. Security & Port Requirements (Security Groups)

### 5.1. Domain Controller (`domain-controller-sg`)
* **Inbound from VPC (`10.0.0.0/16`):**
  * DNS: Port `53` (UDP/TCP)
  * Kerberos: Port `88` (UDP/TCP)
  * LDAP / LDAPS: Port `389` / `636` (TCP/UDP)
  * SMB: Port `445` (TCP)
  * RPC Endpoint Mapper: Port `135` (TCP)
  * Dynamic RPC: Ports `49152 - 65535` (TCP)

### 5.2. Docker Host with AdGuard (`adguard-docker-sg`)
* **Inbound from Domain Controller (`10.0.1.10/32`):**
  * DNS: Port `53` (UDP/TCP)
* **Inbound from Admin IP / Bastion:**
  * Web Management UI: Port `80` / `3000` (TCP)
  * SSH / Admin Access: Port `22` (TCP)
* **Outbound to Internet (`0.0.0.0/0`):**
  * DNS / DoT / DoH: Ports `53`, `853`, `443` (UDP/TCP)
