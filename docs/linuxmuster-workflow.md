# Linuxmuster workflow

## Target
Convert a captured Windows `WIM` into a Linuxmuster-compatible Windows partition image bundle.

## Important model
The tested Linuxmuster path uses a `qcow2` that contains the Windows NTFS partition content, not a complete bootable disk layout. EFI/MSR/cache are created and managed by LINBO according to `start.conf`.

## Important built-in fixes
The converter bakes in the fixes that were required during real testing:
- offline setup state reset
- removal of Panther/rollback/unattend leftovers
- donor-based BCD seeding
- generated `prestart` and `postsync` hooks
- `BootNext` handoff to `Windows Boot Manager`

## Tested deployment pattern
1. Run the converter locally.
2. Copy the generated bundle to the Linuxmuster image server.
3. Rebuild the torrent metadata.
4. On the client run `Sync+Start`.
5. Reboot back to LINBO and test `Start` again.

## Current tested output set
- `win11_pro.qcow2`
- `win11_pro.qcow2.info`
- `win11_pro.qcow2.desc`
- `win11_pro.prestart`
- `win11_pro.postsync`
- `win11_pro.reg`
