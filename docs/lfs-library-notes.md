# LFS/BLFS Library Notes

## How Libraries Work in LFS

Everything lives in `/usr/lib`. The other paths are just symlinks for compatibility:
```
/lib   → /usr/lib
/lib64 → /usr/lib
```

---

## The Golden Rule for BLFS

Always set `--prefix=/usr` when compiling. This ensures libraries land in `/usr/lib` where ldconfig can find them.

| Build System | Command |
|---|---|
| autotools | `./configure --prefix=/usr` |
| cmake | `cmake -DCMAKE_INSTALL_PREFIX=/usr ..` |
| meson | `meson setup build --prefix=/usr` |

---

## Why Packages Fail Without --prefix=/usr

Non-BLFS packages default to `/usr/local` which means:
- Libraries go to `/usr/local/lib` — ldconfig may not scan here
- pkg-config can't find them — dependent packages fail to compile
- Results in: `error while loading shared libraries: libfoo.so not found`

---

## Fix When a Library is Not Found

```bash
# Option 1 — recompile with correct prefix
./configure --prefix=/usr

# Option 2 — add /usr/local/lib to ldconfig scan
echo "/usr/local/lib" >> /etc/ld.so.conf.d/local.conf
ldconfig
```

---

## After Every Install — Run These

```bash
ldconfig                              # refresh library cache
pkg-config --libs --cflags libfoo    # verify pkg-config finds it
ldconfig -v 2>/dev/null | grep foo   # verify library is found
```

---

## Common Culprits in Hyprland/Wayland Stack

Packages like `aquamarine`, `hyprutils`, `hyprgraphics` use cmake/meson and sometimes default to `/usr/local`. Always force `--prefix=/usr` for these!
