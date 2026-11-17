# Lab 4 — Partition, mirror, fail, recover

Pair: Muratuly Mukhammadali (worked alone)   Driver first half: Muratuly
Machine: MacBook (Apple M2), VMware Fusion; guests: Ubuntu 26.04 LTS aarch64 (lab01), Windows 11 ARM64
Date: 12.11.2026

## What I did

### Windows (disk 1, 2 GB)
```powershell
get-disk
initialize-disk -number 1 -partitionstyle gpt
new-partition -disknumber 1 -usemaximumsize -driveletter e
format-volume -driveletter e -filesystem ntfs -newfilesystemlabel "LAB4"
get-disk | select number,partitionstyle,size | format-list
get-volume -driveletter e
```
The disk is **GPT**. Reasons: Windows 11 boots only in UEFI mode and its system disk is GPT too (disk 0 shows GPT); GPT has no 2 TB / four-primary-partition limit and keeps a backup copy of the partition table. MBR would also work for a 2 GB data disk, but there is no reason to choose it.

### Linux (disks nvme0n1, nvme0n2 = mirror, nvme0n4 = replacement; nvme0n3 is the system disk)
```bash
sudo apt install -y mdadm
sudo parted /dev/nvme0n1 --script mklabel gpt mkpart primary 0% 100% set 1 raid on
sudo parted /dev/nvme0n2 --script mklabel gpt mkpart primary 0% 100% set 1 raid on
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme0n1p1 /dev/nvme0n2p1
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/mirror && sudo mount /dev/md0 /mnt/mirror
echo "lab04 mirror test file" | sudo tee /mnt/mirror/important.txt
sha256sum /mnt/mirror/important.txt
sudo mdadm /dev/md0 --fail /dev/nvme0n2p1
sudo mdadm /dev/md0 --remove /dev/nvme0n2p1
sha256sum /mnt/mirror/important.txt
sudo parted /dev/nvme0n4 --script mklabel gpt mkpart primary 0% 100% set 1 raid on
sudo mdadm /dev/md0 --add /dev/nvme0n4p1
```

## Result

| State | Evidence | Key lines |
|---|---|---|
| Healthy | array-healthy.txt | `[2/2] [UU]`, State: clean, 2 active sync |
| Degraded | array-degraded.txt | `[2/1] [U_]`, State: clean, degraded |
| Rebuilt | array-rebuilt.txt | `[2/2] [UU]`, State: clean, nvme0n1p1 + nvme0n4p1 active sync |

SHA-256 of important.txt, identical in all three moments:
`141ac3064e9fd5eff8d0ffed5231912429a72de1999f03f63739596057a25c05`
(before failure, with one disk removed, after rebuild).

Layout before: only the system disk nvme0n3 had partitions; nvme0n1, nvme0n2, nvme0n4 were empty (lsblk-before.txt).
Layout after: md0 (raid1, /mnt/mirror) sits on nvme0n1p1 and nvme0n4p1; nvme0n2p1 is left outside the array (lsblk-after.txt).

## What did not work the first time

1. I pasted a multi-line command into the VM console. The lines merged into one and bash returned `syntax error near unexpected token '}'`. I typed the commands one at a time instead.
2. In the `array-healthy.txt` capture I typed `--detali` instead of `--detail`; mdadm answered `unrecognized option` and the file had a header with no output. I noticed it with `cat`, and recreated the file.
3. I slowed the rebuild with `dev.raid.speed_limit_max=20000` to watch it, but it would have taken about 20 minutes, so I raised the limit again; the rebuild continued from where it was.

## Evidence

- [evidence/lsblk-before.txt](evidence/lsblk-before.txt) — three empty disks before any work
- [evidence/array-healthy.txt](evidence/array-healthy.txt) — mirror with two active disks
- [evidence/array-degraded.txt](evidence/array-degraded.txt) — one disk removed, array degraded
- [evidence/checksum-before.txt](evidence/checksum-before.txt) — hash before the failure
- [evidence/checksum-after-failure.txt](evidence/checksum-after-failure.txt) — same hash with one disk gone
- [evidence/array-rebuilt.txt](evidence/array-rebuilt.txt) — third disk added, rebuild finished
- [evidence/checksum-after-rebuild.txt](evidence/checksum-after-rebuild.txt) — same hash after rebuild
- [evidence/lsblk-after.txt](evidence/lsblk-after.txt) — layout after
- [evidence/windows-disk.txt](evidence/windows-disk.txt) — Windows disk 1 is GPT, volume E NTFS

## Does RAID 1 help if a colleague deleted the wrong file an hour ago?

No. RAID 1 copies every write to both disks immediately, so the deletion was replicated to the mirror within milliseconds and the file is gone from both. A mirror protects against hardware failure of a disk, not against human error or corruption. Recovering the file needs a backup or snapshot that keeps earlier versions on separate storage.