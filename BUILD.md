# Ticket 1: Rebuildable Windows and Linux VMs

## Host and hypervisor

- Host: Mac with Apple Silicon M2 and 16 GB RAM
- Hypervisor: VMware Fusion
- Network mode: NAT / “Share with my Mac”

## Build choices

I used ARM64 guest operating systems because the host uses Apple Silicon. Earlier x64 installation media did not boot correctly, so I switched to ARM64 ISOs.

| VM | Guest OS | CPU | Memory | Disk |
|---|---|---:|---:|---:|
| `lab01-ubuntu` | Ubuntu 26.04.1 LTS ARM64 | 2 cores | 2 GB | about 10 GB |
| `ticket1-win11-arm` | Windows 11 Pro ARM64, build 26300.9457 | 2 cores | 4 GB | 64 GB |

Both VMs use NAT networking. Windows was installed from the ARM64 ISO. I installed VMware Tools using the Typical setup.

## Rebuild procedure

1. In VMware Fusion, create a new ARM virtual machine.
2. Select the ARM64 installer ISO for the guest OS.
3. Assign the CPU, memory, and disk values shown above.
4. Use NAT networking (“Share with my Mac”).
5. Start the VM and complete the OS installer.
6. For Windows, install VMware Tools using the Typical option and restart the guest.
7. Create a snapshot named `ticket1-final`.

## Verification

- Ubuntu reached the internet using `ping -c 4 example.com`.
- Windows reached the internet by opening `example.com` in Microsoft Edge.
- Ubuntu reached Windows at `172.16.255.130`: 4 packets received, 0% loss.
- Windows initially blocked incoming ping. I added an inbound ICMPv4 echo firewall rule for the lab, then repeated the ping successfully.
- I collected guest hardware information in `linux-specs.txt` and `windows-specs.txt`.

## Assumptions and trade-offs

- The assignment did not specify guest operating-system versions, CPU allocation, or memory allocation. I chose ARM64 guests to match the Apple Silicon host.
- I used 2 CPU cores for each VM and allocated 2 GB to Ubuntu and 4 GB to Windows so both guests could run on a 16 GB host.
- NAT provides internet access through the Mac without exposing the guests directly to the local network.
- The VM hardware reports virtual devices presented by VMware, not the Mac’s physical components.

## What I would do differently

I would check the installer architecture before creating the VM. Starting with x64 media caused a boot failure and wasted setup time; using ARM64 media resolved it.