# Task 2 — Plan and justify


## 0. Figures taken from Task 1

| Figure | Value | Command it came from |
|---|---|---|
| CPU | Apple Silicon, 8 cores (4 performance + 4 efficiency) | `sysctl -n machdep.cpu.brand_string` |
| Cores / logical processors | 8 / 8 | `sysctl hw.physicalcpu hw.logicalcpu` |
| Hardware virtualization | supported and enabled (`kern.hv_support: 1`) | `sysctl kern.hv_support` |
| Memory total | 16 GB (LPDDR5) | `sysctl hw.memsize` |
| System disk | 256 GB Apple SSD (AP0256Z), **42.8 GB free** | `diskutil info /` |
| Partition style | GUID Partition Table, APFS container | `diskutil list` |
| OS | macOS 27.0.1 | `sw_vers` |

**Assumption about the host itself.** macOS, a browser, an editor and the hypervisor need about **6 GB of RAM**, **2 logical CPUs** and **15 GB of free disk** (swap, updates, snapshots). I keep at least these free for the host at all times.

**Architecture note.** This Mac is ARM64. Both guests must therefore be **ARM64 builds**: Ubuntu Server for ARM64 and Windows 11 ARM. An x86 guest would have to be emulated, which is very slow and cannot use hardware virtualization.

## 1. Allocation

| | Ubuntu Server 24.04 (ARM64) | Windows 11 (ARM64) |
|---|---|---|
| vCPUs | 1 | 2 |
| Memory | 2 GB | 4 GB |
| Disk size | 20 GB | 64 GB |
| Disk type | dynamically allocated | dynamically allocated |
| Network adapters | 1 | 1 |
| Network mode | NAT / Shared network | NAT / Shared network |
| Disk location | internal SSD | **external SSD** (see section 2) |

Hypervisor: the one used in Labs 1–2 (VMware Fusion, UTM or Parallels). The mode names above are the VMware names; UTM calls the same mode "Shared Network".

## 2. Arithmetic

| Resource | Host has | Guests use | Left for host | Host needs | Result |
|---|---|---|---|---|---|
| CPU | 8 logical | 2 + 1 = 3 vCPU | 5 | 2 | **fits** (no over-commit, 3 < 8) |
| Memory | 16 GB | 4 + 2 = 6 GB | 10 GB | 6 GB | **fits**, 4 GB margin |
| Disk, worst case | 42.8 GB free | 64 + 20 = 84 GB | 42.8 − 84 = **−41.2 GB** | 15 GB | **does not fit** |
| Disk, expected (dynamic) | 42.8 GB free | ≈ 6 GB (Ubuntu) + ≈ 30 GB (Windows) = 36 GB | 42.8 − 36 = **6.8 GB** | 15 GB | **does not fit** (short by ≈ 8 GB) |

The disk sizes of the real installs (≈ 6 GB and ≈ 30 GB) are estimates for a fresh install with updates. Measure them after the build.

**Verdict.** CPU and memory are fine. **The internal disk is the limit.** Both guests cannot be run as asked on the internal 256 GB SSD while the host stays usable: only 6.8 GB would remain, and Windows updates and snapshots would shrink that further.

**Closest configuration that works (chosen):**
- Ubuntu disk on the internal SSD: 42.8 − 6 = **36.8 GB** free, which is above the 15 GB the host needs.
- Windows disk file (64 GB, dynamic) on an **external USB 3 SSD** of at least 80 GB, formatted APFS.
- Both guests still run at the same time and the Mac stays usable.

**What it gives up:** the external drive must be plugged in whenever Windows runs, and disk speed is limited by USB.

**Alternative without a drive:** free at least **10 GB** more on the internal disk (target ≥ 53 GB free before building). Both guests then fit, but with a very thin margin and no room for snapshots.

## 3. Justification

- **ARM64 guests.** The CPU is ARM. Hardware virtualization (`hv_support = 1`) only helps ARM64 guests.
- **Ubuntu: 1 vCPU, 2 GB, 20 GB.** It is a server with no desktop. The minimum is about 1 GB RAM and 5 GB disk. I give 2 GB so `apt upgrade` does not run out of memory, and 20 GB for updates and logs. One vCPU is enough for an SSH test machine.
- **Windows: 2 vCPU, 4 GB, 64 GB.** These are exactly Microsoft's minimums for Windows 11. I do not go higher because 16 GB must also serve the host, and I do not go lower because Windows 11 would not install or would be unusable. **Nothing is set below a published minimum.**
- **Dynamic, not fixed disks.** Fixed disks would reserve 84 GB at once, which is more than the 42.8 GB free, so the build would fail. Dynamic disks only use what the guest writes. The price is slightly slower writes, and the risk that a growing guest fills the host disk (the reason Windows goes to the external drive).
- **Network mode: NAT / Shared network for both guests, not bridged and not host-only.**
  - *NAT/Shared:* the guest gets an address such as `192.168.x.x` on a private network. The guest reaches the internet through the host, and the host can open `ssh` to the guest's IP. This is the one mode that satisfies both requirements with one adapter.
  - *Host-only:* the host can reach the guest, but the guest has **no internet**. It fails the requirement.
  - *Bridged:* works, but on campus or Wi-Fi networks that allow one device per login the guest may be blocked. Its address can change, and SSH would be exposed to everyone on the LAN.
  - *Windows:* also NAT, so it has internet but is not visible on the LAN.
- **SSH.** Ubuntu needs `openssh-server` installed and running (`sudo apt install openssh-server`), because the server image may not enable it by default.

## 4. Verification plan

After both guests are built and **running at the same time**.

### In the Ubuntu guest

| Command | Expected output |
|---|---|
| `nproc` | `1` |
| `free -h` | Mem total ≈ `1.9Gi` |
| `lsblk` | disk of `20G` |
| `df -h /` | about `19G` if the whole disk is used (≈ `10G` with default LVM until extended) |
| `ip -4 addr show` | an `inet 192.168.x.x/24` on `enp0sX` |
| `ip route` | `default via 192.168.x.1` |
| `systemctl status ssh` | `active (running)` |
| `ping -c 3 8.8.8.8` | `3 received, 0% packet loss` (internet works) |
| `curl -I https://ubuntu.com` | `HTTP/2 200` or `301` (DNS and internet work) |

### On the host (Mac Terminal)

| Command | Expected output |
|---|---|
| `ping -c 3 <ubuntu-ip>` | replies, 0% loss |
| `ssh <user>@<ubuntu-ip>` | host-key prompt, then the Ubuntu shell (**proves SSH**) |
| `ssh <user>@<ubuntu-ip> hostname` | prints the guest's hostname |
| `top -l 1 \| grep PhysMem` | used memory about 6 GB higher than before the guests started |
| `memory_pressure \| tail -1` | "free percentage" still well above 30% (host stays usable) |
| `df -h /System/Volumes/Data` | at least 15 GB free |
| `top -l 1 \| grep "CPU usage"` | idle above 50% while the guests are idle |

### In the Windows guest (PowerShell)

| Command | Expected output |
|---|---|
| `$env:NUMBER_OF_PROCESSORS` | `2` |
| `(Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB` | about `4` (slightly under) |
| `Get-Disk` | disk size `64 GB` |
| `Get-PSDrive C` | free space matches the guest's actual use |
| `winver` | Windows 11 |
| `ping 8.8.8.8` | 0% loss |
| `Test-NetConnection ubuntu.com -Port 443` | `TcpTestSucceeded : True` (internet) |

**What would prove the plan failed:** `ssh` times out (wrong network mode), `ping 8.8.8.8` fails inside Ubuntu (host-only mode), or `df -h` on the host shows under 15 GB free (disk plan broken).
