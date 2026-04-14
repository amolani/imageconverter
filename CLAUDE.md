# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains `wim2qcow2`, a single-file Python 3 CLI tool (~2800 lines, no external Python dependencies) that converts Windows WIM images to qcow2 virtual disk images. It runs as root and orchestrates system-level tools (NBD, partitioning, NTFS formatting, offline registry cleanup, Linuxmuster boot metadata seeding).

All user-facing strings and log messages are in German.

## Running

```bash
# Requires root. Interactive wizard (no args):
./wim2qcow2

# Inspect a WIM without converting:
./wim2qcow2 --input image.wim --inspect

# Full-disk GPT (bootable KVM/Proxmox image):
./wim2qcow2 --input image.wim --output image.qcow2 --layout full-disk-gpt

# Linuxmuster partition image (LINBO deployment):
./wim2qcow2 --input image.wim --output image.qcow2 --layout linuxmuster-partition \
    --start-conf /srv/linbo/start.conf.logo --disk-size 64G

# With env-file for defaults:
./wim2qcow2 --input image.wim --output image.qcow2 --env-file defaults.env
```

There is no test suite, build system, or linter configuration.

## Architecture

The tool is a single executable Python script (no `setup.py`, no package structure). Key sections in order:

1. **Constants & embedded data** (lines 1-670): BCD donor registry blob (base64), setup cleanup paths, problematic driver lists, required tool tuples, architecture map.

2. **Data classes** (lines 672-715): `WimImage`, `WimMetadata`, `ConversionState`, `StartConfHints` -- state containers passed through the pipeline.

3. **Logger** (lines 717-780): Structured step/debug/warn/error logging with optional logfile output.

4. **WIM parsing** (`parse_wim_metadata`, `choose_image`): Calls `wimlib-imagex info --xml` and parses the XML to extract image metadata.

5. **Interactive wizard** (`run_interactive_wizard`): Prompts for all conversion parameters when run without arguments.

6. **Disk operations**: NBD connect/disconnect, GPT partitioning via `sgdisk`, NTFS/FAT formatting, mount/unmount management tracked via `ConversionState.mountpoints`.

7. **Windows sanitization** (`sanitize_applied_windows_image`): Removes Sysprep/setup artifacts, problematic FTDI drivers, and applies offline registry patches via `reged`.

8. **Boot handling**:
   - `full-disk-gpt`: `install_boot_files`, `build_patched_bcd_template`, `write_seeded_donor_bcd` create a bootable standalone disk image.
   - `linuxmuster-partition`: `apply_linuxmuster_reference_boot_seed` copies Linuxmuster boot metadata from a working reference qcow2 into the Windows partition itself.

9. **Linuxmuster sidecar generation** (`write_linuxmuster_info/desc/hooks/reg`): Produces `.info`, `.desc`, `.prestart`, `.postsync`, and `.reg` for LINBO integration. In donor-based Linuxmuster mode the current target is to move boot state into the partition image and emit only `no-op` compatibility hooks.

10. **Conversion pipelines**: `convert_full_disk` (EFI+MSR+Windows GPT layout) and `convert_linuxmuster_partition` (raw NTFS partition image), orchestrated by `convert()`.

## System Dependencies

Common: `wimlib-imagex`, `qemu-img`, `qemu-nbd`, `mkntfs`, `mount`, `umount`, `modprobe`, `blkid`, `reged`

Full-disk-gpt additionally: `sgdisk`, `mkfs.fat`, `partprobe`

## Key Design Decisions

- For Linuxmuster, the preferred design is to mimic `linbo_create_image` rather than invent custom boot repair logic. A working Linuxmuster reference qcow2 is used as donor for `.guid.*` and `EFI/Microsoft/Boot`.
- In Linuxmuster mode the converter should regenerate Linuxmuster artifacts from the finished partition image where possible: `BCD.<group>.<part>` should be copied from the active `BCD`, `ntfs.id` should be read from the freshly generated NTFS volume, and only donor data that cannot be recreated locally should remain donor-based.
- The embedded BCD donor blob is still used for full-disk mode and as a fallback path, but it is no longer the preferred Linuxmuster mechanism.
- Cleanup is aggressive: `ConversionState` tracks all NBD connections, mounts, and tempdirs; `cleanup()` in a `finally` block tears everything down even on failure. Partial output files are preserved for diagnostics.
- The `.partial` rename pattern ensures atomic output -- the final qcow2 only appears at the target path after `check_qcow2` validates it.
- `start.conf` parsing (`parse_start_conf`, `derive_start_conf_hints`) extracts partition size, label, and device path from LINBO configuration files to auto-populate conversion defaults.

## Linuxmuster Goal

The long-term Linuxmuster target is documented in [docs/linuxmuster-target.md](docs/linuxmuster-target.md).

Verified milestone:

- On April 14, 2026, a donor-based Linuxmuster build booted successfully with both `.prestart` and `.postsync` set to `no-op`.
- Treat this as proof that the preferred Linuxmuster path is to embed boot state in the partition image, not in custom hooks.
- Any remaining issue where the LINBO GUI start path does not visibly trigger a reboot is a separate LINBO wrapper/UI topic, not a converter architecture reason to reintroduce hook logic.

This is the key rule for future work:

- Prefer reproducing what `linbo_create_image` and `linbo_sync` expect inside the Windows partition.
- Keep `.prestart` / `.postsync` empty in donor-based Linuxmuster mode unless a verified regression requires hook logic again.
- Do not add new boot workarounds if the same result can be achieved by making the generated partition image more Linuxmuster-like.
