# Kingston KC3000 / FURY Renegade NVMe FUA workaround

A Linux kernel patch for a persistent overheating issue affecting some Kingston KC3000 and FURY Renegade SSDs running firmware `EIFK51.2`.

This is an out-of-tree workaround, not an upstream kernel fix. The exact cause inside the SSD firmware is still unknown.

## The problem

On my Kingston KC3000 2 TB, NVMe writes using Force Unit Access (FUA) appear to trigger a high-power state that doesn't go away when the drive becomes idle. Once triggered, the SSD keeps heating up even with no significant disk activity. I've seen temperatures around 80-85°C at idle.

Resetting the controller or rebooting clears the state, but FUA traffic can trigger it again. I haven't observed the same behavior under Windows.

A FURY Renegade owner reported a similar problem in [Issue #1](https://github.com/Frausty45/linux-kc3000-no-fua/issues/1). Their testing also showed that turning FUA off *after* the drive had entered the hot state did not cool it down. FUA had to be disabled before normal disk activity resumed after a reboot.

We don't know exactly what the firmware is doing internally, but the results so far point strongly toward FUA as the trigger.

## The workaround

The patch adds `NVME_QUIRK_NO_FUA` to the Linux NVMe driver. When the quirk applies, the driver stops advertising native FUA support to the block layer, while leaving the SSD's write cache enabled. The block layer can then use its normal write and flush handling instead of issuing native FUA requests.

This does not disable the write cache or filesystem barriers. It also doesn't block raw NVMe commands issued directly to the controller.

Earlier versions of this patch matched PCI device IDs. The current version uses the kernel's existing `core_quirks[]` table to match the NVMe vendor ID, model string and firmware revision. This avoids disabling FUA on drives with firmware that hasn't shown the problem.

### Device matching

| Model | Firmware | Behavior with this patch | Test status |
| --- | --- | --- | --- |
| `KINGSTON SKC3000D2048G` | `EIFK51.2` | FUA disabled | Tested on Linux 7.2.15 |
| `KINGSTON SFYRD2000G` | `EIFK51.2` | FUA disabled | FUA-off workaround independently tested; this exact patch not yet tested |
| `KINGSTON SFYRD2000G` | `EIFK31.6` | FUA unchanged | Independently observed working normally with FUA enabled |
| Other model or firmware combinations | Any | FUA unchanged | Not covered by this patch |

The entries use NVMe vendor ID `0x2646`. The PCI IDs seen in the reports are `2646:5013` and `2646:501f`, but PCI ID alone is no longer used to enable `NO_FUA`.

## Implementation

The current patch changes two files:

- `drivers/nvme/host/nvme.h`: adds `NVME_QUIRK_NO_FUA` at bit 24 and its name mapping.
- `drivers/nvme/host/core.c`: adds the firmware-specific entries to `core_quirks[]` and skips `BLK_FEAT_FUA` when the quirk is active.

`drivers/nvme/host/pci.c` is unchanged. Existing PCI-specific quirks, including `NVME_QUIRK_NO_SECONDARY_TEMP_THRESH` on `2646:5013`, are preserved.

The core quirk is selected during initial controller identification, before the namespace queue limits are set. Linux currently doesn't rescan these firmware-dependent quirks on every controller reinitialization, so reboot and verify the result after a firmware update.

## Test results

### My KC3000

Tested on November 4, 2026:

| Item | Result |
| --- | --- |
| SSD | Kingston KC3000 2 TB (`KINGSTON SKC3000D2048G`) |
| PCI ID | `2646:5013` |
| Firmware | `EIFK51.2` |
| Kernel | `7.2.15-arch1-1-kc3000` (Arch Linux) |
| `/sys/block/nvme0n1/queue/fua` | `0` |
| Initial composite temperature | 37°C |
| Composite temperature after a write test | 42°C |

The patched kernel built, installed and booted successfully, including the NVIDIA DKMS driver. The write test did not trigger the persistent overheating behavior. This is an initial functional test, not a long-term reliability result. Write-cache status should also be checked on each test system.

### Independent FURY Renegade test

The tester in [Issue #1](https://github.com/Frausty45/linux-kc3000-no-fua/issues/1) reported a FURY Renegade (`KINGSTON SFYRD2000G`, PCI `2646:501f`, firmware `EIFK51.2`) at about 68.8°C while affected. With FUA disabled before normal filesystem activity after a clean reboot, its temperature dropped to about 34.9°C after roughly an hour.

That test used a separate workaround on Linux 6.6, not this Linux 7.2.15 patch. It supports the approach, but the firmware-specific FURY Renegade entry still needs direct testing.

The same tester also had a FURY Renegade with firmware `EIFK31.6` that operated normally with FUA enabled. That difference is why the current patch matches firmware rather than just the PCI ID.

## Building on Arch Linux

The current patch was generated against Linux 7.2.15 and tested with the Arch Linux `7.2.15.arch1-1` packaging sources. It may need rebasing for other kernel versions.

Start with the matching official Arch `PKGBUILD` and `config.x86_64` in a separate build directory:

1. Change `pkgbase=linux` to `pkgbase=linux-kc3000`.
2. Add `kc3000-no-fua.patch` to `source=()` after the Arch kernel patch. The stock `prepare()` function already applies `.patch` files from `source=()`.
3. Run `updpkgsums`.
4. Run `makepkg -o` to check that the patch applies and source preparation succeeds.
5. Run `makepkg -s` to build the packages.
6. Install the resulting kernel and headers packages with `sudo pacman -U`.

Keep your regular Arch kernel installed as a fallback. Before rebooting, check that your initramfs or UKI was generated correctly, any DKMS modules built successfully, and your bootloader entry points to the new kernel.

The PKGBUILD also produces a documentation package, but you don't need it to boot the kernel.

## Verifying the workaround

First, confirm the model and firmware. Replace `/dev/nvme0` with the appropriate controller:

```bash
sudo nvme id ctrl /dev/nvme0 | grep -E '^(vid|mn|fr)[[:space:]]*:'
```

After booting the patched kernel:

```bash
uname -r
cat /sys/block/nvme0n1/queue/fua
cat /sys/block/nvme0n1/queue/write_cache
sudo nvme log smart /dev/nvme0
```

For a matching device, `fua` should be `0`, while `write_cache` should normally report `write back` if the drive advertises a volatile write cache. Make sure `/dev/nvme0` and `/sys/block/nvme0n1` refer to the same physical drive on your system.

Check temperatures during normal use and after writes have stopped. I don't recommend deliberately re-enabling FUA on an affected drive just to reproduce the issue.

## Current limitations

- Only the listed model and firmware combinations are matched. Other capacities and firmware revisions may need separate entries.
- The FURY Renegade entry has not yet been tested with this exact kernel patch.
- This patch has not yet been accepted upstream.

## Links

- [Issue #1: investigation and independent test results](https://github.com/Frausty45/linux-kc3000-no-fua/issues/1)
- [Compiled Release](https://github.com/Frausty45/linux-kc3000-no-fua/releases/tag/Compiled)

If you're seeing similar behavior, please open an issue with your exact model string, firmware revision, PCI ID, kernel version, FUA status and temperature readings. Reports from unaffected drives are useful too.
