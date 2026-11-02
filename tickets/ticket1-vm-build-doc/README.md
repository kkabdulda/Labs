# Ticket 1 — New Developer Starts Monday, Needs a Test Machine

> **Credit.** The two virtual machines this ticket documents ('win11', 'ubuntu-server', NatNetwork) were built in Lab 2, which I did in a pair with **Aiym Kabdulda** (I was the driver for the first half). The evidence linked from `../../labs/lab02-windows-vm/` comes from that joint work. This ticket — the report, BUILD.md, the new evidence in `evidence/` and the screenshots — are our submission.

---

## 1. What Was Asked

On Thursday afternoon, our internal IT helpdesk received the following ticket from Aigerim T., Office Manager at the Almaty office:

```text
Ticket ID:  #IT-10492
From:       Aigerim T. (Office Manager)
Date:       Thu, Oct 29, 2026 14:15
Subject:    New developer starts Monday, needs a test machine

Hi IT support,

We have a new backend developer starting next Monday! Can you set up a test 
computer for them? They will need to test their code before pushing to production. 

Nothing special, just a standard computer with internet.

Thanks,
Aigerim T. (Office Manager)
```

### Contextual Analysis of the Request
At first glance, the ticket reads as a simple hardware provisioning request. However, from a systems administration and architectural perspective, this request is severely underspecified and contains multiple hidden assumptions:
1. It requests "a test computer", implying dedicated physical hardware that would require a lengthy procurement cycle and unnecessary capital expenditure.
2. It states "nothing special", overlooking that developer test environments are subject to frequent misconfigurations, software crashes, and dirty package states that require reliable snapshot recovery.
3. It requests "internet", without specifying whether the developer needs to simulate cross-platform client-server interactions, backend-to-database communication, or isolated network segmentation.

---

## 2. What Was Actually Needed

To translate this vague request into a production-grade, reproducible engineering solution, I performed a structured Gap Analysis.

### 2.1 Gap Analysis Table

| Gap Identified | What I Would Have Asked Aigerim / The Developer | What I Assumed / Architecturally Decided |
| :--- | :--- | :--- |
| **Physical vs. Virtual** | "Do you expect a new physical desktop box, or can we host isolated test environments on an existing developer workstation?" | **Virtual Machines.** Procuring, unboxing, and cabling a physical PC takes days and costs budget. A virtualized testbed on an existing PC is instantly deployable, free, and recoverable. |
| **Operating System** | "What software stack is the developer building? Does the code run on Linux servers, Windows desktops, or a distributed architecture?" | **Both Linux and Windows.** Modern backend development requires testing services in a true production-like Linux server environment (Ubuntu Server 24.04 LTS), alongside a Windows 11 client environment to test client desktop integration, APIs, and GUI tools. |
| **Host Hardware Limits** | "What are the hardware specifications of the workstation hosting this test environment?" | **16.0 GB RAM Host (Windows 11).** Measured the physical host workstation to enforce strict resource bounds so the host never freezes or thrashes. |
| **Networking Architecture** | "Does the developer need to test inter-service communication (e.g., client app talking to an API/database), or only basic web surfing?" | **NAT Network (`NatNetwork`, `10.0.2.0/24`).** A standard VirtualBox NAT adapter isolates machines into private silos. A shared NAT Network allows both VMs to talk to each other while sharing the host's internet connection. |
| **Software Licensing** | "Does the company have a spare retail Windows 11 Pro/Enterprise license key for this test machine?" | **Windows 11 Enterprise Evaluation (90-day).** Utilized the official Microsoft evaluation ISO, avoiding license costs and avoiding pirated/pre-activated software. |

---

### 2.2 Deep Dive: Three Incomplete / Flawed Assumptions in the Original Request

#### Assumption 1: "Nothing special"
* **The Flaw:** In development environments, "nothing special" is a dangerous misconception. Developers test experimental code, run unverified scripts, tweak system configuration files, and install competing runtime versions. Inevitably, the system gets broken.
* **The Solution:** A test environment must be built with an immutable recovery mechanism. I established baseline **Hypervisor Snapshots (`clean-install`)** on both machines, allowing the developer to roll back any catastrophic failure in 10 seconds without rebuilding from an ISO. Furthermore, I authored a comprehensive **[`BUILD.md`](BUILD.md)** guide with Russian UI cross-references so that Aigerim or a junior IT technician can understand, recreate, or roll back the setup independently.

#### Assumption 2: "A test computer" (Single Machine vs. Multi-Tier Architecture)
* **The Flaw:** Modern software is rarely deployed on a monolithic, isolated machine. Backend developers write APIs, microservices, and background workers that interact across network boundaries with client applications and external services. Providing a single standalone machine forces the developer to test client and server on `localhost` (loopback), which masks real-world network failure modes (DNS delays, socket timeouts, firewall blocks, MTU limits).
* **The Solution:** Delivering a **two-node virtual environment** (Ubuntu Server 24.04 LTS as the backend/database node and Windows 11 Enterprise as the client node) running concurrently on the same host provides an authentic multi-tier development environment on a single physical workstation.

#### Assumption 3: Default VirtualBox NAT is Sufficient for "Internet"
* **The Flaw:** The default networking mode in VirtualBox is standard "NAT". Under standard NAT, VirtualBox places each virtual machine into an isolated private sandbox behind an internal NAT engine. Both VMs are assigned the exact same IP address (`10.0.2.15`), and neither machine can ping, route to, or discover the other.
* **The Solution:** A dedicated **NAT Network (`NatNetwork`)** must be explicitly created. It functions as an internal virtual switch with a built-in DHCP server and default gateway (`10.0.2.1`), giving both VMs unique IP addresses on the same subnet (`10.0.2.0/24`) while seamlessly routing outbound WAN traffic through the host's physical network adapter.

---

## 3. What I Did

### 3.1 Resource Calculation Table (16.0 GB RAM Host)

Sizing the virtual machines required strict mathematical justification against the physical host workstation (Windows 11 Home, 12th Gen Intel Core i7-12700H with 14 cores / 20 threads, 16.0 GB DDR5 RAM, and 512 GB NVMe SSD with ~64 GB unallocated space):

| Environment Component | RAM Allocated | vCPUs | Virtual Disk Size | Actual Disk on Host | Architectural Role & Rationale |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Windows 11 Host OS** | ~5,800 MB (Reserved) | 16 Threads | 512 GB NVMe | Active Host OS | Reserved for Windows 11 kernel, DWM, background services, and host IDE/browser. |
| **Ubuntu Server 24.04 VM** | 2,048 MB (2.0 GB) | 2 vCPUs | 25.0 GB (VDI) | ~4.18 GB (Sparse) | Headless backend server. Runs Docker, databases, and APIs without GUI memory overhead. |
| **Windows 11 Guest VM** | 4,096 MB (4.0 GB) | 2 vCPUs | 64.0 GB (VDI) | ~18.45 GB (Sparse) | Client testing node. Satisfies official Windows 11 hardware minimums (vTPM 2.0, Secure Boot). |
| **Total Committed Demand** | **11,944 MB (~11.9 GB)** | **4 vCPUs** | **89.0 GB Virtual** | **~22.63 GB Physical** | **Safety Margin:** Leaves **~4.1 GB of uncontested physical RAM** on the host, preventing disk swapping. |

$$\text{Headroom Margin} = 16.0\text{ GB (Physical RAM)} - [5.8\text{ GB (Host)} + 2.0\text{ GB (Ubuntu)} + 4.0\text{ GB (Win11)}] = 4.2\text{ GB Free Headroom}$$

---

### 3.2 Settings That Differ from the Default, and Why

| Setting Category | Default Setting | Customized Production Setting | Architectural Justification |
| :--- | :--- | :--- | :--- |
| **Virtual Network** | NAT (Isolated Per-VM) | **NAT Network (`NatNetwork`)** | Default NAT prevents inter-guest communication. NAT Network bridges both guests into `10.0.2.0/24` with inter-VM routing and shared outbound Internet. |
| **Ubuntu Resources** | 1024 MB RAM / 1 CPU / 10 GB | **2048 MB RAM / 2 vCPUs / 25 GB Disk** | 1 GB is tight for modern Ubuntu Server with systemd and Docker; 2 GB / 2 vCPU ensures smooth multithreading while leaving host memory free. |
| **Windows 11 Resources** | 2048 MB RAM / 1 CPU / 32 GB | **4096 MB RAM / 2 vCPUs / 64 GB Disk** | Windows 11 installer hard-blocks if RAM < 4 GB or disk < 64 GB. Configured vTPM 2.0 and EFI Secure Boot. |
| **Ubuntu OpenSSH** | Disabled / Not installed | **OpenSSH Server Enabled** | Allows headless terminal management, remote code deployment, and script extraction without console clipboard issues. |
| **Windows Defender Firewall** | Inbound ICMP Blocked | **Inbound ICMPv4-In Enabled** | Enabled pre-defined rule `File and Printer Sharing (Echo Request - ICMPv4-In)`. Restores ping reachability without disabling the firewall. |
| **State Protection** | No Snapshots | **Snapshot `clean-install` on both VMs** | Captured pristine state immediately following initial boot. Enables 10-second disaster recovery. |
| **Hypervisor UI Language** | Default English | **Russian Interface Support** | Matched the Oracle VM VirtualBox Russian UI localization installed in the Almaty office workstation, with cross-references in `BUILD.md`. |

---

### 3.3 Decision: NAT Network vs. The Alternatives

Choosing the network topology required evaluating four candidate modes in the context of our corporate/campus network environment:

```text
       +-------------------------------------------------------------------------+
       |                  NAT Network Topology (NatNetwork)                      |
       |                                                                         |
       |  +-------------------------------------------------------------------+  |
       |  | Host OS: Windows 11 (Authenticated Campus Physical MAC & IP)      |  |
       |  +-------------------------------------------------------------------+  |
       |                                   |                                     |
       |              [Hypervisor NAT Engine & Virtual Switch]                   |
       |              Gateway: 10.0.2.1 | Virtual DHCP: 10.0.2.3                 |
       |                      |                                |                 |
       |                      v                                v                 |
       |        +---------------------------+    +---------------------------+   |
       |        | Guest 1: ubuntu-server    |    | Guest 2: win11            |   |
       |        | IP: 10.0.2.15/24          | <->| IP: 10.0.2.16/24          |   |
       |        | Port: SSH (22), App (8080)|    | Client / Testing Browser  |   |
       |        +---------------------------+    +---------------------------+   |
       +-------------------------------------------------------------------------+
```

1. **Standard NAT (Rejected):**  
   Provides outbound Internet via SNAT, but each guest is confined to its own private translation namespace. Guests cannot ping or communicate with each other.
2. **Bridged Networking (Rejected Due to Campus Security):**  
   Bridged mode attaches the virtual NIC directly to the physical host adapter. In our office/campus network, switches enforce **802.1X Port Security** (`switchport port-security maximum 1`). When the switch detects frames from the VM's virtual MAC address (`08:00:27:...`), it immediately drops the packets or shuts down the physical port. The VM fails to obtain an IP address.
3. **NAT + Host-Only Dual Adapters (Rejected Due to Complexity):**  
   Attaching Adapter 1 to NAT (for internet) and Adapter 2 to Host-Only (for inter-VM traffic) works, but introduces multi-homed routing issues, dual default gateways, and unnecessary configuration complexity for a non-technical manager.
4. **NAT Network (Selected):**  
   A single virtual adapter per VM attached to `NatNetwork`. All outbound WAN packets are translated to the host's authenticated physical MAC/IP, bypassing 802.1X port security, while internal traffic between `10.0.2.15` and `10.0.2.16` routes seamlessly over the virtual software switch.

---

### 3.4 What Went Wrong and How I Fixed It

* **Issue 1: Ping Blocked by Windows Defender Firewall**  
  * *Symptom:* After configuring both VMs on `NatNetwork`, Ubuntu Server could ping the gateway (`10.0.2.1`), but pinging Windows 11 (`10.0.2.16`) resulted in `100% packet loss` (`Destination Host Unreachable`).
  * *Diagnosis:* Windows Defender Firewall enforces a default drop policy for unsolicited ICMPv4 Echo Requests.
  * *The Wrong Fix:* Disabling Windows Firewall completely (`Set-NetFirewallProfile -Enabled False`). This is a severe security anti-pattern that exposes SMB/RPC ports to lateral movement.
  * *The Correct Engineering Fix:* Enabled the specific inbound ICMPv4 rule via administrative PowerShell:
    ```powershell
    Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
    ```
    This instantly allowed ICMP Echo Requests while leaving all firewall profiles actively protecting the system.

* **Issue 2: Virtualization-Based Security (VBS) Coexistence on Windows 11 Host**  
  * *Symptom:* On Windows 11 host machines, VirtualBox initially displayed a green turtle icon, indicating that it was running under Hyper-V execution mode, resulting in reduced VM performance.
  * *Fix:* Verified that CPU virtualization (Intel VT-x) was enabled in UEFI firmware, and configured VirtualBox hardware acceleration to **Hyper-V / Default Paravirtualization Interface**, ensuring optimal scheduling coexistence between Windows 11 Core Isolation and VirtualBox.

---

## 4. Evidence Index

### 4.1 New Evidence Files Created for This Ticket

All files are located in [`evidence/`](evidence/):

| File Name | Format | Technical Verification & What It Proves |
| :--- | :---: | :--- |
| [`evidence/01-new-vm-wizard.png`](evidence/01-new-vm-wizard.png) | Image | VirtualBox VM Creation Wizard demonstrating custom sizing: 2048 MB RAM, 2 vCPUs, EFI enabled, and unattended installation bypassed. |
| [`evidence/02-disk-settings.png`](evidence/02-disk-settings.png) | Image | Storage Controller settings showing AHCI SATA controller, dynamically allocated VDI formatting (25 GB Ubuntu / 64 GB Windows), and SSD caching flags. |
| [`evidence/03-network-adapter.png`](evidence/03-network-adapter.png) | Image | Network settings proving attachment to **NAT Network (`NatNetwork`)**, Intel PRO/1000 MT adapter type, and active cable connection. |
| [`evidence/04-guest-uname.txt`](evidence/04-guest-uname.txt) | Text | Linux guest kernel release (`uname -a`, kernel 6.8.0-40-generic), Ubuntu 24.04.1 LTS identity, CPU count, and memory allocation (`free -m`). |
| [`evidence/05-ping-between-guests.txt`](evidence/05-ping-between-guests.txt) | Text | Full transcript demonstrating bidirectional ping between Ubuntu (`10.0.2.15`) and Windows (`10.0.2.16`), with 0% packet loss and sub-millisecond latency, plus external internet reachability (`1.1.1.1`). |
| [`evidence/06-snapshot-list.png`](evidence/06-snapshot-list.png) | Image | VirtualBox Snapshot Manager displaying the **`clean-install`** baseline snapshot hierarchy for both `ubuntu-server` and `win11`. |
| [`evidence/07-host-specs.txt`](evidence/07-host-specs.txt) | Text | Comprehensive audit of the host workstation: Windows 11 Home, 12th Gen Intel Core i7-12700H (14 cores / 20 threads), 16.0 GB DDR5-4800 RAM, and NVMe SSD free space. |
| [`evidence/08-vm-config.txt`](evidence/08-vm-config.txt) | Text | Detailed `VBoxManage showvminfo` dump containing exact UUIDs, MAC addresses, storage controller ports, and memory configurations for both virtual machines. |

---

### 4.2 Cross-Referenced Evidence from Lab 2

The following artifacts from our joint Lab 2 work substantiate this report:

| Lab 2 Evidence File | Relative Markdown Link | What It Demonstrates |
| :--- | :--- | :--- |
| **Windows IP Configuration** | [`../../labs/lab02-windows-vm/evidence/ipconfig-windows.txt`](../../labs/lab02-windows-vm/evidence/ipconfig-windows.txt) | Windows 11 guest network adapter configuration (`10.0.2.16/24`, Gateway: `10.0.2.1`, DHCP: `10.0.2.3`). |
| **Linux IP Configuration** | [`../../labs/lab02-windows-vm/evidence/ip-addr-linux.txt`](../../labs/lab02-windows-vm/evidence/ip-addr-linux.txt) | Ubuntu Server network interface `enp0s3` configured via DHCP on `10.0.2.15/24`. |
| **Diagnostic Ping Logs** | [`../../labs/lab02-windows-vm/evidence/ping-both-ways.txt`](../../labs/lab02-windows-vm/evidence/ping-both-ways.txt) | Initial firewall drop, specific firewall rule remediation commands, and bidirectional ping proof. |
| **Host Memory Baseline** | [`../../labs/lab02-windows-vm/evidence/host-memory-before.png`](../../labs/lab02-windows-vm/evidence/host-memory-before.png) | Host Task Manager showing stable 55% memory utilization (8.8 GB used / 16.0 GB physical) under normal dual-VM execution. |
| **Memory Stress Overcommit** | [`../../labs/lab02-windows-vm/evidence/host-memory-after.png`](../../labs/lab02-windows-vm/evidence/host-memory-after.png) | Host Task Manager demonstrating memory exhaustion (15.6 GB used, 24.6 GB committed) and active swapping to `pagefile.sys`. |

---

## 5. What I Would Do Differently

Reflecting on this provisioning cycle, I identified three areas for improvement in future workstation setups:

1. **Standardizing the Hypervisor UI Language:**  
   The workstation in the Almaty office was configured with a Russian VirtualBox UI. While this is convenient for local staff, technical documentation and industry standards are predominantly English. Translating settings terms (e.g., *«Сеть NAT»*, *«Носители»*, *«Снимок»*) introduced minor cognitive overhead during documentation. In future deployments, I would set the VirtualBox language to English or provide dual-language callouts across all configuration screens.
2. **Third-Party Validation of `BUILD.md`:**  
   As sysadmins, it is easy to take implicit domain knowledge for granted (such as knowing where to place ISO files or knowing that virtualization must be enabled in BIOS). Before handing this off, I would conduct a blind walkthrough test with Aigerim or a junior colleague to verify that a non-engineer can follow `BUILD.md` from start to finish without asking for assistance.
3. **Automating the Build Pipeline via Infrastructure as Code (IaC):**  
   Deploying VMs through GUI wizards requires manual clicking and is prone to human error. For future developer onboarding tickets, I would author a declarative **Vagrantfile** or an automated PowerShell script leveraging `VBoxManage`. This would allow Aigerim or the developer to type a single command:
   ```bash
   vagrant up
   ```
   and have both virtual machines provisioned, networked, and snapshot-ready in under 10 minutes without touching the hypervisor GUI.
