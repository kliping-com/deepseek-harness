# 04 — Status saat ini: untuk super admin, belum untuk pengguna

Pertanyaannya: *"apakah kondisi saat ini adalah untuk super admin, dan belum untuk user? karena aku melihat ini murni framework?"*

Jawabannya: **benar, dengan satu koreksi.** Benar bahwa ini dirancang untuk satu operator berhak penuh di mesinnya sendiri, dan belum untuk pengguna awam. Koreksinya: ini bukan *murni* framework — ada produk nyata di atasnya (GUI Web, CLI, mode headless, server ACP). Hanya saja produk itu ditujukan untuk pengembang, bukan konsumen.

## Bukti bahwa ini bermodel operator tunggal

**Server hanya untuk loopback, tanpa TLS dan tanpa autentikasi.** [packages/host/webserver](../packages/host/webserver/README.md) menyatakannya sendiri di bagian batasan: *"No TLS, auth, or origin policy — binding a non-loopback address exposes the server to that network; deployment hardening is deliberately out of scope for the dev-facing v1."* Konfigurasi host hanya menerima `127.0.0.1` (postur bawaan) atau `0.0.0.0` (paparan jaringan yang disengaja). Tidak ada sesi login, cookie, token, ataupun kebijakan origin di antara keduanya.

**Identitas bersifat anonim.** Grup `packages/identity/` menyediakan identitas anonim bersama — untuk telemetri dan korelasi, bukan untuk membedakan pengguna. Tidak ada entitas akun, tidak ada kepemilikan sumber daya, tidak ada peran.

**Kredensial adalah milik mesin, bukan milik akun.** Kunci API tersimpan di `$DSH_HOME/.credentials.yaml`, dan penyedia kredensial membaca variabel lingkungan di atas berkas `.env`. Siapa pun yang bisa membuka GUI itu memakai kunci yang sama.

**Agen berjalan dengan hak akun sistem.** Direktori pemanggil menjadi workspace bawaan. Preset izin bawaan `workspace-write` membatasi penulisan ke dalam workspace, tetapi ada `danger-full-access` sejauh satu klik, dan dialog konfirmasinya berbunyi: *"Full access reduces confirmation steps and lets the agent perform more actions directly, including sensitive operations, file changes, or external commands."* Itu kalimat yang ditujukan kepada operator yang paham risikonya.

**Workspace adalah direktori lokal.** Memilih workspace berarti menunjuk folder di mesin yang menjalankan server. Tidak ada konsep proyek milik pengguna yang terisolasi dari proyek pengguna lain.

**Prasyarat pemakaiannya berupa perkakas pengembang.** Menjalankan dari sumber butuh Node ^22.19 atau lebih baru, pnpm, `pnpm run build`, dan kunci API yang dipasang sendiri. Jalur `npx @deepseek-ai/dsh web` memang lebih pendek, tetapi tetap menuntut Node dan terminal.

**Statusnya diumumkan sendiri sebagai pratinjau pengembang.** [README.md](../README.md) menulis *developer preview* dengan peringatan perubahan yang merusak kompatibilitas, dan pemberitahuan sambutan di GUI menyebut versi 0.1 masih dalam pengujian untuk pengembang Harness.

## Bukti bahwa ini bukan sekadar framework

Yang membuatnya lebih dari pustaka: ada GUI Web lengkap (percakapan, sidebar sesi dan workspace, setelan model, plan, goal, todo, lampiran, deliverables, tema terang/gelap), ada CLI dengan mode headless sekali-jalan, ada server ACP untuk otomasi, dan ada dua SDK. Yang belum ada adalah **lapisan produk konsumen** di atasnya.

Cara paling jujur menyebutnya: *framework-first product* — fondasi kelas platform dengan satu aplikasi rujukan yang matang untuk pengembang.

## Peta celah menuju produk untuk pengguna awam

Tabel ini memakai istilah yang sama dengan roadmap di [dokumen 05](05-roadmap-app-builder.md).

| Kebutuhan produk konsumen | Kondisi sekarang | Besar pekerjaan |
|---|---|---|
| Akun, login, sesi pengguna | Tidak ada | Besar — butuh paket auth baru di sisi host dan pemeriksa kepercayaan di lapisan koneksi |
| Isolasi data antar pengguna | Tidak ada; satu `$DSH_HOME`, satu berkas kredensial | Besar — kepemilikan pada workspace, sesi, setelan, dan kredensial |
| Eksekusi tepercaya di infrastruktur kita | Sandbox lokal ada; POC E2B ada | Sedang — tukar penyedia fs dan subprocess, tambah manajemen siklus hidup kontainer |
| Kuota, batas laju, penagihan | Tidak ada; statistik token ada | Sedang — dapat diturunkan dari session log dan telemetri |
| Rantai alat Android | Tidak ada sama sekali | Besar — seam baru plus citra kontainer |
| Pratinjau aplikasi berjalan | Tidak ada | Sedang sampai besar, tergantung strategi (Expo Go, emulator terstreaming, atau pratinjau web) |
| Penandatanganan dan publikasi | Tidak ada | Sedang — keystore sebagai kredensial, plus kebijakan penyimpanan rahasia |
| Onboarding untuk orang awam | GUI menuntut pilih workspace, atur provider, pahami mode izin | Sedang — alur terpandu dan preset yang menyembunyikan konsep |
| Keamanan aplikasi web | Tanpa TLS, auth, kebijakan origin | Besar — wajib sebelum ada pengguna eksternal |

## Rekomendasi urutan

Pengalaman produk sejenis menunjukkan urutan yang salah akan mahal. Urutan yang disarankan:

1. **Keamanan dan multi-user lebih dulu.** Tanpa autentikasi dan isolasi, setiap fitur berikutnya dibangun di atas asumsi yang harus dibongkar lagi.
2. **Baru eksekusi jarak jauh.** Setelah ada pemilik untuk setiap sesi, kontainer per sesi menjadi masuk akal dan bisa ditagihkan.
3. **Baru rantai alat Android.** Membangun alat build sebelum ada tempat berdaulat untuk menjalankannya berarti menguji di laptop orang.
4. **Baru pengalaman "ketik ide → jadi aplikasi".** Lapisan ini murni produk: template, pratinjau, tombol unduh, galeri.

Poin 1 dan 2 adalah pekerjaan platform yang tidak terlihat oleh pengguna, tetapi menentukan apakah produk ini bisa dijual sama sekali.
