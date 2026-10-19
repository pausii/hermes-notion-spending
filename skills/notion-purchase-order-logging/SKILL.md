---
name: notion-purchase-order-logging
description: "Logging pengeluaran user ke Notion Spending DB."
---

# Spending Logging

Mencatat pengeluaran/pembelian user ke database Notion **"Spending"**. Database punya DUA ID yang berbeda tergantung endpoint:
- `data_source_id` (id level-atas hasil search) = PATCH struktur kolom + query baris.
- `database_id` (field `parent.database_id` di hasil search / URL browser user) = POST /v1/pages (create page).

## Akses & Konstanta

- Token: `NOTION_API_KEY` di `~/.hermes/.env` (integration Notion milik user).
- data_source_id: `<YOUR_DATA_SOURCE_ID>`
- database_id: `<YOUR_DATABASE_ID>`
- ID prefix: **TRX-** (auto-increment unique_id)

## Struktur Kolom (14 kolom)

| Kolom | Tipe | Aturan |
|---|---|---|
| ID | unique_id (prefix TRX) | Auto — referensi utama untuk revisi |
| Item | title | Nama barang/transaksi — **tanpa tanggal** (Date column sudah cukup) |
| Date | date | ISO YYYY-MM-DD |
| Total Amount | number (format: rupiah) | WAJIB dalam IDR — konversi dari valas jika perlu |
| Real Amount | number (format: number_with_commas) | Nominal asli — sama dengan Total Amount untuk IDR |
| Currency | select: IDR / USD / EUR / SGD | WAJIB diisi untuk SEMUA baris |
| Status | status: Draft / Sent / Approved / Received / Completed | Completed untuk transaksi lunas |
| Kategori | select: Bahan Baku / Alat & Perlengkapan / Transportasi / Operasional / Lainnya | Konvensi: makanan/minuman → Lainnya; bensin/parkir → Transportasi; server/internet/langganan → Operasional. Tebak wajar kalau tidak disebut, tandai di Catatan |
| Merchant | rich_text | Nama toko/penjual |
| Metode | multi_select (opsi: QRIS, Tunai, Transfer Bank, Kartu Kredit, Kredit / Tempo, OVO, GoPay, DANA, ShopeePay, LinkAja, BCA, Mandiri, BNI, BRI, Jenius, Seabank, Bank Lain) | **Format: [App, Method]** — nama app dulu, lalu metode. Contoh: OVO+QRIS untuk QRIS lewat OVO; DANA/GoPay saja untuk transfer dompet; Tunai untuk cash |
| Urgency Level | select: Low / Medium / High / Critical | Default Low |
| Expected Delivery | date | Opsional |
| Attachment | files | Upload lampiran, lalu hapus file asli |
| Catatan | rich_text | Asumsi, kurs valas, info tambahan |

## Aturan Wajib (Preferensi User)

1. **Data tidak lengkap?** — isi tebakan wajar (kategori, merchant, metode), tandai di Catatan. Data dari teks user menang atas isi file.
2. **Baca file yang dikirim** — priority: `vision_analyze` DULU (butuh `auxiliary.vision` terkonfigurasi di `~/.hermes/config.yaml`; jauh lebih akurat — baca digit, subtotal, pajak, metode bayar sekaligus), OCR tesseract hanya fallback kalau vision error. Jika nilai masih ambigu, tandai tebakan di Catatan + sebut di pesan konfirmasi supaya user koreksi. Jika user menyatakan nilai/merchant/metode untuk suatu entri, nilai user MENANG atas isi file (kadang nama merchant di struk beda dari sebutan user — isi Item pakai sebutan user, merchant asli di kolom Merchant).
3. **Item title TIDAK perlu tanggal** — Date column sudah cukup.
4. **Currency & Real Amount WAJIB** — untuk IDR: Currency=IDR, Real Amount = Total Amount.
5. **Metode multi-select** — isi nama App dulu baru metode. Contoh: OVO + QRIS, bukan QRIS + OVO.
6. **Revisi** — user sebut "TRX-N". Query semua baris (sort ID descending) lalu cari nomornya di Python: filter `unique_id.number` di query body sering return kosong pada versi API ini — jangan dipakai. PATCH page, konfirmasi.
7. **Hapus entri** — PATCH page `{"archived": true}` (soft delete). Nomor unique_id TIDAK dipakai ulang: setelah TRX-N diarsipkan, entri baru dapat nomor berikutnya yang lebih besar (TRX-24 diarsip → entri baru TRX-25) — kabari user kalau nomor yang dia sebut melompat.
8. **Satu screenshot berisi banyak transaksi** (mis. history OVO dengan 2+ pembayaran) — buat SATU entri per pembayaran, tapi upload file sekali dan pakai file_upload_id yang sama untuk semua entri (upload once, attach many).
9. **Konfirmasi** — setelah create, kirim ringkasan: ID (TRX-N), Item, Total, Kategori, Metode, status lampiran.
10. **Privasi** — setelah file terupload ke Notion, hapus file asli dari disk (`rm`). Jangan simpan salinan.
11. **Cek duplikat lampiran** — jika user mengirim image/attachment yang terlihat sama dengan entri sebelumnya, cek dulu apakah sudah pernah dilampirkan di Notion (bandingkan merchant+tanggal+nominal); jika mendekati duplikat, konfirmasi ke user sebelum upload. File asli sudah dihapus tiap upload, jadi cek dilakukan dari data di Notion, bukan file lokal.

## Workflow Create Page

```bash
source ~/.hermes/.env; export NOTION_API_KEY
DS="<YOUR_DATA_SOURCE_ID>"
DB="<YOUR_DATABASE_ID>"

# 1 — upload file (jika ada)
UP=$(curl -s -X POST "https://api.notion.com/v1/file_uploads" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"filename":"...","content_type":"...","mode":"single_part"}')
FU_ID=$(echo "$UP" | python3 -c "import json,sys; print(json.load(sys.stdin).get('id',''))")
SEND_URL=$(echo "$UP" | python3 -c "import json,sys; print(json.load(sys.stdin).get('upload_url',''))")

# 2 — kirim bytes (WAJIB Notion-Version header!)
curl -s -X POST "$SEND_URL" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -F "file=@path;type=mime/type"
# cek status: harus "uploaded" — kalau "pending" berarti header Notion-Version terlewat

# 3 — create page (pakai database_id, bukan data_source_id!)
curl -s -X POST "https://api.notion.com/v1/pages" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "parent": {"database_id": "'$DB'"},
    "properties": {
      "Item": {"title": [{"text": {"content": "Nama Item"}}]},
      "Date": {"date": {"start": "YYYY-MM-DD"}},
      "Total Amount": {"number": <IDR>},
      "Currency": {"select": {"name": "IDR"}},
      "Real Amount": {"number": <IDR>},
      "Status": {"status": {"name": "Completed"}},
      "Merchant": {"rich_text": [{"text": {"content": "..."}}]},
      "Kategori": {"select": {"name": "..."}},
      "Metode": {"multi_select": [{"name": "App"}, {"name": "Method"}]},
      "Catatan": {"rich_text": [{"text": {"content": "..."}}]},
      "Attachment": {"files": [{"name": "...", "type": "file_upload", "file_upload": {"id": "'$FU_ID'"}}]}
    }
  }'

# 4 — hapus file asli
rm -f "$PATH_TO_FILE"

# 5 — verifikasi
curl -s -X POST "https://api.notion.com/v1/data_sources/$DS/query" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"page_size": 5, "sorts": [{"property": "ID", "direction": "descending"}]}'
```

## Revisi Baris

```bash
# Cari page_id via ID
# Query full data dulu
curl -s -X POST "https://api.notion.com/v1/data_sources/$DS/query" \
  -H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"page_size": 20}' | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r['properties'].get('ID',{}).get('unique_id',{}).get('number'), r['id']) for r in d['results']]"

# PATCH page
curl -s -X PATCH "https://api.notion.com/v1/pages/{page_id}" \
  -H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"properties": {"Item": {"title": [{"text": {"content": "..."}}]}}}'
```

## Pitfall

- **Dua ID berbeda**: `data_source_id` untuk query+PATCH struktur, `database_id` untuk create page. Salah = 404 "Could not find".
- **Search ≠ akses tulis**: `/v1/search` bisa menampilkan database yang belum di-connect ke integration. Bukti akses yang benar: query `{"page_size": 1}` ke data source mengembalikan 200.
- **File upload Notion-Version**: header `Notion-Version: 2025-09-03` WAJIB disertakan pada POST upload_url. Tanpa header, HTTP 200 tapi status tetap `pending` — page create gagal dengan "File upload has an invalid status of pending".
- **Upload method**: `POST` + `multipart/form-data`, bukan PUT raw binary.
- **File reference format**: bukan `file: {url: ...}` — pakai `{"type": "file_upload", "file_upload": {"id": "..."}}`.
- **Metode ganti tipe**: dari select ke multi_select data lama tetap dipertahankan (Notion menjaga nilai yang sudah ada).
- **Operasi struktur database**: rename kolom, ganti format angka, hapus kolom, ganti prefix ID — lihat `references/database-structure-ops.md`.
