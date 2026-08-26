# Rencana Development SIPRO — Fase 58

> **STATUS PER PEMBARUAN TERAKHIR**
>
> | Bagian | Status |
> |---|---|
> | Pemulihan lingkungan dari repo `sjsjjdbfbd/Sipro` di container baru | **SELESAI** — `backend/.env` dibuat ulang (`JWT_SECRET`, `DEFAULT_ORG_ID`, `PORTAL_MASTER_OTP`), dependensi dipasang, seed jalan di DB bersih, login OK |
> | 58A — Cacat tempat development berhenti: unit di stok yang masih mengaku "terjual" | **SELESAI** — direproduksi, akar masalah ditutup, gate 47 diperkuat (110 pemeriksaan), `mutasi_56.py` 49 mutan / 49 TERTANGKAP |
> | 58B — Fitur: **Toleransi keterlambatan & denda keterlambatan berjurnal** | **SELESAI** — gate 49 `verify_late_fee.py` (67 pemeriksaan), `mutasi_58.py` (31 mutan) |
> | 58C — Dokumentasi & penutupan | **SELESAI** — `docs/v2/51_LATE_FEE_TOLERANCE_SPEC.md`, bagian FASE 58 di `CODEBASE_MAP.md`, `memory/PRD.md` |

---

## 0) Kenapa dev sempat berhenti (dan apa yang sebenarnya terjadi)

Gate `verify_data_integrity.py` merah dengan satu baris: **"unit terjual tanpa ikatan
lead/deal: 1"** pada unit `A-06` yang justru berstatus `available`.

Root cause-nya BUKAN gate yang salah (dugaan pertama yang paling menggoda, karena unitnya
"kan available"): `cancellation_engine._release_unit` melepas unit ke stok dan mengosongkan
`booked_by_deal`, `deal_id`, `lead_id` — tetapi **membiarkan `sold_by_deal` dan `sold_at`**.
Rumah yang sudah dikembalikan ke stok tetap mengaku terjual kepada site plan, invarian
bisnis, dan gate integritas. Lebih jauh: `seed_phase31._fix_unit_defects` → `sync_unit_binding`
membaca tautan basi itu dan **mengikat ulang** unitnya ke pembeli yang justru mundur.

Diperbaiki: pelepasan unit mengosongkan tautan penjualan, `_buyer_binding` menolak mengikat
unit berstatus `available`, dan `seed_phase56.repair_stale_sold_links()` membersihkan basis
data yang sudah pernah menjalankan pembatalan (idempoten).

Pelajaran yang dicatat: **gate yang merah lebih sering benar daripada kode yang merasa benar.**
Reproduksi dulu (`scripts/repro_stale_sold_link.py`), baru perbaiki.

---

## 1) 58A — Cacat pelepasan unit (SELESAI)

* `cancellation_engine._release_unit`: `sold_by_deal`/`sold_at` dikosongkan; unit yang tertaut
  HANYA lewat penjualan tetap bisa dilepas (dulu tidak cocok dengan filter atomiknya).
* `build_engine._buyer_binding`: unit `available` tidak punya pembeli, apa pun sisa tautannya.
* `seed_phase56.repair_stale_sold_links()`: perbaikan data lama, idempoten, jalan saat startup.
* Guardrail: gate 47 K10b2/K10b3/D14b (**D14b memeriksa SELURUH stok**, bukan hanya unit uji)
  + `mutasi_56.py` M14/M47/M48 → 49 mutan, 49 TERTANGKAP / 0 LOLOS.

---

## 2) 58B — Toleransi keterlambatan & denda berjurnal (SELESAI)

Utang yang **diakui sendiri oleh aplikasi** di tab Rencana Bayar ("yang belum dibangun:
toleransi keterlambatan"). Rinciannya ada di `docs/v2/51_LATE_FEE_TOLERANCE_SPEC.md`.

Yang dibangun:
1. **Satu mesin** `backend/late_fee_engine.py` — kebijakan dari Pusat Konfigurasi
   (`payment.late.*`), tenggang milik TERMIN menang atas bawaan, keadaan `dalam_tenggang`
   yang sebelumnya tidak ada, denda prorata berbatas atas & bawah.
2. **Denda berjurnal** (Dr `1-1300` / Cr **`4-1400` akun baru**), idempoten per (termin,
   bulan); yang ditagihkan adalah SELISIH — klik dua kali bukan denda kedua.
3. **Keringanan** hanya Manajer Keuangan (`late_fee:override`), wajib alasan ≥10 huruf,
   membalik jurnal, dan **tidak bisa dianulir** dengan menagihkan denda yang sama lagi.
4. **Tidak ada mesin kedua**: `finance_reports` (tombol "Denda" lama, daftar penagihan,
   konfigurasi penagihan) dilimpahkan ke mesin di atas; `compute_scheme_items` akhirnya
   membawa `grace_days` ke jadwal tagihan.
5. **Layar**: panel pada tab Rencana Bayar (keadaan per termin dari server, sebab denda belum
   bisa ditagihkan, tagihkan, keringanan) + kartu toleransi & denda di portal pembeli.
6. **Guardrail**: gate 49 (67 pemeriksaan) + `mutasi_58.py` (31 mutan).

---

## 3) Kriteria selesai Fase 58

- `bash scripts/run_all_gates.sh` → OVERALL PASS (**49 gates**).
- `python3 scripts/mutasi_56.py --ringkas` → 49/49 TERTANGKAP.
- `python3 scripts/mutasi_58.py --ringkas` → 31/31 TERTANGKAP.
- Tidak ada layar yang mengaku "belum dibangun" untuk sesuatu yang sudah ada.
- Pembeli membaca toleransi & dendanya sendiri dengan angka yang sama dengan pembukuan.

---

## 3b) Fase 58D — Pemulihan lingkungan & batas ukuran berkas (SELESAI, sesi lanjutan)

Development terhenti pada gate `validate_compliance.py`: `backend/rbac.py` (809) dan
`backend/reference.py` (801) melewati batas **800 baris**. Merapatkan komentar hanya menunda
masalahnya, jadi kedua berkas DIPECAH — SSOT tetap satu:

* `backend/rbac_matrix.py` (487) — `DEFAULT_PERMISSIONS`; `rbac.py` (330) menyimpan
  `require_permission`, `can`, `scope_query`, `audit_log`, `ROLE_GRANTS`.
* `backend/reference_groups.py` (527) — `GROUPS` dasar + `_o`; `reference.py` (284) memuatnya
  lalu melengkapi dengan grup per-fase (`reference_p<NN>.py`) seperti sebelumnya.
* Guardrail yang MEMBACA/MEMUTASI matriks diarahkan ke berkas barunya
  (`verify_late_fee.py` membaca `rbac.py` + `rbac_matrix.py`; `mutasi_45/52/58`).
* Pemulihan lingkungan: `backend/.env` (`JWT_SECRET`, `DEFAULT_ORG_ID`, `PORTAL_MASTER_OTP`),
  `reportlab`/`APScheduler`/`tzlocal`, `yarn install`.

Bukti: `run_all_gates.sh` **PASS (49 gates)**, `mutasi_58` 31/31, `mutasi_56` 49/49,
testing agent iterasi 94: backend 11/11, UI panel denda + dialog keringanan + kartu portal
(angka sama dengan pembukuan), 0 isu.

---

## 4) Tugas berikutnya (untuk sesi lanjutan)

1. **Denda otomatis terjadwal** (opsional per organisasi): sekarang denda ditagihkan lewat
   tombol; scheduler harian bisa menerbitkannya + pengingat WhatsApp (`payment.late.auto_apply`
   belum ada — sengaja, karena menagih otomatis adalah keputusan bisnis).
2. **Laporan denda & keringanan** (siapa meringankan apa, berapa, kapan) untuk rapat direksi.
3. **Pembatalan sepihak karena tunggakan** (`payment.staged.arrears_months_to_cancel` = 2
   bulan) — pasalnya sudah tercetak, mesinnya belum menyambung ke Fase 56.
4. Laporan **utang refund** (`2-1460`) tersendiri + arus kas proyeksi.
