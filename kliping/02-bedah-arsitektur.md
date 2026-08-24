# 02 — Bedah arsitektur

Dokumen ini membedah repositori lapis demi lapis. Tujuannya supaya siapa pun yang akan membangun produk di atasnya tahu persis di mana harus menempel, dan di mana jangan menyentuh.

## Peta direktori

```text
vendor/      Salinan-pin sumber Cordis (kerangka plugin). Jangan diedit langsung; ada prosedur sinkronisasi.
packages/    Lebih dari 50 grup paket npm @deepseek-ai/dsh-*. Inilah produknya.
apps/cli     Peluncur `dsh`: mem-parse flag miliknya sendiri, memilih profil, mem-boot pohon plugin.
apps/web     Shell Vite untuk GUI browser. Bukan aplikasi mandiri: butuh window.__DSH_BOOT__ dari `dsh web`.
examples/    Berkas cordis.yml yang bisa dijalankan di atas bundle demo.
python/      SDK Python plus runtime yang dibundel.
native/      Addon Node untuk Landlock (pengurungan proses di Linux).
docs/        Arsitektur, katalog tergenerasi, buku resep, postmortem — dwibahasa Inggris/Mandarin.
.agents/     Alur kerja agen dan Agent Notes (catatan keputusan).
scripts/     Gerbang mutu dan generator.
website/     Proyeksi VitePress atas sebagian docs/.
```

Grup paket dikelompokkan per kemampuan, bukan per lapisan teknis: `core/`, `llm/`, `shell/`, `fs/`, `terminal/`, `sandbox/`, `subagent/`, `workflow/`, `session/`, `client/`, `host/`, `api/`, dan seterusnya. Tabel lengkap beserta ekspektasi rilis tiap grup ada di [packages/README.md](../packages/README.md).

## Boot: profil, bundle, dan patch

Tidak ada berkas konfigurasi tunggal. Yang ada adalah **tumpukan lapisan** yang menghasilkan satu pohon plugin.

- **Bundle** — format distribusi berisi baris konfigurasi Cordis plus kode yang dipasangnya. Yang dikapalkan: `dsh-base`, `dsh-web-app`, `dsh-headless` (lihat [packages/bundle/](../packages/bundle/README.md)).
- **Profil** — komposisi bernama yang tersimpan di home Harness; menyebut bundle yang ditumpuk, menyimpan plugin luar-pohon yang dipasang pengguna, dan `cordis.patch.yml` milik pengguna.
- **Urutan pelapisan** — tiap bundle sesuai urutan profil, lalu `cordis.patch.yml` profil, lalu milik home, lalu overlay `--patch`. Satu patch menyasar baris berdasarkan id dan mengganti seluruh konfigurasinya, atau menyisipkan baris baru.

Perintah `dsh --profile web --dump-config` mencetak pohon yang benar-benar akan di-boot mesin ini. Untuk membangun produk turunan, inilah mekanisme utamanya: **buat bundle sendiri, jangan fork inti**.

## Tulang punggung produk (`packages/core/`)

| Paket | Milik | Kunci `ctx` |
|---|---|---|
| `core/session` | Log `SessionEvent` yang hanya bisa ditambah, plus penyimpanan dalam memori | `ctx.sessions` |
| `core/system-prompt` | Perakitan bagian prompt dan skema alat | `ctx.systemPrompt` |
| `core/tools` | Registri alat berlingkup dan pipeline eksekusi berpenjaga | `ctx.tools` |
| `core/agent` | Antarmuka `Agent`, registri hidup, event `agent/*` | `ctx.agents` |
| `core/agent-loop` | Driver bawaan yang mengimplementasikan antarmuka itu | `ctx.agentLoop` |
| `core/scope` | Primitif registrasi berlingkup per agen | pustaka |
| `llm/llm` | Kosakata pesan dan stream plus sambungan adapter | `ctx.llm` |

Perhatikan pemisahan `core/agent` dan `core/agent-loop`: antarmuka agen dan implementasi loop-nya adalah dua paket berbeda, sehingga **loop bawaan pun bisa diganti** tanpa menyentuh plugin UI, alat, atau hook yang hanya bergantung pada antarmukanya.

## Session log: satu sumber kebenaran

Session log adalah sumber konteks yang dilihat model. `deriveMessages()` memproyeksikan riwayat model dari log; event `assistant/chunk` mentah dipertahankan supaya replay dan tampilan UI setia pada aslinya. Fork, resume, transkrip, telemetri, dan persistensi semuanya diturunkan dari aliran yang sama.

Aturan yang mengikat: **apa pun yang dilihat model harus ada di log**, dan ada runtime invariant yang menegakkannya. Anggota `SessionEventMap` bersifat wajib-saat-baca secara bawaan — build yang tidak mengenali tipe suatu event akan menolak log tersebut, kecuali event itu membawa penanda `ignorable: true`.

Persistensi punya sambungannya sendiri: `session-persistence` (definisi) dengan penyedia `session-persistence-jsonl` dan `session-persistence-sqlite`, ditambah proyeksi, cache proyeksi, statistik, telemetri, dan penghasil judul sesi berbasis LLM. Untuk pencarian, `session-query` menyediakan korpus logis, pembacaan berbatas, silsilah, dan pencarian teks penuh berbasis SQLite.

## Capability seam: alasan produk ini bisa berpindah dunia

Satu *seam* terdiri dari tiga peran: **Service Definition** (kelas Cordis yang memiliki `ctx.<key>`), **Service Provider**, dan **Consumer**. Yang penting dipahami: seam adalah kemampuan utuh, bukan satu peran saja.

Seam yang sudah ada, beserta yang menariknya untuk pengembangan produk:

| Seam | Definisi | Penyedia yang ada | Konsumen |
|---|---|---|---|
| LLM | `ctx.llm` | DeepSeek, plus katalog provider lain dan endpoint OpenAI-compatible | agent loop |
| Shell | `ctx.shell` | `bash-local`, `bash-sandbox`, `pwsh` | `tool-bash`, `tool-pwsh` |
| Subprocess | `ctx.subprocess` | proses lokal (pohon proses), `subprocess-e2b` | shell, terminal, LSP |
| Filesystem | `ctx.fs` | lokal, `fs-e2b` | `tool-fs`, `tool-fs-search`, `str-replace-editor` |
| Terminal | `ctx.terminals` | PTY lokal | `tool-terminal` (buka/baca/kirim/sinyal/tutup) |
| Sandbox | `ctx.sandbox` | `sandbox-local` (bwrap/Landlock/Seatbelt), `sandbox-windows-acl` | pembungkus argv sebelum spawn |
| Code runtime | — | worker thread, CPython | `run_code` (Code Mode) |
| Web | — | pencarian dan fetch | `web_search`, `web_fetch` |
| LSP | `ctx.lsp` | penyedia stdio generik | `tool-lsp` |
| Skill | — | `skill-filesystem` | `tool-skill` (katalog + pemuat) |
| Subagent | — | `fork-in-process`, `spawn-in-process`, `dsh-sdk`, `acp`, `claude-code`, `codex` | `tool-subagent`, kontrol, laporan |
| Workflow | — | worker thread | `workflow`, `ralph` |
| Compaction | — | penyedia dasar | perintah manusia |
| Spill | — | penyimpanan lokal | kebijakan luapan hasil alat |
| Settings | `ctx.settings` | berkas | GUI setelan |
| Credentials | — | env di atas `.env` | alur otorisasi |
| Storage | — | backend non-sesi | domain penyimpanan |

Nilai praktisnya: filesystem dan subprocess berbagi satu "dunia eksekusi". Mengarahkan keduanya ke sandbox jarak jauh **memindahkan Bash, PTY, dan LSP sekaligus**, tanpa mem-fork satu pun alat. Inilah yang membuat rencana build farm di [dokumen 05](05-roadmap-app-builder.md) realistis.

## Alat yang dilihat model

Katalog tergenerasi ada di [docs/tool-catalog.md](../docs/tool-catalog.md). Ringkasan kelompoknya:

- **Berkas**: `read`, `write`, `edit`, `read_image`, `str_replace_editor`, `glob`, `grep`
- **Eksekusi**: `bash`, `pwsh`, versi persisten keduanya, `terminal_*` (enam alat PTY), `run_code`
- **Pengetahuan**: `web_search`, `web_fetch`, `lsp`, `skill`, `session_search` dan kerabatnya
- **Koordinasi**: `subagent`, `list_agents`, `send_message`, `interrupt_agent`, `report`, `workflow`, `ralph`
- **Manajemen kerja**: `todo_write`, `create_goal` / `get_goal` / `update_goal`, `schedule_*`, `job_list` / `job_output` / `job_kill`, `exit_plan_mode`
- **Refleksi diri**: `cordis_define`, `cordis_run`, `cordis_stop`, `cordis_undefine`, `cordis_inspect_*` — agen memeriksa dan memodifikasi pohon plugin-nya sendiri saat berjalan
- **Interaksi**: `ask_user_question`

Setiap alat mendeklarasikan **niat render UI**-nya di muka (`generic` / `terminal` / `diff`, plus `locations`), dan metode presentasinya adalah fungsi murni dari argumen. Karena itu GUI bisa menampilkan kartu diff, blok terminal, dan tautan berkas tanpa mengenal alat tertentu satu per satu.

## Lapisan Web: dua paruh dan satu gerbang bertipe

`packages/host/` adalah paruh Node (server HTTP, rute, proxy API, pemilih direktori, inventaris plugin), `packages/client/` adalah paruh browser (shell React, koneksi, slot UI, dan puluhan plugin `ui-*`).

Dua mekanisme yang layak dicatat:

**Typert — gerbang RPC bertipe.** Metode layanan bisnis ditandai `@Remote`; saat build, generator menganalisis tanda tangan TypeScript-nya dan menghasilkan deskriptor sisi Host dan kontrak sisi Client, lengkap dengan kodek dan validasi. Klien memanggil `ctx.remote.<namespace>.<method>()` sebagai fungsi konkret, bukan proxy. Rinciannya: [docs/api-gateway.md](../docs/api-gateway.md).

**Slot — sistem penempatan UI.** Shell mendeklarasikan lubang bernama (`sidebar.brand.mark`, `conversation.hero.brand.mark`, `sidebar.workspaces`, `conversation.chat.turnTail`, dan seterusnya), lalu paket lain mengisinya lewat `slots.inject()` yang sadar-deklarasi. Registrasinya reversibel dan tahan HMR. Rebranding pada [dokumen 06](06-rebranding.md) memakai persis mekanisme ini.

## Kolaborasi manusia

- **Permission preset** (`packages/interaction/permission-presets`): bawaan `workspace-write` (menulis di dalam workspace dan direktori sementara yang diizinkan; lebih luas dari itu butuh persetujuan) dan `danger-full-access` (akses berkas penuh tanpa dialog).
- **Persetujuan dan pertanyaan** (`user-approval`, `user-questions`, `tool-ask-user`): agen bisa meminta keputusan manusia di tengah kerja.
- **Perintah manusia** (`ctx.commands`): instruksi berawalan garis miring yang dieksekusi tanpa giliran model.
- **Plan mode** (`packages/plan`): mode rencana sebagai keadaan tercatat, dengan pintu keluar yang ditinjau.
- **Agent preset** (`packages/preset`): komposisi plugin per sesi. Yang dikapalkan: Standard mode, PTC mode (alat diekspos lewat Code Mode SDK sehingga model menggabungkan operasi multi-langkah dalam satu program TypeScript), Minimal mode, dan Creator mode (untuk membuat preset baru, termasuk oleh agen itu sendiri).

## Gerbang mutu

Repositori ini menjalankan disiplin yang jauh di atas rata-rata proyek sejenis, dan itu relevan untuk penilaian kelayakan: fondasinya rapi, jadi ongkos membangun di atasnya lebih rendah.

| Perintah | Yang dijaga |
|---|---|
| `pnpm run test` | Uji unit Vitest |
| `pnpm run test:coverage` | Gerbang cakupan CI: **100% per berkas** di `packages/*/*/src` |
| `pnpm run test:e2e` | Uji API sungguhan; melewatkan diri tanpa `DEEPSEEK_API_KEY` |
| `pnpm run test:snapshot` | Replay ACP/headless tanpa kunci, dibandingkan dengan keluaran terekam |
| `pnpm run typecheck` | `tsc -b` untuk muka Host lalu Client |
| `pnpm run lint`, `duplication` | oxlint dan deteksi klon lintas berkas |
| `pnpm run hygiene` | knip, publint, batasan workspace, pemeriksaan konsumen NodeNext |
| `pnpm run doc-sync` | Semua gerbang dokumentasi: tautan, pembungkusan baris, anggaran kata, pasangan dwibahasa, kesegaran katalog |
| `pnpm run build` | `tsc` mengeluarkan lib/types, tsdown membundel runtime |

Ditambah aturan proses: setiap perubahan non-trivial wajib membawa **Agent Note** (catatan keputusan) di PR yang sama, dan setiap dokumen dalam cakupan wajib punya pasangan Mandarin plus catatan konsistensi ber-hash.

## Yang sebaiknya tidak disentuh

- **`vendor/`** — salinan-pin Cordis. Perbarui lewat prosedur sinkronisasi di [vendor/README.md](../vendor/README.md), bukan dengan mengedit di tempat.
- **`core/agent-loop`** — mengubahnya mengharuskan pembaruan [docs/architecture.md](../docs/architecture.md), dan hampir selalu ada extension point yang lebih tepat.
- **Format on-disk** — `SESSION_FORMAT_VERSION` dan `SCHEMA_VERSION` SQLite punya aturan sendiri; jangan menambah kompatibilitas diam-diam.
- **Katalog tergenerasi** (`docs/tool-catalog.md`, `docs/config-catalog.md`, `docs/module-graph.md`, `slot-catalog.ts`) — hasil generator, disegarkan lewat `pnpm run gen-*`, bukan diedit tangan.
