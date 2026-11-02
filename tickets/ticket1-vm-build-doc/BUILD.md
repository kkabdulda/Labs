# BUILD.md: Step-by-Step VM Deployment Guide

**Target Audience:** Aigerim T. (Office Manager) / Junior IT Support  
**Environment:** Oracle VM VirtualBox 7.0 on Windows 11 (16 GB RAM Host)  
**Deliverables:** Two connected virtual machines (`ubuntu-server` and `win11`) on a private shared network with Internet access and baseline recovery snapshots.

*(Примечание: Все пункты содержат названия кнопок на русском языке в скобках, так как на офисном компьютере интерфейс VirtualBox установлен на русском языке).*

---

## Quick Reference Summary (Cheat Sheet)

| Virtual Machine | Name | OS Type | RAM | vCPUs | Virtual Disk | Network Mode | Snapshot Name |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- | :--- |
| **Backend Node** | `ubuntu-server` | Ubuntu (64-bit) | **2048 MB** | **2 CPU** | **25.0 GB** (VDI, Dynamic) | Сеть NAT (`NatNetwork`) | `clean-install` |
| **Client Node** | `win11` | Windows 11 (64-bit) | **4096 MB** | **2 CPU** | **64.0 GB** (VDI, Dynamic) | Сеть NAT (`NatNetwork`) | `clean-install` |

---

## Step 0: Prerequisites Checklist

Before starting, ensure you have:
1. **VirtualBox 7.0+** installed on the workstation.
2. At least **40 GB of free space** on disk `C:`.
3. The following two official ISO files downloaded and placed in `C:\ISO\`:
   * `ubuntu-24.04.1-live-server-amd64.iso` (Ubuntu Server LTS)
   * `Win11_23H2_Enterprise_Eval_x64.iso` (Official Microsoft Evaluation ISO)

---

## Step 1: Configure the Shared NAT Network (`NatNetwork`)

This step creates the virtual router that allows both virtual machines to talk to each other and access the Internet.

1. Open **Oracle VM VirtualBox**.
2. In the top menu, click **File** -> **Tools** -> **Network Manager** (*Файл -> Инструменты -> Сетевой менеджер*).
3. Switch to the **NAT Networks** tab (*Сети NAT*).
4. Click the **Create** button (*Создать* / зеленая иконка с плюсом).
5. A new network named **`NatNetwork`** will appear:
   * **Network Name:** `NatNetwork`
   * **IPv4 Prefix:** `10.0.2.0/24`
   * Check the box: ☑ **Enable DHCP** (*Включить DHCP*).
   * Uncheck IPv6 (not needed).
6. Click **Apply** (*Применить*).

---

## Step 2: Build the Ubuntu Server VM (`ubuntu-server`)

1. In the main VirtualBox window, click **New** (*Создать* / синяя кнопка).
2. **Name and Operating System (*Имя и операционная система*):**
   * **Name (*Имя*):** `ubuntu-server`
   * **Folder (*Папка*):** `C:\Users\student\VirtualBox VMs` (default)
   * **ISO Image (*Образ ISO*):** Click the arrow, browse to `C:\ISO\ubuntu-24.04.1-live-server-amd64.iso`.
   * **Type (*Тип*):** `Linux`
   * **Version (*Версия*):** `Ubuntu (64-bit)`
   * ☑ Check the box: **Skip Unattended Installation** (*Пропустить автоматическую установку*).
   * Click **Next** (*Далее*).
3. **Hardware (*Аппаратное обеспечение*):**
   * **Base Memory (*Основная память*):** Set slider to **`2048 MB`** (2.0 GB).
   * **Processors (*Процессоры*):** Set slider to **`2 CPU`**.
   * ☑ Check the box: **Enable EFI** (*Включить EFI*).
   * Click **Next** (*Далее*).
4. **Virtual Hard Disk (*Виртуальный жесткий диск*):**
   * Select: **Create a Virtual Hard Disk Now** (*Создать новый виртуальный жесткий диск*).
   * **Disk Size (*Размер диска*):** Set to **`25.00 GB`**.
   * Click **Next** (*Далее*), then click **Finish** (*Готово*).
5. **Attach to Network (*Настройка сети*):**
   * Right-click `ubuntu-server` -> **Settings** (*Настроить*).
   * Go to **Network** (*Сеть*) -> **Adapter 1** (*Адаптер 1*).
   * **Attached to (*Тип подключения*):** Select **NAT Network** (*Сеть NAT*).
   * **Name (*Имя*):** Select **`NatNetwork`**.
   * Click **OK**.
6. **Install the OS (*Установка системы*):**
   * Click **Start** (*Запустить*).
   * Choose English language -> Minimal Server install (no GUI!).
   * For network, leave default DHCP (it will automatically get `10.0.2.15`).
   * For disk, select "Use an entire disk" (default ext4).
   * Create username: `student`, password: `password123` (or company standard).
   * In "SSH Setup", check the box: ☑ **Install OpenSSH server**.
   * Finish install, reboot, and unmount the ISO.

---

## Step 3: Build the Windows 11 VM (`win11`)

1. In VirtualBox, click **New** (*Создать*).
2. **Name and Operating System:**
   * **Name (*Имя*):** `win11`
   * **ISO Image (*Образ ISO*):** Select `C:\ISO\Win11_23H2_Enterprise_Eval_x64.iso`.
   * **Type:** `Microsoft Windows`, **Version:** `Windows 11 (64-bit)`.
   * ☑ Check: **Skip Unattended Installation** (*Пропустить автоматическую установку*).
   * Click **Next** (*Далее*).
3. **Hardware:**
   * **Base Memory:** Set slider to **`4096 MB`** (4.0 GB).
   * **Processors:** Set slider to **`2 CPU`**.
   * ☑ Check: **Enable EFI** (*Включить EFI*).
   * Click **Next**.
4. **Virtual Hard Disk:**
   * Set size to **`64.00 GB`** (VDI, dynamically allocated).
   * Click **Finish** (*Готово*).
5. **Attach to Network:**
   * Right-click `win11` -> **Settings** (*Настроить*) -> **Network** (*Сеть*).
   * **Attached to (*Тип подключения*):** Select **NAT Network** (*Сеть NAT*).
   * **Name (*Имя*):** Select **`NatNetwork`**.
   * Click **OK**.
6. **Install Windows 11:**
   * Start the VM, complete the standard Windows setup wizard.
   * Do not enter a license key (select 90-day Evaluation).

---

## Step 4: Fix Windows Firewall for Ping (Crucial Step!)

By default, Windows blocks ping requests. **DO NOT turn off the firewall!** Follow these steps:

1. Inside the Windows 11 VM, click the **Start Menu**, type **PowerShell**.
2. Right-click **Windows PowerShell** -> select **Run as Administrator** (*Запуск от имени администратора*).
3. Copy and paste this single command and press Enter:

```powershell
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
```

*(You will see the command finish silently. Windows Defender Firewall remains 100% active, but will now respond to ping from the Linux server).*

---

## Step 5: Create Baseline Snapshots (`clean-install`)

This is the safety net that lets the developer reset the computer in 10 seconds if anything breaks.

1. **Shut down both virtual machines** properly from inside their operating systems.
2. In the VirtualBox window, click on **`ubuntu-server`**:
   * Click the menu icon next to the VM name and select **Snapshots** (*Снимки*).
   * Click **Take** (*Сделать снимок* / иконка фотоаппарата с плюсиком).
   * **Name (*Имя снимка*):** `clean-install`
   * **Description (*Описание*):** `Baseline clean install. Ubuntu 24.04 LTS, SSH active, IP 10.0.2.15.`
   * Click **OK**.
3. Repeat the exact same steps for **`win11`**:
   * Select `win11` -> **Snapshots** (*Снимки*) -> **Take** (*Сделать снимок*).
   * **Name:** `clean-install`
   * **Description:** `Baseline clean install. Windows 11 Enterprise Eval, ICMP rule enabled, IP 10.0.2.16.`
   * Click **OK**.

---

## Step 6: 2-Minute Verification Checklist

Start both virtual machines and run these quick checks:

### 1. Check IP Addresses
* On **Ubuntu**, run `ip -brief addr show enp0s3`. It should display `10.0.2.15`.
* On **Windows**, open Command Prompt, run `ipconfig`. It should display `10.0.2.16`.

### 2. Check Inter-VM Communication
* On **Ubuntu**, run:
  ```bash
  ping -c 4 10.0.2.16
  ```
  *(Should say: `4 packets transmitted, 4 received, 0% packet loss`).*
* On **Windows**, run:
  ```cmd
  ping 10.0.2.15
  ```
  *(Should say: `Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)`).*

### 3. Check Internet Access
* On both machines, ping `1.1.1.1`. Both should successfully receive replies.

---

## Disaster Recovery: How to Roll Back in 10 Seconds

If the developer breaks their environment or installs corrupted software:
1. In VirtualBox, right-click the broken VM -> **Close** -> **Power Off** (*Закрыть -> Выключить*).
2. Open **Snapshots** (*Снимки*).
3. Click on the snapshot named **`clean-install`**.
4. Click **Restore** (*Восстановить* / иконка со стрелочкой).
5. Start the machine — it will instantly boot in its original, pristine state!
