# XR/iPhoneOS port notes

This branch is based on Fauxly's `t8020` Apple TV jbinit integration.  The
runtime code already has iOS and tvOS paths; the main blocker for iPhone XR was
that the build system was hardwired for AppleTVOS/tvOS artifacts.

## What changed

- Default build target is now `TARGET_SDK=iphoneos` with `JBINIT_ARCH=arm64e`.
- The previous AppleTVOS target remains available with `TARGET_SDK=appletvos`.
- Common target variables are exported from the top Makefile:
  - `TARGET_SDK`
  - `TARGET_VERSION_MIN`
  - `TARGET_TRIPLE`
  - `TARGET_MIN_FLAG`
  - `DEPLOYMENT_TARGET_VAR`
- Component Makefiles now use those target variables instead of hardcoded
  `-mappletvos-version-min` or `arm64-apple-tvos` flags.
- iPhoneOS builds only require `palera1nLoader.ipa`; `tvloader.dmg` is built and
  copied only when `BUILD_TV_LOADER=1` or `TARGET_SDK=appletvos`.
- Submodule URLs were changed from SSH to HTTPS so GitHub Actions can fetch them
  without deploy keys.
- The workflow now builds XR/iPhoneOS arm64e artifacts and uploads:
  - `ramdisk.dmg`
  - `binpack.dmg`
  - development variants

## Build

macOS/Xcode:

```sh
brew install make gnu-sed ldid-procursus fakeroot
curl -fL https://static.palera.in/artifacts/loader/universal_lite/palera1nLoader.ipa -o src/palera1nLoader.ipa
curl -fL https://static.palera.in/binpack.tar -o src/binpack.tar
gmake -j1 tools
gmake -j$(sysctl -n hw.ncpu) TARGET_SDK=iphoneos JBINIT_ARCH=arm64e
```

Outputs:

```text
src/ramdisk.dmg
src/binpack.dmg
```

## Laika/Pongo boot flow

After Laika reaches Pongo and embedded KPF is loaded:

```text
/send src/ramdisk.dmg
ramdisk
/send src/binpack.dmg
overlay
palera1n_flags 0x2
xargs
bootx
```

Do not pass a ramdisk size for these DMGs.  `ramdisk <size>` is only for LZMA
compressed ramdisks where `<size>` is the uncompressed size.

## Still needs runtime validation

The build-system port does not prove the iOS 18.7.10 userspace path is complete.
Runtime validation points:

1. Confirm fakedyld patches the XR/iOS dyld version used by 18.7.10.
2. Confirm KPF ramdisk/rootdev path is correct for `rootdev=md0`/Laika Pongo.
3. Confirm `loader.dmg` mounts and `palera1nLoader.app` gets installed/uicached.
4. If userspace panics, collect framebuffer/panic text and check `/cores/panic.txt`
   if the ramdisk reaches payload code.
