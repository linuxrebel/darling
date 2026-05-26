# Darling

![Darling logo](https://darlinghq.org/img/darling250.png)

Darling is a runtime environment for macOS applications.

Please note that most GUI applications will not run at the moment.

> **Note for Fedora 44 / Clang 22 users:** This fork includes all fixes required
> to build Darling on Fedora 44 with Clang 22. See the
> [Fedora 44 Build Guide](#building-on-fedora-44--clang-22) section below.

## Download

Packages for some distributions are available for download
under [releases](https://github.com/darlinghq/darling/releases).

## Build Instructions

For general build instructions, visit [Darling Docs](https://docs.darlinghq.org/build-instructions.html).

---

## Building on Fedora 44 / Clang 22

Clang 22 (shipped with Fedora 44) promotes `-Wincompatible-pointer-types` to a
hard error by default. This fork contains all fixes required to build successfully.

### Status

Tested and working on:
- **OS:** Fedora 44
- **Compiler:** Clang 22.1.5
- **Result:** Darling shell working, reporting macOS 11.7.4 (Big Sur) x86_64

### Prerequisites

```bash
sudo dnf install git cmake clang clang-devel llvm llvm-devel \
  bison flex pkg-config fuse-devel cairo-devel freetype-devel \
  zlib-devel libpng-devel libtiff-devel libjpeg-devel giflib-devel \
  fontconfig-devel mesa-libGL-devel ffmpeg-devel pulseaudio-libs-devel \
  dbus-devel mesa-libEGL-devel libX11-devel dsymutil setcap \
  openssl-devel python3
```

### Clone This Fork

```bash
git clone --recursive git@github.com:linuxrebel/darling.git
cd darling
git checkout james-dev
```

### Restoring Submodule Fixes

The fixes in this fork are committed both in the parent repo and as branches
on the backup remote. After cloning, the submodules will be at the correct
patched commits automatically via `--recursive`. However, if you ever need to
restore a specific submodule manually, each one has a corresponding branch on
the fork named `src-external-<name>-james-dev`. For example:

```bash
# Example: restore xnu submodule fixes
cd src/external/xnu
git fetch git@github.com:linuxrebel/darling.git src-external-xnu-james-dev
git checkout FETCH_HEAD
cd ../../../
```

The submodules with fixes applied are:
- `src/external/xnu`
- `src/external/Libinfo`
- `src/external/commoncrypto`
- `src/external/corecrypto`
- `src/external/corefoundation`
- `src/external/coretls`
- `src/external/libkqueue`
- `src/external/Heimdal`
- `src/external/foundation`
- `src/external/bash`
- `src/external/OpenDirectory`
- `src/external/security`
- `src/external/cocotron`
- `src/external/IOKitUser`
- `src/external/pyobjc`
- `src/external/passwordserver_sasl`
- `src/external/dtrace`
- `src/frameworks`

### Helper Script

The xnu submodule can occasionally be reset by git operations. A helper script
is included to reapply the `mach_vm.c` patch if needed:

```bash
python3 misc_scripts/fix_mach_vm.py
```

This is only needed if you see errors like:
```
mach_vm.c: error: incompatible pointer types passing 'vm_address_t *'
to parameter of type 'mach_vm_address_t *'
```

### Build

```bash
mkdir build && cd build
cmake .. -DTARGET_i386=OFF 2>&1 | tee cmake.log
make -j$(nproc) 2>&1 | tee build.log
sudo make install
```

### Verify

```bash
darling shell
```

Inside the Darling shell:
```bash
sw_vers        # Should show: ProductVersion: 11.7.4
uname -m       # Should show: x86_64
defaults read  # Should return macOS preference data
```

### Known Limitations

- **ARM64 binaries will not run.** Darling only supports x86_64. Apps built
  for Apple Silicon (post-2020) are not compatible.
- **Modern frameworks** — Many newer macOS frameworks are stubbed or unimplemented.
- **Metal** — Not supported in this build (requires Vulkan + LLVM).
- **Most GUI apps** — Do not work yet (upstream limitation).

### SELinux (Fedora)

SELinux may block `systemd-coredump` from handling Darling crash reports.
This is harmless but can be silenced with:

```bash
ausearch -c 'systemd-coredum' --raw | audit2allow -M my-systemdcoredum
sudo semodule -X 300 -i my-systemdcoredum.pp
```

### Summary of Fixes Applied

All fixes were required because Clang 22 promotes pointer type mismatches to
hard errors. They fall into these categories:

| Category | Files Affected |
|----------|---------------|
| `vm_address_t` vs `mach_vm_address_t` size mismatch | `xnu/mach_vm.c` |
| Transparent union consistency in corecrypto headers | `cczp.h`, `ccdh.h` |
| CF/NS toll-free bridging without explicit casts | Multiple `NS*.m` files |
| `WORD_LIST *` vs `GENERIC_LIST *` in bash | `stringlist.c`, `stringvec.c`, `execute_cmd.c`, `pcomplete.c`, `subst.c` |
| `int *` / `uint16_t *` / `UInt32 *` size mismatches | `resolver.c`, `UserGroup.c` |
| `CGRef` vs `O2Ref` typedef mismatches | cocotron AppKit/CoreGraphics |
| `kevent_internal_s` vs `kevent64_s` | `libkqueue/timer.c` |
| CMakeLists warning suppression flags | `commoncrypto`, `coretls`, `security`, `Heimdal`, `corefoundation`, `libsyscall`, `AppKit`, `CoreGraphics`, `CoreText`, `passwordserver_sasl` |

---

## Prefixes

Darling has support for DPREFIXes, which are very similar to WINEPREFIXes. They
are virtual "chroot" environments with a macOS-like filesystem structure, where
you can install software safely. The default DPREFIX location is `~/.darling`,
but this can be changed by exporting an identically named environment variable.
A prefix is automatically created and initialized on first use.

Please note that we use `overlayfs` for creating prefixes, and so we cannot
support putting a prefix on a filesystem like NFS or eCryptfs. In particular,
the default prefix location won't work if you have an encrypted home directory.

## Hello World

```bash
$ darling shell echo Hello world
Hello world
```

Congratulations, you have printed Hello world through Darling's OS X system
call emulation and runtime libraries.

## Installing Software

You can install `.pkg` packages with the installer tool available inside shell:

```bash
$ darling shell
Darling [~]$ installer -pkg mc-4.8.7-0.pkg -target /
```

The Midnight Commander package from the above example is
[available for download](https://darling-misc.s3.eu-central-1.amazonaws.com/mc-4.8.7-0.pkg).

You can uninstall and list packages with the `uninstaller` command.

## Working with DMG Images

DMG images can be attached and detached from inside `darling shell` with `hdiutil`:

```bash
Darling [~]$ hdiutil attach Xcode_7.2.dmg
/Volumes/Xcode_7.2
Darling [~]$ cp -r /Volumes/Xcode_7.2/Xcode.app /Applications
Darling [~]$ export SDKROOT=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk
Darling [~]$ echo 'void main() { puts("Hello world"); }' > helloworld.c
Darling [~]$ /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang helloworld.c -o helloworld
Darling [~]$ ./helloworld
Hello world
```

## Working with XIP Archives

Xcode is now distributed in `.xip` files. These can be installed using `unxip`:

```bash
cd /Applications
unxip Xcode_11.3.xip
```
