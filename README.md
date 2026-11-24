# IIO demux: take the timestamp source offset from all active channels

This repository holds a fix for `iio_buffer_update_demux()` in `drivers/iio/industrialio-buffer.c`, the patch sent to linux-iio, and the STM32MP157 board test evidence used to find and check the bug.

## Contents

- `patches/0001-iio-buffer-take-the-source-timestamp-offset-from-all.patch`: patch against Linux master (commit `8d7fe98`), as sent to linux-iio. Submission status: sent, not yet reviewed or merged.
- `patches/bsp-5.4.31-tested.diff`: the same change as applied to the Linux 5.4.31 BSP that the board test ran on. The code shape differs slightly from the master patch (`masklength` vs `iio_get_masklength()`, `int` vs `ret` for the axis length). The algorithm is the same.
- `evidence/channels-126-summary.tsv`: per-case results of 126 channel-subset and timestamp-switch configurations on the BSP kernel, with the post-fix results.
- `evidence/build-and-limits.txt`: kernel identity and the acceptance limits file used.

## Problem

With `scan_timestamp` enabled, the demux computes where the timestamp sits in the raw scan from the end of the last *selected* channel. The driver pushes a scan where the timestamp follows *all* active channels, aligned to 8 bytes (`iio_compute_scan_bytes()`). When the last selected channel is not the last active channel, the demux reads the timestamp from the wrong offset.

Example: only accelerometer X and the timestamp are enabled. The raw scan is 24 bytes (six 16-bit axes, 4 bytes padding, 8-byte timestamp at offset 16). The old demux read the timestamp from offset 8, which holds gyro Y/Z and padding.

Before the fix, the 13 of 25 timestamp-enabled subsets that were tested (earlier run on kernel `#5`, not in this repository) produced 435 non-positive or non-increasing timestamps. For the `X + timestamp` case, 34 of 50 steady-state frames were invalid.

## Fix

Walk every active channel to compute the source offset of the timestamp, aligned the way `iio_compute_scan_bytes()` aligns it, and use that as `in_loc`. The destination offset is unchanged.

## Board test result (BSP kernel, after the fix)

Kernel: `Linux ATK-stm32mp1 5.4.31 #6`, STM32MP157D, ICM-20608 SPI IIO driver, 50 Hz, 1 s warm-up, 1 s capture per case.

- 126 of 126 cases PASS (63 subsets with timestamp off, 63 with timestamp on, from the 7-bit scan mask encoded as `layout-XX`; bit `0x40` is the timestamp switch).
- `timestamp_errors` is 0 for every case; delivery is 49–50 Hz; no saturated channels.
- See `evidence/channels-126-summary.tsv`.

## What this does not show

- The master patch was not built or run on mainline hardware. The board result is from the BSP 5.4.31 tree.
- Timestamps are software timestamps from the kernel. They do not prove hardware sampling precision or the number of lost samples.
- The channel checks verify subset layout and timestamp monotonicity at rest. They do not calibrate axis accuracy or angular rate.
- The raw sample archive (about 1.7 MB, including `.bin` and `.csv` for all 126 cases) and the full `results.txt` are not in this repository yet.

## Reproduce the kernel side

```sh
# Build the kernel with the patch applied, copy to the board, then on the board:
board_test --device /sys/bus/iio/devices/iio:device0 --report /tmp/demux-126 \
  --test channels --channel-cases 126 --interactive \
  --limits fifo-lab.limits --warmup 1 --settle 1 --duration 1
```

The test program is part of the ICM-20608 driver project and is not included here.
