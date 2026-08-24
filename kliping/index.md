# Kliping — Dokumentasi Bedah & Roadmap

Kumpulan dokumen ini adalah hasil bedah total repositori ini (`deepseek-harness`, kini dirilis dengan nama produk **Kliping**) plus rencana pengembangannya menuju produk *app builder* yang bisa dipakai orang awam untuk membangun aplikasi Android.

Semua isi dokumen ini ditulis dari pembacaan kode dan dokumen di repositori pada versi `0.1.1-rc.2`. Setiap klaim penting menyertakan jalur berkasnya supaya bisa diperiksa ulang.

## Peta dokumen

| Dokumen | Isi |
|---|---|
| [01 — Ini framework apa?](01-apa-ini.md) | Identitas teknis produk: jenis framework, model mental, posisi terhadap LangChain, Claude Code, dan Emergent |
| [02 — Bedah arsitektur](02-bedah-arsitektur.md) | Anatomi lengkap: boot, profil, bundle, agent loop, capability seam, session log, lapisan Web, gerbang mutu |
| [03 — Killer features](03-killer-features.md) | Keunggulan nyata yang sulit ditiru pesaing, beserta bukti kodenya |
| [04 — Status: super admin, belum untuk user](04-status-super-admin.md) | Jawaban atas pertanyaan "ini masih murni framework, ya?" beserta daftar celah menuju produk konsumen |
| [05 — Roadmap Kliping App Builder](05-roadmap-app-builder.md) | Rencana bertahap dari framework developer menjadi "ketik ide → jadi APK", lengkap dengan paket baru yang perlu dibuat |
| [06 — Catatan rebranding](06-rebranding.md) | Apa yang sudah diganti menjadi Kliping, apa yang sengaja dibiarkan, dan cara melanjutkan |

## Ringkasan eksekutif

**Ini bukan aplikasi, ini fondasi.** Yang ada di repositori adalah *agent harness*: mesin yang menjalankan siklus "model berpikir → memanggil alat → hasil masuk ke log → model berpikir lagi", ditambah katalog alat yang lengkap (bash, PTY, editor berkas, pencarian, LSP, web, subagent, workflow, skill) dan satu GUI Web sebagai etalase. Semuanya disusun di atas [Cordis](../docs/cordis-primer.md), sebuah kerangka plugin: **setiap bagian produk adalah plugin yang bisa dicabut dan diganti dari berkas konfigurasi**, termasuk agent loop-nya sendiri.

**Keunggulan utamanya adalah komposabilitas dan mutu rekayasa**, bukan fitur pengguna akhir. Provider model, filesystem, shell, sandbox, subagent, sampai mesin workflow semuanya berdiri di atas *capability seam* (definisi layanan + penyedia + konsumen) sehingga mengganti satu penyedia memindahkan seluruh produk — misalnya menukar penyedia filesystem dan subprocess ke sandbox jarak jauh langsung memindahkan Bash, terminal, dan LSP ke sana tanpa mengubah alat mana pun.

**Kondisi saat ini memang "untuk super admin", bukan untuk pengguna awam.** Server Web mengikat `127.0.0.1`, tanpa TLS, tanpa autentikasi, tanpa konsep akun; identitas pengguna bersifat anonim; kunci API disimpan di berkas home pengguna; agen berjalan dengan hak penuh milik akun sistem yang menjalankannya. Rinciannya di [dokumen 04](04-status-super-admin.md).

**Jalan menuju app builder Android ada, dan jalurnya sudah setengah dibangun.** Yang sudah tersedia: eksekusi perintah, terminal persisten, jobs latar belakang, sandbox, workspace, plan mode, goal, deliverables (berkas hasil kerja yang bisa diklik), preset agen per sesi, dan POC sandbox jarak jauh E2B. Yang belum ada sama sekali: rantai alat Android (JDK/SDK/Gradle), pratinjau perangkat, penandatanganan APK, multi-user dengan autentikasi, dan kuota. [Dokumen 05](05-roadmap-app-builder.md) memetakan keduanya menjadi enam fase.

## Cara membaca

Kalau tujuannya memahami produk secepatnya, baca [01](01-apa-ini.md) lalu [03](03-killer-features.md). Kalau tujuannya mulai membangun di atasnya, baca [02](02-bedah-arsitektur.md) sampai habis, lalu [05](05-roadmap-app-builder.md). Kalau tujuannya menentukan kelayakan bisnis, baca [04](04-status-super-admin.md) dan bagian risiko di [05](05-roadmap-app-builder.md).

Dokumen resmi bawaan repositori tetap menjadi sumber kebenaran teknis: [docs/architecture.md](../docs/architecture.md), [docs/capability-seams.md](../docs/capability-seams.md), [docs/agent-lifecycle.md](../docs/agent-lifecycle.md), [docs/tool-catalog.md](../docs/tool-catalog.md), dan [AGENTS.md](../AGENTS.md).
