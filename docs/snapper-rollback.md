# Recovering Bogart Linux After a Failed systemd Upgrade (Btrfs)

## 1. Boot a live environment

Mount the Btrfs top-level subvolume:

```bash
mkdir -p /mnt/btrfs_root
mount -t btrfs -o subvolid=5 /dev/nvme1n1p2 /mnt/btrfs_root
```

Confirm the mount:

```bash
btrfs subvolume get-default /mnt/btrfs_root
btrfs subvolume list /mnt/btrfs_root
```

## 2. Find the stable snapshot

Inspect snapshot metadata to find the last stable state before the failed upgrade:

```bash
for n in 2200 2201; do
    echo "===== $n ====="
    cat /mnt/btrfs_root/@snapshots/$n/info.xml
    echo
done
```

Example of what you're looking for:

```
<num>2201</num>
<date>2026-10-23 13:48:59</date>
<description>prior to upgrade systemd-262 to systemd-262.1</description>
```

Verify it's readable:

```bash
btrfs subvolume show /mnt/btrfs_root/@snapshots/2201/snapshot
ls -l /mnt/btrfs_root/@snapshots/2201/snapshot/usr/lib/systemd/systemd
```

Adjust the snapshot number to match what you found.

## 3. Preserve the broken root

Move it aside — don't delete it yet:

```bash
mv /mnt/btrfs_root/@ /mnt/btrfs_root/@_broken_systemd2621
```

## 4. Restore a writable root

Create a writable snapshot from the stable snapshot:

```bash
btrfs subvolume snapshot \
    /mnt/btrfs_root/@snapshots/2201/snapshot \
    /mnt/btrfs_root/@
```

Get its subvolume ID:

```bash
btrfs subvolume show /mnt/btrfs_root/@
```

Set it as the default (replace `2471` with the actual ID):

```bash
btrfs subvolume set-default 2471 /mnt/btrfs_root
btrfs subvolume get-default /mnt/btrfs_root
```

## 5. Check boot configuration

Verify the restored systemd binary and GRUB kernel parameters:

```bash
ls -l /mnt/btrfs_root/@/usr/lib/systemd/systemd
grep -nE 'root=|rootflags=' /mnt/btrfs_root/@/boot/grub/grub.cfg
```

The kernel line must include:

```
rootflags=subvol=@
```

Don't reboot until the restored root, default subvolume, and boot config are all confirmed.

## 6. Reboot and verify

```bash
reboot
```

After booting:

```bash
systemctl --version
systemctl is-system-running
findmnt /
```

A `degraded` state means some units failed — check before calling the recovery done:

```bash
systemctl --failed
journalctl -b -p err
```

## 7. Clean up after validation

Only after the restored system is confirmed working, boot the live environment again and delete the broken root:

```bash
mount -t btrfs -o subvolid=5 /dev/nvme1n1p2 /mnt/btrfs_root
btrfs subvolume show /mnt/btrfs_root/@_broken_systemd2621
btrfs subvolume delete /mnt/btrfs_root/@_broken_systemd2621
umount /mnt/btrfs_root
```

Don't touch `@home`, `@cache`, `@log`, `@tmp`, or `@snapshots`.
