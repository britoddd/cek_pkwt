# CekPKWT

Asisten AI untuk pekerja baru di Indonesia yang ingin memeriksa **kontrak kerja waktu tertentu (PKWT)** *sebelum* menandatanganinya. CekPKWT membaca draf kontrak, mencocokkannya dengan pasal regulasi ketenagakerjaan yang tersimpan, lalu menyusun laporan potensi masalah lengkap dengan sitasi pasal dan pertanyaan yang bisa diajukan ke HR.

> ⚠️ **Bukan nasihat hukum.** Hasil CekPKWT dibuat oleh AI dari basis data regulasi yang terbatas dan bisa keliru. Untuk masalah serius, hubungi LBH (Lembaga Bantuan Hukum) atau Dinas Tenaga Kerja setempat.

## Fitur

| Flow | Fungsi |
|---|---|
| `check_work_contract` | Memeriksa teks kontrak PKWT: menyamarkan data pribadi, mengambil pasal relevan dan data upah minimum 2026, lalu menyusun laporan potensi masalah (keparahan, tingkat keyakinan, rujukan pasal, pertanyaan untuk HR). |
| `answer_labor_rule_question` | Menjawab pertanyaan tentang aturan PKWT **hanya** dari kutipan regulasi yang tersimpan, dengan sitasi pasal. Membantah klaim pengguna yang tidak sesuai regulasi, dan menjawab "tidak ditemukan" bila sumbernya tidak ada. |
| `draft_hr_questions` | Mengubah temuan pemeriksaan menjadi email pertanyaan yang sopan untuk HR, lalu menyimpannya sebagai **draf** Gmail (tidak pernah dikirim otomatis). Tanpa alamat HR, teks email ditampilkan untuk disalin. |
| `ingest_labor_corpus` | Memuat korpus regulasi (`data/labor_corpus.csv`) ke vector store. Flow internal, tidak diekspos ke pengguna akhir. |

## Arsitektur

Semua logika dibangun sebagai flow [Langflow](https://www.langflow.org/) (folder `langflow/`):

```
ingest_labor_corpus:
  Read File (CSV) → CSV Rows to Data (1 baris = 1 pasal) → Astra DB (corpus_v1)

check_work_contract:
  Chat Input → PII Redactor ─┬→ Astra DB (cari pasal relevan) ─→ Parser ─┐
                             │   Astra DB (cari upah minimum 2026) → Parser ─┤
                             └──────────────────────────────────────────────→ Prompt → Agent → Chat Output

answer_labor_rule_question:
  Chat Input → Astra DB (top-4 pasal) → Parser <kutipan> → Prompt → Agent → Chat Output

draft_hr_questions:
  Chat Input → Agent (+ tool Gmail "Create Email Draft" via Composio) → Chat Output
```

**Komponen yang dipakai**

- **LLM:** OpenAI `gpt-5.4-mini` (komponen Agent Langflow, dengan tool kalkulator).
- **Embedding:** Google `models/gemini-embedding-001`.
- **Vector store:** DataStax Astra DB — database `cek_pkwt`, koleksi `corpus_v1`, keyspace `default_keyspace`.
- **Gmail:** Composio Gmail (OAuth2), hanya untuk membuat draf.
- **Komponen kustom:**
  - `PII Redactor (CekPKWT)` — menyamarkan NIK, NPWP, email, nomor telepon, nomor rekening, serta baris nama/alamat/TTL (`[NIK]`, `[NAMA]`, `[ALAMAT]`, dst.) sebelum teks kontrak dikirim ke LLM atau vector store. Label milik perusahaan (PT, direktur, HRD, dsb.) dibiarkan.
  - `CSV Rows to Data` — mengubah tiap baris CSV menjadi satu dokumen: kolom `text` menjadi konten, kolom lain menjadi metadata.

### Pengaman di prompt

- Teks kontrak diperlakukan sebagai **data**, bukan instruksi (perlindungan dari *prompt injection*); teks mencurigakan dilaporkan di bagian "Catatan".
- Rujukan pasal dan angka hanya boleh berasal dari kutipan hasil pencarian, tidak boleh dikarang.
- Bahasa netral: "mungkin bertentangan dengan", "perlu dikonfirmasi" — tidak pernah "ilegal" atau "melanggar hukum".
- Ruang lingkup hanya PKWT; PKWTT, magang, dan kemitraan/freelance ditolak dengan sopan.
- Agent email hanya boleh membuat draf, tidak boleh mengirim atau menghapus email.

## Korpus regulasi

`data/labor_corpus.csv` berisi **64 entri** (satu baris = satu pasal/ketentuan), semuanya rezim `current`.

Kolom:

| Kolom | Keterangan |
|---|---|
| `id` | ID unik, mis. `pp35-12` |
| `regulation`, `article` | Nama regulasi dan pasal/ayat |
| `topic` | Topik, mis. `masa_percobaan`, `upah_minimum`, `thr`, `bpjs` |
| `status` | `berlaku`, `diubah`, `disisipkan`, `dimaknai_putusan_mk`, `berlaku_imbauan`, `catatan_sistem` |
| `amended_by` | Regulasi pengubah (jika ada) |
| `regime` | Rezim aturan (`current`) |
| `region`, `province` | `nasional` atau wilayah kabupaten/kota/provinsi |
| `source_url`, `retrieved_on` | Sumber resmi dan tanggal pengambilan |
| `verification` | `checked_official`, `checked_secondary`, `accepted_single_source`, `system_note` |
| `text` | Teks pasal, diawali header `[Regulasi \| Pasal \| status \| rezim \| wilayah \| topik]` yang dipakai LLM untuk sitasi |

Cakupan utama:

- UU 13/2003 sebagaimana diubah UU 6/2023, Putusan MK No. 168/PUU-XXI/2023
- PP 35/2021 (PKWT, alih daya, waktu kerja, PHK) dan PP 36/2021 (Pengupahan)
- UU 24/2011 (BPJS), UU 4/2024 (KIA), Permenaker 6/2016 (THR), SE Menaker M/5/HK.04.00/V/2025
- **Upah minimum 2026** untuk DKI Jakarta, Jawa Barat (Bekasi, Karawang, Depok, Bogor, Bandung), Banten (Tangerang, Tangerang Selatan, Cilegon), Jawa Tengah (Semarang), DIY (Yogyakarta), dan Jawa Timur (Surabaya, Sidoarjo, Gresik, Malang)

UU Ketenagakerjaan baru yang sedang dibahas DPR **belum** tercakup. Wilayah di luar daftar di atas akan dilaporkan sebagai "Tidak dapat diperiksa" untuk pengecekan upah.

## Persiapan

### Prasyarat

- Langflow (versi yang mendukung komponen Agent dan bundle DataStax/Composio)
- Akun DataStax Astra DB dengan database `cek_pkwt`
- API key OpenAI dan Google AI (Gemini)
- Akun Composio yang terhubung ke Gmail (opsional, untuk `draft_hr_questions`)

### Variabel lingkungan

Salin `.env.example` menjadi `.env` lalu isi:

```env
ASTRA_DB_PKWT=      # Application token Astra DB
COMPOSIO_PKWT=      # API key Composio (Gmail)
GOOGLE_API_KEY=     # Untuk embedding gemini-embedding-001
OPENAI_API_KEY=     # Untuk Agent gpt-5.4-mini
```

Flow merujuk nama-nama variabel ini sebagai *Global Variables* Langflow, jadi nilainya tidak tersimpan di file JSON. Daftarkan variabelnya dengan salah satu cara berikut:

- **Manual:** Settings → Global Variables, buat variabel dengan nama yang sama persis.
- **Otomatis dari environment:** jalankan Langflow dengan
  ```env
  LANGFLOW_STORE_ENVIRONMENT_VARIABLES=true
  LANGFLOW_VARIABLES_TO_GET_FROM_ENVIRONMENT=ASTRA_DB_PKWT,COMPOSIO_PKWT,GOOGLE_API_KEY,OPENAI_API_KEY
  ```

### Langkah setup

1. Jalankan Langflow, lalu impor keempat file JSON dari folder `langflow/`.
2. Komponen **Astra DB** sudah membaca token dari `ASTRA_DB_PKWT`. Jika database Anda berbeda, pilih ulang database dan endpoint-nya (daftar endpoint akan terisi otomatis dari token). Pastikan nama koleksi `corpus_v1` dan model embedding sama di semua flow.
3. Tambahkan API key model untuk Agent di **Settings → Model Providers** (OpenAI, dan Google jika ingin memakai model Gemini). Field `api_key` di komponen Agent hanya *override*; jika dikosongkan, Agent memakai key dari Model Providers sesuai model yang dipilih di dropdown "Language Model".
4. Buka flow `ingest_labor_corpus`, unggah `data/labor_corpus.csv` ke komponen **Read File**, lalu jalankan untuk mengisi vector store.
5. Pada flow `draft_hr_questions`, hubungkan komponen **Gmail** ke akun Composio Anda (OAuth).
6. Coba setiap flow di Playground Langflow, atau panggil lewat API/MCP Langflow.

## Contoh penggunaan

**Memeriksa kontrak** — tempelkan seluruh teks kontrak ke `check_work_contract`. Laporan yang dihasilkan berformat:

```markdown
**Ringkasan:** Ditemukan 2 potensi masalah (1 tinggi, 0 sedang, 1 rendah). ...

**Potensi masalah**
### 1. Masa percobaan pada PKWT
- **Isi kontrak:** "Karyawan menjalani masa percobaan selama 3 bulan."
- **Potensi masalah:** ...
- **Rujukan:** PP 35/2021 Pasal 12, status: berlaku
- **Keparahan:** tinggi · **Keyakinan:** tinggi
- **Pertanyaan untuk HR:** ...

**Klausul tanpa masalah yang ditemukan:** ...
**Tidak dapat diperiksa:** ...
```

**Bertanya soal aturan** — kirim ke `answer_labor_rule_question`, misalnya:
> "Katanya kontrak PKWT boleh pakai masa percobaan 3 bulan, bener kan?"

**Membuat email untuk HR** — kirim daftar temuan (dan, bila ada, alamat email HR) ke `draft_hr_questions`. Periksa dan sunting draf sebelum mengirimnya sendiri.

## Struktur repository

```
.
├── .env.example                 # Template variabel lingkungan
├── data/
│   └── labor_corpus.csv         # Korpus regulasi (64 pasal)
└── langflow/
    ├── answer_labor_rule_question.json
    ├── check_work_contract.json
    ├── draft_hr_questions.json
    └── ingest_labor_corpus.json
```

## Memperbarui korpus

1. Tambahkan atau ubah baris di `data/labor_corpus.csv` sesuai kolom di atas. Sertakan header `[...]` di awal kolom `text`, serta `source_url`, `retrieved_on`, dan `verification`.
2. Jika pasal lama diubah, ganti `status` dan isi `amended_by`, jangan menghapus barisnya.
3. Jalankan ulang `ingest_labor_corpus`. Pertimbangkan koleksi baru (mis. `corpus_v2`) agar tidak ada entri ganda, lalu perbarui nama koleksi di semua flow.
