# Panduan Lengkap: Temporary Root Tanpa Unlock Bootloader
## (GhostLock + ReSukiSU — Xiaomi/POCO & HP Android lain)

> **Status metode ini (per 3 Okt 2026):** terverifikasi legit oleh riset independen (skor 9/10).
> GhostLock = CVE-2026-43499 resmi (NVD). ReSukiSU = open source di GitHub (GPL).
> **Tapi ingat: root-nya TEMPORER (hilang tiap reboot), dan panduan ini tetap berisiko. Baca bagian Risiko sampai habis sebelum mencoba.**

---

## 1. Cara kerjanya (singkat)

1. Aplikasi **GhostLock** menjalankan exploit kernel (bug use-after-free di futex, CVE-2026-43499) → dapat akses tulis ke kernel → jadi root sementara.
2. Dengan akses itu, tool me-load modul kernel **ReSukiSU** langsung ke kernel yang sedang berjalan (dari memori, tanpa menyentuh partisi / tanpa flash).
3. Hasil: root penuh untuk sesi ini. **Reboot = hilang total, HP kembali stock.**

Karena tidak ada partisi yang ditulis, **bootloader tidak perlu di-unlock** dan tetap terkunci.

---

## 2. Prasyarat — CEK DULU, jangan skip

Metode ini **hanya work jika SEMUA kondisi di bawah terpenuhi.** Kalau satu saja tidak cocok, jangan lanjut.

- [ ] **Versi kernel rentan.** Buka Pengaturan → Tentang Ponsel → tap "Versi kernel" (atau via ADB: `adb shell uname -r`). Syarat: kernel **di bawah** versi patch, yaitu:
  - Kernel 6.1.x → harus **di bawah 6.1.175** (contoh yang terbukti: 6.1.118)
  - Kernel 6.6.x → harus **di bawah 6.6.140**
  - Kernel 6.12.x → harus **di bawah 6.12.86**
- [ ] **Security Patch Level (SPL) ≤ ~Juni/Juli 2026.** Cek di Pengaturan → Tentang Ponsel → Patch keamanan Android. Kalau HP-mu sudah update patch Agustus 2026 ke atas, kemungkinan besar sudah kebal.
- [ ] **Build kernel cocok persis** dengan tabel offset di aplikasi GhostLock (cek `uname -r`, string-nya harus sama persis). Kernel tidak terdaftar = aplikasi menolak jalan (ini fitur keselamatan).
  - **Kabar baik untuk Redmi Note 11 Pro 5G / POCO X4 Pro 5G (veux):** profil kustom untuk kernel `5.4.274-qgki-g82179e362f33` sudah dibuat dan tervalidasi — unduh di https://github.com/HirokiAkihiko/ghostlock-veux-profile lalu impor `.conf`-nya (lihat bagian 4F).
- [ ] Baterai > 50%.
- [ ] **Backup data penting.** Wajib, tanpa kecuali.

> Catatan: bertahan di firmware lama demi exploit = kamu melewatkan patch keamanan lain. Ini trade-off nyata.

---

## 3. Bahan yang dibutuhkan — HANYA dari sumber resmi

> ⚠️ **Peringatan keras:** APK "GhostLock"/"ReSukiSU" dari link acak (grup FB, Telegram tidak resmi, situs download) adalah umpan malware yang sempurna. Hanya unduh dari bawah ini:

1. **ReSukiSU Manager** — https://github.com/ReSukiSU/ReSukiSU (menu Releases, atau channel Telegram resmi **t.me/ReSukiSU**)
2. **GhostLock App** — https://github.com/YuKongA/ghostlock-app (atau port resmi sesuai merek HP: mis. "Root My Galaxy" untuk Samsung)
3. **Shizuku** (cadangan, kalau aplikasinya minta) — https://github.com/RikkaApps/Shizuku — via wireless debugging, tanpa PC.
4. **Profil kustom veux** (khusus Redmi Note 11 Pro 5G / POCO X4 Pro 5G, kernel `5.4.274-qgki-g82179e362f33`) — https://github.com/HirokiAkihiko/ghostlock-veux-profile → unduh file `5.4.274-qgki-g82179e362f33.conf`.

Jangan sentuh: KingRoot, iRoot, dan APK "one-click root" lain — exploitnya mati sejak 2018 dan banyak yang malware.

---

## 4. Langkah-langkah

### A. Persiapan
1. Pastikan semua prasyarat bagian 2 terpenuhi.
2. Backup data penting (foto, chat, dsb).
3. Download & install **ReSukiSU Manager** dari GitHub resmi (jangan dibuka dulu).
4. Download & install **GhostLock App** dari GitHub resmi.

### B. (Opsional, jika diminta aplikasi) Siapkan Shizuku
1. Aktifkan Opsi Pengembang (tap 7x "Nomor build").
2. Aktifkan **Wireless Debugging**.
3. Buka Shizuku → "Start via Wireless Debugging" → ikuti proses pairing sampai status "Shizuku is running".

### C. Eksekusi
1. **Reboot HP**, lalu segera (disarankan dalam ±30 detik setelah boot) buka aplikasi **GhostLock**.
2. Tap tombol **Run** → tunggu proses exploit berjalan. Jangan sentuh HP selama proses.
3. Jika sukses: aplikasi ReSukiSU menampilkan status **Working** (mode jailbreak/LKM). Root aktif untuk sesi ini.
4. Jika gagal: biasanya hanya kernel panic → HP reboot sendiri (aman). Jangan spam tap Run berulang-ulang.

### D. Verifikasi root berhasil
- Buka ReSukiSU Manager → status harus "Berfungsi/Working", mode LKM/jailbreak.
- Install aplikasi **Root Checker** → harus menunjukkan akses root.
- Atau via ADB: `adb shell su -c id` → harus keluar `uid=0(root)`.

### E. Setelah reboot
- **Root hilang total.** HP kembali 100% stock (aplikasi bank, Play Integrity normal lagi).
- Untuk root lagi: ulangi langkah C (atau pakai fitur auto-re-trigger jika tersedia).

### F. Jika kernel-mu tidak ada di tabel (opsi lanjutan)
Aplikasi mendukung impor offset kustom: ekstrak offset dari `boot.img`/OTA resmi pakai tool extractor dari repo, lalu impor file `.conf` ke aplikasi. **Butuh keahlian teknis — bukan untuk pemula.**

**Sudah tersedia — Redmi Note 11 Pro 5G / POCO X4 Pro 5G (veux):**
1. Pastikan `uname -r` di HP persis `5.4.274-qgki-g82179e362f33` (cek via Termux atau `adb shell uname -r`).
2. Unduh `5.4.274-qgki-g82179e362f33.conf` dari https://github.com/HirokiAkihiko/ghostlock-veux-profile, pindahkan ke HP.
3. Di aplikasi GhostLock → menu impor profil → pilih file `.conf` tersebut.
4. Lanjut ke langkah C seperti biasa. Profil ini sudah lolos 27/27 cek validasi schema, tapi **belum pernah diuji di perangkat nyata** — perlakukan sebagai eksperimen.

---

## 5. Risiko — baca sebelum memutuskan

1. **APK palsu = risiko terbesar.** Salah unduh = malware dengan akses kernel. Hanya dari GitHub resmi di atas.
2. **Bootloop tanpa jalan pulih.** Modul root yang tidak cocok bisa bikin HP bootloop, dan karena bootloader terkunci (tanpa UBL), kamu **tidak bisa flash ulang sendiri** → harus ke service center.
3. **Selama sesi root aktif**, aplikasi jahat lain di HP juga bisa memanfaatkan akses root (pencurian data). Jangan buka m-banking/e-wallet saat sesi root aktif.
4. **Garansi:** bootloader tidak di-unlock jadi status unlock-based aman, tapi syarat garansi tetap bisa mengecualikan kerusakan akibat modifikasi software.
5. **Aplikasi bank:** menurut liputan Android Authority, bank & Play Integrity tetap jalan karena perangkat terbaca stock. Tapi aplikasi yang mendeteksi artefak root tetap bisa menolak. Setelah reboot (tanpa re-run), semua normal.
6. **Jendela exploit menyempit.** Tiap update keamanan menutup celah ini. Metode ini punya tanggal kedaluwarsa.

---

## 6. FAQ singkat

**Q: Ini permanen?**
A: Tidak. 100% temporer. Reboot = hilang.

**Q: Data kehapus?**
A: Tidak (tidak ada wipe seperti UBL). Tapi tetap backup — bootloop bisa terjadi.

**Q: Bisa untuk semua HP?**
A: Klaimnya ya, tapi dukungan nyata tergantung **build kernel persis**. Cek tabel offset di aplikasi. Untuk veux (`5.4.274-qgki-g82179e362f33`) profil kustom sudah tersedia di bagian 4F.

**Q: Bedanya dengan Shizuku?**
A: Shizuku = akses level ADB (bukan root), aman, legal, untuk debloat/blokir iklan. GhostLock+ReSukiSU = root beneran (sementara), jauh lebih powerful dan berisiko.

**Q: "RuSukiSU" itu apa?**
A: Kemungkinan typo dari ReSukiSU. Tidak ada proyek resmi bernama itu.

---

## 7. Sumber

- NVD — CVE-2026-43499: https://nvd.nist.gov/vuln/detail/CVE-2026-43499
- ReSukiSU (GitHub resmi): https://github.com/ReSukiSU/ReSukiSU
- GhostLock App (YuKongA): https://github.com/YuKongA/ghostlock-app
- Panduan komunitas: https://github.com/dwongdev/awesome-android-root/blob/HEAD/docs/rooting-guides/root-without-unlocking-bootloader.md
- Android Authority via GadgetHacks (Root My Galaxy, 2 Okt 2026): https://samsung.gadgethacks.com/news/root-my-galaxy-app-explained-temporary-root-without-knox/
- XDA — KernelSU tanpa UBL untuk Xiaomi: https://xdaforums.com/t/kernelsu-root-without-unlocked-bootloader-for-most-qualcomm-xiaomi-devices.4781779/
- Dokumentasi ReSukiSU: https://resukisu.github.io

---

*Panduan ini disusun dari riset web 3 Okt 2026. Kondisi exploit bisa berubah sewaktu-waktu mengikuti update keamanan. Jangan jalankan di HP utama yang dipakai untuk m-banking kecuali kamu paham risikonya.*
