# Bogart Linux 1.3

LFS/BLFS-based live distribution with Hyprland.

## System
- Base: LFS 13.0 + BLFS (systemd)
- Kernel: 6.18.10
- WM: Hyprland 0.54.1
- Shell: bash 5.3.0
- Terminal: kitty 0.39.1
- Packages: 272 (LFS: 88, BLFS: 181, Config: 3)
- ISO: ~3.7 GB

## Download
- [ISO Download](https://archive.org/details/bogartlinux)
- [VirusTotal Scan](https://www.virustotal.com/gui/url/1b7dad2cb0ec0de1b3993c14b108c7e98fc443f415433acc91c2e63feac3d6f3?nocache=1)
- MD5: f3cc930d88e46584c487edb35c290a6f

## Login
Username: tertol | Password: 123

After login run: ./start-hypr.sh

## Flash to USB
Use Ventoy. Copy ISO to Ventoy partition, boot and select from menu.

## QEMU
qemu-system-x86_64 -enable-kvm -m 8192 -smp 4 \
  -bios /usr/share/ovmf/OVMF.fd \
  -drive file=bogart-linux-1.3-live.iso,media=cdrom,if=ide \
  -boot d -vga virtio -display spice-app \
  -device virtio-net,netdev=net0 -netdev user,id=net0

## NVIDIA Warning
Proprietary drivers not included. If Hyprland crashes, switch to TTY2/TTY3
and run ./start-hypr.sh manually.

## Known Issues
- Baremetal fail to boot — use KVM/QEMU for reliable results.
- Live session is not persistent.
- Swappy may freeze on some hardware.
- Some keybinds may not work.

## License
Built on LFS/BLFS (MIT licensed). All included packages retain their respective licenses.
