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

## Batasan
- pid/tgid keyakinan tinggi tapi belum diverifikasi fungsi kedua.
- sched_task_group/tasks/atomic_flags masih null (opsional).
- ref_images dihitung, tidak dibaca (pointer di-zero-kan vendor).
- Belum diuji di perangkat nyata. Wajib cocokkan `uname -r` persis sebelum pakai.
