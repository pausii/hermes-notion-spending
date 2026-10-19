# hermes-notion-spending

Skill [Hermes Agent](https://github.com/NousResearch/hermes-agent) untuk mencatat pengeluaran ke database Notion cukup lewat chat WhatsApp.

📝 Cerita lengkap dan latar belakangnya: **[Catat Pengeluaran Cukup Lewat Chat WhatsApp](https://pausi.id/blog/catat-pengeluaran-lewat-whatsapp-dengan-hermes-agent)**

Ada dua cara pakai.

**Kirim foto struk saja**, tanpa teks sama sekali. Hermes membaca isinya, mengisi kolom-kolomnya, meng-upload struknya ke kolom Attachment di Notion, lalu menghapus file aslinya dari server.

**Atau ketik seadanya:**

```
kopi susu 28rb di kopi kenangan, gopay qris
```

Hasilnya sama: satu baris rapi di Notion (Item, Total, Merchant, Kategori, Metode, dll.) plus balasan ringkasan berisi ID `TRX-N`. Ada yang salah tebak? Balas `TRX-26 ganti Metode jadi Kartu Kredit` atau `TRX-26 ganti Total jadi 56rb`.

## Isi repo

```
skills/
  notion-purchase-order-logging/
    SKILL.md                              # skema database, aturan pengisian, workflow curl Notion API
    references/
      database-structure-ops.md           # operasi struktur: rename kolom, ganti tipe/format, hapus, prefix ID
```

## Prasyarat

- Hermes Agent sudah terpasang dan gateway WhatsApp sudah jalan (`hermes whatsapp`, `hermes gateway`).
- Integration Notion ([buat di sini](https://www.notion.so/my-integrations)) dengan token `ntn_...`.
- Database Notion "Spending" dengan 14 kolom sesuai tabel di `SKILL.md`. **Tidak perlu dibuat manual satu per satu** — setelah skill terpasang dan integration terhubung, minta saja Hermes yang membuatkan lewat chat. `references/database-structure-ops.md` berisi semua operasi struktur yang ia butuhkan: tambah kolom, ubah tipe, atur format angka, dan set prefix ID.
- **Model vision — opsional tapi sangat disarankan.** Skill ini membaca foto struk lewat `vision_analyze` lebih dulu. Kalau belum dikonfigurasi, Hermes gagal dengan `No LLM provider configured for task=vision` dan agent jatuh ke OCR — jalan, tapi jauh lebih sering salah baca angka. Aktifkan lewat `hermes setup`, atau langsung di `~/.hermes/config.yaml`:

  ```yaml
  auxiliary:
    vision:
      provider: openrouter
      model: google/gemini-3-flash-preview
  ```

  Model utama tidak perlu multimodal — model vision ini bertugas terpisah, hasilnya diserahkan sebagai teks ke model utama.
- **`tesseract` untuk jalur cadangan.** Dipakai agent kalau vision gagal atau belum diatur:

  ```bash
  sudo apt install tesseract-ocr tesseract-ocr-ind
  ```

## Pasang

1. Simpan token di server:
   ```bash
   echo 'NOTION_API_KEY=ntn_xxx' >> ~/.hermes/.env
   ```
2. Di Notion, buka database → menu `...` → **Connect to** → pilih integration kamu. Tanpa langkah ini semua request balas 404 walaupun token-nya benar.
3. Salin skill ke Hermes:
   ```bash
   cp -r skills/notion-purchase-order-logging ~/.hermes/skills/
   ```
4. Cari kedua ID database lewat search API — `id` di level atas = `data_source_id`, `parent.database_id` = `database_id`:
   ```bash
   source ~/.hermes/.env; export NOTION_API_KEY
   curl -s -X POST https://api.notion.com/v1/search \
     -H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2025-09-03" \
     -H "Content-Type: application/json" \
     -d '{"query":"Spending","filter":{"property":"object","value":"data_source"}}'
   ```
5. Masukkan keduanya ke **salinan yang barusan dipasang**, bukan file di repo ini:
   ```bash
   ${EDITOR:-nano} ~/.hermes/skills/notion-purchase-order-logging/SKILL.md
   # ganti <YOUR_DATA_SOURCE_ID> dan <YOUR_DATABASE_ID>
   ```
6. Restart gateway: `systemctl restart hermes-gateway` (atau jalankan ulang `hermes gateway`).

## Catatan

- Nama foldernya masih `notion-purchase-order-logging` — warisan dari saat database ini bernama "Purchase Order". Dibiarkan apa adanya supaya pemasangan yang sudah jalan tidak rusak.
- Skill ini dibuat dan diperbaiki oleh Hermes sendiri dari percakapan sehari-hari (fitur curator). Aturan pengisiannya — konvensi kategori, format kolom Metode, penanganan duplikat lampiran — mengikuti kebiasaan penulis; sesuaikan di bagian "Aturan Wajib".
- Jangan commit file `.env` atau token apa pun.

## Lisensi

[MIT](LICENSE) © Ahmad Pausi
