# SIPRO — Property Development OS (PRD)

Aplikasi manajemen properti & konstruksi (React + FastAPI + MongoDB) dengan RBAC ketat,
keuangan/GL, konstruksi berbukti, portal pembeli, dan dokumen PDF ber-kop.
Bahasa produk & komunikasi: **Indonesia**.

## Aturan kerja yang tidak boleh dilanggar
- `bash scripts/run_all_gates.sh` adalah nyawa proyek. Semua gate harus PASS (sekarang **52 gate**).
- Batas ukuran berkas: Python < 800 baris, JS < 500 baris (`validate_compliance.py`).
- Form: tidak boleh `<Input>` bebas untuk nilai enum/relasi (`audit_forms_deep.py`); setiap
  `<Input>` wajib punya label/placeholder/aria-label.
- Kosakata enum hanya dari SSOT `/api/reference` (`reference_groups.py` + `reference_p<NN>.py`).
- Kredensial uji: `/app/memory/test_credentials.md` (sandi demo `Sipro#2026`).

## Riwayat implementasi (terbaru di atas)
### 27 Jun 2026 — Fase 61: cetak SPK & PO (SELESAI, gate 52 hijau)
- `backend/docgen_p61.py`: isi SPK (identitas pihak, nilai kontrak, retensi, masa
  pemeliharaan, rincian lingkup dari `spk_scope_items`, 5 ketentuan) & PO (penyedia, jenis,
  jatuh tempo, rincian item + total, 4 ketentuan). Dokumen berstatus `draft` DIPAKSA
  bertanda watermark DRAFT. Nama pihak kedua = subkontraktor/vendor (bukan "Pemesan").
- `pdf_layout.render_letter(..., item_table=...)` + helper `_grid` (dipakai bersama laporan).
- Endpoint: `GET /api/subcon/spk/{id}/pdf`, `GET /api/procurement/pos/{id}/pdf`.
- UI: `patterns/PrintDocButton.js` dipakai di `SPKDetailSheet` & `PODetailSheet`
  (testId `spk-print-pdf`, `po-print-pdf`).
- Target layout baru di Pusat Konfigurasi Dokumen: `SPK`, `PO`.
- Gate baru `scripts/verify_p61.py` (24 pemeriksaan). Uji UI: iteration_97 (bersih).
- PDF diperiksa visual (render PNG): kop, rincian, ketentuan, dua kolom tanda tangan OK.

### 27 Jun 2026 — Fase 60: konfigurasi tampilan dokumen (SELESAI, gate 51 hijau)
- Panel `Master Data → Template Dokumen → Tampilan & kop surat` (`DocLayoutPanel`) dengan
  pratinjau PDF BERDAMPINGAN yang dirender mesin cetak yang sama (`pdf_layout.py`).
- Kop/footer 2 mode (dirakit sistem / gambar desain), watermark, kertas & margin, baris
  biaya (urut, sembunyikan, sembunyikan bila Rp 0, baris manual), tanda tangan dinamis.
- Hak akses ubah = `settings:update` (identitas perusahaan = pengaturan organisasi);
  baca = `documents:view`.
- Bidang usaha jadi dropdown SSOT (`reference_p60.business_field`).
- Jalur cetak yang memakai layout: dokumen staf, **portal pembeli** (diperbaiki), kwitansi,
  penawaran, BAST.
- Gate baru `scripts/verify_p61.py`→(60) `scripts/verify_p60.py` (38 pemeriksaan). UI: iteration_96.
- Perbaikan gate lain: `audit_forms_deep.py` (tagline → dropdown; aria-label RowsForm &
  CostsDialog) dan `verify_analytics.py` (`analytics_engine.rebuild_snapshots` sekarang
  MEMPERBAIKI seluruh riwayat snapshot, bukan hanya hari ini).

### Sebelumnya
- Fase 59: laporan keringanan denda, kandidat tunggakan (2 bulan → usulan pembatalan), utang refund.
- Fase 58: toleransi & keringanan denda keterlambatan.
- Fase ≤57: CRM, kontrak & skema pembayaran, konstruksi berbukti, pengadaan 3-way match,
  subkon/opname/retensi, GL & pajak, portal pembeli, WA/omnichannel, analitik BI.

## Backlog
### P1
- Surat Peringatan Tunggakan (SP1/SP2/SP3) — surat berkop otomatis dari data tunggakan.
- Berita Acara Opname / Punch List PDF.
- Lampiran gambar/spesifikasi pada SPK (berkas dari master proyek).
### P2
- Pengingat WhatsApp untuk pembeli menunggak.
- Peringatan dini tunggakan 1 bulan sebelum batas pembatalan kontrak.
- Ringkasan direksi: email digest laporan keringanan & utang refund setiap awal bulan.
