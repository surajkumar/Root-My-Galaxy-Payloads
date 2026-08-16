# SM-S937B / S937BXXS7CZE1 porting record

Offline port from firmware analysis only; **not yet executed on hardware**.
Sister profile of [`SM-S9370-S9370ZCS9CZG1.md`](SM-S9370-S9370ZCS9CZG1.md)
(same `psq` board), which was device-tested on 2026-08-07.

## 1. Firmware identity

| field | value |
| --- | --- |
| model | `SM-S937B` (Galaxy S25 Edge, international) |
| AP/PDA | `S937BXXS7CZE1` |
| CSC | `S937BOXM7CZE1` (OXM/EUX) |
| codename | `psq` |
| build fingerprint | `samsung/psqxxx/psq:16/BP4A.251205.006/S937BXXS7CZE1:user/release-keys` |
| kernel release | `6.6.98-android15-8-pe17667d-abogkiS937BXXS7CZE1-4k` |
| kernel size | 38849024 |
| kernel SHA-256 | `344aa4dd9f0506430927443f48a7a45a83c77dd2d8ae940a7f4db68547dd0e92` |

Fingerprint, model, and incremental read from `meta-data/fota.zip`
(`SYSTEM/build.prop`, `VENDOR/build.prop`) of the AP archive. The ramdisk and
vendor `ro.*.build.fingerprint` values are stale AP3A/Android 15 placeholders;
the composed runtime fingerprint is the one above.

The GKI base is `pe17667d`, not the `p5a696e2` of `S9370ZCS9CZG1`. The
`.text` of both kernels is identical at the symbol level (every offset below
was re-derived from this build), but `.data`/`.rodata` placement shifted.

## 2. Offsets that differ from psq-S9370ZCS9CZG1

Only two values differ. Everything else in `target.h` matches the CZG1
profile bit-for-bit and was re-verified against this build's `vmlinux.elf`
and BTF, not copied.

| macro | CZG1 | this build | source |
| --- | ---: | ---: | --- |
| `KMALLOC_CACHES_OFF` | `0x017dac30` | `0x017da850` | `kmalloc_caches` symbol |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `0x0175e6e9` | `0x0175e41e` | `nfulnl_logger.name` → `"nfnetlink_log"` |

This build's `kmalloc_caches` (`0x017da850`) also differs from `pa3q`
(`0x017da710`), so the shared `pa3q` payload fails this device at the pipe
cache gate exactly as documented in section 2-3 of the CZG1 record.

`SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` is `0x02439490`, numerically the
same as CZG1 but independently confirmed here: it is the single pointer to
`sysctl_bootid` (`0xffffffc0826426d8`) in the raw Image, preceded by the
`"boot_id"` name pointer, mode `0444`.

## 3. Re-verified identical values

- All `.text` symbol offsets (`ashmem_*`, `configfs_*_iter`,
  `copy_splice_read`, `noop_llseek`, `call_usermodehelper_exec_work`,
  `anon_pipe_buf_ops`, `ashmem_fops`, `system_unbound_wq`, `init_task`,
  `root_task_group`, `selinux_state`, `sysctl_bootid`, `nfulnl_logger`,
  `ashmem_misc`) — from `llvm-nm` on this build's recovered ELF.
- BTF layouts: `file_operations` (0x108), `task_struct` (`pi_lock` 0x90c
  etc.), `miscdevice.fops` 0x10, `nf_logger` 32, `rt_mutex_waiter` 112,
  `configfs_buffer` (`page` 0x10, `needs_read_fill` 0x50, `bin_buffer` 0x58,
  `bin_buffer_size` 0x60, `cb_max_size` 0x64), workqueue structs, and
  `selinux_state.enforcing` at +0.
- `struct page` has **no** slab fields in 6.6; SLUB uses `struct slab` with
  `slab_cache` at `0x08`. Hence `STRUCT_SLAB_CACHE_OFF 0x08` (the 6.1 value
  `0x18` does not apply). `compound_head` 0x08, `page_type` 0x30.
- `SLIDE_TRACEFS_WORKER_CALLER_OFF 0x000d97ec`: `bl schedule` at
  `0xffffffc0800d97e8` inside `worker_thread`; return PC `0xd97ec`.
- `SLIDE_TRACEFS_EVENT_ID 109`: `__event_sched_blocked_reason` is event
  index 89 (`(0x022b1be8 - 0x022b1920) / 8`) and `__TRACE_LAST_TYPE` is 20
  on this branch. Defined explicitly in `target.h`.
- `SLIDE_PSELECT_WORD_SHIFT 0`: same reasoning as CZG1 — non-LEGACY
  `rt_mutex_waiter` occupies words 0-13 and `nfds=320` gives a 15-qword
  logical fd_set, so only 0 or 1 are possible; 0 is the device-proven value
  on this board.

## 4. Physical load address

```c
#define P0_PHYS_OFFSET       0x80000000ULL
#define P0_KERNEL_PHYS_LOAD  0xa8000000ULL
```

Qualcomm device: the BL archive has **no `sboot.bin`** (that is the Exynos
procedure in `PORTING.md`). `uefi.elf` is an ELF64 AArch64 XBL/UEFI image
(entry `0xA7000000`, single LOAD at `0xa7000000`); `abl.elf` is a 32-bit ARM
wrapper. The `0xa8000000` byte pattern in `xbl_config.elf` is a false
positive: that config store is big-endian, so LE `00 00 00 a8` decodes as
the small integer `0xa8`, not an address.

The value rests on: (a) the device-verified CZG1 profile on the same board,
(b) `pa3q`/`q7q` agreement on the same Pakala platform, and (c) the CZG1
runtime log arithmetic: observed write target `0xffffff802a5ad7f0` =
`DIRECT_MAP_BASE + (P0_KERNEL_PHYS_LOAD - P0_PHYS_OFFSET) + slide(0x130000)
+ ASHMEM_MISC_FOPS_OFF(0x247d7f0)`, which closes exactly.

## 5. P0 fingerprint

Regenerated from this build's raw Image with
`tools/generate_p0_fingerprint.pl` (`PROBE_OFFSET=0x1f0000`); the tool
verified all 256 source qwords. Row 0 word 1 is `0xeb00029f943c7b5c`
(CZG1: `...802c`, pa3q: `...77ac`), so neither sibling table can be reused.

## 6. KernelSU

Same as CZG1: the existing `kernelsu/ksud-s25u-kdp` (`android15-6.6` KMI)
loads unmodified in LKM jailbreak mode. No KernelSU rebuild is needed.

## 7. Build (WSL, Windows NDK via interop)

Only a Windows NDK (`windows-x86_64` prebuilts) was present. The extensionless
`aarch64-linux-android35-clang` wrapper in the Windows NDK is a bash script
and works from WSL; override `TARGET_CC`:

```sh
NDK="/mnt/c/Users/Development/AppData/Local/Android/Sdk/ndk/28.2.13676358"
CC="$NDK/toolchains/llvm/prebuilt/windows-x86_64/bin/aarch64-linux-android35-clang"
make TARGET=psq-S937BXXS7CZE1 ANDROID_NDK_HOME="$NDK" TARGET_CC="$CC" all release
```

`cve-2026-43499-app.release.so` passes the fixed 104128-byte gate and is
copied to `artifacts/psq-S937BXXS7CZE1/cve-2026-43499-app.so`.

## 8. Support feed

`SM-S937B` was removed from the `galaxy-s25-series-2026-06-07` (pa3q) entry:
that payload deterministically cannot work on `psq` hardware, and model
matching must resolve `SM-S937B` to exactly one payload. Other `SM-S937x`
regional models remain on the generic entry until their own `psq` profiles
are ported.

## 9. Scope

Verified only by offline analysis against `S937BXXS7CZE1`. Hardware
execution on `SM-S937B` is still pending; expected behavior matches the
device-verified CZG1 profile, given the identical `.text` and the two
corrected data offsets.
