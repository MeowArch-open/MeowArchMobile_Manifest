# MeowArch zorn manifest

This is the `repo` manifest for the Xiaomi `zorn` / SM8650 Linux bring-up.
It locks every project to a commit instead of following moving `main` heads.
The `upstream` attributes tell repo which branch contains each locked commit,
so `repo sync -c` can fetch the required revision efficiently.

## Checkout profiles

The default profile contains the public Linux-side projects:

```text
kernel common display audio touch wifi common_rootfs
```

The complete authenticated profile adds the private Modem repository and all
UEFI inputs:

```sh
repo init -u https://github.com/MeowArch-open/MeowArchMobile_Manifest \
  -b main -m default.xml
repo sync -j8 -g default,private,uefi
```

The Modem repository is intentionally `private,notdefault`; a user without
access will receive a normal GitHub permission error when requesting the
`private` group.

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
   `common_rootfs/scripts/build-rootfs.sh`, including the subsystem runtime
   artifacts supplied by `kernel`, `common`, `display`, `audio`, `touch`,
   `modem`, and `wifi`.

`repo` itself only checks out sources. The ESP and rootfs filesystem-image
packing step belongs to the release builder and must consume the fixed output
names above; it must not silently select a different DTB or kernel.

The current device's ESP selection is documented by the Display component's
`dts/ESP-current.md`: the default entry loads `/Image` with
`/dtb/zorn-display.dtb`, whose content is the validated `audio-micb` DT.

## Updating locks

After deliberately updating a component, update its `revision` and
`upstream` together. For a checked-out workspace, `repo manifest -r` can
produce a revision-locked manifest; review it before committing to this
repository.
