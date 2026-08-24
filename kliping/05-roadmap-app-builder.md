# 05 — Roadmap: dari harness menjadi Kliping App Builder

Sasaran akhir: pengguna mengetik satu kalimat ide, lalu menerima aplikasi Android yang bisa dipasang — dengan pengalaman sekelas Emergent, tetapi lebih terbuka dan lebih ramah pengguna teknis.

Dokumen ini menerjemahkan sasaran itu menjadi enam fase yang masing-masing punya keluaran nyata, definisi selesai, dan risiko. Semua nama paket yang diusulkan mengikuti konvensi repositori: satu grup di `packages/<grup>/<paket>/`, npm `@deepseek-ai/dsh-<nama>`, dan setiap kemampuan dibangun sebagai *capability seam* utuh (definisi layanan, penyedia, konsumen).

## Prinsip yang tidak boleh dilanggar

**Jangan fork inti.** Semua tambahan masuk sebagai plugin dan bundle. Begitu ada logika Android di `core/agent-loop`, keunggulan komposabilitas produk ini hilang.

**Setiap kemampuan baru adalah seam utuh.** Bukan "satu alat yang memanggil Gradle", melainkan definisi layanan build, penyedia lokal dan jarak jauh, lalu alat sebagai konsumen. Ini yang memungkinkan build pindah dari laptop ke build farm tanpa menyentuh alatnya.

**Yang dilihat model harus tercatat.** Status build, log build, dan artefak wajib menjadi event sesi, bukan keadaan sampingan. Tanpa itu, resume, fork, audit, dan penagihan tidak bisa diturunkan.

**Manusia awam tidak boleh berhadapan dengan konsep harness.** Profil, bundle, patch, preset izin, dan pemilihan workspace harus tersembunyi di balik alur terpandu. Konsep itu tetap ada untuk pengguna teknis.

## Aset yang sudah ada dan langsung terpakai

| Kebutuhan app builder | Sudah tersedia sebagai |
|---|---|
| Menjalankan perintah build | seam shell (`ctx.shell`) dan subprocess (`ctx.subprocess`) |
| Menjalankan build panjang tanpa memblokir | `packages/jobs` plus alat `job_list`, `job_output`, `job_kill` |
| Terminal interaktif untuk diagnosis | seam terminal PTY plus enam alat `terminal_*` |
| Memindahkan eksekusi ke kontainer | POC `packages/e2b` (`fs-e2b`, `subprocess-e2b`) |
| Mengurung proses | `packages/sandbox` (bubblewrap/Landlock, Seatbelt, ACL Windows) |
| Menyusun agen spesialis | `packages/preset` plus Creator mode |
| Kerja panjang berbatas | `goal`, `workflow`, `ralph`, `schedule` |
| Menyerahkan berkas hasil | `packages/client/ui-deliverables` |
| Menyimpan artefak | seam storage, attachment, dan spill |
| Menyembunyikan langkah teknis | plan mode, preset izin, perintah manusia |
| Panel UI baru | sistem slot klien plus HMR plugin |

Yang belum ada sama sekali: rantai alat Android, pratinjau perangkat, penandatanganan, akun pengguna, dan kuota.

## Fase 0 — Bedah dan rebranding (selesai)

Keluaran: dokumentasi bedah ini, rebranding permukaan produk menjadi Kliping tanpa logo, dan pemetaan celah. Rinciannya di [dokumen 06](06-rebranding.md).

## Fase 1 — Fondasi multi-pengguna

Tanpa fase ini, tidak ada fase lain yang boleh menyentuh pengguna luar.

**Yang dibangun.**

- `packages/host/auth` — seam autentikasi: definisi layanan `ctx.auth` dengan penyedia awal berbasis penyedia identitas OIDC, plus penyedia token untuk otomasi. Pemeriksaan kepercayaan dipasang di lapisan koneksi (`packages/client/connection` sudah punya titik pemeriksaan terpadu untuk `/api`), bukan disebar ke tiap rute.
- **Kepemilikan sumber daya** — workspace, sesi, setelan, dan kredensial memperoleh pemilik. `packages/workspace` dan `packages/session` adalah tempat perubahan ini bermuara.
- **Isolasi home per pengguna** — satu direktori Harness per pengguna, sehingga `.credentials.yaml`, setelan, dan sesi tidak bercampur.
- **Postur jaringan** — TLS diselesaikan di depan (reverse proxy) plus kebijakan origin; dokumentasikan sebagai kontrak penyebaran, karena webserver sendiri sengaja tidak mengurusnya.
- **Kuota dan pengukuran** — turunkan dari session log dan `session-stats`; batasi lewat kebijakan pada `jobs` dan pada admisi giliran.

**Definisi selesai.** Dua pengguna berbeda pada satu instans tidak dapat melihat sesi, workspace, kredensial, atau artefak satu sama lain, dibuktikan dengan uji end-to-end. Permintaan tanpa kredensial valid ditolak sebelum mencapai gerbang API.

**Risiko.** Ini pekerjaan terbesar dan paling tidak terlihat. Godaan untuk melewatinya demi demo APK sangat kuat dan akan menghasilkan perombakan berlipat kemudian.

## Fase 2 — Dunia eksekusi jarak jauh

**Yang dibangun.**

- **Penyedia kontainer produksi** — naikkan POC E2B menjadi penyedia yang didukung, atau tambahkan penyedia kontainer sendiri (`packages/sandbox` untuk kebijakan, `packages/subprocess` dan `packages/fs` untuk penyedia). Satu sesi memperoleh satu dunia eksekusi.
- **Siklus hidup sandbox** — pembuatan, pemanasan, batas waktu hidup, pembersihan, dan pemulihan saat sesi di-resume. Ikuti pola pengendali siklus hidup tunggal yang dianut repositori ([docs/defensive-patterns.md](../docs/defensive-patterns.md)).
- **Citra dasar** — Linux dengan Node, pnpm, Git, dan cache paket yang sudah hangat.

**Definisi selesai.** Sesi berjalan penuh di kontainer: `bash`, `terminal_*`, `read`/`write`/`edit`, `glob`/`grep`, dan `lsp` semuanya bekerja tanpa perubahan pada paket alat mana pun.

**Risiko.** Latensi filesystem jarak jauh terasa pada alat pencarian; siapkan pengukuran sejak awal.

## Fase 3 — Rantai alat Android

Inilah kemampuan yang benar-benar baru.

**Keputusan teknologi yang harus diambil lebih dulu.** Rekomendasi: **React Native dengan Expo** sebagai jalur utama, dan Kotlin dengan Jetpack Compose sebagai jalur lanjutan untuk pengguna teknis.

| Kriteria | Expo / React Native | Kotlin / Compose | Flutter |
|---|---|---|---|
| Kecepatan pratinjau | Sangat baik (Expo Go, pratinjau web) | Lambat (emulator) | Sedang |
| Kecocokan dengan kekuatan model | Tinggi (TypeScript) | Sedang | Sedang (Dart) |
| Kompleksitas rantai alat build | Sedang | Tinggi | Tinggi |
| Kualitas aplikasi akhir | Baik untuk mayoritas kasus | Terbaik | Baik |

**Yang dibangun.**

- `packages/mobile/app-build` — Service Definition `ctx.appBuild`: kosakata permintaan build (varian debug atau rilis, target ABI, nama paket, versi), spesifikasi hasil (jalur artefak, ringkasan log, kode keluar), dan pemisahan `resolve(request): Spec` dari `run()` seperti pola `dsh-shell`.
- `packages/mobile/app-build-expo` — penyedia untuk alur Expo: `prebuild` lalu Gradle assemble, atau build lokal EAS.
- `packages/mobile/app-build-gradle` — penyedia Gradle langsung untuk proyek Android asli.
- `packages/mobile/tool-app-build` — Consumer: alat `app_build` yang dilihat model, dengan niat render `terminal` untuk aliran log dan `locations` yang menunjuk artefak APK.
- **Citra build** — JDK 17, Android SDK command-line tools, platform-tools, build-tools, Gradle, cache dependensi. Menjadi varian citra pada penyedia kontainer Fase 2.
- `packages/mobile/app-scaffold` — skill dan template proyek yang dimuat lewat seam skill yang sudah ada, bukan generator berkas ad hoc.

**Definisi selesai.** Dari sesi kosong, agen dapat membuat proyek baru, menjalankan `app_build`, dan menghasilkan APK debug yang terpasang di perangkat sungguhan. Jalur build tercakup uji snapshot pada transkrip aplikasi terpasang.

**Risiko.** Waktu build dingin bisa menyentuh belasan menit; strategi cache Gradle dan pemanasan citra menentukan pengalaman pengguna. Lisensi Android SDK harus disetujui secara non-interaktif di dalam citra.

## Fase 4 — Pratinjau dan iterasi

Tanpa pratinjau cepat, "ketik ide → jadi aplikasi" hanya terasa seperti antrean build.

**Yang dibangun.**

- `packages/mobile/app-preview` — Service Definition `ctx.appPreview`: memulai, menghentikan, dan melaporkan alamat sesi pratinjau.
- **Penyedia pratinjau web** — jalankan target web React Native, sajikan lewat rute host, tampilkan di panel GUI. Paling murah dan paling cepat.
- **Penyedia Expo Go** — server pengembangan plus kode QR untuk perangkat fisik pengguna.
- **Penyedia emulator terstreaming** — emulator di kontainer dengan aliran layar ke browser. Paling mahal; jadikan opsi lanjutan.
- `packages/client/ui-app-builder` — panel pratinjau, status build, riwayat versi, dan tombol unduh APK, semuanya mengisi slot yang sudah dideklarasikan shell.

**Definisi selesai.** Perubahan yang diminta pengguna terlihat di pratinjau dalam hitungan detik untuk jalur web dan Expo Go, dan pengguna dapat mengunduh APK dari GUI.

## Fase 5 — Rilis: penandatanganan dan publikasi

**Yang dibangun.**

- `packages/credentials/keystore` — keystore penandatanganan sebagai kredensial, memakai seam kredensial yang ada; nilai rahasia tidak pernah kembali ke klien, hanya deskriptor tersunting.
- **Build rilis** — varian rilis dengan penandatanganan, penomoran versi otomatis dari session log, dan AAB untuk toko aplikasi.
- **Penyimpanan artefak** — APK dan AAB sebagai artefak berumur panjang lewat seam storage, dengan retensi dan rute unduh yang terautentikasi.
- **Jalur publikasi** — unggahan ke Google Play lewat API penerbitan sebagai penyedia terpisah, dengan persetujuan manusia wajib sebelum publikasi.

**Definisi selesai.** Pengguna dapat menghasilkan build rilis bertanda tangan dan mengunggahnya ke jalur pengujian internal tanpa meninggalkan produk.

**Risiko.** Penyimpanan keystore adalah tanggung jawab hukum dan keamanan. Pertimbangkan menjadikan kepemilikan keystore sebagai pilihan pengguna sejak awal.

## Fase 6 — Pengalaman untuk orang awam

**Yang dibangun.**

- **Onboarding tanpa konsep harness** — pilih template, jelaskan ide, mulai. Workspace dibuat otomatis; provider model disediakan produk.
- **Preset "App Builder mode"** — komposisi alat khusus: alat berkas, build, pratinjau, skill template; tanpa alat inspeksi Cordis dan tanpa akses penuh sistem.
- **Bundle profil** `packages/bundle/app-builder` — satu lapisan patch yang menyusun semua di atas, sehingga `dsh --profile app-builder` menghasilkan produk konsumen, sementara profil `web` tetap menjadi alat pengembang.
- **Galeri template dan contoh** — masuk lewat seam skill supaya bertambah tanpa rilis kode.
- **Pagar pengaman** — batas ronde goal, batas biaya per proyek, dan ringkasan biaya yang terlihat pengguna.

**Definisi selesai.** Pengguna tanpa latar belakang teknis dapat menghasilkan APK yang berjalan dari satu kalimat ide, tanpa pernah membuka terminal.

## Urutan dan ketergantungan

```text
Fase 1 (multi-user, keamanan)
   └─> Fase 2 (dunia eksekusi jarak jauh)
          ├─> Fase 3 (rantai alat Android)
          │      └─> Fase 5 (rilis dan publikasi)
          └─> Fase 4 (pratinjau)
                 └─> Fase 6 (pengalaman awam)
```

Fase 3 dan 4 dapat berjalan paralel setelah Fase 2 selesai. Fase 6 menunggu keduanya karena pengalaman awam bergantung pada pratinjau yang cepat.

## Pekerjaan yang mudah diremehkan

**Biaya token dan waktu.** Membangun aplikasi utuh adalah pekerjaan berjam-jam. Kompaksi konteks, Code Mode, dan delegasi ke subagent murah bukan optimasi belakangan, melainkan penentu kelayakan biaya.

**Determinisme rantai alat.** Versi Gradle, JDK, dan Android SDK harus dipatok di dalam citra. Build yang tidak reprodusibel akan menghabiskan dukungan pelanggan.

**Uji end-to-end yang jujur.** Repositori ini menuntut bukti berupa transkrip aplikasi terpasang, bukan sekadar uji unit. Rencanakan cakupan snapshot untuk alur build sejak awal, termasuk dukungan harness snapshot yang perlu ditambahkan.

**Dokumentasi dwibahasa.** Setiap dokumen dalam cakupan wajib punya pasangan Mandarin dan catatan konsistensi. Anggarkan waktunya, atau tempatkan dokumen produk di luar cakupan seperti direktori `kliping/` ini.

## Ukuran keberhasilan

| Fase | Metrik |
|---|---|
| 1 | Nol kebocoran data lintas pengguna pada uji; waktu masuk di bawah tiga detik |
| 2 | Sesi dingin siap pakai di bawah sepuluh detik; paritas alat seratus persen |
| 3 | Build APK debug pertama di bawah sepuluh menit; build berikutnya di bawah tiga menit |
| 4 | Pratinjau perubahan di bawah lima detik untuk jalur web |
| 5 | Build rilis bertanda tangan dalam satu kali klik persetujuan |
| 6 | Lebih dari separuh pengguna baru menghasilkan APK berjalan pada sesi pertama |
