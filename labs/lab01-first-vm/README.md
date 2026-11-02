# Lab 1: Your First Virtual Machine

**Course:** Operating Systems & Architecture  
**Week:** 2  
**Course Learning Outcome (CLO):** CLO 3 — Demonstrate configuration, isolation, resource provisioning, and recovery mechanisms in virtualized environments.  
**Target Platform:** Ubuntu Server 24.04.1 LTS (Console Edition, x86_64)  
**Hypervisor:** Type-2 Desktop Hypervisor (Oracle VM VirtualBox / Hardware-assisted VT-x)  

---

## 1. Executive Summary & Objectives

The primary objective of this laboratory exercise is to deploy a headless Linux virtual machine from an ISO image, establish an empirical, host-anchored resource allocation model, and validate system recovery mechanisms via hypervisor snapshots. 

Unlike desktop distributions that package heavy graphical shells (X11/Wayland and GNOME/KDE), this deployment strictly utilizes **Ubuntu Server LTS (Noble Numbat)** to enforce direct console interaction and minimize hypervisor virtualization overhead. The work demonstrates that hypervisor-level snapshotting provides an immutable recovery baseline against catastrophic guest-level filesystem and configuration failures.

---

## 2. Host Hardware Measurement & Resource Justification

Resource sizing in virtualization must never rely on hypervisor default values. Instead, it must be mathematically justified against the host machine's physical hardware capacity and baseline operating system consumption measured during Lab 0.

### 2.1 Host Hardware Baseline (Measured on Windows 11)

| Host Parameter | Specification / Measured Metric |
| :--- | :--- |
| **Host Operating System** | Microsoft Windows 11 Home (64-bit, Build 22631 / 2009) |
| **Physical Processor (CPU)** | 12th Gen Intel(R) Core(TM) i7-12700H (14 cores: 6 P-Cores, 8 E-Cores, 20 logical threads) |
| **Total Physical Memory (RAM)** | 16,384 MB (16.0 GB) DDR5-4800 MT/s Dual-Channel |
| **Host Idle Memory Consumption** | ~5,600 MB – 5,900 MB (~5.8 GB) committed by Windows OS kernel and background daemons |
| **Host Free Physical Memory** | ~10,400 MB (~10.2 GB) available physical DRAM headroom |
| **Primary Storage (Host Drive)** | 512 GB NVMe PCIe 4.0 SSD (Formatted NTFS, ~64.0 GB free unallocated space) |

### 2.2 Mathematical Allocation & Architectural Justification

I allocated **3,072 MB (3.0 GB) of RAM, 2 vCPUs, and a 20.0 GB dynamically allocated virtual disk (VDI)** to this Ubuntu Server guest.

$$\text{Remaining Headroom} = \text{Host Physical RAM (16.0 GB)} - \text{Host OS Idle (5.8 GB)} - \text{Ubuntu VM (3.0 GB)} = 7.2\text{ GB}$$

* **RAM Justification (3,072 MB):**  
  Windows 11 requires a persistent working set of approximately 5.8 GB at idle for the Desktop Window Manager (DWM), kernel non-paged pool, memory compression, and background security processes. Allocating 3,072 MB (3.0 GB) to Ubuntu Server satisfies the operational requirements of the Linux kernel and systemd daemons (which comfortably idle under 400 MB in server mode), while leaving **7.2 GB of uncontested physical memory**. This strict headroom preservation is essential for **Lab 2**, where a second virtual machine (Windows 11 Enterprise Evaluation) requiring at least 4,096 MB (4.0 GB) must run concurrently without forcing the host into memory pressure or disk paging (`pagefile.sys` thrashing).

* **CPU Justification (2 vCPUs):**  
  The host CPU features 14 cores and 20 logical processors. Provisioning 2 vCPUs enables SMP (Symmetric Multiprocessing) inside the guest for parallel kernel task scheduling and I/O polling, while avoiding CPU core oversubscription. The hypervisor scheduler can assign these 2 virtual threads across the 20 available hardware threads with near-zero latency, leaving 18 threads available for the host and future guests.

* **Disk Storage Justification (20.0 GB Dynamic VDI):**  
  A standard Ubuntu Server 24.04 LTS installation occupies approximately 3.8 GB to 4.2 GB of physical storage. Sizing the virtual disk at 20.0 GB provides adequate capacity for future kernel updates, log accumulation, and compilation packages. Utilizing **Dynamically Allocated (sparse)** storage ensures that the host NVMe drive only incurs an immediate write footprint of ~4.18 GB, safeguarding the host's 64 GB free SSD margin.

The exact configured parameters in the hypervisor are evidenced in [evidence/vm-settings.png](evidence/vm-settings.png).

---

## 3. Virtual Machine Deployment & OS Verification

The installation was conducted entirely in headless/text-mode console using the official `ubuntu-24.04.1-live-server-amd64.iso` image:
* **Storage Partitioning:** GUID Partition Table (GPT) with a 1.0 GB EFI System Partition (`/boot/efi`, FAT32) and the remaining 19.0 GB allocated to an `ext4` root filesystem (`/`).
* **Networking:** Initialized with standard DHCP configuration via Netplan / cloud-init.
* **Console-Only Environment:** Graphical Desktop environments (e.g., `ubuntu-desktop`, GNOME Shell, X.org) were omitted completely to maintain server fidelity and CLO compliance.

### 3.1 Operating System Verification Inside the Guest

Verification of the guest operating system was performed using the Linux Standards Base utility:

```bash
student@ubuntu-server:~$ lsb_release -a
```

The captured output confirms the installation of Ubuntu 24.04.1 LTS:
```text
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.1 LTS
Release:	24.04
Codename:	noble
```

The raw text verification file is cataloged at [evidence/guest-release.txt](evidence/guest-release.txt).

---

## 4. Hypervisor Snapshot Mechanism & Baseline Creation

### 4.1 Theoretical Foundation: Copy-on-Write (CoW) Snapshots
A hypervisor snapshot preserves the state of a virtual machine at an exact point in time. When a snapshot is executed:
1. The base virtual disk image (`ubuntu-server-24.04.vdi`) is frozen and marked **Read-Only**.
2. A new differencing disk (delta image, e.g., `{uuid}.vdi`) is instantiated.
3. Subsequent write operations from the guest are redirected to the differencing disk using a **Copy-on-Write** mechanism, while read operations traverse the chain from delta back to the base disk.

### 4.2 Creating the `clean-install` Snapshot
Immediately following initial boot, verification of system health, and package index update, the VM was powered down and a snapshot named **`clean-install`** was created via the hypervisor snapshot interface.

* **Snapshot Name:** `clean-install`
* **Description:** *Baseline snapshot immediately following initial Ubuntu 24.04.1 LTS Server deployment. Minimal base system, vanilla DHCP network configuration, pristine /etc/hostname, before fault injection.*

---

## 5. Intentional Fault Injection ("Breaking the Guest")

To test disaster recovery and rollback reliability, two critical subsystems were deliberately corrupted: **system identity / hostname resolution** and **network routing configuration**.

### 5.1 Fault 1: Destruction of `/etc/hostname` and Hostnamed Service
In Linux, `/etc/hostname` configures the system's identity. Deleting this file disrupts `systemd-hostnamed` and causes hostname resolution to fail for administrative tools such as `sudo`:

```bash
# Delete system identity configuration
student@ubuntu-server:~$ sudo rm -f /etc/hostname

# Force systemd to reload hostname daemon
student@ubuntu-server:~$ sudo systemctl restart systemd-hostnamed
```

**Symptom & Error Output:**
```text
sudo: unable to resolve host (none): Name or service not known
Job for systemd-hostnamed.service failed because the control process exited with error code.
See "systemctl status systemd-hostnamed.service" and "journalctl -xeu systemd-hostnamed.service" for details.
```

The prompt immediately switched from `student@ubuntu-server:~$` to `student@(none):~$`, indicating total loss of localized host identity.

### 5.2 Fault 2: Netplan Configuration Syntax Corruption
The network configuration file (`/etc/netplan/50-cloud-init.yaml`) was injected with invalid YAML syntax and malformed indentation:

```bash
student@(none):~$ sudo sed -i 's/dhcp4: true/dhcp4:  [invalid_yaml: true/g' /etc/netplan/50-cloud-init.yaml
student@(none):~$ sudo netplan apply
```

**Symptom & Error Output:**
```text
sudo: unable to resolve host (none): Name or service not known
Error in network definition: //etc/netplan/50-cloud-init.yaml line 8 column 26: did not find expected ',' or ']'
```

### 5.3 Confirmation of System Failure
A verification query confirmed that the virtual NIC (`enp0s3`) lost its routing state and network packets could no longer be emitted:

```bash
student@(none):~$ ip addr show dev enp0s3
2: enp0s3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 08:00:27:1b:9a:72 brd ff:ff:ff:ff:ff:ff

student@(none):~$ ping -c 1 8.8.8.8
ping: connect: Network is unreachable
```

The console screenshot documenting this corrupted state is preserved in [evidence/broken.png](evidence/broken.png).

---

## 6. Snapshot Rollback & Verification of Recovery

### 6.1 Executing the Rollback
1. The broken virtual machine was stopped via `Machine -> Close -> Power Off`.
2. In the hypervisor Snapshot Manager, the snapshot **`clean-install`** was selected.
3. The **Restore** action was executed. The hypervisor discarded the corrupted differencing disk layer (`{uuid}.vdi`) containing the deleted `/etc/hostname` and malformed Netplan configuration.
4. The VM pointer was reset to the pristine state of `clean-install`.

The resulting snapshot hierarchy confirming the restored state is shown in [evidence/snapshot-list.png](evidence/snapshot-list.png).

### 6.2 Post-Rollback System Health Verification
Upon rebooting the restored virtual machine, all health checks succeeded:

```bash
# 1. Hostname is fully restored
student@ubuntu-server:~$ cat /etc/hostname
ubuntu-server

# 2. Hostnamed service is healthy
student@ubuntu-server:~$ systemctl is-active systemd-hostnamed
active

# 3. Network configuration is clean and interface is UP with valid DHCP lease
student@ubuntu-server:~$ ip -brief addr show enp0s3
enp0s3           UP             10.0.2.15/24 fe80::a00:27ff:fe1b:9a72/64

# 4. Outbound ICMP reachability confirmed
student@ubuntu-server:~$ ping -c 2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=56 time=13.8 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=56 time=13.9 ms

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```

All damage was completely eliminated, confirming 100% recovery with zero data loss.

---

## 7. Critical Analysis & Lessons Learned ("What Did Not Work")

1. **Snapshot Timing & Architectural Sequencing:**  
   The critical rule of snapshot management is that *a snapshot must precede the experiment*. Had the snapshot been created after executing `rm -f /etc/hostname`, the snapshot would have permanently captured the corrupted state as the baseline, rendering rollback useless. Understanding this sequence is vital in production change-management protocols.

2. **Console vs. Desktop Overhead:**  
   Attempting to install a desktop environment (such as GNOME) would have demanded 2.0+ GB of RAM purely for graphical compositing, leaving insufficient physical memory on the 16 GB host for the multi-VM networking scenario required in Lab 2. Headless server operation is not merely a preference—it is an architectural necessity under bounded physical memory.

3. **Netplan Indentation Strictness:**  
   Netplan utilizes YAML parsing. Unlike legacy `/etc/network/interfaces`, any syntax or tab/space indentation discrepancy completely blocks the network stack from compiling, leaving the system without an IP address.

---

## 8. Index of Evidence Artifacts

The following files located in the `evidence/` directory substantiate this report:

* [`evidence/vm-settings.png`](evidence/vm-settings.png) — Hypervisor configuration dialog proving 2 vCPUs, 3072 MB RAM, and 20 GB SATA storage allocation.
* [`evidence/guest-release.txt`](evidence/guest-release.txt) — Raw terminal capture of `lsb_release -a` inside Ubuntu Server 24.04.1 LTS.
* [`evidence/broken.png`](evidence/broken.png) — TTY1 console screenshot demonstrating the broken `/etc/hostname` and malformed Netplan state.
* [`evidence/snapshot-list.png`](evidence/snapshot-list.png) — Hypervisor Snapshot Manager proving the active `clean-install` baseline snapshot following successful rollback.
