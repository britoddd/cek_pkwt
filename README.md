# CekPKWT

**AI pemeriksa kontrak kerja PKWT sebelum tanda tangan.**

CekPKWT adalah AI agent yang membantu *fresh graduate* memeriksa kontrak kerja waktu tertentu (PKWT) **sebelum** menandatanganinya. Pengguna cukup menempelkan teks kontrak ke **IBM Bob**. Bob lalu memanggil flow **IBM Langflow** lewat MCP untuk menyamarkan data pribadi, mengekstrak klausul penting, mencari pasal yang relevan dari basis regulasi resmi, dan menyusun laporan potensi masalah. Laporan itu memuat pasal rujukan beserta status berlakunya, tingkat keparahan, dan pertanyaan yang bisa diajukan ke HR.

Tema: **Education & Future of Work**

> ⚠️ **Bukan nasihat hukum.** Hasil CekPKWT dibuat oleh AI dari basis regulasi yang terbatas dan bisa keliru. Untuk masalah serius, hubungi LBH (Lembaga Bantuan Hukum) atau Dinas Tenaga Kerja setempat.

## Daftar isi

- [Latar belakang](#latar-belakang)
- [Fitur utama](#fitur-utama)
- [Arsitektur](#arsitektur)
- [Penjelasan setiap flow](#penjelasan-setiap-flow)
- [Korpus regulasi](#korpus-regulasi)
- [Setup](#setup)
- [Menghubungkan IBM Bob ke Langflow](#menghubungkan-ibm-bob-ke-langflow)
- [Contoh penggunaan](#contoh-penggunaan)
- [Hasil evaluasi](#hasil-evaluasi)
- [Responsible AI](#responsible-ai)
- [Keterbatasan](#keterbatasan)
- [Rencana pengembangan](#rencana-pengembangan)
- [Struktur repository](#struktur-repository)

## Latar belakang

### Masalah

Fresh graduate yang menerima kontrak PKWT pertama biasanya menandatanganinya dalam waktu singkat, tanpa tahu klausul mana yang mungkin bertentangan dengan aturan. Contohnya masa percobaan pada PKWT, upah di bawah upah minimum, tidak adanya uang kompensasi saat kontrak berakhir, atau denda jika mengundurkan diri lebih awal.

Ada tiga hal yang membuatnya sulit:

1. **Aturannya tersebar** di UU Ketenagakerjaan beserta perubahannya, peraturan pemerintah turunannya, keputusan upah minimum daerah, dan putusan Mahkamah Konstitusi.
2. **Aturannya sedang berubah.** UU Ketenagakerjaan baru sedang dibahas DPR dan ditargetkan disahkan pada Oktober 2026.
3. **Bantuan sulit dijangkau.** Pengacara terlalu mahal bagi pekerja baru, sementara bertanya ke HR terasa canggung karena mereka takut tawaran kerjanya dibatalkan.

Kerugiannya baru terasa setelah kontrak ditandatangani, saat posisi tawar pekerja sudah jauh lebih lemah.

### Target pengguna

- **Pengguna utama:** fresh graduate dan mahasiswa tingkat akhir yang akan menandatangani kontrak PKWT pertama mereka, dimulai dari wilayah DKI Jakarta.
- **Pengguna sekunder:** pusat karier kampus yang mendampingi lulusan dalam mencari kerja.
- **Di luar cakupan MVP:** magang, PKWTT, dan outsourcing.

### Bedanya dengan cara yang ada

| Cara saat ini | Kelemahan |
|---|---|
| Membaca kontrak sendiri | Tidak tahu aturan mana yang berlaku |
| Bertanya ke teman | Jawaban berbeda-beda dan tidak berdasar |
| Chatbot umum | Bisa memberi pasal yang salah atau sudah tidak berlaku, tanpa sumber yang bisa dicek |
| Konsultasi ke pengacara | Mahal dan lambat untuk kebutuhan sebelum tanda tangan |

CekPKWT:

1. Hanya menjawab berdasarkan basis regulasi resmi yang terverifikasi.
2. Mencantumkan pasal rujukan, status berlakunya, dan rezim aturan untuk setiap temuan.
3. Menyatakan dengan jelas jika suatu hal tidak dapat diperiksa.
4. Mengubah temuan menjadi tindakan nyata, yaitu pertanyaan dan draf email untuk HR.

## Fitur utama

1. **Pemeriksaan Kontrak** (`check_work_contract`): mengekstrak klausul penting dari teks kontrak, membandingkannya dengan pasal regulasi dan upah minimum 2026 per wilayah, lalu menyusun laporan potensi masalah beserta tingkat keparahan, tingkat keyakinan, dan pasal rujukan.
2. **Tanya Aturan PKWT** (`answer_labor_rule_question`): menjawab pertanyaan lanjutan hanya dari regulasi yang tersimpan, dengan sitasi pasal. Klaim pengguna yang keliru ("katanya boleh kan?") langsung dibantah. Jika jawabannya tidak ada di sumber, sistem menyatakannya dan menyarankan bertanya ke Disnaker atau LBH.
3. **Draf Pertanyaan untuk HR** (`draft_hr_questions`): mengubah temuan menjadi pertanyaan yang sopan dan membuat draf Gmail yang tidak terkirim otomatis. Pengguna meninjau dan mengirimnya sendiri.
4. **Kesadaran Perubahan Regulasi:** setiap pasal di korpus ditandai status (`berlaku`, `diubah`, `dimaknai_putusan_mk`, …) dan rezim aturan. Laporan selalu menyebut rezim yang dipakai, sehingga korpus dapat diperbarui saat UU baru berlaku tanpa membangun ulang sistem.
5. **Perlindungan Data Pribadi:** NIK, NPWP, email, nomor telepon, nomor rekening, serta baris nama, alamat, dan tempat/tanggal lahir disamarkan otomatis sebelum teks diproses model. Kontrak pengguna tidak pernah disimpan ke basis data vektor.

## Arsitektur

```
 Pengguna
    │  "tolong cek kontrak ini sebelum saya tanda tangan"
    ▼
 IBM Bob  (antarmuka percakapan, MCP client)
    │  memilih tool berdasarkan nama & deskripsi
    ▼  MCP (Streamable HTTP, API key)
 IBM Langflow  (MCP server, seluruh logika workflow)
    ├── check_work_contract         ──┐
    ├── answer_labor_rule_question  ──┼──► Astra DB (corpus_v1, embedding Gemini)
    ├── draft_hr_questions ──► Composio Gmail (hanya draf)
    └── ingest_labor_corpus (internal) ─► memuat data/labor_corpus.csv ke Astra DB
```

**Pembagian peran**

- **Langflow** menjalankan seluruh logika: redaksi data pribadi, ekstraksi klausul, pengambilan pasal dari Astra DB, perbandingan oleh LLM, dan pembuatan draf email.
- **IBM Bob** menjadi antarmuka percakapan: menafsirkan permintaan pengguna, memilih tool Langflow yang tepat, lalu menjelaskan hasilnya dengan bahasa yang mudah dipahami.

**Teknologi**

| Bagian | Yang dipakai |
|---|---|
| Orkestrasi workflow | IBM Langflow 1.12.4 |
| Antarmuka & MCP client | IBM Bob |
| LLM (Agent) | OpenAI `gpt-5.4-mini` (bawaan flow). Bisa diganti ke model Gemini lewat dropdown "Language Model". |
| Embedding | Google `models/gemini-embedding-001` |
| Vector store | DataStax Astra DB: database `cek_pkwt`, koleksi `corpus_v1` |
| Email | Composio Gmail (OAuth2) |

**Alur penggunaan**

1. Pengguna menempelkan teks kontrak di IBM Bob.
2. Bob memilih tool `check_work_contract`, lalu MCP mengirim teks ke Langflow.
3. Langflow menyamarkan data pribadi dan mencari pasal serta upah minimum yang relevan di Astra DB.
4. Agent mengekstrak klausul, membandingkannya dengan pasal, lalu menyusun laporan.
5. Laporan dikembalikan ke Bob, dan Bob menjelaskannya kepada pengguna.
6. Pengguna bertanya lebih lanjut (`answer_labor_rule_question`) atau membuat draf email HR (`draft_hr_questions`).
7. Pengguna meninjau draf di Gmail dan mengirimnya sendiri.

## Penjelasan setiap flow

Semua flow ada di folder `langflow/` dan dapat diimpor langsung ke Langflow.

### 1. `check_work_contract`
![Flow check_work_contract di Langflow](docs/screenshots/langflow-workflow.png)

```
Chat Input → PII Redactor ─┬─► Astra DB (pasal relevan, top-4) ─► Parser ─┐
                           │   Astra DB (upah minimum 2026)   ─► Parser ─┤
                           └────────────────────────────────────────────►├─► Prompt Template ─► Agent ─► Chat Output
```

- **Input:** teks kontrak.
- **PII Redactor (CekPKWT):** komponen kustom berbasis regex yang mengganti data pribadi dengan label (`[NIK]`, `[NPWP]`, `[EMAIL]`, `[TELEPON]`, `[NO_REKENING]`, `[NAMA]`, `[ALAMAT]`, `[TTL]`). Label milik perusahaan (PT, CV, direktur, HRD) dibiarkan.
- **Retrieval:** teks yang sudah disamarkan dipakai sebagai query ke Astra DB. Query kedua yang tetap mengambil data upah minimum 2026 per wilayah.
- **Agent** (dengan tool kalkulator dan tanggal):
  1. mengecek cakupan (hanya PKWT; PKWTT, magang, dan kemitraan ditolak dengan sopan),
  2. mengekstrak klausul ke struktur tetap: jenis kontrak, jenis pekerjaan, jangka waktu dan perpanjangan, masa percobaan, upah dan lokasi kerja, uang kompensasi, BPJS, pengakhiran dini dan denda, penahanan dokumen, serta bentuk dan bahasa kontrak,
  3. mencocokkan setiap klausul dengan pasal yang ditemukan,
  4. menandai masalah **hanya** jika didukung teks pasal,
  5. menyusun laporan.
- **Output:** ringkasan, daftar potensi masalah (isi kontrak, penjelasan, rujukan pasal dan statusnya, keparahan, keyakinan, pertanyaan untuk HR), klausul tanpa masalah, klausul yang tidak dapat diperiksa, catatan, dan disclaimer.
- Pesan masukan tidak disimpan ke riwayat pesan Langflow (`should_store_message = false`).

### 2. `answer_labor_rule_question`

```
Chat Input ─┬─► Astra DB (top-4) ─► Parser <kutipan> ─┐
            └─────────────────────────────────────────┴─► Prompt Template ─► Agent ─► Chat Output
```

- **Input:** pertanyaan pengguna.
- **Aturan jawaban:** hanya dari kutipan yang ditemukan. Sitasi disalin persis dari header kutipan dengan format `(Regulasi, Pasal, status)`. Jika jawabannya tidak ada, sistem menjawab "Saya tidak menemukan jawabannya di sumber regulasi yang saya miliki." Jika pertanyaan mengandung klaim yang bertentangan dengan kutipan, kalimat pertama jawaban langsung membantahnya.
- **Output:** jawaban beserta sitasi pasal.

### 3. `draft_hr_questions`

```
Chat Input ─► Agent (+ Composio Gmail sebagai tool) ─► Chat Output
```

- **Input:** daftar temuan dari laporan pemeriksaan, dan bila ada, alamat email HR.
- **Proses:** menulis email berbahasa Indonesia yang sopan dan tidak menuduh (subjek "Pertanyaan Terkait Draf Kontrak Kerja (PKWT)", satu pertanyaan bernomor per temuan, nama pengirim `[Nama Anda]`).
  - Jika alamat HR tersedia, Agent memakai aksi Gmail **Create Email Draft**.
  - Jika tidak, teks email ditampilkan untuk disalin.
- **Output:** status ("Draf dibuat di Gmail (belum terkirim)" atau "Teks email siap disalin"), subjek dan isi email, serta pengingat untuk meninjau sebelum mengirim.
- **Agent tidak bisa mengirim email.** Di komponen Gmail hanya dua aksi yang aktif: **Create Email Draft** dan **List Drafts**. Aksi kirim (Send Email, Send Draft) dan semua aksi lain dimatikan, sehingga pembatasnya bersifat teknis, bukan hanya instruksi prompt. Prompt Agent juga mengabaikan instruksi di dalam input yang meminta mengirim atau menghapus email.

![Isi draf email HR di IBM Bob](docs/screenshots/bob-hr-draft.png)

### 4. `ingest_labor_corpus` (internal, tidak diekspos ke Bob)

```
Read File (CSV) ─► CSV Rows to Data ─► Astra DB (embedding gemini-embedding-001)
```

- **CSV Rows to Data:** komponen kustom yang mengubah satu baris CSV menjadi satu dokumen. Kolom `text` menjadi konten, kolom lain menjadi metadata.

## Korpus regulasi

`data/labor_corpus.csv` berisi **64 entri**. Satu baris berisi satu pasal atau ketentuan, dan semuanya berada di rezim `current`.

| Kolom | Keterangan |
|---|---|
| `id` | ID unik, mis. `pp35-12` |
| `regulation`, `article` | Nama regulasi dan pasal/ayat |
| `topic` | Topik, mis. `masa_percobaan`, `upah_minimum`, `thr`, `bpjs` |
| `status` | `berlaku`, `diubah`, `disisipkan`, `dimaknai_putusan_mk`, `berlaku_imbauan`, `catatan_sistem` |
| `amended_by` | Regulasi pengubah (jika ada) |
| `regime` | Rezim aturan (`current`; rezim UU baru akan ditambahkan setelah disahkan) |
| `region`, `province` | `nasional`, atau kabupaten/kota/provinsi untuk upah minimum |
| `source_url`, `retrieved_on` | Sumber resmi dan tanggal pengambilan |
| `verification` | `checked_official` (39), `checked_secondary` (20), `accepted_single_source` (4), `system_note` (1) |
| `text` | Teks pasal, diawali header `[Regulasi \| Pasal \| status \| rezim \| wilayah \| topik]` yang dipakai LLM untuk sitasi |

**Cakupan regulasi:**

- UU 13/2003 sebagaimana diubah UU 6/2023, Putusan MK No. 168/PUU-XXI/2023
- PP 35/2021 (PKWT, alih daya, waktu kerja, PHK) dan PP 36/2021 (Pengupahan)
- UU 24/2011 (BPJS), UU 4/2024 (KIA), Permenaker 6/2016 (THR), SE Menaker M/5/HK.04.00/V/2025

**Upah minimum 2026:**

- DKI Jakarta (UMP)
- Jawa Barat: Kota/Kab. Bekasi, Karawang, Depok, Kota/Kab. Bogor, Bandung
- Banten: Kota/Kab. Tangerang, Tangerang Selatan, Cilegon
- Jawa Tengah: Semarang
- DI Yogyakarta: Kota Yogyakarta
- Jawa Timur: Surabaya, Sidoarjo, Gresik, Malang

## Setup

### Prasyarat

- IBM Langflow 1.12.4 atau yang lebih baru (versi flow diekspor)
- IBM Bob
- Akun DataStax Astra DB dengan database `cek_pkwt`
- API key OpenAI dan/atau Google AI (Gemini)
- Akun Composio yang terhubung ke Gmail (untuk `draft_hr_questions`)

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
5. Pada flow `draft_hr_questions`, hubungkan komponen **Gmail** ke akun Composio Anda (OAuth). Aksi yang aktif sudah dibatasi ke Create Email Draft dan List Drafts; jangan aktifkan aksi kirim.
6. Uji setiap flow di Playground Langflow sebelum menghubungkannya ke Bob.

## Menghubungkan IBM Bob ke Langflow

Langflow mengekspos flow dalam satu project sebagai **MCP server**, dan IBM Bob terhubung sebagai **MCP client**.

1. **Siapkan MCP server di Langflow.**
   - Buka project yang berisi flow CekPKWT, lalu buka tab **MCP Server**.
   - Aktifkan tiga flow sebagai tool: `check_work_contract`, `answer_labor_rule_question`, dan `draft_hr_questions`. Jangan aktifkan `ingest_labor_corpus`.
   - Pastikan nama dan deskripsi setiap tool jelas, karena Bob memilih tool berdasarkan keduanya.
2. **Siapkan autentikasi.** Pakai autentikasi API key, lalu buat API key Langflow di Settings → API Keys.
3. **Salin konfigurasi.** Ambil JSON konfigurasi untuk transport **Streamable HTTP** dari tab MCP Server.
4. **Daftarkan di Bob.** Tambahkan konfigurasi itu ke `mcp.json` milik IBM Bob. Bentuknya kurang lebih seperti ini; pakai nilai yang disalin dari Langflow:

   ```json
   {
     "mcpServers": {
       "cekpkwt": {
         "type": "streamable-http",
         "url": "http://localhost:7860/api/v1/mcp/project/<PROJECT_ID>/streamable",
         "headers": {
           "x-api-key": "<LANGFLOW_API_KEY>"
         }
       }
     }
   }
   ```

   Jangan commit file konfigurasi yang berisi API key asli.
5. **Uji di Bob.** Pastikan ketiga tool muncul di daftar tool MCP Bob, lalu coba contoh di bawah.

Bob mengirim teks pengguna ke Chat Input setiap flow dan menerima Chat Output sebagai hasil tool. Logika workflow tetap berada di Langflow, sedangkan logika percakapan berada di Bob.

## Contoh penggunaan

| Pengguna menulis di Bob | Tool yang dipanggil |
|---|---|
| "Tolong cek kontrak ini sebelum saya tanda tangan: …" | `check_work_contract` |
| "Apa itu kompensasi akhir PKWT?" | `answer_labor_rule_question` |
| "Katanya kontrak PKWT boleh pakai masa percobaan 3 bulan, bener kan?" | `answer_labor_rule_question` |
| "Bantu buatkan email ke HR (hr@perusahaan.co.id) soal temuan tadi" | `draft_hr_questions` |

### Hasil nyata di IBM Bob

**Pemeriksaan kontrak:** Bob memanggil `check_work_contract` dan menampilkan 8 potensi masalah.

![Hasil check_work_contract di IBM Bob](docs/screenshots/bob-check-contract.png)

**Tanya aturan dan permintaan draf email:** Bob memanggil `answer_labor_rule_question` untuk UMK Karawang, lalu `draft_hr_questions`.

![Tanya UMK dan permintaan draf email di IBM Bob](docs/screenshots/bob-rule-question.png)

## Hasil evaluasi

> Bagian ini diisi dari hasil pengujian nyata. Jangan isi dengan perkiraan.

| Metrik | Hasil |
|---|---|
| Masalah yang sengaja ditanam dan berhasil terdeteksi | [X dari Y] pada [N] kontrak uji fiktif ([%]) |
| Alarm palsu pada kontrak bersih | [Z] pada [M] kontrak |
| Sitasi pasal yang sesuai teks resmi (verifikasi manual) | [X%] |
| Waktu review kontrak | dari [A] menit menjadi [B] menit ([N] responden) |

## Responsible AI

- **Hanya dari sumber:** jawaban dan temuan hanya boleh berasal dari pasal yang diambil dari korpus. Pasal, angka, atau akibat hukum tidak boleh dikarang.
- **Jujur saat tidak tahu:** klausul yang tidak bisa dicocokkan dengan pasal dimasukkan ke "Tidak dapat diperiksa". Pertanyaan tanpa sumber dijawab "tidak ditemukan", disertai saran ke Disnaker atau LBH.
- **Transparansi rezim aturan:** setiap rujukan mencantumkan status pasal, dan setiap laporan menyebut rezim aturan yang dipakai.
- **Privasi:** data pribadi disamarkan sebelum teks dikirim ke LLM atau dipakai sebagai query. Kontrak tidak pernah dimasukkan ke basis data vektor, dan pesan masukan pemeriksaan kontrak tidak disimpan ke riwayat pesan Langflow.
- **Tahan prompt injection:** teks kontrak diperlakukan sebagai data, bukan instruksi. Teks yang mencoba memberi perintah ke AI dilaporkan di bagian "Catatan".
- **Bahasa yang tidak menghakimi:** sistem memakai "mungkin bertentangan dengan" atau "perlu dikonfirmasi", tidak pernah "ilegal" atau "melanggar hukum".
- **Kendali di tangan pengguna:** agent hanya bisa membuat draf email. Aksi kirim Gmail dimatikan di level komponen, jadi agent secara teknis tidak dapat mengirim email. Pengguna meninjau, mengedit, dan mengirim sendiri.
![Draf tersimpan di Gmail dan belum terkirim](docs/screenshots/gmail-draft.png)
- **Bukan nasihat hukum:** setiap laporan ditutup dengan disclaimer dan rujukan ke LBH atau Disnaker.
- **Kredensial aman:** semua key disimpan sebagai Global Variables Langflow, tidak di file flow.

## Keterbatasan

- Hanya memeriksa **PKWT**. PKWTT, magang, outsourcing, dan kemitraan/freelance berada di luar cakupan.
- Korpus hanya berisi **64 entri** dari rezim aturan saat ini. UU Ketenagakerjaan baru yang sedang dibahas DPR belum tercakup.
- Data upah minimum 2026 hanya mencakup wilayah yang tercantum di atas. Lokasi kerja di luar daftar itu dilaporkan sebagai "Tidak dapat diperiksa".
- Sebagian entri korpus diverifikasi dari sumber sekunder (`checked_secondary`) atau satu sumber saja (`accepted_single_source`).
- Retrieval mengambil 4 pasal teratas per query, sehingga pasal relevan bisa saja terlewat pada kontrak yang panjang.
- Redaksi data pribadi berbasis regex dan bisa melewatkan format yang tidak umum.
- Input berupa teks; kontrak dalam bentuk PDF atau foto perlu disalin menjadi teks terlebih dahulu.
- Saat ini hanya bisa digunakan lewat IBM Bob (IDE) atau Playground Langflow.

## Rencana pengembangan

- **Regulasi:** memperbarui korpus ke UU Ketenagakerjaan baru setelah disahkan, cukup dengan menambahkan pasal berlabel rezim baru.
- **Wilayah:** menambahkan upah minimum provinsi dan kabupaten/kota lain.
- **Jenis kontrak:** memperluas cakupan ke PKWTT, magang, dan outsourcing.
- **Distribusi:** integrasi dengan platform lowongan kerja sebagai fitur "cek kontrak", serta lisensi untuk pusat karier kampus dan serikat pekerja.
- **Pasar kedua:** pemeriksaan kepatuhan template kontrak untuk UMKM sebelum merekrut.
- **Antarmuka:** front end web agar dapat digunakan tanpa IDE.

### Memperbarui korpus

1. Tambahkan atau ubah baris di `data/labor_corpus.csv` sesuai kolom di atas. Sertakan header `[...]` di awal kolom `text`, serta `source_url`, `retrieved_on`, dan `verification`.
2. Jika pasal lama diubah atau diganti, ubah `status` dan isi `amended_by`; jangan menghapus barisnya. Pasal dari UU baru diberi `regime` baru.
3. Jalankan ulang `ingest_labor_corpus`. Pertimbangkan koleksi baru (mis. `corpus_v2`) agar tidak ada entri ganda, lalu perbarui nama koleksi di semua flow.

## Struktur repository

```
.
├── .env.example                 # Template variabel lingkungan (tanpa nilai)
├── data/
│   └── labor_corpus.csv         # Korpus regulasi (64 entri) beserta sumbernya
└── langflow/                    # Ekspor JSON flow (berisi prompt lengkap)
    ├── answer_labor_rule_question.json
    ├── check_work_contract.json
    ├── draft_hr_questions.json
    └── ingest_labor_corpus.json
```
