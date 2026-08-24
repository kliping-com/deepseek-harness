# 01 — Ini framework AI agent apa?

## Jawaban singkat

**Kliping (nama asal: DeepSeek Harness, perintah CLI `dsh`) adalah *agent harness* berbasis plugin.** Bukan pustaka orkestrasi seperti LangChain, bukan aplikasi jadi seperti Emergent, bukan hanya CLI seperti Claude Code — melainkan mesin lengkap untuk menjalankan agen pemrogram, yang setiap bagiannya dapat dibongkar-pasang dari berkas konfigurasi.

Definisi resminya ada di [README.md](../README.md): *"an open-source agent harness ... architecture where everything is a plugin, powered by Cordis"*.

## Tiga lapisan yang perlu dibedakan

Kebingungan paling umum saat membaca repositori ini adalah menyamakan tiga hal yang sebetulnya terpisah rapi.

**Lapisan 1 — Cordis (kerangka plugin).** Ada di [vendor/](../vendor/README.md), disalin-pin dari proyek [Cordis](https://github.com/cordiverse/cordis). Cordis menyediakan konteks berbagi (`ctx`), layanan bernama (`ctx.<key>`), event bertipe, dan *effect* yang bisa dibatalkan. Semua registrasi adalah efek: saat sebuah plugin dilepas, semua kontribusinya ikut tercabut. Ini bukan kode buatan repositori ini; ini fondasi yang dipinjam. Pengantarnya: [docs/cordis-primer.md](../docs/cordis-primer.md).

**Lapisan 2 — Harness (isi `packages/`).** Inilah produknya: sesi, prompt sistem, registri alat, agen, agent loop, penyedia LLM, seluruh katalog alat, persistensi, sandbox, subagent, workflow, dan seterusnya. Lebih dari lima puluh grup paket, semuanya plugin Cordis. Peta grupnya ada di [packages/README.md](../packages/README.md).

**Lapisan 3 — Aplikasi (isi `apps/` dan `examples/`).** `apps/cli` adalah peluncur `dsh` (memilih profil lalu mem-boot pohon plugin), `apps/web` adalah shell Vite untuk GUI browser. Aplikasi hanyalah komposisi; tidak ada logika produk yang eksklusif di sini.

Konsekuensi praktisnya: **menambah kemampuan tidak pernah berarti mengubah inti.** Tidak ada "core" yang perlu ditambal — alat baru, provider baru, bahkan agent loop pengganti, semuanya dipasang sebagai baris plugin di sebelah yang lain.

## Model mental: satu proses, satu pohon plugin

Saat `dsh web` dijalankan, yang terjadi adalah:

1. Peluncur membaca **profil** (`web` atau `headless`) dari direktori home Harness.
2. Profil menyebutkan **bundle** yang ditumpuk berurutan: `dsh-base` (model, alat, persistensi, sandbox, kebijakan persetujuan, setelan, kredensial, telemetri), lalu `dsh-web-app` (aplikasi browser) atau `dsh-headless` (pelari sekali jalan).
3. Setiap bundle menyumbang baris konfigurasi Cordis; lapisan berikutnya boleh menambal baris mana pun berdasarkan id-nya — `cordis.patch.yml` milik profil, lalu milik home, lalu overlay `--patch`.
4. Pohon hasil komposisi itulah yang di-boot. `dsh --profile web --dump-config` mencetaknya tanpa menjalankan.

Artinya konfigurasi produk ini bukan sekadar berkas setelan, melainkan **daftar plugin yang aktif beserta konfigurasinya** — bisa dilihat, ditambal, dan diganti seluruhnya.

## Siklus kerjanya

Satu **turn** adalah satu penarikan masukan yang diterima; satu **step** adalah satu permintaan ke model plus alat yang dipanggilnya. Alurnya (versi ringkas dari [docs/architecture.md](../docs/architecture.md)):

```text
turn/start
  klaim masukan + satu pesan antrean
  rakit bagian prompt + skema alat
  -> agent/pre-step        (waterfall: boleh menolak atau menulis ulang pesan)
     step/start
     turunkan riwayat model dari session log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     masih ada utang alat / masukan baru -> step berikutnya
  -> agent/turn-stopping
turn/end
```

Yang membuatnya berbeda dari kebanyakan kerangka agen: **titik-titik itu semuanya adalah extension point publik**. `agent/pre-step`, `agent/request`, `llm/stream`, dan tiga event `tools/*` adalah *waterfall* — pendengar wajib memanggil `next()` untuk meneruskan, atau memutus rantai untuk mengambil alih. Kebijakan seperti mode rencana, penjaga perulangan, tenggat waktu alat, persetujuan pengguna, dan kompaksi konteks semuanya adalah plugin yang menempel di sini, bukan cabang `if` di dalam loop.

## Aturan yang membentuk seluruh kode

Tiga aturan berikut menjelaskan hampir semua keputusan desain di repositori ini.

**"Model-visible ⟺ logged".** Apa pun yang sampai ke permintaan model harus dapat direkonstruksi dari session log. Menambah masukan baru yang dilihat model berarti menambah event sesi baru, bukan menyisipkan teks di tengah jalan. Inilah yang membuat fork sesi, resume, replay, transkrip, dan telemetri bisa diturunkan dari satu sumber.

**"A capability seam comprises Service Definition / Service Provider / Consumer".** Satu kemampuan selalu terdiri dari tiga peran: kelas layanan abstrak yang memiliki `ctx.<key>`, satu atau lebih penyedia, dan konsumen (biasanya alat yang dilihat model). Contoh kanonnya `packages/shell`: `dsh-shell` (definisi), `dsh-bash-local` / `dsh-bash-sandbox` (penyedia), `dsh-tool-bash` (konsumen). Rinciannya di [docs/capability-seams.md](../docs/capability-seams.md).

**"Plugins, not loop changes".** Perilaku baru menempel di extension point yang terdokumentasi; mengubah `agent-loop` mengharuskan pembaruan dokumen arsitektur. Ini yang menjaga agar percabangan produk tidak menumpuk di satu berkas.

## Posisi terhadap produk lain

| Produk | Bentuk | Perbedaan inti dengan Kliping |
|---|---|---|
| LangChain / LangGraph | Pustaka orkestrasi | Menyediakan primitif graf; tidak membawa alat pemrogram, sandbox, GUI, persistensi sesi, atau kebijakan izin |
| Claude Code / Codex CLI | Produk CLI tertutup | Pengalaman siap pakai, tetapi komposisinya tidak terbuka; di sini justru mereka bisa dipasang **sebagai subagent** (`packages/subagent/subagent-claude-code`, `subagent-codex`) |
| OpenHands / Devin | Agen pemrogram | Fokus pada satu produk agen; di sini inti produknya adalah kerangka komposisi yang bisa melahirkan banyak agen berbeda per sesi |
| Emergent / Lovable / Bolt | App builder untuk pengguna awam | Menjual hasil (aplikasi jadi) dengan alur terpandu; di sini yang tersedia adalah bahan bakunya — belum ada rantai alat build aplikasi, pratinjau, akun, maupun kuota |

Kesimpulan posisi: **Kliping hari ini adalah "Claude Code yang bisa dibongkar", bukan "Emergent".** Menjadikannya Emergent-untuk-Android adalah pekerjaan produk di atas fondasi ini, dan itulah isi [dokumen 05](05-roadmap-app-builder.md).

## Untuk siapa repositori ini ditulis

Perlu dicatat karena mempengaruhi ekspektasi: hampir seluruh dokumentasi internal ditujukan untuk **agen AI yang mengerjakan repositori ini**, bukan untuk pengguna akhir. Lihat [AGENTS.md](../AGENTS.md) — berisi standing order tentang konvensi, gerbang mutu, dan cara menulis catatan keputusan. Dokumen pengguna hanya ada empat halaman di [docs/user/](../docs/user/index.md). Rasio itu sendiri adalah sinyal kuat tentang tahap kematangan produk.
