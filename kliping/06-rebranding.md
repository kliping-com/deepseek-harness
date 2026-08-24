# 06 — Catatan rebranding menjadi Kliping

Dokumen ini mencatat apa yang sudah diganti, mengapa cakupannya dibatasi seperti itu, dan apa langkah berikutnya bila ingin rebranding total.

## Prinsip cakupan

Rebranding dilakukan pada **permukaan yang dilihat manusia dan dilihat model**, bukan pada identitas teknis paket. Alasannya: nama paket npm (`@deepseek-ai/dsh-*`), perintah CLI (`dsh`), direktori home (`$DSH_HOME`), dan nama repositori adalah pengenal teknis yang tersebar di ribuan berkas, lockfile, tsconfig, dan katalog tergenerasi. Menggantinya adalah proyek tersendiri dengan risiko tinggi dan nilai kosmetik rendah pada tahap ini.

Yang diganti karena benar-benar terlihat pengguna atau memasuki permintaan model:

## Yang sudah diganti

**Baris merek di sidebar.** Sebelumnya lambang paus DeepSeek plus label `DSH Local Build`; sekarang teks `Kliping` tanpa lambang, tetap dengan lencana revisi build tujuh karakter supaya build lokal tidak tertukar dengan rilis. Berkas: `packages/client/ui-sidebar/src/client/SidebarRoot.tsx` dan `SidebarRoot.module.css`.

**Judul hero pada sesi kosong.** Sebelumnya lambang paus plus `Into the Unknown` (Mandarin: `探索未至之境`); sekarang `Kliping` dengan lencana `Preview`, tanpa lambang. Berkas: `packages/client/ui-conversation/src/client/locales.ts` dan `skeleton/EmptyHero.tsx`.

**Rel sidebar saat menyempit.** Dulu tombol beristirahat sebagai lambang paus dan berganti ikon panel saat disentuh kursor. Karena produk tidak lagi mengapalkan lambang, tombol itu kini selalu menampilkan ikon panel — kalau tidak, tombolnya akan tak kasatmata.

**Judul jendela peramban dan metadata pemasangan.** `Kliping` di `apps/web/index.html`, `apps/web/vite.config.ts`, `packages/client/ui-renderer/src/client/DocumentTitle.tsx`, dan `apps/web/public/manifest.webmanifest` (`name` dan `short_name`).

**Ikon tab.** `apps/web/public/favicon.svg` diganti dari lambang paus menjadi monogram huruf K netral, tetap mengikuti aturan tema terang dan gelap yang diuji.

**Judul produk untuk build resmi.** `scripts/client-build-environment.ts` menetapkan `DSH_CLIENT_TITLE` menjadi `Kliping`.

**Paket merek resmi.** `packages/client/ui-brand-official` sekarang hanya mengisi slot `sidebar.brand.name` dengan teks produk, dan tidak lagi mengisi slot lambang mana pun. Ketergantungannya pada paket percakapan dan primitif ikut dicabut.

**Karya seni merek DeepSeek dihapus dari klien Web.** `FishLogo.tsx` dan `BrandWordmark.tsx` dihapus dari `packages/client/ui-primitives`, beserta ekspor dan ujinya.

**Identitas yang dilihat model.** Tiga kalimat prompt sistem kini menyebut Kliping: pembuka identitas di `packages/core/system-prompt`, kalimat lokasi checkout di `packages/boot/app-boot`, dan kalimat konteks GUI Web di `packages/bundle/web-app`. Seluruh keluaran snapshot terekam ikut diperbarui agar tetap konsisten.

**Salinan pemberitahuan sambutan.** `packages/client/ui-settings-models/src/onboarding-copy.ts` menyebut Kliping, dan versi pemberitahuannya dinaikkan supaya pengguna lama melihatnya sekali lagi.

## Yang sengaja dibiarkan

**Nama paket npm dan lingkup `@deepseek-ai`.** Tersebar di seluruh workspace, lockfile, tsconfig, dan katalog tergenerasi.

**Perintah `dsh`, variabel `DSH_*`, dan direktori `$DSH_HOME`.** Pengenal teknis, bukan merek yang dilihat pengguna awam.

**Nama provider model "DeepSeek".** Ini nama vendor API yang sesungguhnya, bukan merek produk. Kartu provider di halaman Setelan memang harus tetap menyebut DeepSeek.

**Dokumentasi internal, JSDoc, dan Agent Notes.** Ratusan berkas dwibahasa; menggantinya menuntut pembaruan pasangan Mandarin plus pencatatan ulang hash konsistensi untuk setiap berkas.

**Nama repositori dan URL GitHub.** Milik pemilik repositori.

**Situs dokumentasi.** `website/public/wordmark.svg` masih memuat wordmark DeepSeek, dan judul navigasinya menyisipkan berkas itu. Situs tersebut menerbitkan korpus dokumentasi hulu yang memang belum di-rebrand, jadi menggantinya sendirian justru menghasilkan campuran yang tidak konsisten.

## Cara melanjutkan ke rebranding total

Bila suatu saat diperlukan, urutan yang paling tidak menyakitkan:

1. **Ganti nama lingkup npm** lebih dulu, dalam satu PR mekanis: `@deepseek-ai/dsh-*` menjadi lingkup baru, perbarui setiap `package.json`, referensi tsconfig, impor, lalu jalankan ulang setiap generator katalog. Repositori sudah menyediakan preseden prosedurnya di [docs/rescope.md](../docs/rescope.md).
2. **Ganti nama perintah dan variabel lingkungan** (`dsh`, `DSH_*`, `$DSH_HOME`) dengan jalur migrasi untuk profil pengguna yang sudah ada.
3. **Sapu dokumentasi** per pasangan bahasa, dan catat ulang setiap berkas `.i18n.yaml`.

Langkah 1 dan 2 sebaiknya menunggu sampai ada rilis bertag pertama, karena sikap pra-rilis repositori saat ini justru menganjurkan penggantian nama bebas tanpa lapisan kompatibilitas.

## Cara mengganti merek tanpa menyentuh kode inti

Untuk penyebaran turunan, tidak perlu mengedit shell sama sekali. Buat satu paket klien yang mengisi slot merek, lalu pasang barisnya lewat patch:

- `sidebar.brand.name` — nama produk di baris merek sidebar.
- `sidebar.brand.mark` — lambang di baris merek. Shell sengaja tidak mengapalkan isinya.
- `conversation.hero.brand.mark` — lambang pada hero sesi kosong.
- `hero.headline` pada kamus lokal paket percakapan — teks judul hero.
- `DSH_CLIENT_TITLE` saat build — judul jendela peramban.

Paket `packages/client/ui-brand-official` adalah contoh lengkapnya, termasuk cara mendaftar yang sadar-deklarasi sehingga urutan aktivasi tidak berpengaruh dan pencabutan bersih saat hot reload.

## Verifikasi yang dijalankan

| Perintah | Hasil |
|---|---|
| Uji unit paket terdampak (sidebar, percakapan, merek, primitif, renderer, setelan model, prompt sistem, app-boot, agent-loop, bundle web) | Lulus |
| `pnpm run typecheck` | Lulus |
| `pnpm run verify-translation-pairing` | 1003 pasangan konsisten |
| `pnpm run gen-client-catalog`, `pnpm run gen-module-graph` | Katalog tergenerasi disegarkan |

Uji end-to-end peramban dan replay snapshot belum dijalankan di lingkungan ini karena keduanya menuntut artefak hasil `pnpm run build` yang lengkap; keluaran terekam sudah diperbarui agar sejalan dengan sumbernya, dan CI memiliki sinyal itu.
