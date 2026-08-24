# 03 — Killer features

Bagian ini menyaring apa yang benar-benar sulit ditiru pesaing. Urutannya menurut nilai strategis untuk rencana app builder, bukan menurut kerumitan teknis.

## 1. Agen bisa memodifikasi runtime-nya sendiri

Paket `packages/extensions/` memberi model alat `cordis_define`, `cordis_run`, `cordis_stop`, `cordis_undefine`, dan `cordis_inspect_*`. Artinya agen dapat memeriksa pohon plugin yang sedang menjalankannya, menulis plugin baru, memasangnya, menjalankannya, lalu mencabutnya kembali — semua dalam satu sesi, tanpa restart.

Digabung dengan **Creator mode** di [packages/preset](../packages/preset/README.md), agen bahkan bisa menyusun *preset agen* baru: komposisi alat, prompt, dan kemampuan untuk sesi lain. Demonstrasinya ada sebagai perintah siap jalan: `pnpm run demo:cordis`.

Kenapa ini killer: produk app builder pada akhirnya perlu agen-agen terspesialisasi (perancang UI, penulis Gradle, penguji, pemaket rilis). Di sini melahirkan spesialis bukan berarti menulis kode baru di inti — cukup menyusun preset.

## 2. Satu pertukaran penyedia memindahkan seluruh produk

Karena filesystem dan subprocess berdiri di atas seam yang sama, mengarahkan keduanya ke sandbox jarak jauh memindahkan Bash, terminal PTY, dan LSP sekaligus. Tidak ada alat yang perlu di-fork, tidak ada percabangan "kalau remote maka…" di dalam alat.

Buktinya sudah ada dalam bentuk POC: `packages/e2b/` berisi `fs-e2b` dan `subprocess-e2b` yang menjalankan dunia eksekusi di sandbox E2B.

Kenapa ini killer: build APK menuntut mesin bertenaga dengan JDK, Android SDK, dan Gradle. Kemampuan memindahkan seluruh dunia eksekusi ke kontainer jarak jauh — tanpa menyentuh lapisan alat — adalah prasyarat build farm, dan itu sudah tersedia arsitekturnya.

## 3. Agen lain bisa dipasang sebagai subagent

`packages/subagent/` memuat penyedia untuk fork dalam proses, spawn dalam proses, SDK `dsh`, protokol ACP, **Claude Code**, dan **Codex**. Satu antarmuka, banyak dunia.

Kenapa ini killer: strategi model tidak terkunci. Pekerjaan yang lebih murah bisa didelegasikan ke model murah, pekerjaan sulit ke agen lain yang sudah terbukti, dan semuanya tetap terekam di satu session log dengan silsilah induk-anak.

## 4. Session log sebagai basis data, bukan sekadar riwayat obrolan

Log hanya-tambah, dengan aturan "yang dilihat model harus tercatat". Dari satu aliran itu diturunkan: riwayat model, transkrip, fork sesi, resume, statistik, telemetri OpenTelemetry, judul sesi, dan pencarian teks penuh (`session-query` dengan SQLite FTS).

Kenapa ini killer: produk konsumen butuh audit, "lanjutkan proyek saya kemarin", dan penagihan berdasarkan penggunaan. Ketiganya adalah turunan langsung dari log yang sudah ada, bukan fitur baru yang harus dibangun dari nol.

## 5. Pengurungan proses yang sungguhan

`packages/sandbox/` menyediakan seam pengurungan dengan backend nyata: bubblewrap dan Landlock di Linux (dengan addon Node sendiri di [native/](../native/README.md)), Seatbelt di macOS, dan ACL di Windows. Ditambah `sandbox-policy` untuk kebijakannya.

Kenapa ini killer: begitu pengguna awam boleh menjalankan agen di infrastruktur kita, pengurungan bukan lagi opsi. Fondasinya sudah ada dan sudah teruji lintas platform di CI.

## 6. Kolaborasi manusia yang matang

Bukan sekadar tombol "approve": ada preset izin (`workspace-write` versus `danger-full-access`), seam persetujuan, alat bertanya kepada pengguna, perintah manusia berawalan garis miring, plan mode sebagai keadaan tercatat, goal dengan batas ronde, penjadwalan tindak lanjut, dan jobs latar belakang yang bisa diperiksa serta dihentikan.

Kenapa ini killer: inilah beda antara demo dan produk. Alur "agen mengerjakan sesuatu berjam-jam sambil pengguna sesekali menyetujui langkah" sudah punya rumahnya.

## 7. GUI Web berbasis slot dengan HMR plugin

Antarmuka bukan monolit React. Shell mendeklarasikan lubang bernama; plugin `ui-*` mengisinya lewat registrasi reversibel yang sadar-deklarasi. Ada lebih dari tiga puluh paket `ui-*` (percakapan, sidebar, setelan, model, plan, goal, todo, skill, subagent, jobs, lampiran, deliverables, tema, dan seterusnya). Menjalankan `pnpm dsh web` bersama `pnpm run dev:web` memberi hot reload untuk plugin klien.

Kenapa ini killer: membangun antarmuka app builder — panel pratinjau, galeri template, tombol unduh APK — berarti menambah paket UI, bukan mengoperasi ulang shell.

## 8. Deliverables: hasil kerja yang bisa diklik

`packages/client/ui-deliverables` melipat setiap pemanggilan alat yang berhasil mengubah berkas menjadi baris "berkas yang dihasilkan" di akhir giliran, dan menautkan penyebutan berkas di dalam prosa penutup. Pengenalannya berdasarkan *niat render* alat, bukan nama alat — jadi alat mutasi baru ikut terdaftar otomatis.

Kenapa ini killer: "aplikasimu sudah jadi, ini APK-nya" adalah tepat pola ini. Alat build APK yang baru cukup mendeklarasikan niat render dan lokasi keluarannya untuk masuk ke baris deliverables.

## 9. Code Mode: model menulis program, bukan rentetan panggilan alat

PTC mode mengekspos alat lewat Code Mode SDK sehingga model menggabungkan banyak langkah dalam satu program TypeScript, dijalankan oleh runtime kode (worker thread atau CPython). Konsumennya adalah alat `run_code`.

Kenapa ini killer: alur build aplikasi penuh langkah berantai (scaffold, pasang dependensi, generate ikon, build, verifikasi). Satu program jauh lebih murah dalam token dan jauh lebih cepat dibanding lima belas giliran alat.

## 10. Loop otonom yang sudah punya nama dan batas

`workflow` menjalankan skrip alur kerja di worker thread; `ralph` menjalankan ronde-ronde agen-segar terhadap satu objektif tetap, dengan *handoff* terstruktur antar-ronde dan tanpa menyeret percakapan lama. `goal` menyimpan objektif per sesi dengan fase `active` / `paused` / `blocked` / `complete` dan batas ronde.

Kenapa ini killer: "bangun aplikasi dari satu kalimat" adalah pekerjaan berjam-jam yang harus tahan gagal-ulang. Primitifnya sudah ada, lengkap dengan batas agar tidak berputar tanpa akhir.

## 11. Multi-provider LLM, bukan terkunci pada satu vendor

Ada katalog provider (DeepSeek, Anthropic, OpenAI, dan lainnya), dukungan endpoint OpenAI-compatible untuk gateway perusahaan atau server sendiri, plus autentikasi native untuk Bedrock, Vertex, Azure, dan Codex. Deklarasi modalitas gambar per model bisa ditulis di setelan. Panduannya: [docs/user/guide/providers.md](../docs/user/guide/providers.md).

## 12. Jembatan protokol: ACP, MCP, hook, dan dua SDK

Tersedia server ACP (Agent Client Protocol) untuk otomasi, klien MCP, jembatan hook untuk Claude Code dan Codex, SDK TypeScript berbasis JSON-RPC, dan SDK Python. Artinya harness ini bisa menjadi mesin di balik editor, CI, atau produk lain.

## 13. Disiplin rekayasa sebagai fitur

Cakupan uji 100% per berkas di CI, replay snapshot tanpa kunci API, dokumentasi dwibahasa dengan gerbang konsistensi ber-hash, katalog tergenerasi yang dijaga kesegarannya, deteksi duplikasi kode, dan kewajiban Agent Note untuk setiap keputusan.

Kenapa ini killer: untuk produk yang akan dititipi kode orang lain dan menjalankan perintah di mesin orang lain, disiplin ini adalah aset komersial — bukan pemanis.

## Yang justru belum ada

Supaya penilaian seimbang, ini yang **tidak** ditemukan di repositori dan sering dikira ada:

- Tidak ada rantai alat aplikasi seluler apa pun. Pencarian menyeluruh atas kata `android`, `apk`, `gradle`, `expo`, `react-native`, dan `flutter` tidak menghasilkan satu pun kode produk.
- Tidak ada autentikasi, akun, organisasi, peran, atau kuota.
- Tidak ada penagihan, pembatasan laju, atau pengukuran penggunaan per pengguna.
- Tidak ada pratinjau aplikasi (emulator, streaming layar, atau kanal pratinjau).
- Tidak ada penyimpanan artefak rilis, penandatanganan, maupun jalur publikasi ke toko aplikasi.
