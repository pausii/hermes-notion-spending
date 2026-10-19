# Notion Data Source — Structure Operations

Semua operasi struktur kolom: `PATCH /v1/data_sources/{data_source_id}` + header `Notion-Version: 2025-09-03`. Semua terverifikasi jalan lewat integration internal Notion.

## Rename database (judul)
```json
{"title":[{"type":"text","text":{"content":"Spending"}}]}
```

## Rename kolom
```json
{"properties":{"Old Name":{"name":"New Name","type":"<same type>","<type>":{...}}}}  
```
WAJIB sertakan ulang tipe + definisi penuh. Untuk `select`/`status`/`multi_select` harus re-list SEMUA opsi, kalau tidak opsi hilang. Untuk `title` cukup `{"type":"title","title":{}}`. Data baris yang sudah ada otomatis ikut (nilai tetap, nama berubah).

## Rubah tipe select → multi_select
Kirim ulang properti dengan `type: "multi_select"` + semua opsi yang diinginkan. Nilai single-select yang sudah ada dipertahankan Notion. (Metode: select → multi_select, terverifikasi.)

## Hapus kolom
```json
{"properties":{"Nama Kolom":null}}
```

## Ganti format angka
```json
{"properties":{"Real Amount":{"name":"Real Amount","type":"number","number":{"format":"number_with_commas"}}}}
```
`number_with_commas` = pemisah ribuan koma (48,000).

## Ganti prefix unique_id (ID)
```json
{"properties":{"ID":{"name":"ID","type":"unique_id","unique_id":{"prefix":"TRX"}}}}
```
Prefix bersifat level-database: SEMUA baris (lama & baru) menampilkan prefix baru di response query. Nomor tidak di-reset oleh perubahan ini.

## Hapus entri (arsip)
`PATCH /v1/pages/{page_id}` dengan `{"archived":true}`. Nomor unique_id TIDAK dipakai ulang oleh baris baru — counter menerus dari angka terbesar yang pernah dibuat. Setelah hapus TRX-24, create berikutnya dapat TRX-25, bukan TRX-24.

## Mencari page_id dari nomor TRX
Filter body `{"filter":{"property":"ID","unique_id":{"number":N}}}` TIDAK reliabel (return kosong). Cara yang jalan: query `page_size:100`, sort ID descending, scan nomor di Python.
```bash
curl -s -X POST "https://api.notion.com/v1/data_sources/$DS/query" -H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2025-09-03" -H "Content-Type: application/json" -d '{"page_size":100,"sorts":[{"property":"ID","direction":"descending"}]}' | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r['properties'].get('ID',{}).get('unique_id',{}).get('number'), r['id'], r['properties'].get('Item',{}).get('title',[{}])[0].get('plain_text','') if r['properties'].get('Item',{}).get('title') else '') for r in d['results']]"
```
