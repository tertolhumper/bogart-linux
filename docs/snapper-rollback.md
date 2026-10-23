# Recover Bogart Linux After a Failed systemd Upgrade (Btrfs)

## 1. Boot a Linux live environment

Identify the Bogart Linux Btrfs partition and mount its top-level subvolume (ID 5).

```bash
sudo mkdir -p /mnt/btrfs_root
sudo mount -t btrfs -o subvolid=5 /dev/nvme1n1p2 /mnt/btrfs_root
```

Verify the mounted filesystem:

```bash
sudo btrfs subvolume get-default /mnt/btrfs_root
sudo btrfs subvolume list /mnt/btrfs_root
```

## 2. Find the stable snapshot

List available snapshots and inspect their metadata. Snapshot numbers vary.

```bash
for n in 2200 2201; do
    echo "===== SNAPSHOT $n ====="
    sudo cat "/mnt/btrfs_root/@snapshots/$n/info.xml"
    echo
done
```

Select the snapshot whose description identifies it as the last stable state before the failed upgrade.

Example:

```text
<num>2201</num>
<date>2026-10-23 13:48:59</date>
<description>prior to upgrade systemd-262 to systemd-262.1</description>
```

Verify that the selected snapshot exists and is readable:

```bash
sudo btrfs subvolume show /mnt/btrfs_root/@snapshots/2201/snapshot
sudo ls -l /mnt/btrfs_root/@snapshots/2201/snapshot/usr/lib/systemd/systemd
```

**Replace `2201` with the snapshot number selected from the metadata.**

## 3. Preserve the broken root

Move the existing root subvolume out of the way. Do not delete it before the restored system has booted successfully.

```bash
sudo mv /mnt/btrfs_root/@ /mnt/btrfs_root/@_broken_systemd2621
```

Choose a descriptive backup name for the failed upgrade.

## 4. Restore a writable root

Create a writable snapshot of the selected read-only recovery snapshot:

```bash
sudo btrfs subvolume snapshot \
    /mnt/btrfs_root/@snapshots/2201/snapshot \
    /mnt/btrfs_root/@
```

Verify the new root and obtain its numeric subvolume ID:

```bash
sudo btrfs subvolume show /mnt/btrfs_root/@
```

Set the new root as the default. Substitute the actual ID reported by the previous command.

```bash
sudo btrfs subvolume set-default 2471 /mnt/btrfs_root
sudo btrfs subvolume get-default /mnt/btrfs_root
```

The ID above (`2471`) is specific to this recovery and will differ in future recoveries.

## 5. Verify boot configuration

Check the restored systemd binary and the GRUB kernel parameters:

```bash
sudo ls -l /mnt/btrfs_root/@/usr/lib/systemd/systemd
sudo grep -nE 'root=|rootflags=' /mnt/btrfs_root/@/boot/grub/grub.cfg
```

Confirm that the kernel command line points to the correct Btrfs filesystem UUID and uses:

```text
rootflags=subvol=@
```

Do not reboot until the restored root, default subvolume, and boot configuration have been checked.

## 6. Reboot and verify

```bash
sudo reboot
```

After booting Bogart Linux:

```bash
systemctl --version
systemctl is-system-running
findmnt /
```

Confirm that the expected systemd version is running and `/` is mounted from the restored `@` subvolume.

A `degraded` system state means some systemd units are not fully operational; inspect the failures before considering the recovery complete.

```bash
systemctl --failed
journalctl -b -p err
```

## 7. Remove the fallback only after successful validation

Boot the live environment again if necessary, mount the Btrfs top-level subvolume, and verify the exact name of the preserved broken root:

```bash
sudo mount -t btrfs -o subvolid=5 /dev/nvme1n1p2 /mnt/btrfs_root
sudo btrfs subvolume show /mnt/btrfs_root/@_broken_systemd2621
```

Only after the restored system has been tested and the fallback is no longer needed, delete that specific subvolume:

```bash
sudo btrfs subvolume delete /mnt/btrfs_root/@_broken_systemd2621
```

Unmount the filesystem when finished:

```bash
sudo umount /mnt/btrfs_root
```

**Important:** Keep the snapshot and broken-root fallback until the restored installation has been validated. Do not delete unrelated Btrfs subvolumes such as `@home`, `@cache`, `@log`, `@tmp`, or `@snapshots`.
