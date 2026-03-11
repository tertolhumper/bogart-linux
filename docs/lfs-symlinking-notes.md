# LFS/BLFS Shared Library Symlinking Notes

## The Three Symlinks Explained

Every shared library has THREE versions of its filename:

```
libfoo.so.2.3.1   ← REAL FILE    — installed by the package
libfoo.so.2       ← SONAME       — created by ldconfig (runtime)
libfoo.so         ← LINKER NAME  — needed at COMPILE TIME (often missing!)
```

### What each one does

| File | Purpose | Who creates it |
|---|---|---|
| `libfoo.so.2.3.1` | Actual library binary | `make install` |
| `libfoo.so.2` | Runtime linker finds this when program runs | `ldconfig` |
| `libfoo.so` | Compiler finds this when building dependent packages | Sometimes missing! |

---

## Why Libraries Are "Missing" Even Though They're Installed

This is the most common BLFS headache. The package IS there but the
**linker symlink** (`libfoo.so`) is missing so the next package can't compile.

Example error:
```
cannot find -lfoo
```
or:
```
error while loading shared libraries: libfoo.so not found
```

---

## The Fix Workflow

**Step 1 — Check what symlinks exist:**
```bash
ls -la /usr/lib/libfoo.so*
# Example output:
# libfoo.so.2.3.1   ← exists
# libfoo.so.2       ← exists (ldconfig created it)
# libfoo.so         ← MISSING! this is the problem
```

**Step 2 — Create the missing linker symlink:**
```bash
ln -sfv libfoo.so.2.3.1 /usr/lib/libfoo.so
```

**Step 3 — Always run ldconfig after:**
```bash
ldconfig
```

**Step 4 — Verify pkg-config finds it:**
```bash
pkg-config --libs --cflags libfoo
```

---

## After Every Package Install — Checklist

```bash
# 1. Check symlinks are all present
ls -la /usr/lib/libfoo.so*

# 2. Create linker symlink if missing
ln -sfv /usr/lib/libfoo.so.X.X.X /usr/lib/libfoo.so

# 3. Refresh library cache
ldconfig

# 4. Verify pkg-config
pkg-config --libs libfoo

# 5. Verify ldconfig found it
ldconfig -v 2>/dev/null | grep libfoo
```

---

## Why pkg-config Also Fails

Even with correct symlinks, pkg-config may not find the library if
it was installed to `/usr/local` instead of `/usr`.

pkg-config searches:
```
/usr/lib/pkgconfig
/usr/share/pkgconfig
```

It does NOT search `/usr/local/lib/pkgconfig` by default!

**Fix — add to /etc/profile or /root/.bashrc:**
```bash
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/share/pkgconfig:$PKG_CONFIG_PATH
```

---

## The Golden Rules for BLFS

1. **Always compile with `--prefix=/usr`**
   ```bash
   ./configure --prefix=/usr          # autotools
   cmake -DCMAKE_INSTALL_PREFIX=/usr  # cmake
   meson setup build --prefix=/usr    # meson
   ```

2. **Always run `ldconfig` after every install**

3. **Always check symlinks after install**
   ```bash
   ls -la /usr/lib/libfoo.so*
   ```

4. **Always verify pkg-config after install**
   ```bash
   pkg-config --libs libfoo
   ```

---

## With GNU Stow (Bare Metal Build)

Install to stow directory first, then stow it:
```bash
./configure --prefix=/usr/local/stow/packagename-version
make
make install
cd /usr/local/stow
stow packagename-version
ldconfig
```

Stow creates ALL symlinks including `.so` linker symlinks cleanly.
`stow -D packagename-version` removes ALL of them cleanly — no orphans!

---

## Real World Example — Hyprland Stack

These packages are common culprits for missing symlinks:

| Package | Library | Common issue |
|---|---|---|
| hyprutils | `libhyprutils.so` | linker symlink missing |
| aquamarine | `libaquamarine.so` | wrong prefix |
| hyprgraphics | `libhyprgraphics.so` | linker symlink missing |
| wayland | `libwayland-client.so` | usually fine |
| wlroots | `libwlroots.so` | version mismatch |

**Always check after installing any hypr* package:**
```bash
ls -la /usr/lib/libhypr*.so*
ldconfig
pkg-config --libs hyprutils
```

---

## Quick Reference — Find a Library

```bash
# Find where a library is installed
find /usr /usr/local -name "libfoo.so*" 2>/dev/null

# Check if ldconfig knows about it
ldconfig -v 2>/dev/null | grep foo

# Check if pkg-config finds it
pkg-config --libs --cflags foo

# After plocate/mlocate is installed (much faster)
locate libfoo.so
```
