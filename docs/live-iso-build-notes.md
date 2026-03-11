# Bogart Linux 1.3 — Live ISO Build Notes
## fstab, squashfs, initramfs, and xorriso Workflow

---

## Overview

Building a live ISO from an installed LFS/BLFS system requires careful handling
of the fstab file. This document covers the full workflow, automation attempts,
real outcomes, and why the manual approach was chosen as the final solution.

---

## The Core Problem — fstab in a Live ISO

An installed LFS system has a real fstab pointing to actual disk partitions:

```
/dev/vda3      /          ext4   defaults        1  1
/dev/vda1      /boot/efi  vfat   defaults        0  2
/dev/vda2      swap       swap   pri=1           0  0
/swapfile      none       swap   sw              0  0
```

When this fstab gets squashed into the live ISO and the system boots from USB,
systemd reads it and tries to mount `/dev/vda3`, `/dev/vda2`, and `/swapfile`
— none of which exist on the live boot target machine. This causes:

```
[FAILED] Failed to activate swap /swapfile
[DEPEND] Dependency failed for Swaps
```

And in worst cases — emergency mode or complete boot failure.

**The fstab inside the squash MUST only contain:**
```
# Begin /etc/fstab
tmpfs   /tmp    tmpfs   defaults    0  0
# End /etc/fstab
```

---

## The Manual Workflow (Current Solution)

This is the reliable, battle-tested workflow used for Bogart Linux 1.3:

### Prerequisites
- `/etc/fstab.install` — backup of original fstab (vda partitions + swapfile)
- `/etc/fstab.live` — tmpfs-only fstab for live boot

### Step 1 — Clear bash history
```bash
cat /dev/null > /home/tertol/.bash_history
cat /dev/null > /root/.bash_history
```

### Step 2 — Switch to live fstab
```bash
cat > /etc/fstab << 'EOF'
# Begin /etc/fstab
tmpfs   /tmp    tmpfs   defaults    0  0
# End /etc/fstab
EOF
```

### Step 3 — Squash the filesystem
```bash
rm -f /iso/LiveOS/squashfs.img
mksquashfs / /iso/LiveOS/squashfs.img \
  -comp zstd \
  -Xcompression-level 15 \
  -wildcards \
  -no-sparse \
  -no-xattrs \
  -e boot \
  -e dev \
  -e proc \
  -e sys \
  -e run \
  -e tmp \
  -e iso \
  -e sources \
  -e swapfile \
  -e bogart-linux-1.3-live.iso \
  -e root/.cargo \
  -e root/.cache \
  -e home/tertol/.bash_history \
  -e root/.bash_history \
  -e "*.pyc" \
  -e var/log \
  -b 131072 \
  -progress
```

### Step 4 — IMMEDIATELY restore real fstab (in another terminal)
```bash
cp /etc/fstab.install /etc/fstab
```

> ⚠️ Do this as soon as squash starts — do NOT wait for squash to finish.
> The squash captures the filesystem state at start time so restoring fstab
> immediately is safe and prevents accidental reboot with wrong fstab.

### Step 5 — Rebuild initramfs
```bash
dracut --force --kver 6.18.10 \
  --add "dmsquash-live" \
  --omit "iscsi fcoe fcoe-uefi nbd" \
  --no-hostonly \
  /boot/initramfs-6.18.10-live.img

cp /boot/initramfs-6.18.10-live.img /iso/boot/initramfs.img
cp /boot/vmlinuz-6.18.10-lfs-13.0-systemd /iso/boot/vmlinuz
```

### Step 6 — Build the ISO
```bash
rm -f /bogart-linux-1.3-live.iso
xorriso -as mkisofs \
  -iso-level 3 \
  -volid "BOGARTLIVE" \
  -full-iso9660-filenames \
  -R -J \
  --efi-boot boot/grub/efi.img \
  -efi-boot-part \
  --efi-boot-image \
  -o /bogart-linux-1.3-live.iso \
  /iso

ls -lh /bogart-linux-1.3-live.iso
```

### Step 7 — Transfer to host
```bash
# On host machine
sftp root@192.168.122.157
get /bogart-linux-1.3-live.iso /home/motchi/bogart-linux-1.3-live.iso
```

---

## Automation Attempt — dracut livefstab Module

### What We Tried

To eliminate the manual fstab swap step, a custom dracut module was created
to automatically replace fstab during initramfs boot:

**Module structure:**
```
/usr/lib/dracut/modules.d/99livefstab/
  module-setup.sh
  live-fstab.sh
```

**live-fstab.sh:**
```bash
#!/bin/bash
if [ -f /sysroot/etc/fstab.live ]; then
    cp /sysroot/etc/fstab.live /sysroot/etc/fstab
fi
```

**module-setup.sh (first attempt):**
```bash
#!/bin/bash
check() { return 0; }
depends() { return 0; }
install() {
    inst_hook initqueue/settled 99 "$moddir/live-fstab.sh"
}
```

### Outcome in KVM VM

- `initqueue/settled 99` — hook ran too late, swap errors still appeared
- `pre-pivot 01` — hook ran too early, sysroot not fully mounted
- `pre-pivot 10` — worked in KVM! Boot time dropped to **17 seconds**
- graphical.target reached in **1.5 seconds** ✅

### Outcome on Real Hardware (Ventoy USB)

- `pre-pivot 10` — system booted into initqueue for **26 seconds then powered off**
- Root cause unknown — sysroot mount timing differs between KVM virtio and
  real hardware SATA/USB boot path
- The hook timing that works perfectly in KVM does not translate reliably
  to real hardware USB boot

### Why Manual Was Chosen

The dracut hook approach introduced an unpredictable variable — hook timing
behaves differently depending on:
- KVM virtio vs real hardware SATA
- USB 3.0 vs USB 2.0 Ventoy boot speed
- Hardware detection timing (NVMe, sound, rfkill all settle at different times)

The manual fstab swap is:
- **Predictable** — always works regardless of hardware
- **Simple** — one command before squash, one after
- **Safe** — no initramfs complexity
- **Fast** — no debugging hook timing across different hardware

The 30 second live boot time on real hardware is acceptable for a live ISO.

---

## Boot Optimization — grub.cfg

These kernel parameters were added to significantly reduce boot time:

```
rd.lvm=0 rd.luks=0 rd.md=0 rd.dm=0
```
Prevents dracut from probing for LVM, LUKS, MD RAID, and device mapper
devices that don't exist in a live environment.

```
systemd.mask=dev-virtio\x2dports-com.redhat.spice.0.device
```
This was the biggest win — without it systemd waited **1 minute 30 seconds**
for a SPICE virtio port that only exists in KVM. On real hardware this device
never appears causing a full timeout on every boot.

**Result:** Boot time on real hardware dropped from ~2 minutes to ~30 seconds.

---

## Current grub.cfg

```
set default=0
set timeout=10
insmod all_video
insmod iso9660

menuentry "Bogart Linux 1.3 Live" {
    search --no-floppy --label --set=root BOGARTLIVE
    linux /boot/vmlinuz root=live:CDLABEL=BOGARTLIVE rd.live.image rd.lvm=0 rd.luks=0 rd.md=0 rd.dm=0 net.ifnames=0 biosdevname=0 systemd.mask=dev-vda1.device systemd.mask=dev-vda2.device systemd.mask=boot-efi.mount systemd.mask=swap.target systemd.mask=swapfile.swap systemd.mask=dev-virtio\x2dports-com.redhat.spice.0.device
    initrd /boot/initramfs.img
}
menuentry "Bogart Linux 1.3 Live (RAM)" {
    search --no-floppy --label --set=root BOGARTLIVE
    linux /boot/vmlinuz root=live:CDLABEL=BOGARTLIVE rd.live.image rd.live.ram rd.lvm=0 rd.luks=0 rd.md=0 rd.dm=0 net.ifnames=0 biosdevname=0 systemd.mask=dev-vda1.device systemd.mask=dev-vda2.device systemd.mask=boot-efi.mount systemd.mask=swap.target systemd.mask=swapfile.swap systemd.mask=dev-virtio\x2dports-com.redhat.spice.0.device
    initrd /boot/initramfs.img
}
```

---

## Key Lessons

1. **KVM and real hardware boot differently** — what works in VM does not
   always work on real hardware, especially initramfs hook timing.

2. **SPICE virtio device timeout** — always mask this when building a live ISO
   intended for real hardware. KVM users are unaffected since the device exists.

3. **rd.lvm=0 rd.luks=0 rd.md=0 rd.dm=0** — always add these to live ISO
   kernel parameters. Saves significant boot time by skipping device probing
   that is irrelevant to a live environment.

4. **Simple is reliable** — the manual fstab swap is one extra step but
   eliminates an entire class of unpredictable boot failures across hardware.

5. **Always backup fstab** — keep `/etc/fstab.install` as a permanent backup.
   One wrong reboot with the live fstab on the build VM breaks everything.
