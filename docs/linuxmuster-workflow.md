# Linuxmuster workflow

## Target
Convert a captured Windows `WIM` into a Linuxmuster-compatible Windows partition image bundle.

## Important model
The tested Linuxmuster path uses a `qcow2` that contains the Windows NTFS partition content, not a complete bootable disk layout. EFI/MSR/cache are created and managed by LINBO according to `start.conf`.

## Important built-in fixes
The converter now follows the Linuxmuster-native direction as closely as possible:
- offline setup state reset
- removal of Panther/rollback/unattend leftovers
- donor-based Linuxmuster boot metadata import
- regeneration of `BCD.<group>.<partition>` from the active `BCD`
- regeneration of `ntfs.id` from the freshly created NTFS volume
- donor normalization of `bsmbr.<group>`
- no-op compatibility hooks for donor-based Linuxmuster builds

## Tested deployment pattern
1. Run the converter locally.
2. Copy the generated bundle to the Linuxmuster image server.
3. Rebuild the torrent metadata.
4. On the client run `Sync+Start`.
5. Reboot back to LINBO and test `Start` again.

## Verified result
On April 14, 2026, the donor-based Linuxmuster path was verified on real hardware:
- `Sync+Start` booted successfully with `postsync = no-op`
- a later reboot/start path also booted successfully with both `prestart = no-op` and `postsync = no-op`

This is the practical proof that the preferred Linuxmuster path is:
- embed boot state in the Windows partition image
- do not rely on custom hook logic as the primary mechanism

## Current tested output set
- `win11_pro.qcow2`
- `win11_pro.qcow2.info`
- `win11_pro.qcow2.desc`
- `win11_pro.prestart`
- `win11_pro.postsync`
- `win11_pro.reg`
