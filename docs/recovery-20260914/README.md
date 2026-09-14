# Dorothy CMP 170HX source recovery — 2026-09-14

This branch preserves the unlocker customizations recovered before any power cycle
following Dorothy's NVMe failure. It is based on upstream commit
`76f0954fbb9864df0938f9c7bd9bfdc5bf2d6636` (Floorsweep guard being overprotective).
The surviving on-disk `refs/heads/master` and the September 9 tool history both
identify that base. The initial recovery was published on a separate branch;
the verified recovery is now also published on the fork's `master`.

## Fork alignment

Before this recovery, the fork's `master` was at
`f902c4416ac5b68fcc9700abe4b81a46919183c0`, an ancestor **16 upstream commits
behind** Dorothy's base. Those 16 commits include the WPR2 reserved-memory fix,
GPU profiling, passthrough support, and floorsweep changes. The recovery branch
already includes that complete history as well as the two local customizations.
Promotion to `master` is a fast-forward, preserving the existing fork history.

On September 14, a fresh read of Dorothy's `refs/heads/master` again returned
`76f0954fbb9864df0938f9c7bd9bfdc5bf2d6636`. The loaded NVIDIA module's
`srcversion` was `B33D7C08666F586299BF97A`, matching the final 128 MiB module
recorded in the September 9 tool evidence. Live installation metadata still
reported driver `610.43.02` and profile `8gb`.

Upstream's current `master`, `88e39ce67488796b2c6c716fe8f9b4e6e943a55e`, has
12 further commits beyond Dorothy's base, including driver 615.71.09 support
and changes to the floorsweep handling. They are outside this recovery: its
target is the version used on Dorothy. Unreadable-file limitations below apply.

## Preserved customizations

- `driver/patches/bar1-resize-unlock.patch`: copied directly from Dorothy. For CMP
  PCI IDs 0x20C2 and 0x2082, the final September 9 patch sets XVE CFG to
  `0x80000001` and requests a 128 MiB BAR1 aperture. Other devices retain upstream
  sizing. This caps the PCI aperture; the 8gb profile still unlocks 64 GiB VRAM.
  SHA-256: `1b01eb1b1788840fb4dbc9bbc070a8f9c924f9d29ea20ab61d9b49f7e98ce8d8`.
- `driver/build.sh`: reconstructed by replaying the exact five-line September 9
  patch from Loom onto the pinned upstream file. `CMPUNLOCKER_SKIP_RELOAD=1`
  exits after installing modules, running depmod and rebuilding initramfs, before
  stopping NVIDIA services or attempting live module unload/reload. This is an
  install operation, not a dry run. The default behavior is unchanged.
  Reconstructed Git blob: `43872251c807cc4994a09884643fe5384122b796`, matching
  the `4387225` blob prefix in the historical diff. Its 10,520-byte size also
  matches Dorothy's surviving file metadata; the live file itself returned EIO.
- All 12 active driver patches were recovered directly. The remaining 23 readable
  source files match the pinned upstream base byte for byte.
- `historical/bar1-force64mib-20260826.patch`: the older 64 MiB experiment, recovered
  from the existing copy on Rehoboam and matched to the August tool evidence.
  SHA-256: `5c5da45972d5cc7855e106cde69036cdff868ebabf7a4f8c325a6d49aa01ce6d`.
  It is superseded by the active 128 MiB patch and is outside the build patch list.

## Recovery completeness

`source-recovery-manifest.json` records the 24 directly copied files and read
failures. Thirteen individual files and the `driver/passthrough` directory were
unreadable. Those tracked files are supplied by the pinned upstream base, except
for the build script reconstructed above. This does not prove that unreadable
files had no other local edits. The September 9 final Git status identified only
`driver/build.sh` and `driver/patches/bar1-resize-unlock.patch` as modified.

The older acceptance checkout at
`170hx-acceptance/2026-08-21-1322821093001/tools/cmpunlocker-pr32-dd03cda`
was also inaccessible. Generated build output, installed module binaries, logs,
Git object storage, and the full acceptance workspace are not backed up by this
branch. This is a source recovery, not a complete backup of Dorothy.

The related fan-control code under `ops/170hx-fan-control` had no reported local
changes, and its latest commit was verified on GitHub:
[f9f9875: restore CMP cooling after four-GPU topology change](https://github.com/eugeneware/dorothy-services/commit/f9f9875124b55f9c440afe1b8d1940c0041996ce).

## Rebuild context

The live installation metadata read on September 14 reports NVIDIA `610.43.02`
and profile `8gb`. September 9's recorded build explicitly selected that version;
the upstream VERSION file defaults to a newer release, so do not rely on its
default when reproducing this installation. The recorded kernel was
`7.0.0-29-generic`. The build script uses the running kernel's headers.

For a future rebuild on a repaired machine, with matching NVIDIA userspace and
kernel headers, the historical build invocation was equivalent to:

```bash
sudo env CMPUNLOCKER_SKIP_RELOAD=1 \
  CMPUNLOCKER_DRIVER_VERSION=610.43.02 \
  CMPUNLOCKER_CARD_PROFILE=8gb \
  CMPUNLOCKER_GPU_INVENTORY="$(cat /lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory)" \
  ./driver/build.sh
```

That command assumes an existing installation with its inventory file. It writes
modules and initramfs and requires a later separately planned boot to activate
them. It was not executed during this recovery. Preserve the existing boot and
IOMMU configuration separately; this branch does not install a machine's GRUB
configuration.

## Provenance

Original conversation: Debug the kernel panic, September 9, 2026.

- [Exact skip-reload edit](https://loom.zapus-dragon.ts.net/conversations/e8b15909-ecd1-5f60-9aa0-cca614cbf5ea?item=40055057-ec91-5f9d-9dfa-0148f645c3c0)
- [Historical build-script diff and blob prefix](https://loom.zapus-dragon.ts.net/conversations/e8b15909-ecd1-5f60-9aa0-cca614cbf5ea?item=fa6e9c85-b690-5693-97bd-69260f7cb835)
- [Final staged 128 MiB implementation and modified-file status](https://loom.zapus-dragon.ts.net/conversations/e8b15909-ecd1-5f60-9aa0-cca614cbf5ea?item=bb41d1a7-94de-5516-ab45-559a46a90889)

Loom links require the owner's access. Raw conversations and host credentials
are not included in this repository.

## Validation performed during recovery

- All 24 directly recovered file contents match their recorded SHA-256 hashes.
- `bash -n driver/build.sh` and `git diff --check` pass.
- The repository's `tools/read-constants.py` validates the restored patch set with
  the `8gb` profile (64 GiB).
- All 12 active patches apply in the build script's explicit order to NVIDIA
  610.43.02 source at commit `57130a2702d565be81200ea2e114abcb0455e8bb`, using
  the same patch defaults as the build script. The unchanged upstream
  `name-string.patch` needs fuzz 1; a stricter zero-fuzz attempt stops at that
  existing patch. The recovered BAR1 patch applies without fuzz (a line offset
  is reported). See `patch-validation.log` and `nvidia-input-hashes.json`.
- No kernel-module compilation, installation, GPU workload, or power cycle was
  performed for this recovery. Patch application is not a new hardware test.
