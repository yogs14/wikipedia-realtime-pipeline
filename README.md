# 🌐 Wikipedia Real-Time Data Pipeline

> End-to-End Modern Data Stack Implementation · Materi Pendamping Pelatihan Big Data

Proyek ini mengekstrak aliran data *real-time* dari **Wikipedia API**, memprosesnya dengan **Apache Spark**, mengorkestrasinya via **Apache Airflow**, menyimpannya ke **ClickHouse**, dan memvisualisasikannya di **Metabase** — dikemas dalam satu perintah Docker.

Arsitektur mengadopsi prinsip dari *"Mining of Massive Datasets"* (MMDS) dengan pendekatan **micro-batching** setiap 2 menit.

---

## 🏗️ Arsitektur Sistem

```
Wikipedia API
     ↓  (500 edits / 2 menit)
[Ingestion — Python requests]
     ↓  simpan .parquet
[Data Lake — folder lokal]
     ↓  baca & agregasi
[Processing — Apache Spark]
     ↓  truncate-insert
[Data Warehouse — ClickHouse]
     ↓  koneksi langsung
[Dashboard — Metabase]

↻  Seluruh siklus diatur oleh Apache Airflow
```

**Metrik yang dianalisis:**
- **Top 30 Heavy Hitters** — artikel paling sering diedit / sedang edit war
- **Volume Bytes** — total karakter ditambah/dihapus per artikel
- **Bot vs Human Edits** — seberapa dominan bot dalam merawat artikel hari ini

---

## 🛠️ Tech Stack

| Komponen | Teknologi |
|----------|-----------|
| Orchestration | Apache Airflow 2.9 |
| Processing | Apache Spark / PySpark 3.5 |
| Data Warehouse | ClickHouse (column-oriented OLAP) |
| BI & Dashboard | Metabase |
| Infrastructure | Docker & Docker Compose |
| Language | Python 3.11 |

---

## 📂 Struktur Proyek

```
wikipedia-realtime-pipeline/
├── dags/
│   ├── scripts/
│   │   ├── fetch_wikipedia.py       # Ekstraksi API → Data Lake
│   │   └── process_spark.py         # PySpark: agregasi & load ke ClickHouse
│   └── wikipedia_pipeline.py        # Definisi DAG Airflow
├── data_lake/                       # Penyimpanan sementara file .parquet
├── docker-compose.yml               # Konfigurasi seluruh service
├── Dockerfile                       # Custom Airflow image (+ Java JRE)
├── requirements.txt                 # Dependensi Python
└── .gitignore
```

---

## 🚀 Tutorial — Menjalankan dari Nol

### Prasyarat
- [Docker Desktop](https://docs.docker.com/get-docker/) sudah terinstal dan **sedang menyala**

---

### Step 1 — Buat Struktur Folder

```bash
mkdir wikipedia-realtime-pipeline
cd wikipedia-realtime-pipeline
```

```bash
# Buat subfolder Airflow dan penyimpanan data mentah
mkdir -p dags/scripts data_lake
```

```bash
# Buat semua file (masih kosong, diisi di Step 2)
touch docker-compose.yml Dockerfile requirements.txt .gitignore
touch dags/wikipedia_pipeline.py
touch dags/scripts/fetch_wikipedia.py
touch dags/scripts/process_spark.py
```

> `dags/` → dibaca otomatis oleh Airflow  
> `dags/scripts/` → logika utama agar folder `dags/` tetap rapi  
> `data_lake/` → penyimpanan sementara `.parquet` hasil ingest

---

### Step 2 — Isi File Konfigurasi & Kode

Isi masing-masing file dengan script dari repo referensi:

| File | Fungsi |
|------|--------|
| `requirements.txt` | Library Python yang diinstal otomatis (`requests`, `pandas`, dll) |
| `Dockerfile` | Instruksi merakit container Airflow + dependensi custom |
| `docker-compose.yml` | Urutan nyala service: Postgres → Airflow → ClickHouse |
| `fetch_wikipedia.py` | Tarik 500 data terbaru dari API, simpan sebagai `.parquet` |
| `process_spark.py` | Baca `.parquet` → agregasi → load ke ClickHouse → hapus file mentah |
| `wikipedia_pipeline.py` | DAG Airflow: jadwal & urutan task |

> ⚠️ **Perhatikan dua hal ini sebelum lanjut:**  
> • Di `wikipedia_pipeline.py` — atur interval ke `*/2 * * * *` (setiap 2 menit)  
> • Di `fetch_wikipedia.py` — ganti bagian `email` dengan email kamu sendiri

---

### Step 3 — Jalankan Docker

```bash
# Build image dari Dockerfile (unduh & rakit semua komponen)
docker-compose build
```

```bash
# Inisialisasi database internal Airflow — cukup sekali saja
docker-compose up airflow-init
```

```bash
# Jalankan seluruh stack di background
docker-compose up -d
```

> Tunggu 1–2 menit setelah `up -d`, lalu buka **http://localhost:8080**

---

### Step 4 — Aktifkan Pipeline di Airflow

1. Buka **http://localhost:8080** → login `admin` / `admin`
2. Temukan DAG **`wikipedia_realtime_stream`**, geser sakelar untuk mengaktifkan
3. Klik ▶️ **Trigger DAG** untuk memaksanya jalan sekarang

**Yang terjadi di balik layar:**

```
[Trigger]
    ↓
[Task 1: fetch_wikipedia]  →  API Wikipedia → 500 data → simpan .parquet ✅
    ↓
[Task 2: process_spark]    →  baca .parquet → agregasi → load ClickHouse → hapus .parquet ✅
    ↓
[Menunggu 2 menit berikutnya...]
```

---

### Step 5 — Validasi Data di ClickHouse

```bash
# Masuk ke shell ClickHouse di dalam container
docker exec -it wikipedia-realtime-pipeline-clickhouse-server-1 clickhouse-client --user admin --password rahasia
```

```sql
-- Pastikan database 'analytics' sudah ada
SHOW DATABASES;
USE analytics;

-- Cek struktur tabel dan jumlah data
DESCRIBE analytics.wikipedia_trending;
SELECT COUNT(*) FROM analytics.wikipedia_trending;
```

```sql
-- 10 artikel paling banyak diedit
-- Kolom: title, total_edits, total_bytes_changed, bot_edits
SELECT * FROM wikipedia_trending LIMIT 10;
```

```sql
-- Artikel yang aktif diedit oleh bot
SELECT title, total_edits, bot_edits
FROM wikipedia_trending
WHERE bot_edits > 0
ORDER BY bot_edits DESC;
```

```sql
exit
```

---

### Step 6 — Visualisasi di Metabase

1. Buka **http://localhost:3000**, isi data diri (boleh dummy)
2. Di halaman **Add your data**, isi koneksi berikut:

| Field | Value |
|-------|-------|
| Database type | ClickHouse |
| Display name | Data Warehouse Wikipedia |
| Host | `clickhouse-server` ⚠️ *bukan* `localhost` |
| Port | `8123` |
| Database name | `analytics` |
| Username | `admin` |
| Password | `rahasia` |

3. Klik **+ New → Question** → pilih **wikipedia_trending** → klik **Visualize**
4. Pilih format grafik (misal: Bar Chart)
5. Di Dashboard, atur **auto-refresh = 1 menit**

---

### Step 7 — Matikan Infrastruktur

```bash
# Jalankan dari dalam folder project
docker-compose down
```

> Mematikan semua container dengan aman dan membebaskan RAM

---

## 🔐 Referensi Akses Layanan

| Layanan | URL | Username | Password |
|---------|-----|----------|----------|
| Apache Airflow | http://localhost:8080 | `admin` | `admin` |
| Metabase | http://localhost:3000 | *(buat saat setup)* | — |
| ClickHouse TCP | `localhost:9000` | `admin` | `rahasia` |

---

*Dibuat untuk keperluan Pelatihan Data Engineering & Big Data Analytics.*  
*Sebelum push ke GitHub, ganti `USERNAME_ANDA` di URL clone dengan username GitHub kamu.*
