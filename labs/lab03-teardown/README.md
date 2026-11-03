 Lab 3 — Teardown without a screwdriver

## The machine
Apple MacBook Air (M2, 2022), Model Identifier: Mac14,2.
Model identified with `system_profiler SPHardwareDataType`.
[Official Apple repair manual](https://support.apple.com/en-kz/100603).

## Components
![Apple exploded view of MacBook Air M2](evidence/manual-figures/macbook-air-m2-parts.png)


Ubuntu and Windows were tested in VMware Fusion virtual machines. Their CPU, memory, and disk values describe the virtual hardware assigned to each VM, not the physical Mac.

Captions use the numbers in Apple's exploded view.
For each part: what it is, what it connects to, and its limits.

| No. | Part | What / connects to / limits |
|---:|---|---|
| 1 | Bottom case | Outer enclosure; fastens to the top case; not an upgrade part. |
| 2 | Right display hinge cover | Covers the right display hinge; protective part, not an upgrade. |
| 3 | Left and right speakers with antennas | Speaker and antenna assemblies; connect to the computer’s audio and wireless systems; not CPU or memory upgrades. |
| 4 | Right speaker connector cowling | Holds the right speaker connector cover in place; protective piece, not an upgrade. |
| 5 | Interposer board connector cowling | Covers the interposer-board connector; protective piece, not an upgrade. |
| 6 | Lid angle sensor/audio board connector cowling | Covers the named connectors; protective piece, not an upgrade. |
| 7 | Touch ID board connector cowling | Covers the Touch ID board connector; protective piece, not an upgrade. |
| 8 | Trackpad connector cowling | Covers the trackpad connector; protective piece, not an upgrade. |
| 9 | Left display hinge cover | Covers the left display hinge; protective part, not an upgrade. |
| 10 | Display connectors cowling | Covers the display connectors; protective piece, not an upgrade. |
| 11 | Antenna coaxial cables connector cowling | Covers the antenna cable connectors; protective piece, not an upgrade. |
| 12 | Left speaker connector cowling | Covers the left speaker connector; protective piece, not an upgrade. |
| 13 | MagSafe 3/USB-C boards connector cowling | Covers connectors for the charging and USB-C boards; protective piece, not an upgrade. |
| 14 | Battery connector cowling | Covers the battery connector; protective piece, not an upgrade. |
| 15 | Lid angle sensor | Detects the display lid’s position; works with the computer’s sensor system; not a performance upgrade. |
| 16 | Touch ID board flexible cowling | Covers the Touch ID flex connection; protective piece, not an upgrade. |
| 17 | Audio board | Audio component board; connects with the computer’s audio components; does not upgrade CPU or memory. |
| 18 | Touch ID board | Touch ID component; connects to the computer through its board connector; not a CPU or memory upgrade. |
| 19 | MagSafe 3 board wedge | Supports or positions the MagSafe board; mechanical piece, not an upgrade. |
| 20 | MagSafe 3 board | Charging-port board; connects the MagSafe port to the computer; does not upgrade battery capacity. |
| 21 | USB-C boards | USB-C port boards; connect the ports to the computer; do not upgrade CPU, memory, or storage. |
| 22 | Trackpad flex cable | Connects the trackpad to the computer; cable only, not an upgrade. |
| 23 | Heat sink | Transfers heat away from the processor area; the MacBook Air has no user-upgradeable CPU cooling module. |
| 24 | Logic board | Main board connecting the computer’s components; Apple lists fixed M2, memory, and storage configurations for replacement boards. |
| 25 | Battery | Powers the computer and connects to the logic board; replaceable, but not a memory or storage upgrade. |
| 26 | Top case | Main case assembly including the keyboard area; holds internal components; not a CPU or memory upgrade. |
| 27 | Trackpad | Pointing-device surface; connects through the trackpad flex cable; replacement part, not a performance upgrade. |
| 28 | Display | Screen assembly; connects to the computer through display connectors and hinges; replacement part, not a compute upgrade. |

## Upgrade verdict

| Component | Verdict | Evidence |
|---|---|---|
| Memory | No independent upgrade | Apple lists logic-board configurations with different memory amounts. Memory is not listed as a separate replaceable module. |
| Storage | No independent upgrade documented | Apple lists logic-board configurations with different storage capacities. The exploded-parts list does not show a separate SSD module. |
| CPU | No independent upgrade | The M2 is part of the logic-board assembly. Apple lists logic-board replacements, not a separate CPU part. |

These verdicts are based on the [Apple repair manual](https://support.apple.com/en-kz/100603) and its [exploded-parts list](https://support.apple.com/en-kz/100604). The manual documents replacing assemblies; it does not document upgrading these components separately.

| Fact | macOS host | Linux (Ubuntu VM) | Windows VM |
|---|---|---|---|
| CPU model | `sysctl machdep.cpu.brand_string`: Apple M2 | `lscpu Model name`: not reported; `Architecture`: aarch64 | `Win32_Processor Name`: Apple silicon (VM-reported) |
| Cores / logical CPUs | `hw.physicalcpu`: 8; `hw.logicalcpu`: 8 | `lscpu CPU(s)`: 2; `Core(s) per socket`: 2 | `Win32_Processor`: 2 entries, each reports 1 core and 1 logical processor |
| Total memory | `hw.memsize` / Memory: 16 GB | `free total`: 1,666,128 KiB (about 1.59 GiB) | `Win32_PhysicalMemory Capacity`: 4,294,967,296 bytes (4 GiB) |
| Memory type | `SPMemoryDataType`: LPDDR5 | Not reported | `SMBIOSMemoryType`: 3; VMware Virtual RAM |
| Disk | `SPStorageDataType`: Apple SSD, Apple Fabric, 251 GB | `lsblk`: VMware virtual NVMe, 20 GB | `Get-PhysicalDisk`: VMware Virtual NVMe Disk, SSD, 64 GiB |
| System board | `Model Identifier`: Mac14,2 | VMware virtual hardware | `Win32_BaseBoard`: VMware, Inc. / VBSA |

The Mac is the physical host; Ubuntu and Windows are VMware virtual machines, so their reported CPU, memory, and disk describe virtual hardware. Windows reports two virtual processor entries, each with one core. The Ubuntu output does not identify a specific CPU model or memory type.
