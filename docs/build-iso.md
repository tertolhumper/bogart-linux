# IMPORTANT: Backup fstab first before running step 1
cp /etc/fstab /etc/fstab.install

# 1. Change fstab
cat > /etc/fstab << 'EOF'
# Begin /etc/fstab
tmpfs   /tmp    tmpfs   defaults    0  0
# End /etc/fstab
EOF

# 2. Clear history
cat /dev/null > /home/tertol/.bash_history
cat /dev/null > /root/.bash_history
history -c

# 3. Squash
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
  -e "bogart-linux-1.3-live.iso" \
  -e "root/.cargo" \
  -e "root/.cache" \
  -e "home/*/.git" \
  -e "var/tmp/.*" \
  -e "var/log" \
  -e "usr/share/doc" \
  -e "usr/share/man" \
  -e "var/cache" \
  -b 131072 \
  -progress


#After the squashing starts fstab immediately

cp /etc/fstab.install /etc/fstab

# 4. Initramfs
dracut --force --kver 6.18.10 \
  --add "dmsquash-live" \
  --omit "iscsi fcoe fcoe-uefi nbd" \
  --no-hostonly \
  /boot/initramfs-6.18.10-live.img

cp /boot/initramfs-6.18.10-live.img /iso/boot/initramfs.img
cp /boot/vmlinuz-6.18.10-lfs-13.0-systemd /iso/boot/vmlinuz

# 5. xorriso
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
md5sum /bogart-linux-1.3-live.iso 

# 6. SFTP & check the md5sum are identical

sftp root@192.168.xxx.xx

get bogart-linux-1.3-live.iso /home/motchi/

exit

md5sum bogart-linux-1.3-live.iso




