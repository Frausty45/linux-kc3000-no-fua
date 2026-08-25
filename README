# linux-kc3000-no-fua

Linux kernel workaround for a Kingston KC3000 / FURY Renegade NVMe issue triggered by Force Unit Access (`FUA`) writes.

On affected drives, FUA writes under Linux can leave the Phison E18 controller in an abnormally high-power state. The SSD may continue heating even after all I/O has stopped.

This patch adds a device-specific `NVME_QUIRK_NO_FUA` quirk for the affected controller.

## Problem

On my Kingston KC3000, certain writes under Linux caused the drive to enter a persistent high-power state.

Once triggered, the drive continued heating even when:

- Disk I/O was at 0
- The filesystem was idle
- The filesystem was unmounted
- NVMe APST was enabled
- PCIe ASPM was enabled
- The controller was configured to enter its non-operational power state

The composite temperature eventually reached approximately 85°C while the drive was completely idle.

Example from the affected state:

```text
temperature                             : 83 °C
Warning Temperature Time                : 1183
Critical Composite Temperature Time     : 4
Temperature Sensor 2                    : 90 °C
```

At the same time, `iostat` reported no activity.

Unmounting the filesystem did not cause the drive to cool down. A reboot, which resets the NVMe controller, immediately restored normal behavior.

After rebooting without mounting the drive, the temperature dropped from approximately 49°C to 46°C and continued towards its normal idle temperature.

The same behavior had also occurred during a previous Linux installation, while the drive did not noticeably exhibit the problem under Windows.

## Cause

The behavior appears to be related to FUA writes sent to the Kingston KC3000 / FURY Renegade controller.

The affected controller has PCI ID:

```text
2646:5013
```

On the unmodified kernel, the NVMe driver advertises both:

```text
BLK_FEAT_WRITE_CACHE
BLK_FEAT_FUA
```

for controllers with a volatile write cache.

After an FUA write triggers the problem, the KC3000 can remain in a high-power state despite having no active I/O.

## Workaround

The patch introduces:

```c
NVME_QUIRK_NO_FUA
```

and applies it specifically to:

```c
PCI_DEVICE(0x2646, 0x5013)
```

This PCI ID covers the Kingston KC3000 and Kingston FURY Renegade.

When the quirk is active, the NVMe driver continues advertising:

```text
BLK_FEAT_WRITE_CACHE
```

but does not advertise:

```text
BLK_FEAT_FUA
```

for the affected controller.

The write cache itself remains enabled. Linux can use cache flush semantics instead of FUA for write durability.

Other NVMe controllers are unaffected.

## Kernel Changes

The patch modifies three files in the NVMe host driver.

### `drivers/nvme/host/nvme.h`

Adds the new quirk:

```c
NVME_QUIRK_NO_FUA = (1 << 23)
```

and its corresponding name:

```c
case NVME_QUIRK_NO_FUA:
    return "no_fua";
```

### `drivers/nvme/host/core.c`

FUA is only advertised when the controller does not have `NVME_QUIRK_NO_FUA`:

```c
if ((ns->ctrl->vwc & NVME_CTRL_VWC_PRESENT) && !info->no_vwc) {
    lim.features |= BLK_FEAT_WRITE_CACHE;

    if (!(ns->ctrl->quirks & NVME_QUIRK_NO_FUA))
        lim.features |= BLK_FEAT_FUA;
    else
        lim.features &= ~BLK_FEAT_FUA;
} else {
    lim.features &= ~(BLK_FEAT_WRITE_CACHE | BLK_FEAT_FUA);
}
```

### `drivers/nvme/host/pci.c`

The existing KC3000 / FURY Renegade entry is changed from:

```c
{ PCI_DEVICE(0x2646, 0x5013),
    .driver_data = NVME_QUIRK_NO_SECONDARY_TEMP_THRESH, },
```

to:

```c
{ PCI_DEVICE(0x2646, 0x5013),
    .driver_data = NVME_QUIRK_NO_SECONDARY_TEMP_THRESH |
                   NVME_QUIRK_NO_FUA, },
```

The existing `NVME_QUIRK_NO_SECONDARY_TEMP_THRESH` quirk is retained.

## Tested Configuration

| Component | Configuration |
| --- | --- |
| SSD | Kingston KC3000 2 TB |
| Controller | Phison PS5018-E18 |
| PCI ID | `2646:5013` |
| Kernel | Linux 7.1.10 / Arch Linux `7.1.10.arch1` |
| Filesystem | Btrfs |
| System | ASUS TUF Gaming A15 FA507NV |
| CPU | AMD Ryzen 5 7535HS |
| GPU | NVIDIA GeForce RTX 4060 Laptop GPU |

## Results

### Unmodified kernel

After triggering the issue:

```text
Composite temperature: ~85°C
Disk I/O:              0
Filesystem:            unmounted
```

The temperature remained high until the controller was reset by rebooting the system.

### Patched kernel

The patched kernel was built as:

```text
Linux 7.1.10-arch1-1-kc3000
```

After booting it, the block layer correctly reported FUA as disabled for the KC3000:

```console
$ cat /sys/block/nvme0n1/queue/fua
0
```

The drive was then mounted and tested with a 1 GiB sequential write followed by `sync`. The test file was deleted and another `sync` was performed.

The drive remained at approximately 42°C.

The runaway idle heating observed with FUA enabled could no longer be reproduced.

## Verification

First identify the NVMe controller:

```console
$ lspci -nn | grep -i 'Non-Volatile'
```

The affected controller should have PCI ID:

```text
2646:5013
```

Determine which block device corresponds to the KC3000:

```console
$ lsblk -o NAME,MODEL
```

After booting a kernel containing the patch, check whether FUA is advertised:

```console
$ cat /sys/block/nvme0n1/queue/fua
0
```

Replace `nvme0n1` with the appropriate block device if necessary.

The composite temperature can be checked using `nvme-cli`:

```console
$ sudo nvme smart-log /dev/nvme0
```

For continuous monitoring:

```console
$ watch -n 5 "sudo nvme smart-log /dev/nvme0 | grep '^temperature'"
```

## Compatibility

The current version has been tested with Linux 7.1.10.

Kernel internals change over time. Before porting the patch to another kernel version, verify that:

1. `NVME_QUIRK_NO_FUA` has not already been implemented upstream.
2. Bit 23 in `enum nvme_quirks` has not been assigned to another quirk.
3. The feature setup in `drivers/nvme/host/core.c` has not changed in a way that requires the patch to be adapted.
4. The KC3000 / FURY Renegade entry still uses PCI ID `2646:5013`.
5. The patch applies cleanly to the target kernel.

## Scope

This patch is a workaround for the behavior observed on the Kingston KC3000 / FURY Renegade controller.

It is not intended as a general method of disabling FUA.

`NVME_QUIRK_NO_FUA` is only assigned to PCI ID `2646:5013`. Other NVMe devices continue using the normal kernel behavior.

## Status

Tested and working on Linux 7.1.10.

The long-term solution would be an appropriate fix or device quirk in the upstream Linux kernel, at which point maintaining this patch would no longer be necessary.

## Disclaimer

This modifies the Linux NVMe driver and should be treated as an experimental workaround.

Keep a known-good kernel available as a fallback and back up important data before testing a modified kernel.

## License

This repository is based on the Linux kernel source tree.

The Linux kernel is licensed under GPL-2.0-only WITH Linux-syscall-note where applicable. See [`COPYING`](COPYING) and [`LICENSES/`](LICENSES/) for details.
