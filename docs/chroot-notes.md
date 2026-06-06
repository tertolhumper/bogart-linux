# Chroot into Bogart Linux

mount -o subvol=@ /dev/nvme1n1p2 /mnt
mount -o subvol=@home /dev/nvme1n1p2 /mnt/home
mount -o subvol=@cache /dev/nvme1n1p2 /mnt/var/cache
mount -o subvol=@log /dev/nvme1n1p2 /mnt/var/log
mount -o subvol=@tmp /dev/nvme1n1p2 /mnt/var/tmp
mount -o subvol=@libvirt /dev/nvme1n1p2 /mnt/var/lib/libvirt/images

mount /dev/nvme1n1p1 /mnt/boot/efi

mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount -t proc proc /mnt/proc
mount -t sysfs sysfs /mnt/sys
mount -t tmpfs tmpfs /mnt/run
mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars

chroot /mnt /usr/bin/env -i \
  HOME=/root \
  TERM=xterm-256color \
  PATH=/usr/bin:/usr/sbin \
  /bin/bash --login

# exit
umount -R /mnt
