# Building Darling on Fedora 44 with Clang 22

## Overview

This guide documents how to successfully build and install [Darling](https://www.darlinghq.org/)
— a macOS compatibility layer for Linux — on Fedora 44 with Clang 22.

Clang 22 (shipped with Fedora 44) promotes several pointer type mismatches to hard errors
that were previously warnings. This required fixes across many source files and CMakeLists.

**Result:** Darling shell working, reporting macOS 11.7.4 (Big Sur), x86_64.

---

## Prerequisites

```bash
sudo dnf install git cmake clang clang-devel llvm llvm-devel \
  bison flex pkg-config fuse-devel cairo-devel freetype-devel \
  zlib-devel libpng-devel libtiff-devel libjpeg-devel giflib-devel \
  fontconfig-devel mesa-libGL-devel ffmpeg-devel pulseaudio-libs-devel \
  dbus-devel mesa-libEGL-devel libX11-devel dsymutil setcap \
  openssl-devel python3
```

---

## Clone and Setup

```bash
git clone --recursive https://github.com/darlinghq/darling.git
cd darling
git checkout -b your-branch-name

# Create build directory
mkdir build && cd build
cmake .. -DTARGET_i386=OFF 2>&1 | tee cmake.log
```

---

## Helper Script

Save this as `~/darling/misc_scripts/fix_mach_vm.py` — the xnu submodule
gets reset by git operations so this needs to be reapplied if the submodule is reset:

```python
import subprocess

path = "/home/james/darling/src/external/xnu/darling/src/libsystem_kernel/libsyscall/mach/mach_vm.c"

with open(path, 'r') as f:
    content = f.read()

content = content.replace(
    "\tkern_return_t rv;\n\n\trv = _kernelrpc_vm_map(target, address, size, mask, flags, object,\n\t    offset, copy, cur_protection, max_protection, inheritance);",
    "\tkern_return_t rv;\n\tmach_vm_address_t mach_addr;\n\n\tmach_addr = (mach_vm_address_t)*address;\n\trv = _kernelrpc_vm_map(target, (vm_address_t *)&mach_addr, size, mask, flags, object,\n\t    offset, copy, cur_protection, max_protection, inheritance);\n\t*address = (vm_address_t)mach_addr;"
)
content = content.replace(
    "\tkern_return_t rv;\n\n\trv = _kernelrpc_vm_remap(target, address, size, mask, flags,\n\t    src_task, src_address, copy, cur_protection, max_protection,\n\t    inheritance);",
    "\tkern_return_t rv;\n\tmach_vm_address_t mach_addr;\n\n\tmach_addr = (mach_vm_address_t)*address;\n\trv = _kernelrpc_vm_remap(target, (vm_address_t *)&mach_addr, size, mask, flags,\n\t    src_task, src_address, copy, cur_protection, max_protection,\n\t    inheritance);\n\t*address = (vm_address_t)mach_addr;"
)
content = content.replace(
    "\tkern_return_t rv;\n\n\t/* {max,cur}_protection is inout */\n\trv = _kernelrpc_vm_remap_new(target, address, size, mask, flags,\n\t    src_task, src_address, copy, cur_protection, max_protection,\n\t    inheritance);",
    "\tkern_return_t rv;\n\tmach_vm_address_t mach_addr;\n\n\tmach_addr = (mach_vm_address_t)*address;\n\t/* {max,cur}_protection is inout */\n\trv = _kernelrpc_vm_remap_new(target, (vm_address_t *)&mach_addr, size, mask, flags,\n\t    src_task, src_address, copy, cur_protection, max_protection,\n\t    inheritance);\n\t*address = (vm_address_t)mach_addr;"
)

with open(path, 'w') as f:
    f.write(content)

result = subprocess.run(['grep', '-n', 'mach_vm_address_t mach_addr', path], capture_output=True, text=True)
print(result.stdout if result.stdout else "WARNING: No matches found")
```

---

## Source File Fixes

Apply all of these before building.

### 1. xnu — mach_vm.c

```bash
python3 ~/darling/misc_scripts/fix_mach_vm.py
```

### 2. Libinfo — resolver.c

```bash
sed -i 's/int servPort = 0;/uint16_t servPort = 0;/' \
  ~/darling/src/external/Libinfo/darling-resolver/resolver.c
```

### 3. libkqueue — timer.c

File: `src/external/libkqueue/src/linux/timer.c`

```c
// Line ~177: fix kevent cast
kevent_int_to_64((const struct kevent_internal_s *)&src->kev, dst);

// Line ~277: fix kevent cast
return evfilt_timer_knote_modify(filt, kn, (const struct kevent64_s *)&kn->kev);
```

### 4. corefoundation — CFXPCBridge.c

File: `src/external/corefoundation/CFXPCBridge.c`

```c
// Add casts to CFDictionaryCreate call
CFDictionaryRef dict = CFDictionaryCreate(NULL, (const void **)keys,
    (const void **)values, count,
    &kCFTypeDictionaryKeyCallBacks,
    &kCFTypeDictionaryValueCallBacks);
```

### 5. corefoundation — NSException.m

```bash
sed -i 's/reason: reason/reason: (NSString *)reason/g' \
  ~/darling/src/external/corefoundation/NSException.m
```

### 6. corefoundation — NSForwarding.m

```c
// Line ~140
id block = *(id *)frame;  // was *(id**)frame
```

### 7. corefoundation — NSInvocation.m, NSKeyedArchiver.m, NSPort.m,
###    NSRunLoop.m, NSScanner.m, NSURL.m, NSUserDefaults.m

**NSInvocation.m** — all `CFStringAppend(@"...")` calls need `(CFStringRef)@"..."` casts.

**NSKeyedArchiver.m** — two `CFDictionaryGetValueIfPresent` calls need `(const void **)` cast on last arg.

**NSPort.m:**
```c
OSAtomicDecrement32((volatile int32_t *)&_reserved)
OSAtomicIncrement32((volatile int32_t *)&_reserved)
```

**NSRunLoop.m:**
```c
[performer->allTimers removeObject: (id)performer->timer];
```

**NSScanner.m** — replace `scanInteger:` implementation with temp variable approach:
```objc
- (BOOL)scanInteger:(NSInteger *)value {
    if (sizeof(NSInteger) == sizeof(int)) {
        int tmp;
        BOOL result = [self scanInt:&tmp];
        if (result && value) *value = (NSInteger)tmp;
        return result;
    } else if (sizeof(NSInteger) == sizeof(long long)) {
        long long tmp;
        BOOL result = [self scanLongLong:&tmp];
        if (result && value) *value = (NSInteger)tmp;
        return result;
    } else {
        DEBUG_BREAK();
    }
}
```

**NSURL.m:**
```c
// Line ~115
return (NSURL *)CFURLCreateWithString(kCFAllocatorDefault, CFSTR(""), NULL);
// Line ~576
self = (NSURL *)CFURLCreateWithString(kCFAllocatorDefault, (CFStringRef)string, [url _cfurl]);
```

**NSUserDefaults.m:**
```c
NSString* appName = (NSString *)APP_NAME;
CFPreferencesAppSynchronize((CFStringRef)appName);
```

### 8. Heimdal — crypto.c

Remove the `(ECDSA *)` casts we added — the Darling version of these functions
takes `EC_KEY *` directly:

```c
// These should have NO cast — pass EC_KEY * directly
ret = ECDSA_verify(-1, digest.data, digest.length, sig->data, sig->length, key);
sig->length = ECDSA_size(signer->private_key.ecdsa);
ret = ECDSA_sign(-1, indata.data, indata.length, sig->data, &siglen, signer->private_key.ecdsa);
```

### 9. bash — stringlist.c, stringvec.c, execute_cmd.c, pcomplete.c, subst.c

All `list_length()` and `list_append()` calls passing `WORD_LIST *` need `(GENERIC_LIST *)` casts:

```bash
sed -i 's/= list_length (list)/= list_length ((GENERIC_LIST *)list)/g' \
  ~/darling/src/external/bash/bash-3.2/lib/sh/stringlist.c \
  ~/darling/src/external/bash/bash-3.2/lib/sh/stringvec.c

sed -i 's/list_len = list_length (list)/list_len = list_length ((GENERIC_LIST *)list)/' \
  ~/darling/src/external/bash/bash-3.2/execute_cmd.c

sed -i 's/nw = list_length (l2)/nw = list_length ((GENERIC_LIST *)l2)/' \
  ~/darling/src/external/bash/bash-3.2/pcomplete.c
```

For `subst.c` lines ~7893 and ~8143:
```c
output_list = (WORD_LIST *)list_append((GENERIC_LIST *)glob_list, (GENERIC_LIST *)output_list);
new_list = (WORD_LIST *)list_append((GENERIC_LIST *)expanded, (GENERIC_LIST *)new_list);
```

### 10. OpenDirectory — UserGroup.c

```c
// Line ~171
status = dsFindDirNodes(gDirRef, nodeBuffer, NULL, eDSSearchNodeName,
    (UInt32 *)&returnCount, &localContext);

// Line ~1248
status = dsDoAttributeValueSearchWithData(gSearchNode, searchBuffer, recType,
    attrType, eDSExact, lookUpPtr, attrsToGet, 0,
    (UInt32 *)&recCount, &localContext);
```

### 11. NSForwarding pattern — all stub files

Fix all `NSStringFromSelector([anInvocation selector])` occurrences at once:

```bash
grep -rl "NSStringFromSelector(\[anInvocation selector\])" \
  ~/darling/src/frameworks ~/darling/src/external ~/darling/src/lib \
  2>/dev/null | xargs sed -i \
  's/NSStringFromSelector(\[anInvocation selector\])/NSStringFromSelector((SEL)[anInvocation selector])/g'
```

### 12. dtrace — plockstat.c

```c
// Line ~520
if (P == NULL || dtrace_proc_lookup_by_addr(g_dtp, P, addr, name,
    sizeof(name), (__GElf_Sym *)&sym, &info) != 0) {
```

### 13. cocotron — various files

**CoreData/NSXMLPersistentStore.h** — change `XMLDocument *` to `NSXMLDocument *`

**Onyx2D/O2ImageSource.m and O2ImageSource_TIFF.m:**
```objc
return (CFDictionaryRef)[[NSDictionary alloc] init];
```

**CoreText/CTStringATtributes.m:**
```c
const CFStringRef kCTLigatureAttributeName = CFSTR("NSLigature");
const CFStringRef kCTUnderlineStyleAttributeName = CFSTR("NSUnderline");
```

**QuartzCore/CARenderer.m:**
```objc
CGImageRef image = (CGImageRef)layer.contents;
```

**AppKit/NSComboBoxCell.m:**
```objc
NSTextView *editor = (NSTextView *)[controlView currentEditor];
string = (NSString *)object;
attstr = (NSAttributedString *)object;
```

**AppKit/NSWorkspace.m:**
```objc
+ (instancetype)configuration {
    return [[self alloc] init];  // was: return self;
}
```

**AppKit/NSCursor.m:**
```objc
return (NSCursor *)[NSNull null];
```

**AppKit/nib.subproj/NSCustomView.m:**
```objc
return (NSCustomView *)newView;
```

**AppKit/NSView.m:**
```objc
[_layer setContents: (id)image];
```

**AppKit/NSAlert.m:**
```objc
[textField setAttributedStringValue: [[[NSAttributedString alloc] ...]]]
// was: setStringValue:
```

**AppKit/NSFont.m:**
```objc
_cgFont = (CGFontRef)(void *)CGFontCreateWithFontName((CFStringRef)_name);
_ctFont = CTFontCreateWithGraphicsFont((O2FontRef)(void *)_cgFont, _pointSize, NULL, NULL);
// Line ~409: fix variable name
name = [NSString stringWithCString: nameStr encoding: NSASCIIStringEncoding];
```

**IOKitUser/hid.subproj/HIDSessionBase.m:**
```objc
_IOHIDSessionReleasePrivate((IOHIDServiceRef)(void *)(__bridge IOHIDSessionRef)self);
```

**frameworks/DiskArbitration/DADisk.c:**
```c
return CFStringGetCStringPtr(disk->path, kCFStringEncodingUTF8);
```

**frameworks/OpenDirectory/src/ODSession.m:**
Already handled by the global sed command in step 11.

**frameworks/ImageIO/src/CGImageSource.m:**
```objc
return (CGImageRef)[self createImageAtIndex:index options:options];
return (CFStringRef)[self type];
```

**pyobjc — module.m:**
```objc
(void)[[self alloc] init];  // was: self = [[self alloc] init];
```

---

## CMakeLists Fixes

These require re-running cmake after applying.

### src/CMakeLists.txt — add global define

```cmake
add_definitions(
    -DDARLING
    -DCORECRYPTO_USE_TRANSPARENT_UNION=1
)
```

### src/external/commoncrypto/CMakeLists.txt

```cmake
add_compile_options(
    -nostdinc
    -Wno-unused-command-line-argument
    -Wno-incompatible-pointer-types
    -Wno-error=incompatible-pointer-types
    -Wno-error
    -Wl,-exported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/exports.exp-in
)
add_definitions(
    -DNDEBUG
    -DCORECRYPTO_DONOT_USE_TRANSPARENT_UNION
)
```

### src/external/coretls/CMakeLists.txt

```cmake
add_compile_options(
    -nostdinc
    -Wno-error=int-conversion
    -Wno-incompatible-pointer-types
    -Wno-error=incompatible-pointer-types
    -Wno-error
)
add_definitions(
    -DCCCBC_RETURN_INT
    -DCORECRYPTO_DONOT_USE_TRANSPARENT_UNION
)
```

### src/external/security/CMakeLists.txt

Add to `add_compile_options`:
```cmake
-Wno-incompatible-pointer-types
-Wno-error=incompatible-pointer-types
-Wno-error
```

### src/external/Heimdal/CMakeLists.txt

Add to `add_compile_options`:
```cmake
-Wno-incompatible-pointer-types
-Wno-error=incompatible-pointer-types
-Wno-error
```

### src/external/corefoundation/CMakeLists.txt

Add to `CMAKE_C_FLAGS`:
```cmake
-Wno-incompatible-pointer-types \
-Wno-error=incompatible-pointer-types \
```

### src/external/xnu/darling/src/libsystem_kernel/libsyscall/CMakeLists.txt

Add to `CMAKE_C_FLAGS`:
```cmake
-Wno-incompatible-pointer-types
```

### src/external/passwordserver_sasl/CMakeLists.txt

Add after `project(...)`:
```cmake
add_compile_options(-Wno-incompatible-pointer-types)
```

### src/external/cocotron/AppKit/CMakeLists.txt
### src/external/cocotron/CoreGraphics/CMakeLists.txt
### src/external/cocotron/CoreText/CMakeLists.txt

Add to `CMAKE_C_FLAGS`:
```cmake
-Wno-incompatible-pointer-types \
```

---

## Header Fixes

### corecrypto — cczp.h

File: `Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/corecrypto/cczp.h`

Change:
```c
#if CORECRYPTO_USE_TRANSPARENT_UNION
```
(This is already the correct value — just ensure it hasn't been changed to `#if 1` or `#if !defined(...)`)

### corecrypto — ccdh.h

File: `Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/corecrypto/ccdh.h`

All three `#if` blocks controlling union typedefs should use:
```c
#if CORECRYPTO_USE_TRANSPARENT_UNION
```

---

## Build

```bash
# Apply mach_vm.c patch (do this every time after a submodule reset)
python3 ~/darling/misc_scripts/fix_mach_vm.py

# Build
cd ~/darling/build
cmake .. -DTARGET_i386=OFF 2>&1 | tee cmake.log
make -j$(nproc) 2>&1 | tee build.log

# Install
sudo make install
```

---

## Verification

```bash
# Open Darling shell
darling shell

# Inside shell:
sw_vers        # Should show macOS 11.7.4
uname -m       # Should show x86_64
defaults read  # Should show macOS preferences
```

---

## Known Limitations

- **ARM64 binaries will not run** — Darling only supports x86_64. Apps built
  for Apple Silicon (post-2020) are incompatible.
- **Modern frameworks** — Many newer macOS frameworks are stubbed or unimplemented.
- **Graphics** — OpenGL works; Metal does not (requires Vulkan + LLVM which
  weren't found in this build).

---

## SELinux Note (Fedora)

SELinux may block `systemd-coredump` from handling Darling crash reports.
This is harmless but can be fixed with:

```bash
ausearch -c 'systemd-coredum' --raw | audit2allow -M my-systemdcoredum
semodule -X 300 -i my-systemdcoredum.pp
```

---

## Background

All fixes were required because Clang 22 (Fedora 44) promotes
`-Wincompatible-pointer-types` to a hard error by default. The Darling codebase
was written against older compilers that treated these as warnings or ignored them.

The fixes fall into these categories:
1. `vm_address_t` vs `mach_vm_address_t` (32 vs 64-bit pointer width)
2. Transparent union consistency across corecrypto headers
3. CF/NS toll-free bridging without explicit casts (Objective-C)
4. `WORD_LIST *` vs `GENERIC_LIST *` in bash
5. Various `int *` / `uint16_t *` / `UInt32 *` size mismatches
6. `CGRef` vs `O2Ref` typedef mismatches in cocotron
