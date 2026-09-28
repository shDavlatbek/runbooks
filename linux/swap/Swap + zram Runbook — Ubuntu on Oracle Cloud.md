# Swap + zram Runbook — Ubuntu on Oracle Cloud

Sep 28, 2026 · @Aziz

## Scope

Sets up 4 GB of disk swap plus compressed zram swap on Ubuntu 24.04 running on an Oracle Cloud Always Free micro instance (VM.Standard.E2.1.Micro, 1 vCPU, 1 GB RAM, 47 GB boot volume).

When finished, zram takes swap traffic first and the disk file catches overflow. Needs a sudo-capable account and about ten minutes.

## Step 0 — Check what already exists

```bash
swapon --show
free -h
```

No output from `swapon --show` means no swap is active. If a `/swapfile` already exists at the size you want, skip to Step 3.

## Step 1 — Create the 4 GB swap file

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

The `chmod 600` is not optional — swap holds whatever was in memory, and `mkswap` warns or refuses on a world-readable file.

If `fallocate` fails (some filesystems don't support it), build the file with `dd` instead, then continue from `chmod`:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
```

## Step 2 — Make it survive reboots

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Skipping this is the most common mistake: swap works until the next boot, then silently vanishes.

## Step 3 — Tune swappiness

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swap.conf
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.d/99-swap.conf
sudo sysctl --system
```

The default swappiness of 60 pages out too eagerly. On a micro instance the swap file lives on network-backed block storage and the CPU share is small, so heavy paging makes the box feel dead. A value of 10 keeps swap as a safety net rather than routine storage.

## Step 4 — Add zram

Install the kernel module package first. Oracle's Ubuntu images ship a trimmed module set without `zram.ko`, so installing `zram-tools` on its own leaves a service that fails at every boot.

```bash
sudo apt install linux-modules-extra-$(uname -r)
sudo apt install zram-tools
```

Set the algorithm and size in `/etc/default/zramswap`:

```
ALGO=zstd
PERCENT=50
```

`PERCENT=50` gives roughly 477 MB of compressed swap on a 1 GB instance. Because compression typically reaches 2–3×, that holds well over 1 GB of real pages.

```bash
sudo modprobe zram
sudo systemctl restart zramswap
```

Confirm the algorithm was accepted — the bracketed entry is the active one:

```bash
cat /sys/block/zram0/comp_algorithm
```

If `zstd` is absent from that list, set `ALGO=lzo-rle` instead and restart the service.

## Step 5 — Load the zram module at boot

`modprobe` by hand lasts only until the next reboot. Without this, `zramswap.service` fails on every boot.

```bash
echo zram | sudo tee /etc/modules-load.d/zram.conf
```

## Step 6 — Verify

Reboot, then check that both devices come back without any manual step:

```bash
sudo reboot
# after reconnecting
swapon --show
free -h
systemctl status zramswap --no-pager
```

Correct output looks like this — zram at priority 100 is used first, the disk file at -2 catches overflow:

```
NAME       TYPE      SIZE  USED PRIO
/swapfile  file        4G    0B   -2
/dev/zram0 partition 477M    0B  100
```

`0B` used on a freshly booted instance is normal; the devices fill only under memory pressure.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `modprobe: FATAL: Module zram not found` | Kernel module package missing — normal on Oracle Ubuntu images | `sudo apt install linux-modules-extra-$(uname -r)` |
| `zramswap.service` failed, exit code 1 | Same missing module, or an `ALGO` this kernel rejects | Check `journalctl -xeu zramswap.service`; compare `ALGO` against `/sys/block/zram0/comp_algorithm` |
| Swap gone after reboot | No `/etc/fstab` entry, or no `/etc/modules-load.d/zram.conf` | Redo Step 2 and Step 5 |
| `E: dpkg was interrupted` | A previous apt run was killed, often by the OOM killer | `sudo dpkg --configure -a`, then retry |
| `mkswap` refuses the file | Permissions too open | `sudo chmod 600 /swapfile` |
| `fallocate` fails | Filesystem doesn't support it | Use the `dd` command in Step 1 |
| `linux-modules-extra-$(uname -r)` not found | Package name differs for this kernel | `apt-cache search linux-modules-extra \| grep oracle` |

## Notes

Swap buys headroom, not memory. If an instance sits at several hundred MB of swap right after boot, the workload does not fit in 1 GB and paging on network-backed storage will make it unresponsive. Cap the offending service with `docker update --memory`, drop it, or move to a VM.Standard.A1.Flex instance, where the Always Free allocation reaches 4 OCPUs and 24 GB of RAM.

A kernel upgrade installs a new `linux-modules-extra` version and reverts to the trimmed set, so zram can break after an unattended upgrade. If `zramswap` starts failing for no apparent reason, rerun Step 4's first command.

Repeated unclean reboots in `last` output usually mean the OOM killer. Confirm with `sudo journalctl -k | grep -i 'out of memory'`.

Disk cost is negligible here: a 4 GB file against a 47 GB minimum boot volume. Sizing swap larger than 4 GB rarely helps — swapping that heavily means thrashing either way.
