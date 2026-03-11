# Bogart Linux 1.3

<p align="center">
  <img src="https://github.com/YOURUSERNAME/bogart-linux/raw/main/assets/bogart-logo.png" width="120"/>
</p>

<p align="center">
  A custom <strong>Linux From Scratch (LFS/BLFS)</strong> based live distribution featuring <strong>Hyprland</strong> on Wayland.
</p>

---

## System Info

| Property | Version |
|---|---|
| Base | LFS 13.0 + BLFS (systemd) |
| Kernel | Linux 6.18.10 |
| Window Manager | Hyprland 0.54.1 (Wayland) |
| Shell | bash 5.3.0 |
| Terminal | kitty 0.39.1 |
| Greeter | greetd + tuigreet |
| Total Packages | 272 (LFS: 88, BLFS: 181, Config: 3) |
| ISO Size | ~3.7 GB |

---

## Download

- 📀 [ISO Download](https://archive.org/details/bogart-linux-1.3-live)
- 🛡️ [VirusTotal Scan](https://virustotal.com/LINK_HERE) *(pending)*
- 🔑 MD5: `d3c97380b59aa9af424f55cc3ae31ec6`

---

## How to Use

### Flashing to USB (Recommended: Ventoy)

1. Download and install [Ventoy](https://www.ventoy.net) on your USB drive
2. Copy `bogart-linux-1.3-live.iso` to the Ventoy partition
3. Boot from USB and select **Bogart Linux 1.3 Live** from the Ventoy menu

### Login

- Username: `tertol`
- Password: `123`

After login, run:
```bash
./start-hypr.sh
```

---

## ⚠️ NVIDIA GPU Warning

If you have an **NVIDIA GPU** (GTX/RTX series), Hyprland may **crash or show a black screen** on the first login attempt due to the open-source nouveau driver conflict.

**Workaround:**
1. When greetd login screen appears, press `Alt + F2 or Alt + F3` to switch to **TTY3**
2. Login as `tertol`
3. Run `./start-hypr.sh` manually

This is a known limitation of the live ISO. NVIDIA proprietary drivers are not included.

---

## Testing in KVM/QEMU (Virtual Machine)

No need for a full XML definition. Just use these parameters:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 8192 \
  -smp 4 \
  -bios /usr/share/ovmf/OVMF.fd \
  -drive file=/path/to/bogart-linux-1.3-live.iso,media=cdrom,if=ide \
  -boot d \
  -vga virtio \
  -display spice-app \
  -device virtio-net,netdev=net0 \
  -netdev user,id=net0
```

Or in **virt-manager**:
- New VM → Import existing disk image
- Select the ISO as **CDROM** with **IDE bus** (not SATA, not VirtIO)
- No hard disk needed
- 8GB RAM, 4 vCPUs recommended
- Firmware: UEFI (OVMF)

---

## Hyprland Dotfiles

The Hyprland configuration used in Bogart Linux is based on the excellent **[Hyprland-Dots](https://github.com/JaKooLit/Hyprland-Dots)** project by **[JaKooLit](https://github.com/JaKooLit)**, with personal modifications to fit the Bogart Linux environment and aesthetic.

> 🙏 **Special thanks to JaKooLit** for creating and sharing such a polished and well-structured Hyprland configuration. His work made it significantly easier to build a functional and beautiful desktop experience on a from-scratch Linux system. Huge thanks as well to all the continuing maintainers and contributors who keep the project alive and improving — your efforts are very much appreciated by the community.

Configuration lives in `~/.config/hypr/`. Key files:

| File | Purpose |
|---|---|
| `hyprland.conf` | Main Hyprland config |
| `configs/keybinds.conf` | Keybindings |
| `configs/WindowRules.conf` | Window rules |
| `configs/animations.conf` | Animations |
| `configs/decoration.conf` | Borders, blur, shadows |
| `configs/monitors.conf` | Monitor setup |

### Waybar
- Config: `~/.config/waybar/config`
- Style: `~/.config/waybar/style.css`

### Theme
- Color scheme: Custom warm brown/gold (`#BD9161`, `#C99962`)
- Font: JetBrains Mono

> For the original dotfiles and installation scripts, visit: https://github.com/JaKooLit/Hyprland-Dots

---

## Build Info

Built entirely from source using [Linux From Scratch 13.0](https://www.linuxfromscratch.org/lfs/) and [Beyond LFS](https://www.linuxfromscratch.org/blfs/).

### Live ISO Stack
| Tool | Version |
|---|---|
| squashfs-tools | 4.7.5 |
| xorriso | 1.5.7 |
| dracut-ng | 107 |
| Compression | zstd level 15 |

### Notable Packages
- Mesa (with llvmpipe software rendering)
- PipeWire + WirePlumber
- NetworkManager
- Firefox
- kitty, btop, fastfetch

---

## Known Issues

- NVIDIA GPU users need to use TTY3 workaround (see warning above)
- `swappy` screenshot tool may freeze on some hardware
- Live session is not persistent — changes are lost on reboot

---

## License

This project is built on top of LFS/BLFS which is MIT licensed.
Hyprland and all included packages retain their respective licenses.

---

<p align="center">Made with ❤️ on Linux From Scratch</p>
