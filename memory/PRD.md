# PRD — SIPRO (Property Development OS)

> Repo asal: `github.com/djskdjdd/sipro`. Dokumen ini adalah ringkasan produk untuk sesi
> lanjutan. SSOT teknis tetap: `CODEBASE_MAP.md`, `plan.md`, `docs/v2/*`, `test_result.md`.

## 1) Pernyataan masalah (dari pengguna)

"Lanjutkan development repo ini — sebelumnya berhenti di sini." Development terhenti tepat
saat memasang `CancellationPanel` ke layar Kontrak & Legal (Fase 56C, *Pembatalan kontrak &
refund berjurnal*). Pilihan pengguna: **(1)** selesaikan pembatalan kontrak (backend +
frontend) lebih dahulu, **(2)** lanjut sesuai backlog `plan.md`, **(3)** tanpa integrasi
pihak ketiga baru, **(4)** fokus fungsionalitas **dan** rapikan UI.

## 2) Arsitektur & pekerjaan pemulihan lingkungan

* React (CRA + Tailwind + shadcn/ui, alias `@/`) · FastAPI (router modular, semua di `/api`)
  · MongoDB (Motor) · APScheduler untuk pekerjaan berjadwal.
* Pemulihan di container baru: `backend/.env` disusun ulang (`JWT_SECRET`, `DEFAULT_ORG_ID`,
  `PORTAL_MASTER_OTP`), dependensi dipasang (`emergentintegrations` dilewati karena bentrok
  dengan wheel `litellm` bawaan image), seed Fase 16..56 jalan pada DB bersih.
* Disiplin repo yang WAJIB dipatuhi penerus: setiap fitur punya **POC isolasi**, **gate**
  di `scripts/run_all_gates.sh`, dan **uji-mutasi** yang membuktikan gate itu bergigi.

## 3) Persona pemakai (statis)

| Persona | Akun demo | Yang ia kerjakan |
|---|---|---|
| Direksi / Owner | `owner@`, `superadmin@` | angka perusahaan, persetujuan besar, nomor dokumen |
| Manajer Sales | `manager@` | lead, reservasi, konversi pembeli, **mengajukan pembatalan** |
| Sales | `sales@`, `sales2@` | lead miliknya sendiri (lingkup data dijaga) |
| Manajer Keuangan | `finlead@` | **memutuskan pembatalan**, mengabaikan penahanan refund, laporan |
| Keuangan (kasir) | `finance@` | penerimaan, pembayaran, **membayar refund** |
| Proyek / Lapangan | `pm@`, `site@` | progres bangunan, upah harian, BAST |
| Pembeli | portal OTP | jadwal bayar, bukti, progres, garansi, **pembatalan & refund** |

## 4) Kebutuhan inti (statis)

1. Uang hanya boleh bergerak lewat **jurnal berimbang**; buku besar dan subledger wajib
   sama (tie-out).
2. **Pemisahan tugas**: yang mengajukan ≠ yang memutuskan ≠ yang membayar.
3. **Jujur soal ketidaktahuan**: "belum ada data" adalah KALIMAT beserta sebabnya, bukan
   `Rp 0`, bukan tabel hampa, bukan tombol mati tanpa penjelasan.
4. **SSOT**: label & pilihan dari Kamus Data; ambang/persentase dari Pusat Konfigurasi
   (bisa diperbaiki admin tanpa deploy).
5. **Pembeli berhak membaca angkanya sendiri** di portal — angka yang SAMA dengan pembukuan,
   tanpa istilah/nomor akun internal.
6. Fitur yang tidak bisa dilihat manusia di layar dianggap **tidak ada**.

## 5) Yang dikerjakan pada sesi ini (26 Agustus 2026) — Fase 58

**58A — cacat tempat development berhenti (gate `verify_data_integrity` MERAH).**
`cancellation_engine._release_unit` melepas unit ke stok tetapi membiarkan
`sold_by_deal`/`sold_at`, sehingga rumah berstatus `available` tetap mengaku "terjual" ke
setiap pembaca data — dan `build_engine._buyer_binding` mengikatnya ULANG ke pembeli yang
justru mundur. Diperbaiki di mesin, ditambah pembersih data lama idempoten
(`seed_phase56.repair_stale_sold_links`). Dijaga gate 47 (K10b2/K10b3/**D14b memeriksa
seluruh stok**) + `mutasi_56.py` 49 mutan / 49 TERTANGKAP.

**58B — fitur: toleransi keterlambatan & denda keterlambatan BERJURNAL** (P0 no. 1 pada
backlog sebelumnya; utang yang diakui sendiri oleh layar Rencana Bayar):

* `backend/late_fee_engine.py` — SATU mesin: kebijakan dari Pusat Konfigurasi
  (`payment.late.grace_days/rate_pct_month/max_pct_of_term/min_charge`), tenggang milik TERMIN
  menang atas bawaan, keadaan baru **`dalam_tenggang`** (lewat tanggal ≠ menunggak), denda
  prorata sesudah tenggang dengan batas atas & minimum.
* **Denda berjurnal**: Dr `1-1300` / Cr **`4-1400` (akun baru)**, idempoten per (termin,
  bulan); yang ditagihkan adalah SELISIH dengan yang sudah ditagihkan/diringankan.
* **Keringanan** hanya Manajer Keuangan (`late_fee:override`), wajib alasan ≥10 huruf,
  membalik jurnal, dan tidak bisa dianulir bawahan dengan menagihkan denda yang sama lagi.
* **Tidak ada mesin kedua**: tombol "Denda" lama, daftar penagihan, dan konfigurasi penagihan
  (`finance_reports`) dilimpahkan ke mesin ini; `compute_scheme_items` akhirnya membawa
  `grace_days` ke jadwal tagihan (sebelumnya toleransi hilang di titik ini).
* **Layar**: panel *Toleransi keterlambatan & denda* pada tab Rencana Bayar (keadaan per
  termin dari server, sebab denda belum bisa ditagihkan, tagihkan, keringanan berdialog) +
  kartu toleransi & denda di **portal pembeli** (angka sama dengan pembukuan, tanpa nomor
  akun GL). Klaim "belum dibangun" pada layar dicabut karena sudah tidak benar.
* **Guardrail**: gate 49 `scripts/verify_late_fee.py` (**67 pemeriksaan**) terdaftar di
  `run_all_gates.sh`; `scripts/mutasi_58.py` (**31 mutan**).
* Dokumen: `docs/v2/51_LATE_FEE_TOLERANCE_SPEC.md`, bagian FASE 58 di `CODEBASE_MAP.md`.

## 6) Backlog berprioritas (sesudah Fase 58)

**P0**
1. **Pembatalan sepihak karena tunggakan** (`payment.staged.arrears_months_to_cancel` = 2
   bulan): pasalnya sudah tercetak di SPR, mesinnya belum menyambung ke Fase 56.
2. Alur **pembatalan setelah AJB/BAST** — kini ditolak jujur; perlu keputusan bisnis apakah
   dibuatkan jalur notaris.

**P1**
3. **Denda otomatis terjadwal** + pengingat WhatsApp saat termin masuk/keluar masa toleransi
   (sengaja belum: menagih otomatis adalah keputusan bisnis, bukan bawaan).
4. **Laporan denda & keringanan** (siapa meringankan apa, berapa, kapan) untuk rapat direksi.
5. Penjualan ulang unit yang batal + pemicu otomatis pelunasan refund saat unit terjual lagi.
6. Laporan **utang refund** (`2-1460`) tersendiri & arus kas proyeksi.

**P2**
7. Ekspor daftar pembatalan/denda (CSV/PDF) untuk rapat direksi.
8. Riwayat pembatalan & denda pada profil pembeli sebagai jejak lintas kontrak.

## 7) Tugas berikutnya (untuk sesi lanjutan)

1. Baca `plan.md` + `CODEBASE_MAP.md` bagian **FASE 58** dan
   `docs/v2/51_LATE_FEE_TOLERANCE_SPEC.md` lebih dahulu.
2. Jalankan `bash scripts/run_all_gates.sh` **sebelum** menyentuh satu baris pun (harus
   OVERALL PASS 49 gates). Bila backend baru saja di-restart, tunggu `/api/health` hijau —
   gate runtime yang berjalan saat startup akan merah tanpa sebab.
3. Ambil P0 no. 1 dengan pola yang sama: reproduksi/POC → mesin → layar → gate → uji-mutasi →
   dokumentasi.


## Fase 59 (26 Agustus 2026) — tiga utang Fase 58 dibayar
1. **Laporan keringanan denda** untuk rapat direksi: siapa meringankan apa, berapa, kapan,
   dengan alasan tertulis + rekap per pemberi keputusan; ekspor CSV & PDF. Dibuka dari tab
   "Riwayat keringanan" pada panel denda (Rencana Bayar) dan tab "Keringanan Denda" di
   Keuangan — satu komponen, satu angka.
2. **Kandidat pembatalan karena tunggakan**: bulan tunggakan dihitung akumulatif DAN
   berurutan sesuai SPR, ambang dari Pusat Konfigurasi. Sistem MENGUSULKAN dan menitipkan
   tugas peninjauan kepada Manajer Keuangan (idempoten per bulan, plus job harian) —
   pembatalan tetap diajukan Manajer Sales & diputus Manajer Keuangan (SoD Fase 56 utuh).
3. **Laporan utang refund `2-1460`**: jatuh tempo = keputusan + `cancellation.refund_due_days`,
   bucket umur, proyeksi kas 6 bulan, refund yang tertahan ketentuan SPR TIDAK diberi tanggal
   karangan, dan laporannya diuji cocok dengan saldo buku besar.

Guardrail: `scripts/verify_p59.py` (gate 50, 53 pemeriksaan). Status: `run_all_gates.sh`
OVERALL PASS (50 gates); testing agent iterasi 95 → backend 17/17, frontend bersih, 0 isu.

### Backlog berikutnya
- P1: `scripts/mutasi_59.py` (uji mutasi untuk ketiga fitur Fase 59).
- P1: denda otomatis terjadwal + pengingat WhatsApp untuk tunggakan mendekati batas.
- P2: laporan keringanan per proyek/periode di Analitik & BI.
