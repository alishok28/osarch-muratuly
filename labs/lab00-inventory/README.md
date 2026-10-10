# Lab 0 — Inventory your own machine

Pair: Individual (Mukhammadali)
Driver first half: Mukhammadali
Machine: MacBook Air (Mac14,2), Apple M2
Date: 10.10.2026

## What I did

```sh
system_profiler SPHardwareDataType
sysctl -n hw.logicalcpu
system_profiler SPMemoryDataType
diskutil info /
df -h /
sysctl kern.hv_support
