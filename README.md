# imageconverter

`imageconverter` converts Windows `WIM` images into `qcow2` images, with a tested workflow for Linuxmuster/LINBO Windows partition images.

## What it does
- inspect WIM contents and list available indices
- build a fresh `qcow2`
- create either
  - a full-disk GPT image, or
  - a Linuxmuster-compatible NTFS partition image
- apply the selected WIM index
- clean up broken/offline Windows setup state from captured images
- generate Linuxmuster sidecars automatically:
  - `.qcow2.info`
  - `.qcow2.desc`
  - `.prestart`
  - `.postsync`
  - `.reg`

## Why this exists
The practical target was not a generic Windows VM conversion alone, but a reproducible path from a LogoDidakt/MDT-style `WIM` to a Linuxmuster image bundle that actually boots and survives `Sync+Start` and later `Start` runs.

The final implementation includes fixes for the problems that showed up during real hardware testing:
- Linuxmuster needs a partition image, not always a full-disk image
- EFI/BCD handling must not blindly overwrite a working EFI store
- setup/sysprep/rollback leftovers from the captured WIM must be removed offline
- LINBO `Start` needs a reliable `BootNext -> Windows Boot Manager` path

## Requirements
Typical Linux dependencies:
- `python3`
- `wimlib-imagex`
- `qemu-img`
- `qemu-nbd`
- `mount`
- `mkntfs`
- `blkid`
- `reged` from `chntpw`
- for full-disk mode also tools such as `sgdisk`, `mkfs.fat`

Root privileges are required for NBD, mount and filesystem operations.

## Usage
Without arguments, the tool starts an interactive wizard:

```bash
./wim2qcow2
```

Inspect a WIM only:

```bash
./wim2qcow2 \
  --input /path/to/image.wim \
  --inspect
```

Linuxmuster partition image:

```bash
./wim2qcow2 \
  --input /path/to/image.wim \
  --output /srv/linbo/images/win11_pro/win11_pro.qcow2 \
  --index 1 \
  --layout linuxmuster-partition \
  --disk-size 80G \
  --target-partition /dev/disk0p3 \
  --ntfs-label windows \
  --copy-sidecars-from /srv/linbo/images/win11_pro_edu \
  --verbose
```

Using an env file:

```bash
./wim2qcow2 --env-file examples/linuxmuster.env.example
```

## Tested input/output
Validated during the current migration work:
- input WIM: `39ef39cf-9848-4766-bb75-6cab64845a06.wim`
- output image: `win11_pro.qcow2`
- resulting SHA256:
  - `19744f7d7e111f25fc4d28e7684eb96ec7869a76f3adb370cb7c39367c4f1af2`

## Repository contents
- `wim2qcow2`: main converter script
- `examples/linuxmuster.env.example`: reusable env defaults
- `docs/linuxmuster-workflow.md`: operational notes from the tested Linuxmuster path

## Notes
This repository currently focuses on the Linuxmuster path because that is the workflow that was tested end-to-end on a real server/client setup.
