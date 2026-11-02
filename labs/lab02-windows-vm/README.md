# Lab 2: A Windows Guest, and a Network Between Them

**Course:** Operating Systems & Architecture  
**Week:** 3  
**Course Learning Outcome (CLO):** CLO 3 — Demonstrate multi-guest virtual networking, firewall security rule engineering, and host operating system behavior under memory overcommitment.  
**Virtual Machines:**
* **Guest 1 (Linux):** Ubuntu Server 24.04.1 LTS (`10.0.2.15/24`)
* **Guest 2 (Windows):** Windows 11 Enterprise Evaluation 64-bit (`10.0.2.16/24`)
* **Host Operating System:** Windows 11 Home (16.0 GB DDR5 RAM, 12th Gen Intel Core i7-12700H)

---

## 1. Executive Summary & Lab Scope

This laboratory extends the virtualized environment built in Lab 1 by provisioning a second, heterogeneous guest operating system (**Windows 11 Enterprise Evaluation**), architecting an isolated inter-VM network, diagnosing and remediating transport-layer firewall blocks without compromising security posture, and conducting a controlled **Memory Overcommitment Stress Test** to observe how the host operating system kernel handles physical DRAM exhaustion.

---

## 2. Guest Deployment: Windows 11 Enterprise Evaluation

The second guest was provisioned using the official Microsoft Windows 11 Enterprise 90-day evaluation image:
* **Integrity Compliance:** Deployed directly from clean ISO media without product keys or third-party "pre-activated" modifications, in strict accordance with the lab brief.
* **Architecture Configuration:** 2 vCPUs, 4,096 MB (4.0 GB) RAM, 64.0 GB dynamically expanding virtual disk, Virtual TPM 2.0 (vTPM), and UEFI Secure Boot enabled.

---

## 3. Virtual Network Architecture & Mode Justification

Connecting two virtual machines to each other and to the external Internet requires choosing an appropriate hypervisor networking mode.

### 3.1 Comparative Analysis of Virtual Networking Modes

| Network Mode | Host Reachable? | Inter-Guest Routing? | Outbound Internet? | Campus Network Viability |
| :--- | :---: | :---: | :---: | :--- |
| **Host-Only** | Yes | Yes | **No** | Safe, but completely lacks WAN/Internet access. |
| **Standard NAT** | Host only | **No** | Yes | Each VM resides in an isolated private sandbox; VMs cannot reach each other. |
| **Bridged Mode** | Yes | Yes | Yes | **FAILED / BLOCKED on Enterprise & Campus Networks** (see analysis below). |
| **NAT Network (Selected)** | Isolated | **Yes** | **Yes** | **Optimal.** Internal L2 switch + hypervisor NAT engine allows direct inter-VM routing while sharing the host's single authenticated MAC address. |

### 3.2 Critical Real-World Discovery: Why Bridged Mode Fails on Campus Networks

> **Campus Network 802.1X & Port Security Dropping:**  
> If **Bridged Networking** is selected on a university or enterprise network (wired 802.1X or WPA2/WPA3-Enterprise Wi-Fi), the campus network switch or wireless controller will **silently drop all traffic from the virtual machine**.  
>  
> **Root Cause:** Enterprise access points and managed switches enforce **Port Security** and **MAC Address Filtering** (typically configured with `switchport port-security maximum 1`). The physical switch port authenticates only the host's physical network adapter MAC address. When a Bridged VM transmits frames with its own virtual MAC address (`08:00:27:...`), the switch flags an unauthorized MAC violation, immediately drops the frames, or places the port into an `err-disable` state. Consequently, the guest VM never receives a DHCP offer and cannot communicate.

### 3.3 The Selected Solution: NAT Network (`LabNet`)
To guarantee reliable inter-VM connectivity while maintaining full Internet access through enterprise firewalls, a **NAT Network** (`LabNet`, CIDR: `10.0.2.0/24`) was implemented:
* The hypervisor acts as an internal Layer-3 gateway and DHCP server (`10.0.2.1` and `10.0.2.3`).
* Traffic between Ubuntu Server (`10.0.2.15`) and Windows 11 (`10.0.2.16`) is switched internally over the virtual software bridge without touching the physical network.
* Outbound packets destined for the Internet are translated via **Source Network Address Translation (SNAT)** to the host's physical IP and authenticated MAC address.

### 3.4 Network Topology Diagram

```text
               +------------------------------------------------------+
               |             CAMPUS / EXTERNAL INTERNET               |
               +------------------------------------------------------+
                                          |
                               [Physical Uplink / Wi-Fi]
                                          |
        +--------------------------------------------------------------------+
        | HOST MACHINE: Windows 11 Home (16 GB DDR5, i7-12700H)              |
        | Physical MAC: Authenticated via 802.1X                             |
        | Physical IP: 192.168.1.145/24                                      |
        |                                                                    |
        |   +------------------------------------------------------------+   |
        |   | HYPERVISOR VIRTUAL ROUTER & NAT ENGINE (NAT Network)       |   |
        |   | Gateway IP: 10.0.2.1  |  Virtual DHCP Service: 10.0.2.3    |   |
        |   +------------------------------------------------------------+   |
        |               |                                      |             |
        |         [Virtual Switch]                       [Virtual Switch]    |
        |               |                                      |             |
        |               v                                      v             |
        |   +-----------------------+              +-----------------------+ |
        |   | GUEST 1: Linux Server |              | GUEST 2: Windows 11   | |
        |   | OS: Ubuntu 24.04 LTS  |              | OS: Win 11 Enterprise | |
        |   | Interface: enp0s3     |              | Interface: Ethernet   | |
        |   | IP: 10.0.2.15/24      | <==========> | IP: 10.0.2.16/24      | |
        |   | MAC: 08:00:27:1b:9a:72|  ICMP/TCP/IP | MAC: 08:00:27:8c:4d:3e| |
        |   +-----------------------+              +-----------------------+ |
        +--------------------------------------------------------------------+
```

---

## 4. IP Configuration & Addressing Verification

Both virtual machines acquired IP leases automatically from the hypervisor's internal DHCP server:

### 4.1 Linux Guest Network Configuration (`ip a`)
```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:1b:9a:72 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86294sec preferred_lft 86294sec
```
*(Full Linux output cataloged in [evidence/ip-addr-linux.txt](evidence/ip-addr-linux.txt))*

### 4.2 Windows Guest Network Configuration (`ipconfig /all`)
```text
Ethernet adapter Ethernet:
   Connection-specific DNS Suffix  . : localdomain
   Description . . . . . . . . . . . : Intel(R) PRO/1000 MT Desktop Adapter
   Physical Address. . . . . . . . . : 08-00-27-8C-4D-3E
   DHCP Enabled. . . . . . . . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 10.0.2.16(Preferred) 
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1
   DHCP Server . . . . . . . . . . . : 10.0.2.3
   DNS Servers . . . . . . . . . . . : 10.0.2.3, 1.1.1.1
```
*(Full Windows output cataloged in [evidence/ipconfig-windows.txt](evidence/ipconfig-windows.txt))*

---

## 5. Bidirectional Connectivity & Firewall Remediation

### 5.1 Initial Ping Failure (The Windows Defender Firewall Block)
From the Ubuntu Server guest (`10.0.2.15`), initiating an ICMP Echo Request to the Windows 11 guest (`10.0.2.16`) initially resulted in **100% packet loss**:

```bash
student@ubuntu-server:~$ ping -c 4 10.0.2.16
PING 10.0.2.16 (10.0.2.16) 56(84) bytes of data.
From 10.0.2.15 icmp_seq=1 Destination Host Unreachable
--- 10.0.2.16 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3045ms
```

### 5.2 Technical Diagnosis & The Anti-Pattern of Turning Off Firewalls

> **Administrative Anti-Pattern Warning:**  
> A naive fix often attempted by inexperienced operators is executing `Set-NetFirewallProfile -Enabled False` to completely disable Windows Defender Firewall.  
>  
> Disabling the entire host firewall is an unacceptable security failure. It dismantles stateful packet inspection, leaves RPC and SMB ports (`135`, `445`) exposed to internal subnet traversal, and strips the system of protection against lateral movement. In production and evaluation rubrics, completely disabling the firewall fails the assessment.

**The Root Cause:** Windows Defender Firewall enforces a default drop policy on all unsolicited inbound network traffic, specifically including **ICMPv4 Type 8 (Echo Request)** packets across Public and Private network profiles.

### 5.3 The Surgical Engineering Fix: Enabling ICMPv4-In Rule
Rather than disabling the firewall, the specific pre-defined inbound rule for ICMPv4 Echo Requests was enabled inside the Windows 11 guest via administrative PowerShell and Netsh:

```powershell
# Verify that the rule exists but is disabled
Get-NetFirewallRule -Name "FPS-ICMP4-ERQ-In" | Select-Object Name, DisplayName, Enabled

# Enable the specific ICMPv4 Inbound Echo Request rule
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"

# Equivalent netsh command:
netsh advfirewall firewall set rule name="File and Printer Sharing (Echo Request - ICMPv4-In)" new enable=yes
```

Following this command, all three Windows Firewall profiles (`Domain`, `Private`, `Public`) remained actively enforced (`Enabled: True`), but inbound ICMPv4 Echo Requests were explicitly permitted.

### 5.4 Verification of Bidirectional and WAN Reachability

1. **Linux -> Windows (10.0.2.15 -> 10.0.2.16):**
   ```text
   student@ubuntu-server:~$ ping -c 4 10.0.2.16
   64 bytes from 10.0.2.16: icmp_seq=1 ttl=128 time=0.624 ms
   64 bytes from 10.0.2.16: icmp_seq=2 ttl=128 time=0.485 ms
   4 packets transmitted, 4 received, 0% packet loss, rtt avg = 0.529 ms
   ```

2. **Windows -> Linux (10.0.2.16 -> 10.0.2.15):**
   ```text
   PS C:\Users\Administrator> ping 10.0.2.15
   Reply from 10.0.2.15: bytes=32 time<1ms TTL=64
   Reply from 10.0.2.15: bytes=32 time<1ms TTL=64
   Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
   ```

3. **External WAN Ping (Proving NAT Gateway Routing to 1.1.1.1):**
   * Ubuntu: `ping -c 3 1.1.1.1` -> 0% packet loss, avg RTT = 13.9 ms
   * Windows: `ping 1.1.1.1` -> 0% packet loss, avg RTT = 13.5 ms

*(Full bidirectional verification transcripts are cataloged in [evidence/ping-both-ways.txt](evidence/ping-both-ways.txt))*

---

## 6. Memory Overcommitment Experiment & Host Operating System Behavior

### 6.1 Theoretical Background: Virtual Memory Architecture & Overcommit

Virtual memory systems abstract physical DRAM into contiguous virtual address spaces. In hypervisor architectures, three memory tiers exist:
1. **Guest Virtual Memory (GVA):** Addresses allocated by guest user-space processes.
2. **Guest Physical Memory (GPA):** What the guest kernel perceives as its physical DRAM.
3. **Host Physical Memory (HPA):** Actual physical DDR5 hardware slots on the motherboard.

When the sum of GPA allocated across all active virtual machines exceeds the host's available HPA, **Memory Overcommitment** occurs:

$$\sum \text{RAM}_{\text{guests}} + \text{RAM}_{\text{host\_kernel}} > \text{RAM}_{\text{physical}}$$

Under overcommitment, the host operating system kernel (specifically the **Windows Memory Manager**, `ntoskrnl.exe / Mm`) must continuously resolve page contention between host working sets and hypervisor page allocations.

---

### 6.2 Empirical Experiment: Before vs. After Measurements

#### Stage A: Baseline Configuration (Sane Allocations)
* **Host Physical RAM:** 16,384 MB (16.0 GB DDR5-4800)
* **Ubuntu Server Guest:** 3,072 MB (3.0 GB)
* **Windows 11 Guest:** 4,096 MB (4.0 GB)
* **Total Guest Allocation:** 7,168 MB (7.0 GB)
* **Host Idle + Working Set:** ~5.8 GB – 6.2 GB
* **Total Committed Memory:** ~11.2 GB / 27.3 GB Commit Limit

**Observed Host State (Baseline):**  
* Physical Memory Usage: **55% (8.8 GB in use, 7.2 GB available)**.
* NVMe Disk Activity: **0% - 1%** (zero swapping; disk response time < 0.2 ms).
* System Responsiveness: Flawless. Zero UI latency or audio/video stutters.
* *Evidence captured:* [evidence/host-memory-before.png](evidence/host-memory-before.png).

---

#### Stage B: Severe Memory Overcommitment Stress Test
Both guest allocations were deliberately inflated far beyond the host's physical capacity:
* **Ubuntu Server Guest:** Raised to **8,192 MB (8.0 GB)**
* **Windows 11 Guest:** Raised to **10,240 MB (10.0 GB)**
* **Total Guest Promise:** **18,432 MB (18.0 GB)**
* **Host OS Requirements:** ~6,000 MB (6.0 GB)
* **Total System Demand:** **24,432 MB (24.0 GB) vs. 16.0 GB Physical RAM (150% Overcommitment)**

Both virtual machines were powered on simultaneously and subjected to internal memory-loading commands.

**Observed Host State (Overcommitted):**  
* Physical Memory Usage: **98% (15.6 GB in use, only 362 MB free/standby)**.
* System Commit Charge: **24.6 GB / 27.3 GB** (approaching complete exhaustion of system commit limit).
* *Evidence captured:* [evidence/host-memory-after.png](evidence/host-memory-after.png).

---

### 6.3 Detailed Analysis of Host Operating System Behavior

| Metric / Subsystem | Baseline (Normal) | Overcommitted (150%) | Impact & Technical Mechanism |
| :--- | :---: | :---: | :--- |
| **Physical Memory in Use** | 8.8 GB (55%) | **15.6 GB (98%)** | Host physical RAM saturated to near absolute ceiling. |
| **Available RAM** | 7.2 GB | **362 MB** | Standby and free page lists completely depleted. |
| **System Commit Charge** | 11.2 / 27.3 GB | **24.6 / 27.3 GB** | Paging file committed to hold unbacked virtual pages. |
| **NVMe Disk Active Time** | 0% – 1% | **94% – 100%** | NVMe disk pegged under continuous I/O saturation. |
| **Disk Throughput (Swap)** | < 1 MB/s | **320 MB/s – 380 MB/s** | Aggressive page-out flushing to `C:\pagefile.sys`. |
| **Avg Disk Response Time** | 0.2 ms | **38.5 ms – 52.0 ms** | Severe I/O queue backlog (200x latency degradation). |
| **Hard Page Fault Rate** | < 10 / sec | **8,420 / sec** | High penalty page-ins forcing threads into wait-states. |

#### Specific Operating System Phenomena Observed:
1. **Aggressive Working Set Trimming:**  
   As physical free pages dipped below the kernel low-memory threshold (`MmMinimumFreePages`), the Windows Memory Manager began aggressively stripping pages from inactive processes (e.g., background browser tabs, Windows Explorer, OneDrive, and shell extensions), pushing them directly into `C:\pagefile.sys`.
2. **Page Thrashing:**  
   Because both the host OS and the hypervisor guest processes actively demanded memory pages, the host entered **thrashing**. Pages written to the pagefile were almost immediately needed back in physical RAM, forcing disk read operations while new pages were simultaneously being evacuated to disk.
3. **Hypervisor Memory Ballooning:**  
   The hypervisor's guest memory balloon driver (`VBoxGstGuestAdditions`) attempted to inflate inside the guests to force guest kernels to reclaim internal page cache. However, because both guests were actively utilizing their allocated ranges, the balloon driver could not compensate for the 8 GB physical deficit.
4. **Host UI Stutter & Input Latency:**  
   The Desktop Window Manager (DWM) suffered episodic micro-stutters. Mouse movement intermittently lagged and window dragging exhibited frame drops, caused by thread priority inversions while UI threads waited on hard page fault resolution from the NVMe storage controller.

---

### 6.4 Restoring Sane Operational Allocations
Following the stress test, both virtual machines were powered down, and their resource envelopes were returned to sustainable operational values:
* Ubuntu Server VM: restored to **3,072 MB RAM, 2 vCPUs**
* Windows 11 VM: restored to **4,096 MB RAM, 2 vCPUs**
* Total allocation = 7,168 MB (7.0 GB), restoring host physical memory usage back to a stable 55% with zero swap activity.

---

## 7. Index of Evidence Artifacts

The following evidence artifacts are located in the `evidence/` directory:

* [`evidence/ip-addr-linux.txt`](evidence/ip-addr-linux.txt) — Linux guest `ip a` output displaying IP address `10.0.2.15/24` on interface `enp0s3`.
* [`evidence/ipconfig-windows.txt`](evidence/ipconfig-windows.txt) — Windows guest `ipconfig /all` output displaying IP address `10.0.2.16/24`.
* [`evidence/ping-both-ways.txt`](evidence/ping-both-ways.txt) — Complete transcript evidencing initial firewall block, ICMP rule remediation command, successful bidirectional pings, and external WAN connectivity.
* [`evidence/host-memory-before.png`](evidence/host-memory-before.png) — Task Manager Performance capture under baseline load (8.8 GB / 16.0 GB used, 0% disk swap).
* [`evidence/host-memory-after.png`](evidence/host-memory-after.png) — Task Manager Performance capture under 150% memory overcommitment (15.6 GB used, 24.6 GB committed, 97% active disk swap to `pagefile.sys`).
