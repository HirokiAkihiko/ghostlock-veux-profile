# Derivasi profil GhostLock `5.4.274-qgki-g82179e362f33` (veux)

Perangkat: Redmi Note 11 Pro 5G / POCO X4 Pro 5G (veux), Snapdragon 695,
HyperOS OS1.0.10.0.TKCIDXM (ID), Android 13, kernel 5.4.274-qgki-g82179e362f33.

## Ringkasan
Extractor resmi (`ghostlock-extract`) berhasil memulihkan kallsyms (179372 simbol)
setelah patch `--kallsyms-base` (vendor men-zero-kan `kallsyms_relative_base`),
tetapi profil yang dihasilkan tidak lengkap: semua field `task_struct`/`cred`
dan `task_offset`/`lock_offset` null karena kernel ini tidak punya BTF.
Field-field tersebut diderivasi manual di bawah ini dari disassembly + analisis
binary. Semua nilai adalah offset dalam byte dari awal struct (bukan tebakan).

## Temuan penting
1. **Vendor memodifikasi `struct cred`**: `usage` adalah `atomic_t` (4 byte),
   bukan `atomic_long_t` (8 byte) seperti vanilla 5.4.274. Terbukti dari pola
   binary init_cred: `usage=4 @0`, lalu `cap_permitted/effective/bset =
   0x3FFFFFFFFF @48/56/64`, `cap_inheritable=0 @40`, `cap_ambient=0 @72`.
2. **Pointer `.data` di-zero-kan vendor** di image yang dikirim (bukan hasil
   ekstraksi — terkonfirmasi ada di boot.img asli). `init_cred.user/user_ns/
   group_info` dibaca 0 padahal harusnya `&root_user/&init_user_ns/&init_groups`.
   Nilai `refN_image` dihitung sebagai `_text + fileoff(simbol)` dari kallsyms.
3. `struct task_struct` vendor lebih besar dari vanilla (batas via kallsyms:
   `init_task@0x29ba4c0` → `init_signals@0x29bb480` = 4032 byte).

## task_struct (offset, sumber verifikasi)
| Field | Offset | Cara verifikasi |
|---|---|---|
| prio | 124 (0x7c) | `task_blocks_on_rt_mutex`: `ldr w8,[x20,#0x7c]` = task->prio → waiter->prio |
| normal_prio | 132 (0x84) | `sched_fork`: `p->prio = current->normal_prio` |
| pi_lock | 2276 (0x8e4) | `task_rq_lock` & `task_blocks_on_rt_mutex`: `&p->pi_lock` untuk raw_spin_lock |
| pi_waiters | 2288 (0x8f0) | `rt_mutex_init_task` (inlined di copy_process): `p->pi_waiters = RB_ROOT_CACHED` |
| pi_top_task | 2304 (0x900) | idem, store berurutan setelah pi_waiters |
| pi_blocked_on | 2312 (0x908) | `task_blocks_on_rt_mutex`: `str x22,[x20,#0x908]` = task->pi_blocked_on = waiter |
| cred | 2064 (0x810) | `copy_creds`/`prepare_creds`: `p->cred`, lalu `cred->thread_keyring@104` cocok |
| real_cred | 2056 (0x808) | `copy_creds`: `p->real_cred = get_cred(p->cred)` |
| comm | 2080 (0x820) | string "swapper" di init_task+0x820 |
| seccomp | 2240 (0x8c0) | `__secure_computing`: `seccomp_mode(&current->seccomp)`, cek mode 1/2 |
| pid | 1624 (0x658) | `copy_process`: `p->pid = pid_nr(pid)` (keyakinan tinggi) |
| tgid | 1628 (0x65c) | `copy_process`: logika tgid/CLONE_THREAD (keyakinan tinggi) |
| tasks | 1368 (0x558) | `mm_update_next_owner`: loop `for_each_process` — `ldr x8,[x26,#0x558]` + `sub x9,x8,#0x558` (container_of), cek `PF_KTHREAD` (bit 21) di hasil (variabel luar `g`, bukan inner `for_each_thread`) |
| sched_task_group | 992 (0x3e0) | `sched_move_task` (inlined `sched_change_group`): `str x8,[x22,#0x3e0]` = `tsk->sched_task_group = tg`; x22=tsk terkonfirmasi via pola `sched_class@0x90` |
| atomic_flags | 1568 (0x620) | struktural: `pid(1624) − sizeof(restart_block)(48) − 8`; urutan vanilla `atomic_flags → restart_block → pid → tgid` |
| sched_task_group, tasks, atomic_flags | null | tidak wajib di schema; belum diverifikasi |

## struct cred (layout vendor, C2)
`usage@0` (4B) `uid@4` `gid@8` `suid@12` `sgid@16` `euid@20` `egid@24`
`fsuid@28` `fsgid@32` `securebits@36` `cap_inheritable@40` `cap_permitted@48`
`cap_effective@56` `cap_bset@64` `cap_ambient@72` `jit_keyring@80`
`session_keyring@88` `process_keyring@96` `thread_keyring@104`
`request_key_auth@112` `security@120` `user@128` `user_ns@136`
`group_info@144` `rcu@152`, sizeof = 168.
Diverifikasi silang: `prepare_creds` (`get_uid(new->user)@0x80`,
`get_group_info(new->group_info)@0x90`, keyring accesses @0x58/0x60/0x68/0x70).

Template: `usage_value=4`, `caps=(offset 48, count 3, value 0x3FFFFFFFFF)`,
refs (dihitung dari kallsyms, bukan dibaca dari binary):
- ref0: offset 128 → `root_user` @ VA 0xffffffc00a9c6a30
- ref1: offset 136 → `init_user_ns` @ VA 0xffffffc00a9c67f8
- ref2: offset 144 → `init_groups` @ VA 0xffffffc00a9c7838

## rt_mutex_waiter
`task@48` (0x30), `lock@56` (0x38) — dari `task_blocks_on_rt_mutex`:
`stp x20,x19,[x22,#0x30]` (waiter->task = task, waiter->lock = lock).
Konsisten dengan `struct rb_node` 24 byte ×2 di 5.4 dan
`CONFIG_DEBUG_RT_MUTEXES` tidak set (cek IKCONFIG).

## Validasi
27/27 cek schema `docs/kernel_profiles/PROFILE_SCHEMA.md` (required-field
matrix + semua bounds) LULUS. `kernel_phys_load` null = fallback ke formula SoC
(diizinkan schema).

**Update 2026-10-04**: aplikasi menggabungkan DUA validator
(`validateProfileFields` + `ProfileResolver.validateMerged`). Yang kedua
mewajibkan **15 field `task_struct` berupa Number** (tidak boleh null/hilang).
Tiga field yang kurang (`tasks`, `sched_task_group`, `atomic_flags`) diderivasi
dari disassembly (lihat tabel di atas). `refN_image` ditulis sebagai **signed
i64** (negatif) sesuai konvensi extractor (`(*image as i64)`), bukan unsigned.
`kernel_phys_offset` diisi eksplisit `0x80000000` (standar Qualcomm SM6375;
native fallback ke nilai ini bila absen). Replika Python kedua validator:
**invalidPaths KOSONG**.

## Batasan
- pid/tgid keyakinan tinggi tapi belum diverifikasi fungsi kedua.
- sched_task_group/tasks/atomic_flags masih null (opsional).
- ref_images dihitung, tidak dibaca (pointer di-zero-kan vendor).
- Belum diuji di perangkat nyata. Wajib cocokkan `uname -r` persis sebelum pakai.

## Update 2026-10-04: mm_struct_sz 1024 -> 1280

**Masalah**: Exploit gagal di W1 heap spray, `KernelSnitch mm_struct leak failed` 4/4.

**Root cause**: `kernelsnitch.mm_struct_sz = 1024` disalin dari profil 5.15 (miracle), tapi untuk kernel 5.4 ini kemungkinan salah. Dari kode sumber GhostLock (`src/core/kernel/constants.hpp`):
```cpp
inline constexpr unsigned long MM_STRUCT_SZ = 0x500;  // = 1280
```
1280 adalah default bawaan aplikasi untuk kernel yang tidak diketahui ukurannya. Karena 1024 gagal konsisten, dikembalikan ke default 1280.

**Catatan**: `/proc/slabinfo` dan `/sys/kernel/slab/mm_struct/object_size` tidak bisa dibaca tanpa root di Android, jadi ukuran pasti belum terverifikasi. Jika 1280 juga gagal, perlu investigasi lebih lanjut.

## Update 2026-10-04: mm_struct_sz 1280 -> 896

**Analisis dari source kernel 5.4.274** (`kernel/fork.c:mm_cache_init`):
```c
mm_size = sizeof(struct mm_struct) + cpumask_size();
mm_cachep = kmem_cache_create_usercopy("mm_struct", mm_size, ...,
        SLAB_HWCACHE_ALIGN|...);
```

- `sizeof(struct mm_struct)` ≈ 880 (dihitung dari `include/linux/mm_types.h` + config)
- `cpumask_size()` = 8 (8 CPU → 1 unsigned long)
- Total = 888, dibulatkan ke 64-byte (SLAB_HWCACHE_ALIGN) → **896**

1280 (default bawaan) kemungkinan untuk kernel lebih baru dengan mm_struct lebih besar.

## Troubleshooting: KernelSnitch mm_struct leak failed (2026-10-04)

**Gejala**: Profil diterima aplikasi (`invalid=0`), exploit jalan tapi gagal di W1 heap spray:
```
[*] [spray] mm_struct leaked=0xffffffffffffffff
[-] KernelSnitch mm_struct leak failed (4/4 retries)
[-] heap spray failed
```

**Nilai `mm_struct_sz` yang dicoba** (tiap ronde: download ulang .conf → re-import → run di HP):
| Nilai | Asal | Hasil |
|-------|------|-------|
| 1024 | Disalin dari profil 5.15 (miracle) | Gagal |
| 1280 | Default bawaan aplikasi (`src/core/kernel/constants.hpp`) | Gagal |
| 896 | Analisis source 5.4.274: `mm_cache_init()` = `sizeof(mm_struct)+cpumask_size()`, SLAB_HWCACHE_ALIGN → ~880+8=888 → 896 | Gagal |
| 960 | Langkah 64-byte berikutnya | Gagal |
| 832 | Langkah 64-byte sebelumnya | Gagal |

**Temuan kunci dari config kernel** (diekstrak via IKCONFIG dari Image + `/proc/config.gz` via Shizuku, 6664 baris, identik):
```
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SHUFFLE_PAGE_ALLOCATOR=y
```

**Kesimpulan**: Kegagalan konsisten di semua nilai `mm_struct_sz` menunjukkan masalah BUKAN pada ukuran, melainkan hardening kernel (`SLAB_FREELIST_RANDOM`, `SHUFFLE_PAGE_ALLOCATOR`) yang membuat teknik heap spray stride-based KernelSnitch tidak kompatibel dengan kernel 5.4 ini.

**Tindak lanjut**: Issue dipost ke `YuKongA/ghostlock-app#270` untuk konfirmasi developer.

## mm_struct_sz = 920 (derivasi 2026-10-05, metode TheFliss)

Di-derive dari disassembly `Image`: string "mm_struct" @ fileoff 0x1e712fb, di-xref dari 2 situs
(0x2504ee8, 0x250c488) dengan pola identik:

```
adrp x0, #0x1e71000
add  x0, x0, #0x2fb      ; x0 = "mm_struct"
mov  w1, #0x398          ; a2 = size = 0x398 = 920
mov  w3, #0x2000
movk w3, #0x404, lsl #16 ; a4 = 0x04042000 = 67379200
mov  w4, #0x158          ; a5
mov  w5, #0x170          ; a6
mov  w2, wzr             ; a3 = 0
mov  x6, xzr             ; a7 = 0
bl   kmem_cache_create_usercopy
```

Cocok dengan `mm_cachep = kmem_cache_create_usercopy("mm_struct", 0x398, 0, 67379200, 0x158, 0x170, 0)`
di `proc_caches_init`. Nilai lama 832 (tebakan) diganti 920. Catatan: ukuran benar TIDAK
memperbaiki W1 heap-spray failure (hardening kernel) — terbukti di moonstone (TheFliss, issue #272).

## kernel_phys_offset = 0x40000000, kernel_phys_load = 0x40080000 (derivasi 2026-10-05)

Semantik dari source GhostLock (`src/core/memory/address_space.cpp`):
- `kernel_phys_offset` = DRAM base utk translasi image -> direct-map
  (default bawaan P0 = 0x80000000 bila null/tidak di-override).
- `kernel_phys_load` = alamat fisik kernel di-load
  (default P0_KERNEL_PHYS_LOAD = 0xa8000000 bila 0).

Derivasi:
- DRAM base = **0x40000000** dari device tree (`memory@40000000`, reg 8GB).
  Nilai lama 0x80000000 hanya default GhostLock — SALAH utk veux.
- `text_offset` dari header Image ARM64 (magic "ARMd" OK) = **0x80000**.
  kernel_phys_load = DRAM base + text_offset = **0x40080000**.
  Nilai lama 0 (-> default 0xa8000000) juga salah.

Sanity translasi: image_addr = KIMAGE_TEXT_BASE+off -> physical = 0x40080000+off
(>= phys_offset OK) -> direct = (0x80000+off) | PAGE_OFFSET. Masuk akal.

Caveat: DRAM base 0x40000000 berasal dari DT kiriman user (belum diverifikasi
independen dari vendor_boot DTB di payload.bin). text_offset dari Image sendiri (solid).
