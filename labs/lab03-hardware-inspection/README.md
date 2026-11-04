# Lab 3 — Teardown and OS Hardware Inspection

## Part 1: Component Analysis & Paper Builds

### 1. Hardware Components

* **AMD Ryzen 5 7600**
  * **Type:** Desktop Central Processing Unit (CPU)
  * **Connects to:** AM5 Socket on motherboard
  * **Limits:** Requires AM5 motherboard, supports DDR5 RAM only (no DDR4 support), 65W TDP.
  * **Official Link:** [AMD Ryzen 5 7600](https://www.amd.com/en/products/processors/desktops/ryzen/7000-series/amd-ryzen-5-7600.html)

* **Intel Core i5-13400**
  * **Type:** Desktop Central Processing Unit (CPU)
  * **Connects to:** LGA 1700 Socket on motherboard
  * **Limits:** Requires LGA 1700 socket, 65W base power / 148W max power.
  * **Official Link:** [Intel Core i5-13400](https://www.intel.com/content/www/us/en/products/sku/230495/intel-core-i513400-processor-20m-cache-up-to-4-60-ghz/specifications.html)

* **ASUS PRIME B650M-A**
  * **Type:** Micro-ATX Motherboard
  * **Connects to:** AM5 Socket CPUs, 24-pin ATX power supply
  * **Limits:** Socket AM5 only (AMD 7000/8000/9000 series), 4x DDR5 DIMM slots only (no DDR4).
  * **Official Link:** [ASUS PRIME B650M-A](https://www.asus.com/motherboards-components/motherboards/prime/prime-b650m-a/)

* **MSI PRO B760M-A WIFI DDR4**
  * **Type:** Micro-ATX Motherboard
  * **Connects to:** LGA 1700 Socket CPUs, 24-pin ATX power supply
  * **Limits:** Socket LGA 1700 only (Intel 12th/13th/14th Gen), 4x DDR4 DIMM slots only (no DDR5).
  * **Official Link:** [MSI PRO B760M-A WIFI DDR4](https://www.msi.com/Motherboard/PRO-B760M-A-WIFI-DDR4)

* **Kingston FURY Beast DDR5-5600 16GB**
  * **Type:** System RAM (DDR5 DIMM)
  * **Connects to:** 288-pin DDR5 DIMM slot on motherboard
  * **Limits:** Compatible only with DDR5 motherboards.
  * **Official Link:** [Kingston FURY Beast DDR5](https://www.kingston.com/en/memory/gaming/kingston-fury-beast-ddr5-memory)

* **Crucial DDR4-3200 16GB DIMM**
  * **Type:** System RAM (DDR4 DIMM)
  * **Connects to:** 288-pin DDR4 DIMM slot on motherboard
  * **Limits:** Compatible only with DDR4 motherboards.
  * **Official Link:** [Crucial DDR4 Desktop Memory](https://www.crucial.com/memory/ddr4/ct16g4dfra32a)

* **Samsung 990 PRO 1TB**
  * **Type:** M.2 NVMe Solid State Drive (SSD)
  * **Connects to:** M.2 slot (PCIe 4.0 x4) on motherboard
  * **Limits:** M.2 2280 form factor, requires M.2 slot supporting PCIe protocol.
  * **Official Link:** [Samsung 990 PRO NVMe SSD](https://semiconductor.samsung.com/consumer-storage/internal-ssd/990-pro/)

* **Samsung 870 EVO 1TB**
  * **Type:** 2.5" SATA Solid State Drive (SSD)
  * **Connects to:** SATA III data port on motherboard & SATA power cable from PSU
  * **Limits:** Max transfer speed capped at SATA III interface limit (~560 MB/s).
  * **Official Link:** [Samsung 870 EVO SATA SSD](https://semiconductor.samsung.com/consumer-storage/internal-ssd/870evo/)

* **Corsair RM750e (750W)**
  * **Type:** ATX Power Supply Unit (PSU)
  * **Connects to:** Wall outlet (AC input) and motherboard/components (DC output cables)
  * **Limits:** Max power output 750W, ATX form factor.
  * **Official Link:** [Corsair RM750e PSU](https://www.corsair.com/us/en/p/psu/cp-9020262-na/rme-series-rm750e-fully-modular-low-noise-atx-power-supply-cp-9020262-na)

* **AMD Wraith Stealth Cooler (Stock Cooler)**
  * **Type:** Air CPU Cooler
  * **Connects to:** AM5/AM4 CPU socket mount & 4-pin PWM fan header on motherboard
  * **Limits:** Ships with AMD Ryzen 5 7600; designed for up to 65W TDP CPUs only.
  * **Official Link:** [AMD Thermal Solutions](https://www.amd.com/en/technologies/cpu-cooler-solution)

---

### 2. Compatible Computer Builds

#### Build 1: AMD DDR5 System
* **CPU:** AMD Ryzen 5 7600
* **Cooler:** AMD Wraith Stealth (Included with CPU)
* **Motherboard:** ASUS PRIME B650M-A
* **RAM:** Kingston FURY Beast DDR5-5600 16GB
* **Storage:** Samsung 990 PRO 1TB M.2 NVMe SSD
* **Power Supply:** Corsair RM750e (750W)
* **Justification:** AM5 socket on CPU matches ASUS B650 motherboard; motherboard supports DDR5 memory and M.2 PCIe 4.0 NVMe SSD.
* **Still Needed:** Computer Case (Micro-ATX or larger), Monitor, Keyboard, Mouse, Operating System.

#### Build 2: Intel DDR4 System
* **CPU:** Intel Core i5-13400
* **Cooler:** Intel RM1 (Included with CPU)
* **Motherboard:** MSI PRO B760M-A WIFI DDR4
* **RAM:** Crucial DDR4-3200 16GB DIMM
* **Storage:** Samsung 870 EVO 1TB SATA SSD
* **Power Supply:** Corsair RM750e (750W)
* **Justification:** LGA 1700 socket matches MSI B760 motherboard; motherboard explicitly uses DDR4 memory slots and has SATA III ports for the SSD.
* **Still Needed:** Computer Case (Micro-ATX or larger), Monitor, Keyboard, Mouse, Operating System.

---

## Part 2: OS Hardware Detection Commands

| Specification | Windows Command | Linux Command | macOS Command |
| :--- | :--- | :--- | :--- |
| **CPU Info** | `wmic cpu get Name, NumberOfCores, MaxClockSpeed` | `lscpu` | `sysctl -n machdep.cpu.brand_string` |
| **Memory Info** | `wmic memorychip get Capacity, Speed, Manufacturer` | `free -h` | `system_profiler SPMemoryDataType` |
| **Disk Info** | `wmic diskdrive get Model, Size, InterfaceType` | `lsblk -o NAME,SIZE,FSTYPE,MODEL` | `system_profiler SPStorageDataType` |

### Evidence

The following evidence files are located in the `evidence/` directory:

* [`evidence/windows-specs.txt`](evidence/windows-specs.txt) — Automated hardware inspection output from Windows PowerShell (`Get-CimInstance`) capturing CPU, memory, and disk specifications of the host machine (12th Gen Intel Core i7-12700H, 2× 8 GB DDR5-4800, Micron 2450 512 GB NVMe).
