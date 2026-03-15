#1. Mounting partion in a BTRFS 

mkdir -p /mnt/lfs
mount -o subvol=@ /dev/sda2 /mnt/lfs
mount -o subvol=@home /dev/sda2 /mnt/lfs/home
mount -o subvol=@cache /dev/sda2 /mnt/lfs/var/cache
mount -o subvol=@log /dev/sda2 /mnt/lfs/var/log
mount -o subvol=@tmp /dev/sda2 /mnt/lfs/var/tmp 
mount -o subvol=@snapshots /dev/sda2 /mnt/lfs/.snapshots
mount -o subvol=@images /dev/sda2 /mnt/lfs/var/lib/libvirt/images
mount /dev/sda1 /mnt/lfs/boot/efi


mount --bind /dev /mnt/lfs/dev
mount --bind /dev/pts /mnt/lfs/dev/pts
mount -t proc proc /mnt/lfs/proc
mount -t sysfs sysfs /mnt/lfs/sys
mount -t tmpfs tmpfs /mnt/lfs/run

#2. Chroot
 
chroot /mnt/lfs /usr/bin/env -i \
HOME=/root \
TERM="$TERM" \
PS1="(lfs chroot) \u:\w\$ " \
PATH=/usr/bin:/usr/sbin:/bin:/sbin \
/bin/bash --login


#3.Umount

exit

umount -l /mnt/lfs/dev/pts
umount -l /mnt/lfs/dev
umount -l /mnt/lfs/proc
umount -l /mnt/lfs/sys
umount -l /mnt/lfs/run

umount -l /mnt/lfs/boot/efi
umount -l /mnt/lfs/var/lib/libvirt/images
umount -l /mnt/lfs/.snapshots
umount -l /mnt/lfs/var/tmp
umount -l /mnt/lfs/var/log
umount -l /mnt/lfs/var/cache
umount -l /mnt/lfs/home
umount -l /mnt/lfs

