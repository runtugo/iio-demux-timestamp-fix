# IIO demux: fix the timestamp source offset

This repository documents an IIO buffer demultiplexing bug found on an STM32MP157 with an ICM-20608 SPI/IIO driver. It contains the Linux IIO mailing-list patches and the separate Linux 5.4.31 BSP fix and board-test summary.

## Upstream submission

- **v2:** [`patches/0001-iio-buffer-use-cached-timestamp-offset-in-demux-v2.patch`](patches/0001-iio-buffer-use-cached-timestamp-offset-in-demux-v2.patch), based on Linux master commit `8d7fe98`. It reuses the cached source timestamp offset instead of recomputing it.
- **v1:** [`patches/0001-iio-buffer-take-the-source-timestamp-offset-from-all.patch`](patches/0001-iio-buffer-take-the-source-timestamp-offset-from-all.patch), retained for the review history.
- [v2 patch on the Linux IIO mailing list](https://lore.kernel.org/linux-iio/tencent_945BB9A3F886BF4E33511C90AD6F89C40809@qq.com/) and [Arnav Kapoor's Reviewed-by](https://lore.kernel.org/linux-iio/20261125134805.57530-1-kapoorarnav43@gmail.com/).
- [Patchwork status](https://patchwork.kernel.org/series/1221852/): **New** as checked on 2026-11-26. Reviewed-by is not a merge; this is a mailing-list patch, not a GitHub pull request.

## Problem and fix

When a buffer requests a subset of the device's active scan channels and a timestamp, the old demux source cursor only advances to the end of the last selected channel. Unselected trailing source channels can cause it to read the timestamp from the wrong offset.

For the reproduced `X + timestamp` case, the device source scan contains six 16-bit axes, 4 bytes of padding and an 8-byte timestamp at offset **16**. The old demux reads offset **8**. In the destination scan, offset **8** is correct because only the X channel precedes the timestamp.

On the v2 mainline base, `iio_compute_scan_bytes()` calculates the source timestamp offset for the actual active scan mask; `iio_enable_buffers()` caches it before building the demux table. The push helper also uses this cached offset. v2 uses `ACCESS_PRIVATE(indio_dev, scan_timestamp_offset)` for the demux source while leaving destination alignment unchanged.

The tested 5.4.31 BSP does not have this cached field. Its [separate BSP patch](patches/bsp-5.4.31-tested.diff) walks all active source channels and aligns the timestamp; it is not the mainline v2 patch and remains unchanged.

## Board test result: BSP 5.4.31, not mainline v2

On STM32MP157D with ICM-20608, Linux 5.4.31 `#6`, 50 Hz configured rate, 1 s warm-up and 1 s capture per case:

- Before the BSP fix, 13 of 25 tested timestamp-enabled configurations had 435 non-positive or non-increasing timestamps. The X + timestamp case had 34 invalid timestamps in 50 steady-state frames.
- After the BSP fix, 126/126 configurations passed: 63 channel subsets with timestamps off and 63 with timestamps on. The X + timestamp case had 0/50 timestamp errors; delivery was 49–50 frames/s.
- See [per-case results](evidence/channels-126-summary.tsv) and [kernel identity and limits](evidence/build-and-limits.txt).

These are results for the separate downstream BSP fix, not a build or hardware test of the mainline v2 patch. Software timestamps and userspace delivery rates do not prove hardware sampling precision or exact loss counts. The channel checks do not calibrate axis accuracy or angular rate. The raw `.bin`/`.csv` samples and full `results.txt` are not in this repository.

## Reproduce the BSP test

The `board_test` program and limits file are part of the ICM-20608 driver project and are not included here. After applying the BSP patch and building and booting the BSP kernel, run on the board:

```sh
board_test --device /sys/bus/iio/devices/iio:device0 --report /tmp/demux-126 \
  --test channels --channel-cases 126 --interactive \
  --limits fifo-lab.limits --warmup 1 --settle 1 --duration 1
```

The upstream v2 patch was not successfully built or run on mainline hardware in this work.
