# SM-S937B / S937BXXS9CZG3 porting record

Device-tested temporary root + KernelSU (LKM jailbreak mode) on 2026-08-23.
Sibling build of [`SM-S937B-S937BXXS7CZE1.md`](SM-S937B-S937BXXS7CZE1.md);
produced with the procedure in [`PORTING-SIBLING.md`](PORTING-SIBLING.md).

## 1. Firmware identity

| field | value |
| --- | --- |
| model | `SM-S937B` (Galaxy S25 Edge, international) |
| AP/PDA | `S937BXXS9CZG3` |
| CSC | `S937BOXM9CZG3` (OXM) |
| build fingerprint | `samsung/psqxeea/psq:16/BP4A.251205.006/S937BXXS9CZG3_OXM9CZG3:user/release-keys` |
| kernel release | `6.6.98-android15-8-pd6ff1cd-abogkiS937BXXS9CZG3-4k` |
| kernel size | 38849024 |
| kernel SHA-256 | `6a16b91dd80a6594d0f829425fff70ae9f5a8f35575d832f21ebf5ab045ae204` |
| security patch | 2026-07-05 |

## 2. What changed vs S937BXXS7CZE1

GKI revision `pd6ff1cd` (CZE1: `pe17667d`). The scheduler region moved
(`schedule` 0x01149164 → 0x011482a4, `__schedule` moved with it), so the P0
fingerprint table differs in the affected rows and was regenerated. All other
`.text` symbols are offset-identical, including `worker_thread` and its
`bl schedule` site, so `SLIDE_TRACEFS_WORKER_CALLER_OFF` stays `0x000d97ec`.

| macro | CZE1 | this build | source |
| --- | ---: | ---: | --- |
| `KMALLOC_CACHES_OFF` | `0x017da850` | `0x017da710` | `kmalloc_caches` symbol |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `0x0175e41e` | `0x0175e2af` | `nfulnl_logger.name` → `"nfnetlink_log"` |

`SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` re-verified at `0x02439490`
(unique `sysctl_bootid` pointer, `"boot_id"` name, mode 0444).

## 3. Validation log notes

The unmodified CZE1 payload found the slide on this kernel (fingerprint
`best=8 second=0` — the probe page at that slide contains no boot-patched
words) but failed the pipe cache gate with `want=ffffffc08187abf8` while the
groomed pages correctly showed `slab=ffffff8001cf4c00` — the textbook
`KMALLOC_CACHES_OFF` drift signature. With the corrected offset the exploit
rooted (attempt 21/24, `uid=2000->0`, `context=u:r:kernel:s0`) and
`ksud-s25u-kdp` late-loaded unmodified (`kernelsu` live, Enforcing restored).

Environment prerequisites are the same as the CZE1 record: phantom process
tracking must be disabled (the setting is wiped with the firmware reflash —
re-apply), and the pushed ksud loader needs `chmod 755`.

## 4. Feed note

Both `psq-S937BXXS7CZE1` and this profile match `SM-S937B` + `6.6.98`; the
feed lists this (current) firmware first so automatic selection picks it.
Devices still on CZE1 should select `psq-S937BXXS7CZE1` manually in the
app's Advanced mode — the wrong-table fingerprint scan fails safe.
