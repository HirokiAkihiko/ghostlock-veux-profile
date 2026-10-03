# Profil Offset Kernel Kustom GhostLock — Redmi Note 11 Pro 5G (veux)

Profil offset kustom untuk GhostLock temp-root, dibuat dari `boot.img` firmware
stok karena kernel perangkat ini tidak ada di daftar built-in aplikasi.

## Perangkat & firmware

- Perangkat: Redmi Note 11 Pro 5G (codename **veux**, Snapdragon 695)
- Firmware: MIUI `OS1.0.10.0.TKCIDXM` (VEUXIDGlobal, Android 13)
- Kernel: `5.4.274-qgki-g82179e362f33`
- Dibuat: 2026-10-03

## Isi repo

- `5.4.274-qgki-g82179e362f33.offsets.json` — profil format JSON
- `5.4.274-qgki-g82179e362f33.conf` — profil format HOCON (self-contained)

9 offset simbol berhasil di-resolve: `init_task`, `init_cred`,
`selinux_enforcing`, `selinux_blob_sizes`, `security_hook_heads`,
`root_task_group`, `empty_zero_page`, `slide_nfulnl_logger`,
`slide_loggers_0_1`, `slide_boot_id`.

Catatan: field struct semuanya `null` — kernel ini tidak membawa BTF,
jadi derivasi struct tidak dimungkinkan; offset simbol inti tercakup semua.

## Cara membuat

1. Download ROM `miui_VEUXIDGlobal_OS1.0.10.0.TKCIDXM_*.zip`, ekstrak
   `payload.bin`, lalu ekstrak `boot.img` → `Image` (kernel).
2. Build extractor: `YuKongA/ghostlock-app` (`tools/extract_rs`,
   commit `123a469`) → `cargo build --release`.
3. Jalankan extractor terhadap `Image` → profil di atas.

## Status

Profil dibuat dan lolos verifikasi format, **belum diuji di perangkat**.
Pengujian temp-root di HP sesungguhnya tetap berisiko (bootloop) —
lakukan dengan kesadaran penuh.
