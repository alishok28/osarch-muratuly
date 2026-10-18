# Lab 1 — First VM

## VM configuration
- Guest OS: Ubuntu Server 26.04.1 LTS, ARM64, console only
- Host: MacBook Air, Apple M2
- Host resources: 8 CPU cores, 16 GB RAM, 245.1 GB disk
- VM: 2 CPU cores, 2048 MB RAM, 20 GB dynamically allocated disk
- Network: NAT
- Hostname: lab01
- Snapshot: clean-install

## Resource justification
The host has 8 CPU cores and 16 GB of RAM. I assigned 2 CPU cores and 2048 MB of RAM to Ubuntu, leaving most host resources available to macOS and other applications. VMware Fusion recommends at least 3072 MB, so this VM uses less than recommended.
The virtual disk is 20 GB and dynamically allocated, so its file grows as the guest writes data. My host disk has 245.1 GB total capacity.
I reserve 4 GB RAM and 40 GB disk for next week's Windows VM.

## Checks inside the guest
- nproc: 2 — matches the 2 CPU cores assigned in VMware Fusion.
- free -h: about 1.6 GiB total memory — slightly less than 2048 MB because some memory is used by the system and reported in GiB.
- df -h /: root filesystem is about 9.8 GB, smaller than the 20 GB virtual disk because the disk also contains boot partitions and unused space in the LVM volume group.

## Break and restore
I changed the hostname from lab01 to broken with hostnamectl.
- Command proving it was broken: hostname — output was broken.
- I restored the clean-install snapshot.
- Command proving it was fixed: hostname — output was lab01.

## Snapshot note
A snapshot taken now would capture the current state. Reverting to clean-install would return the VM to that earlier state and discard later changes unless I save them first.

## Evidence
See the evidence folder for guest-release.txt, vm-settings.png, snapshot-list.png, and broken.png.
16 GR RAM, 12.57 GB USED. 2 GB FOR UBUNTU AND 4 GB RAM  RESERVED FOR THE NEXT WEEK.
DISK IS 241.1 GB, USED 13.7 GB. 40 GB RESERVED, 20 GB GIVEN TO UBUNTU