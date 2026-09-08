# MeowArch zorn manifest

This is the `repo` manifest for the Xiaomi `zorn` / SM8650 Linux bring-up.
It locks every project to a commit instead of following moving `main` heads.
The `upstream` attributes document the branch containing each locked commit;
the release workflow initializes repo with `--no-current-branch` so nested
UEFI projects fetch their pinned SHA even when an old commit is no longer
reachable from the declared upstream branch tip.

## Checkout profiles

The default profile contains the public Linux-side projects:

```text
kernel common display audio touch wifi common_rootfs builder toolchain
```

The complete authenticated profile adds the private Modem repository and all
UEFI inputs:

```sh
repo init -u https://github.com/MeowArch-open/MeowArchMobile_Manifest \
  -b main -m default.xml
repo sync -j8 -g default,private,uefi
```

The `toolchain` project is a small metadata repository whose Release asset is
downloaded by the Builder. It supplies the host x86_64 cross-toolchain for
Kernel/UEFI/DTB/ESP; only the rootfs/AUR stage enters the ARM64 container.

The Modem repository is intentionally `private,notdefault`; a user without
access will receive a normal GitHub permission error when requesting the
`private` group. Redistributable builds use `default,uefi` with the Builder's
`--public-no-modem` profile. That profile never checks out the private project
and explicitly omits its patches, firmware, services and userspace artifacts.

The UEFI projects are also `notdefault`. Their nested paths reproduce the
Project Mu workspace and avoid the old `.gitmodules` URL
`MeowArchMoblie_UEFI_common_Binary`:

```text
uefi/Project_Mu/
├── Binaries
├── Common/Mu
├── Common/Mu_OEM_Sample
├── Mu_Basecore
├── Silicium-ACPI
└── Silicon/Silicium/OpensslPkg/Library/OpensslLib/openssl
```

The `Mu_Basecore` and `Silicium-ACPI` projects are the MeowArch-open mirrors
selected by the current Project Mu gitlinks; they are locked to the same
commits as the Project Mu checkout. The nested MIPI System Trace gitlink is
also represented explicitly because the new Basecore `MdePkg.dec` requires its
include directory during metadata processing. The Basecore SPDM, oniguruma and
libfdt dependencies are also represented explicitly; the SPDM lock uses the
reachable upstream commit ending in `432e` because the mirror gitlink ending in
`4320` is not present in DMTF's repository.

Do not add `--submodules` for this manifest; the submodules are represented as
fixed repo projects.

## Three image outputs

The manifest supplies inputs for the complete zorn release, whose release
pipeline must emit these three artifacts:

1. `ESP.img`: FAT32 ESP containing the UEFI boot files, `/Image`, selected
   `/dtb/*`, and `EFI/BOOT/grub.cfg`.
2. `UEFI`: Project Mu's zorn outputs, currently `Mu-zorn-0.img` and
   `Mu-zorn-1.img` (plus raw `.bin` payloads), built with:

   ```sh
   cd uefi/Project_Mu
   python3 build_uefi.py -d zorn -m 0 -c
   python3 build_uefi.py -d zorn -m 1 -c
   ```

3. `rootfs.img`: an Arch ARM rootfs assembled by
   `common_rootfs/scripts/build-rootfs.sh`. The authenticated full profile
   includes all subsystem runtime artifacts; the automated public profile
   includes `kernel`, `common`, `display`, `audio`, `touch`, and `wifi` and is
   intentionally marked `public-no-modem`.

`repo` itself only checks out sources. The ESP and rootfs filesystem-image
packing step belongs to the release builder and must consume the fixed output
names above; it must not silently select a different DTB or kernel.

The current device's ESP selection is documented by the Display component's
`dts/ESP-current.md`: the default entry loads `/Image` with
`/dtb/zorn-display.dtb`, whose content is the validated `audio-micb` DT.

## Automated public release

`.github/workflows/build-zorn.yml` runs manually and at 03:17 UTC on the first
day of each month. It uses GitHub's Ubuntu 24.04 ARM64 runner, syncs only
`default,uefi`, builds the public-no-Modem rootfs plus UEFI and ESP, validates
and compresses each image with zstd, then uploads it to the private Azure
`zorn-builds` container. Every run is retained under
`runs/<run-id>-<attempt>/`; `latest/` is updated after the immutable upload.
The workflow requires the repository secret `AZURE_STORAGE_CONNECTION_STRING`.

## Updating locks

After deliberately updating a component, update its `revision` and
`upstream` together. For a checked-out workspace, `repo manifest -r` can
produce a revision-locked manifest; review it before committing to this
repository.
