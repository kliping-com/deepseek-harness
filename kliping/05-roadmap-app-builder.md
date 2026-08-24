# 05 — Roadmap: dari harness menjadi Kliping App Builder

Sasaran akhir: pengguna mengetik satu kalimat ide, melihat aplikasinya hidup dalam hitungan detik, lalu menerima aplikasi web yang bisa dibuka dan APK yang bisa dipasang — dengan pengalaman sekelas Emergent, tetapi lebih terbuka dan lebih ramah pengguna teknis.

Dokumen ini menerjemahkan sasaran itu menjadi enam fase yang masing-masing punya keluaran nyata, definisi selesai, dan risiko. Semua nama paket yang diusulkan mengikuti konvensi repositori: satu grup di `packages/<grup>/<paket>/`, npm `@deepseek-ai/dsh-<nama>`, dan setiap kemampuan dibangun sebagai *capability seam* utuh (definisi layanan, penyedia, konsumen).

## Engine kita versus provider yang bisa ditukar

Ini pembedaan terpenting di seluruh dokumen, dan paling mudah salah dibaca.

**Yang sudah ada dan menjadi engine kita** adalah harness ini sendiri: agent loop, registri alat, seam shell dan subprocess, filesystem, sandbox, jobs, session log, sistem slot GUI. Itulah yang membuat agen mampu *mengerjakan* sebuah proyek.

**Expo, EAS, Gradle, Vite, dan sejenisnya bukan engine kita.** Semuanya adalah kandidat *provider* di balik seam yang belum dibuat — apa yang dipanggil agen di dalam kontainer, bukan mesin yang menjalankan agen. Per hari ini pencarian menyeluruh atas `expo`, `eas`, `react-native`, `gradle`, dan `apk` di `packages/` dan `apps/` menghasilkan **nol baris kode produk**; satu-satunya kemunculan kata Android adalah nama proyek palsu di fixture pemilih workspace.

| Lapisan | Isi | Sifat |
|---|---|---|
| Engine | agent loop, alat, seam, sandbox, session log, GUI | milik kita, stabil |
| Provider build | `expo prebuild` + Gradle, atau Gradle langsung, atau Vite untuk web | dapat ditukar tanpa menyentuh alat |
| Provider pratinjau | server dev web, Expo Go, emulator terstreaming | dapat ditukar per tingkat biaya |
| Provider eksekusi | kontainer sendiri, E2B, mesin lokal | dapat ditukar per penyebaran |

Konsekuensinya menyenangkan: mengganti Expo dengan Flutter atau Kotlin nanti bukan penulisan ulang produk, melainkan penggantian satu penyedia di belakang seam yang sama.

### Catatan khusus tentang EAS

EAS punya dua rasa yang konsekuensinya jauh berbeda, dan keduanya sering disebut dengan nama yang sama.

**EAS Build (layanan awan Expo)** menjalankan build di infrastruktur pihak ketiga: berbayar per build, punya antrean, menuntut akun Expo, dan **mengirim kode sumber pengguna keluar dari infrastruktur kita**. Menjadikannya tulang punggung berarti menempelkan margin, SLA, dan posisi privasi produk pada vendor lain.

**`expo prebuild` menjadi proyek native lalu Gradle di citra kontainer kita sendiri** (atau `eas build --local`) menjalankan build di infrastruktur kita: biayanya adalah biaya komputasi kita, tanpa antrean pihak lain, dan kode pengguna tidak pernah meninggalkan sandbox miliknya.

Rekomendasi: **jalur kedua sebagai bawaan**, dengan EAS awan sebagai penyedia cadangan opsional. Seam `ctx.appBuild` membuat keduanya hidup berdampingan tanpa percabangan di alat.

## Kenapa pratinjau adalah produknya

Pengguna awam tidak menilai produk ini dari kualitas kode yang dihasilkan. Mereka menilainya dari satu hal: **berapa lama antara "aku minta ubah tombolnya jadi biru" dan melihat tombol biru itu**. Di situlah Emergent, Lovable, dan Bolt menang, dan di situ pula sebagian besar pesaing kalah.

Karena itu pratinjau bukan pemanis di fase belakang, melainkan **loop inti produk**, dan pekerjaan yang membuatnya cepat dijadwalkan sebelum pekerjaan yang membuat artefak akhir sempurna.

**Tiga tingkat penyedia pratinjau**, dari paling murah ke paling mahal:

| Tingkat | Cara | Waktu tampil | Biaya |
|---|---|---|---|
| Web | server dev di kontainer sesi, ditampilkan di panel GUI | di bawah 5 detik | murah |
| Perangkat asli | Expo Go plus kode QR ke ponsel pengguna | detik sampai puluhan detik | murah |
| Emulator | emulator Android di kontainer, layarnya distream ke browser | puluhan detik sampai menit | mahal |

**Desainnya sudah punya sambungan di repositori.** [`dsh-host-webserver`](../packages/host/webserver/README.md) menyediakan `register(route)` untuk rute HTTP `exact` atau `prefix` dan `registerUpgrade(route)` untuk rute upgrade — persis yang dibutuhkan sebuah proksi pratinjau: satu awalan jalur per sesi untuk halaman aplikasi, plus jalur upgrade untuk WebSocket yang membawa hot reload. Tidak perlu server baru, tidak perlu port baru per pengguna.

**Yang wajib dijaga sejak awal:** pratinjau milik satu pengguna tidak boleh dapat dibuka pengguna lain, umur server dev harus dibatasi agar tidak menumpuk, dan pemakaian sumber dayanya masuk ke kuota yang sama dengan sesi. Ketiganya bergantung pada Fase 1, dan itulah alasan Fase 1 tetap berdiri paling depan.

## Prinsip yang tidak boleh dilanggar

**Jangan fork inti.** Semua tambahan masuk sebagai plugin dan bundle. Begitu ada logika Android di `core/agent-loop`, keunggulan komposabilitas produk ini hilang.

**Setiap kemampuan baru adalah seam utuh.** Bukan "satu alat yang memanggil Gradle", melainkan definisi layanan build, penyedia lokal dan jarak jauh, lalu alat sebagai konsumen. Ini yang memungkinkan build pindah dari laptop ke build farm tanpa menyentuh alatnya.

**Yang dilihat model harus tercatat.** Status build, log build, alamat pratinjau, dan artefak wajib menjadi event sesi, bukan keadaan sampingan. Tanpa itu, resume, fork, audit, dan penagihan tidak bisa diturunkan.

**Manusia awam tidak boleh berhadapan dengan konsep harness.** Profil, bundle, patch, preset izin, dan pemilihan workspace harus tersembunyi di balik alur terpandu. Konsep itu tetap ada untuk pengguna teknis.

## Aset yang sudah ada dan langsung terpakai

| Kebutuhan app builder | Sudah tersedia sebagai |
|---|---|
| Menjalankan perintah build | seam shell (`ctx.shell`) dan subprocess (`ctx.subprocess`) |
| Menjalankan build panjang tanpa memblokir | `packages/jobs` plus alat `job_list`, `job_output`, `job_kill` |
| Terminal interaktif untuk diagnosis | seam terminal PTY plus enam alat `terminal_*` |
| Menyajikan pratinjau dan kanal hot reload | `ctx.webServer` dengan rute prefix dan rute upgrade |
| Memindahkan eksekusi ke kontainer | POC `packages/e2b` (`fs-e2b`, `subprocess-e2b`) |
| Mengurung proses | `packages/sandbox` (bubblewrap/Landlock, Seatbelt, ACL Windows) |
| Menyusun agen spesialis | `packages/preset` plus Creator mode |
| Kerja panjang berbatas | `goal`, `workflow`, `ralph`, `schedule` |
| Menyerahkan berkas hasil | `packages/client/ui-deliverables` |
| Menyimpan artefak | seam storage, attachment, dan spill |
| Menyembunyikan langkah teknis | plan mode, preset izin, perintah manusia |
| Panel UI baru | sistem slot klien plus HMR plugin |

Yang belum ada sama sekali: rantai alat aplikasi apa pun, pratinjau, penandatanganan, akun pengguna, dan kuota.

## Fase 0 — Bedah dan rebranding (selesai)

Keluaran: dokumentasi bedah ini, rebranding permukaan produk menjadi Kliping tanpa logo, dan pemetaan celah. Rinciannya di [dokumen 06](06-rebranding.md).

## Fase 1 — Fondasi multi-pengguna

Tanpa fase ini, tidak ada fase lain yang boleh menyentuh pengguna luar. Pratinjau memperkuat alasannya: sebuah pratinjau adalah URL, dan URL tanpa pemilik adalah kebocoran.

**Yang dibangun.**

- `packages/host/auth` — seam autentikasi: definisi layanan `ctx.auth` dengan penyedia awal berbasis penyedia identitas OIDC, plus penyedia token untuk otomasi. Pemeriksaan kepercayaan dipasang di lapisan koneksi (`packages/client/connection` sudah punya titik pemeriksaan terpadu untuk `/api`), bukan disebar ke tiap rute.
- **Kepemilikan sumber daya** — workspace, sesi, setelan, kredensial, dan nanti pratinjau serta artefak memperoleh pemilik. `packages/workspace` dan `packages/session` adalah tempat perubahan ini bermuara.
- **Isolasi home per pengguna** — satu direktori Harness per pengguna, sehingga `.credentials.yaml`, setelan, dan sesi tidak bercampur.
- **Postur jaringan** — TLS diselesaikan di depan (reverse proxy) plus kebijakan origin; dokumentasikan sebagai kontrak penyebaran, karena webserver sendiri sengaja tidak mengurusnya.
- **Kuota dan pengukuran** — turunkan dari session log dan `session-stats`; batasi lewat kebijakan pada `jobs` dan pada admisi giliran.

**Definisi selesai.** Dua pengguna berbeda pada satu instans tidak dapat melihat sesi, workspace, kredensial, pratinjau, atau artefak satu sama lain, dibuktikan dengan uji end-to-end. Permintaan tanpa kredensial valid ditolak sebelum mencapai gerbang API.

**Risiko.** Ini pekerjaan terbesar dan paling tidak terlihat. Godaan untuk melewatinya demi demo cepat sangat kuat dan akan menghasilkan perombakan berlipat kemudian.

## Fase 2 — Dunia eksekusi jarak jauh

**Yang dibangun.**

- **Penyedia kontainer produksi** — naikkan POC E2B menjadi penyedia yang didukung, atau tambahkan penyedia kontainer sendiri (`packages/sandbox` untuk kebijakan, `packages/subprocess` dan `packages/fs` untuk penyedia). Satu sesi memperoleh satu dunia eksekusi.
- **Siklus hidup sandbox** — pembuatan, pemanasan, batas waktu hidup, pembersihan, dan pemulihan saat sesi di-resume. Ikuti pola pengendali siklus hidup tunggal yang dianut repositori ([docs/defensive-patterns.md](../docs/defensive-patterns.md)).
- **Citra dasar** — Linux dengan Node, pnpm, Git, dan cache paket yang sudah hangat.

**Definisi selesai.** Sesi berjalan penuh di kontainer: `bash`, `terminal_*`, `read`/`write`/`edit`, `glob`/`grep`, dan `lsp` semuanya bekerja tanpa perubahan pada paket alat mana pun.

**Risiko.** Latensi filesystem jarak jauh terasa pada alat pencarian; siapkan pengukuran sejak awal.

## Fase 3 — Pratinjau dan target web

Inilah fase yang membuat produk terasa hidup, dan sengaja mendahului APK: keluarannya lebih cepat terbukti, dan seluruh komponennya dipakai ulang sebagai pratinjau APK di fase berikutnya.

### 3a — Seam pratinjau

- `packages/preview/app-preview` — Service Definition `ctx.appPreview`: memulai, menghentikan, melaporkan alamat, dan mengumumkan status sesi pratinjau sebagai event sesi.
- `packages/preview/app-preview-web` — penyedia server dev web di dalam kontainer sesi.
- `packages/preview/preview-proxy` — rute prefix per sesi plus rute upgrade untuk hot reload di atas `ctx.webServer`, dengan pemeriksaan kepemilikan dari Fase 1.
- `packages/client/ui-app-builder` — panel pratinjau di GUI, indikator status build, dan riwayat versi; mengisi slot yang sudah dideklarasikan shell.

**Definisi selesai.** Perubahan yang diminta pengguna tampak di panel pratinjau dalam hitungan detik, tanpa memuat ulang halaman GUI, dan pratinjau pengguna lain tidak dapat dibuka.

### 3b — Target web sebagai keluaran nyata

Pratinjau membuat aplikasi terlihat; target web membuatnya bisa dibagikan.

- **Build statis** melalui seam build yang sama yang akan dipakai APK, dengan penyedia web.
- **Hosting hasil** — domain atau subdomain per proyek, TLS, dan isolasi antar proyek. Ini pekerjaan platform tersendiri, bukan bonus dari build.
- **Backend aplikasi hasil** — keputusan produk yang harus diambil sadar: aplikasi yang dihasilkan hampir selalu butuh basis data, autentikasi, dan penyimpanan. Rekomendasi: pakai satu penyedia terkelola (misalnya Supabase atau sejenisnya) sebagai integrasi resmi, jangan membangun sendiri. Inilah bagian yang sebenarnya dijual pesaing, dan bagian yang paling sering diremehkan.

**Definisi selesai.** Pengguna dapat membagikan URL aplikasinya kepada orang lain, dan aplikasi itu tetap hidup setelah sesi ditutup.

## Fase 4 — Target APK

Setelah loop iterasi terbukti di web, APK menjadi penambahan penyedia, bukan produk baru — terutama bila kerangka aplikasinya React Native, karena satu basis kode melayani web dan Android sekaligus.

**Keputusan teknologi.** Rekomendasi tetap **React Native dengan Expo** sebagai jalur utama, dan Kotlin dengan Jetpack Compose sebagai jalur lanjutan untuk pengguna teknis.

| Kriteria | Expo / React Native | Kotlin / Compose | Flutter |
|---|---|---|---|
| Kecepatan pratinjau | Sangat baik (web, Expo Go) | Lambat (emulator) | Sedang |
| Satu basis kode untuk web dan Android | Ya | Tidak | Sebagian |
| Kecocokan dengan kekuatan model | Tinggi (TypeScript) | Sedang | Sedang (Dart) |
| Kompleksitas rantai alat build | Sedang | Tinggi | Tinggi |
| Kualitas aplikasi akhir | Baik untuk mayoritas kasus | Terbaik | Baik |

**Yang dibangun.**

- `packages/mobile/app-build` — Service Definition `ctx.appBuild`: kosakata permintaan build (varian debug atau rilis, target ABI, nama paket, versi), spesifikasi hasil (jalur artefak, ringkasan log, kode keluar), dan pemisahan `resolve(request): Spec` dari `run()` seperti pola `dsh-shell`.
- `packages/mobile/app-build-expo` — penyedia `expo prebuild` diikuti Gradle assemble di citra kita sendiri.
- `packages/mobile/app-build-gradle` — penyedia Gradle langsung untuk proyek Android asli.
- `packages/mobile/app-build-eas` — penyedia cadangan opsional untuk EAS awan, dengan peringatan eksplisit bahwa kode sumber meninggalkan infrastruktur kita.
- `packages/mobile/tool-app-build` — Consumer: alat `app_build` yang dilihat model, dengan niat render `terminal` untuk aliran log dan `locations` yang menunjuk artefak APK sehingga otomatis masuk baris deliverables.
- `packages/preview/app-preview-device` — penyedia Expo Go dengan kode QR, dan penyedia emulator terstreaming sebagai tingkat lanjutan.
- **Citra build** — JDK 17, Android SDK command-line tools, platform-tools, build-tools, Gradle, cache dependensi, lisensi SDK disetujui non-interaktif. Menjadi varian citra pada penyedia kontainer Fase 2.
- `packages/mobile/app-scaffold` — skill dan template proyek yang dimuat lewat seam skill yang sudah ada, bukan generator berkas ad hoc.

**Definisi selesai.** Dari sesi kosong, agen dapat membuat proyek baru, menjalankan `app_build`, dan menghasilkan APK debug yang terpasang di perangkat sungguhan. Jalur build tercakup uji snapshot pada transkrip aplikasi terpasang.

**Risiko.** Build dingin bisa menyentuh belasan menit; strategi cache Gradle dan pemanasan citra menentukan pengalaman pengguna.

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

**Definisi selesai.** Pengguna tanpa latar belakang teknis dapat menghasilkan aplikasi web yang bisa dibagikan dan APK yang berjalan, dari satu kalimat ide, tanpa pernah membuka terminal.

## Urutan dan ketergantungan

```text
Fase 1 (multi-user, keamanan)
   └─> Fase 2 (dunia eksekusi jarak jauh)
          └─> Fase 3 (pratinjau + target web)
                 ├─> Fase 4 (target APK, memakai ulang seam pratinjau dan build)
                 │      └─> Fase 5 (rilis dan publikasi)
                 └─> Fase 6 (pengalaman awam)
```

Fase 3 adalah simpul kritis: setelah loop pratinjau hidup, Fase 4 hanyalah menambah penyedia build dan penyedia pratinjau perangkat di atas kerangka yang sama.

## Pekerjaan yang mudah diremehkan

**Hosting dan backend aplikasi hasil.** Membangun artefak web memang lebih mudah daripada APK, tetapi *menghidupkan* aplikasi web — domain, TLS, basis data, autentikasi pengguna akhir — adalah platform tersendiri. APK justru lebih selesai begitu berkasnya jadi. Jangan menyimpulkan "web pasti lebih mudah" untuk keseluruhan produk hanya karena build-nya lebih ringan.

**Biaya token dan waktu.** Membangun aplikasi utuh adalah pekerjaan berjam-jam. Kompaksi konteks, Code Mode, dan delegasi ke subagent murah bukan optimasi belakangan, melainkan penentu kelayakan biaya.

**Determinisme rantai alat.** Versi Node, Gradle, JDK, dan Android SDK harus dipatok di dalam citra. Build yang tidak reprodusibel akan menghabiskan dukungan pelanggan.

**Uji end-to-end yang jujur.** Repositori ini menuntut bukti berupa transkrip aplikasi terpasang, bukan sekadar uji unit. Rencanakan cakupan snapshot untuk alur pratinjau dan build sejak awal, termasuk dukungan harness snapshot yang perlu ditambahkan.

**Dokumentasi dwibahasa.** Setiap dokumen dalam cakupan wajib punya pasangan Mandarin dan catatan konsistensi. Anggarkan waktunya, atau tempatkan dokumen produk di luar cakupan seperti direktori `kliping/` ini.

## Ukuran keberhasilan

| Fase | Metrik |
|---|---|
| 1 | Nol kebocoran data lintas pengguna pada uji; waktu masuk di bawah tiga detik |
| 2 | Sesi dingin siap pakai di bawah sepuluh detik; paritas alat seratus persen |
| 3 | Perubahan tampak di pratinjau di bawah lima detik; URL aplikasi tetap hidup setelah sesi ditutup |
| 4 | Build APK debug pertama di bawah sepuluh menit; build berikutnya di bawah tiga menit |
| 5 | Build rilis bertanda tangan dalam satu kali klik persetujuan |
| 6 | Lebih dari separuh pengguna baru menghasilkan aplikasi berjalan pada sesi pertama |
