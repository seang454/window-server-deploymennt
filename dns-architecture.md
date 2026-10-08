# DNS Architecture, Concepts & System Design

**Target Audience:** Network Engineering, System Administration & Cyber Security  
**Environment:** Windows Server (VMware Workstation) + AdGuard Home (Docker) + Ezecom ISP ($15/month Home Fiber)  
**Document Purpose:** In-depth theoretical concepts, architecture design, cryptographic flow, and design trade-offs.

---

## 1. System Vision & Core Objectives

When designing a modern enterprise or lab network on Windows Server, DNS has three competing requirements:
1. **Directory Integrity & Local Resolution:** Active Directory (AD DS), Windows Domain logons, and local service discovery require strict adherence to Microsoft DNS standards (SRV records, Kerberos discovery, dynamic registration).
2. **Security & Content Filtering:** Modern networks require perimeter defenses against malware domains, phishing sites, invasive tracking telemetry, and advertisements at the network layer.
3. **Data Privacy & Upstream Encryption:** Traditional DNS queries over UDP port `53` are sent in cleartext, enabling the ISP (Ezecom) or anyone on the wire to log every website visited or redirect queries.

A single DNS server cannot solve all three simultaneously out of the box:
* **Native Windows DNS** is excellent at Active Directory and local resolution, but lacks ad-blocking, threat filtering, and native DNS-over-HTTPS (DoH) forwarding.
* **AdGuard Home / Pi-hole** excels at filtering, ad-blocking, and DoH encryption, but lacks native Active Directory SRV record management and Kerberos integration.

**The Solution:** A **Two-Tier Hybrid DNS Architecture**.

---

## 2. High-Level Component Topology

```mermaid
flowchart TD
    subgraph Internet ["🌐 Public WAN"]
        Cloudflare["Cloudflare Anycast DNS (1.1.1.1)\n• Endpoint: https://dns.cloudflare.com/dns-query\n• Port: 443 (TLS 1.3 / DoH)"]
        Google["Google Public DNS (8.8.8.8)\n• Fallback Upstream Resolver"]
    end

    subgraph PhysicalEnvironment ["🏠 Physical Layer (Cambodia / Ezecom ISP)"]
        EzecomGPON["📡 Ezecom GPON Router\n• Gateway IP: 192.168.100.1\n• Role: Physical L2/L3 Gateway\n• Sees only encrypted port 443 traffic"]
        PhysicalHost["🖥️ Physical Host PC (Windows 10/11)\n• Runs VMware Workstation Pro / Player"]
    end

    subgraph VMwareLayer ["📦 VMware Virtualization (Bridged VMnet0)"]
        subgraph WinServerVM ["🪟 Windows Server Guest OS (Static IP: 192.168.100.50)"]
            
            subgraph Tier1 ["Tier 1: Local & Directory Authority"]
                WinDNS["🗄️ Native Windows DNS Server\n• Listening on: 192.168.100.50:53 (UDP/TCP)\n• Authoritative Zone: *.itp.local\n• Forwarder Target: 127.0.0.1:5353"]
            end

            subgraph Tier2 ["Tier 2: Filtering & Cryptographic Gateway"]
                DockerDaemon["🐳 Docker Engine (WSL2 / Hyper-V Engine)"]
                AdGuard["🛡️ AdGuard Home Container\n• Host Port: 5353:53 (UDP/TCP)\n• Web Management: 8080:80 (HTTP)\n• Initial Wizard: 3000:3000\n• Upstream: https://dns.cloudflare.com/dns-query"]
                DockerDaemon --- AdGuard
            end

            WinDNS -- "Forward unresolved queries\n(Loopback :5353)" --> AdGuard
        end

        ClientVM["💻 Client Devices / Lab VMs\n• Primary DNS: 192.168.100.50"]
    end

    %% Network links
    PhysicalHost --- EzecomGPON
    EzecomGPON <--> Cloudflare
    EzecomGPON <--> Google
    ClientVM -- "Port 53 Queries" --> WinDNS
    AdGuard -- "TLS Encrypted Packets (Port 443)" --> EzecomGPON
```

---

## 2.1 Deployment Models: NAT Mode (VMnet8) vs. Bridged Mode (VMnet0)

When deploying this architecture in VMware Workstation, network engineers must choose between two virtual networking modes depending on where client devices reside and whether the host moves between physical locations.

### Comparison Matrix

| Architectural Feature | NAT Mode (`VMnet8`) **[Recommended for Labs]** | Bridged Mode (`VMnet0`) **[Physical Integration]** |
| :--- | :--- | :--- |
| **Virtual Subnet** | Isolated, immutable virtual subnet (e.g., `192.168.1.0/24`) | Inherited directly from external physical router (`192.168.100.0/24` or campus `10.1.64.0/20`) |
| **Default Gateway** | VMware Virtual NAT Router (`192.168.1.1`) | Physical Router (e.g., Ezecom GPON `192.168.100.1` or Campus `10.1.64.1`) |
| **Location Portability** | ✅ **100% Stable:** IP addresses never change whether you are at home, university, or a cafe. | ❌ **Fragile:** Moving between Wi-Fi networks breaks static IPs and requires reconfiguring the VM. |
| **Campus / Enterprise Wi-Fi** | ✅ **Works Everywhere:** Bypasses 802.1X, MAC filtering, and captive portal login screens. | ❌ **Frequently Blocked:** Enterprise Wi-Fi drops secondary virtual MACs or blocks client-to-client traffic. |
| **Client VM Support** | ✅ Fully supported (`pro-win-client`, `pro-win-client2` talk to server over `VMnet8`). | ✅ Supported (if router allows L2 intra-subnet forwarding). |
| **Physical Device Access** | ❌ Requires manual port forwarding on VMware NAT to reach from external phones/PCs. | ✅ **Native:** Physical phones and laptops on the same Wi-Fi can directly query `192.168.100.50:53`. |

---

### Architectural Flow in NAT Mode (`VMnet8`)

In NAT mode, VMware acts as a private Layer-3 router with stateful packet inspection. The entire DNS resolution and ad-blocking chain functions without any dependency on external router configuration:

```mermaid
flowchart TD
    subgraph HostAndInternet ["🌐 External Network & Physical Host"]
        PhysicalWiFi["📡 Physical Wi-Fi / Campus / Home Router\n• Dynamic IP (e.g., 10.1.79.209 or 192.168.100.4)"]
        VMwareNAT["⚙️ VMware NAT Virtual Router (VMnet8)\n• Gateway: 192.168.1.1\n• Translates internal VM traffic to host socket"]
        CloudflareDoH["🔒 Cloudflare DoH (1.1.1.1:443)\n• Upstream encrypted resolver"]
        
        PhysicalWiFi <--> CloudflareDoH
        VMwareNAT <--> PhysicalWiFi
    end

    subgraph VMnet8Subnet ["📦 VMware Private Subnet (192.168.1.0/24)"]
        ClientVM["💻 Client VM (pro-win-client)\n• IP: 192.168.1.20\n• Gateway: 192.168.1.1\n• Primary DNS: 192.168.1.10"]

        subgraph ServerVM ["🪟 Windows Server (pro-win-server)\n• Static IP: 192.168.1.10\n• Gateway: 192.168.1.1"]
            NativeDNS["🗄️ Tier 1: Windows DNS Server\n• Listening on: 192.168.1.10:53\n• Authoritative Zone: itp.local\n• Forwarder Target: 127.0.0.1:5353"]
            AdGuardDocker["🛡️ Tier 2: AdGuard Home (Docker)\n• Container Port: 5353:53\n• Blocklists / Sinkhole (0.0.0.0)\n• Forwarder to https://dns.cloudflare.com/dns-query"]
            
            NativeDNS -- "Forward Unresolved (Loopback :5353)" --> AdGuardDocker
        end

        ClientVM -- "1. Standard UDP:53 Queries" --> NativeDNS
        AdGuardDocker -- "2. Egress TLS Packets (Port 443)" --> VMwareNAT
    end
```

### Architectural Verdict
* **Use NAT Mode** for university coursework, mobile laptops, and multi-VM lab environments where clients are other VMs in VMware.
* **Use Bridged Mode** strictly in fixed home/lab environments where physical smartphones, smart TVs, or external hardware need network-wide DNS filtering from the Windows Server.

---

## 3. Deep Dive: DNS Query Lifecycle & State Flow

The following sequence details how the system arbitrates between internal records, filtered threat domains, and valid public domains.

```mermaid
sequenceDiagram
    autonumber
    actor Client as 💻 Client Device (PC/Phone/VM)
    participant WinDNS as 🗄️ Tier 1: Windows DNS (Port 53)
    participant AdGuard as 🛡️ Tier 2: AdGuard Docker (Port 5353)
    participant Ezecom as 📡 Ezecom Router (192.168.100.1)
    participant Cloudflare as 🌐 Cloudflare Anycast (1.1.1.1:443)

    rect rgb(240, 245, 255)
    Note over Client, WinDNS: PATH A: Authoritative Local Resolution (Active Directory / Internal Lab)
    Client->>WinDNS: 1. A-Record Query: "fileserver.itp.local"
    WinDNS->>WinDNS: Evaluate local Authoritative Zone (*.itp.local)
    WinDNS-->>Client: 2. Authoritative Response: "192.168.100.20"
    Note over WinDNS, AdGuard: AdGuard and Internet are completely bypassed! Zero latency.
    end

    rect rgb(255, 245, 245)
    Note over Client, AdGuard: PATH B: Malicious / Advertisement Query
    Client->>WinDNS: 3. A-Record Query: "telemetry.ads.tracker.com"
    WinDNS->>WinDNS: Not in local zone -> send to Forwarder
    WinDNS->>AdGuard: 4. Forward query over loopback to 127.0.0.1:5353
    AdGuard->>AdGuard: Match against Blocklists (AdGuard SDN / OISD / EasyList)
    AdGuard-->>WinDNS: 5. Response: 0.0.0.0 (Sinkhole / Blocked)
    WinDNS-->>Client: 6. Response: 0.0.0.0
    Note over Client: Client application drops request. No bandwidth wasted, no ad shown.
    end

    rect rgb(245, 255, 245)
    Note over Client, Cloudflare: PATH C: Legitimate Internet Domain Resolution
    Client->>WinDNS: 7. A-Record Query: "github.com"
    WinDNS->>WinDNS: Not in local zone -> send to Forwarder
    WinDNS->>AdGuard: 8. Forward query to 127.0.0.1:5353
    AdGuard->>AdGuard: Check in-memory DNS cache (Cache miss)
    
    Note over AdGuard, Cloudflare: Cryptographic Layer (DNS-over-HTTPS)
    AdGuard->>AdGuard: Wrap DNS request into HTTP/2 POST payload over TLS 1.3
    AdGuard->>Ezecom: 9. Transmit TLS packet to 1.1.1.1:443
    Ezecom->>Cloudflare: 10. Pass packet over fiber optic uplink
    Note over Ezecom: Ezecom sees only: SRC: Your IP, DST: 1.1.1.1, PORT: 443 (HTTPS)
    
    Cloudflare->>Cloudflare: Decrypt payload -> Resolve "github.com" -> Encrypt response
    Cloudflare-->>Ezecom: 11. Encrypted TLS response packet
    Ezecom-->>AdGuard: 12. Forward packet to VM
    AdGuard->>AdGuard: Decrypt TLS packet, cache DNS record in RAM
    AdGuard-->>WinDNS: 13. Return IP: 140.82.121.4
    WinDNS-->>Client: 14. Return IP: 140.82.121.4
    Client->>Ezecom: 15. Initiate direct HTTPS connection to 140.82.121.4
    end
```

---

## 4. Port Conflict Resolution & Socket Arbitration

### The Core Problem
Under Windows Server, any service implementing DNS will attempt to bind to `0.0.0.0:53` (all interfaces, UDP and TCP).
* If Native Windows DNS starts, it acquires port `53`.
* When Docker attempts to start AdGuard Home with `-p 53:53`, the Windows Host OS rejects the binding with:
  ```text
  Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:53:
  bind: An attempt was made to access a socket in a way forbidden by its access permissions.
  ```

### The Architectural Strategy
Instead of fighting Windows kernel socket drivers, we decouple port obligations:

```
┌────────────────────────────────────────────────────────┐
│ WINDOWS SERVER OS (192.168.100.50)                     │
│                                                        │
│  [Network Interface: 192.168.100.50:53]                │
│       ▲                                                │
│       │ (Clients connect here)                         │
│       │                                                │
│  ┌────┴──────────────────────────┐                     │
│  │ Native Windows DNS Server     │                     │
│  │ Listening: Port 53            │                     │
│  │ Forwarder: 127.0.0.1:5353     │                     │
│  └────┬──────────────────────────┘                     │
│       │ (Internal loopback query)                      │
│       ▼                                                │
│  [Host Port Mapping: 127.0.0.1:5353]                   │
│       │                                                │
│       ▼ (Docker Port Translation)                      │
│  ┌───────────────────────────────┐                     │
│  │ AdGuard Home Container        │                     │
│  │ Container Port: 53            │                     │
│  └───────────────────────────────┘                     │
└────────────────────────────────────────────────────────┘
```

1. **Windows DNS** remains the sole listener on standard port `53`. All network clients (PCs, phones, VMs) only talk to port `53`.
2. **AdGuard Home** maps host port `5353` ➔ container port `53`.
3. **Windows DNS Forwarder** is configured to query `127.0.0.1` on port `5353`.
4. Result: Zero port collisions, 100% service uptime.

---

## 5. Security & Cryptographic Analysis: DoH vs Ezecom ISP

### Plain DNS (Traditional RFC 1035)
When a computer queries standard DNS:
* Protocol: Unencrypted UDP on port `53`.
* Vulnerability: The query travels in plain human-readable text. Any router, ISP equipment, or middlebox can inspect, log, or spoof the response (DNS Poisoning / Cache Hijacking).
* On Ezecom: ISP administrators or automated deep packet inspection (DPI) appliances can record every domain query associated with your static/PPPoE IP.

### DNS-over-HTTPS (DoH RFC 8484)
In this architecture, AdGuard Home implements DoH to Cloudflare (`https://dns.cloudflare.com/dns-query`):
* Protocol: Encrypted TLS 1.3 connection encapsulated in HTTP/2 on TCP port `443`.
* What leaves your Ezecom router:

| Packet Property | What is Transmitted | Can Ezecom ISP Read It? |
| :--- | :--- | :---: |
| **Source IP** | Your Home Public IP (`119.82.x.x`) | Yes (Required for routing) |
| **Destination IP**| Cloudflare Anycast (`1.1.1.1`) | Yes (Required for routing) |
| **Port** | TCP `443` (Standard HTTPS) | Yes |
| **Domain Requested**| Encrypted binary blob (AES-256-GCM / ChaCha20-Poly1305) | ❌ **No (Cryptographically Secure)** |
| **DNS Response / IP**| Encrypted inside TLS tunnel | ❌ **No (Cannot tamper or spoof)** |

---

## 6. Architecture Comparison Matrix

| Evaluation Criteria | Pure Windows DNS | Pure AdGuard Docker | Hybrid Two-Tier (Our Design) |
| :--- | :---: | :---: | :---: |
| **Active Directory Integration** | ⭐⭐⭐⭐⭐ Native | ⭐ Incompatible | ⭐⭐⭐⭐⭐ Seamless |
| **Network-Wide Ad-Blocking** | ❌ None | ⭐⭐⭐⭐⭐ Built-in | ⭐⭐⭐⭐⭐ Built-in |
| **DoH / Privacy Encryption** | ❌ Complex / No UI | ⭐⭐⭐⭐⭐ Native | ⭐⭐⭐⭐⭐ Native |
| **Web Dashboard & Metrics** | ❌ Old MMC Snap-in | ⭐⭐⭐⭐⭐ Modern Web UI | ⭐⭐⭐⭐⭐ Modern Web UI |
| **Client Configuration Overhead** | ⭐ Zero (Standard 53) | ⭐ Zero (Standard 53) | ⭐ Zero (Standard 53) |
| **Single Point of Failure** | 1 Service | 1 Container | 2 Services (Mitigated by DNS Cache) |

---

## 7. Summary
This architecture achieves **enterprise-grade directory compliance** without sacrificing **modern privacy, ad-blocking, and threat protection**. It isolates external ISP visibility while providing local systems with sub-millisecond authoritative resolution.

---

## 8. VMware Virtual Network Architecture: Bridged vs. NAT vs. Host-Only

Hypervisors like VMware Workstation provide three primary virtual networking topologies. Selecting the correct type determines how the virtual machines communicate with the host PC, other virtual machines, and the outside internet.

### Comprehensive Comparison Matrix

| Architectural Feature | 1. Bridged (`VMnet0`) | 2. NAT (`VMnet8`) | 3. Host-Only (`VMnet1`) |
| :--- | :---: | :---: | :---: |
| **Internet Access** | ✅ Yes (Direct via physical gateway) | ✅ Yes (Shared via host network) | ❌ **No (Completely Offline)** |
| **Inter-VM Routing** | ✅ Yes (Across the same subnet) | ✅ Yes (Within `192.168.1.0/24`) | ✅ Yes (Within `192.168.127.0/24`) |
| **Host-to-VM Communication** | ✅ Yes | ✅ Yes | ✅ Yes |
| **External Physical Device Access** | ✅ **Direct:** Physical phones/PCs can directly query VM IP | ❌ **Hidden:** Requires manual VMware Port Forwarding | ❌ **Completely Blocked** |
| **IP Address Management** | Assigned by physical router (e.g., Ezecom or Campus DHCP) | Managed by VMware NAT router or internal Windows Server DHCP | Managed by VMware Host-Only switch or internal DHCP |
| **Campus / Enterprise Wi-Fi** | ❌ **Fragile:** Blocked by 802.1X, MAC filters, and captive portals | ✅ **100% Stable:** Bypasses campus restrictions transparently | N/A (Offline network) |
| **Mobility (Home ➔ School ➔ Cafe)** | ❌ Breaks (Subnet changes with physical router) | ✅ **Immutable:** Subnet remains identical anywhere | ✅ **Immutable:** Subnet remains identical anywhere |

---

### Detailed Mechanics & Real-World Analogies

```mermaid
flowchart TD
    subgraph Mode1 ["1. Bridged Mode (VMnet0) - 'Independent House'"]
        PhysRouter1["📡 Physical Wi-Fi Router\n(Gateway: 192.168.100.1)"]
        Laptop1["💻 Host Laptop\n(192.168.100.4)"]
        VM1["🪟 Windows Server\n(192.168.100.50)"]
        PhysPhone["📱 Physical Phone\n(192.168.100.25)"]

        PhysRouter1 <--> Laptop1
        PhysRouter1 <--> VM1
        PhysPhone <--> VM1
    end

    subgraph Mode2 ["2. NAT Mode (VMnet8) - 'Apartment behind Reception'"]
        PhysRouter2["📡 Physical Wi-Fi Router\n(Dynamic Campus/Home IP)"]
        HostNIC2["💻 Host Wi-Fi NIC"]
        VMNATRouter["⚙️ VMware NAT Router (192.168.1.1)"]
        VM2["🪟 Windows Server (192.168.1.10)"]
        ClientVM2["💻 Client VM (192.168.1.20)"]

        PhysRouter2 <--> HostNIC2 <--> VMNATRouter
        VMNATRouter <--> VM2
        VMNATRouter <--> ClientVM2
        VM2 <--> ClientVM2
    end

    subgraph Mode3 ["3. Host-Only Mode (VMnet1) - 'Underground Bunker'"]
        HostNIC3["💻 Host Loopback Virtual NIC"]
        VM3["🪟 Malware Sandbox VM"]
        ClientVM3["💻 Isolated Analysis Client"]

        HostNIC3 <--> VM3
        HostNIC3 <--> ClientVM3
        VM3 <--> ClientVM3
    end
```

#### 1. Bridged Mode (`VMnet0`) - "An Independent House on the Street"
* **How it operates:** VMware binds the virtual network card directly to the physical network card (Wi-Fi or Ethernet) using the `VMware Bridge Protocol` driver. The VM broadcasts its own unique virtual MAC address directly onto the physical router's Layer-2 switch fabric.
* **Real-World Analogy:** A separate house on the same street with its own mailbox.
* **When to use:** When physical hardware on your local network (e.g., your smartphone, smart TV, or a classmate's laptop) must connect directly to services (DNS, Web, SMB) running inside your VM.
* **Limitations:** Every time your host laptop switches Wi-Fi networks (e.g. from home to university), the physical subnet changes, causing static IPs to drop offline. Furthermore, enterprise networks with 802.1X security frequently drop secondary virtual MAC addresses.

#### 2. NAT Mode (`VMnet8`) - "An Apartment behind a Front Desk / Router"
* **How it operates:** VMware provisions an internal software router and virtual switch. The host laptop's physical network adapter acts as the WAN interface, while all VMs sit on a private virtual LAN (`192.168.1.0/24`). Outbound packets have their source IP translated (NATed) to the host PC's IP.
* **Real-World Analogy:** An apartment building with a single front door and front desk. Outside visitors only see the main building, while apartments communicate freely through interior hallways.
* **When to use:** **The industry standard for development laptops and coursework.** It isolates the lab environment from external network changes, provides continuous internet egress across any physical Wi-Fi, and allows all lab VMs (`pro-win-server`, `pro-win-client`) to interconnect without interference.
* **Limitations:** External physical devices cannot initiate connections into the VM without manual port forwarding rules configured in the VMware NAT settings.

#### 3. Host-Only Mode (`VMnet1`) - "An Underground Bunker (Air-Gapped)"
* **How it operates:** Creates an isolated virtual switch between the host operating system and guest virtual machines. There is **no default gateway** and no routing path to any physical network adapter.
* **Real-World Analogy:** An underground bunker with zero telephone lines or windows to the outside world. People inside can only talk to each other.
* **When to use:** **Cybersecurity sandboxing, malware analysis, and strictly isolated penetration testing.** Guarantees that hostile code, untested exploits, or misconfigured routing tables cannot accidentally leak onto the host LAN or public internet.
* **Limitations:** No internet access whatsoever. VMs cannot download packages, update operating systems, or query public upstream resolvers (Cloudflare / Google).

---

### Engineering Recommendation for this Project:
For this two-tier DNS and Active Directory infrastructure, **NAT Mode (`VMnet8`)** is selected because:
1. It maintains an immutable static IP schema (`192.168.1.10`) regardless of whether the physical laptop travels between home, school, or mobile hotspots.
2. It facilitates local DHCP delegation: disabling VMware's built-in DHCP on `VMnet8` allows the Windows Server VM to serve as the authoritative enterprise DHCP and Active Directory Domain Controller for all lab client VMs (`pro-win-client`).
3. Outbound encrypted DoH traffic (TCP 443) passes transparently through the host's existing Wi-Fi socket without being blocked by campus firewall policies.
