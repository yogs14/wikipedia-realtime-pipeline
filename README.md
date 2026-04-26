# 🌐 Wikipedia Real-Time Data Pipeline
> **End-to-End Modern Data Stack Implementation**

Proyek ini adalah implementasi *End-to-End Data Pipeline* yang mengekstrak aliran data *real-time* dari **Wikipedia API**, memprosesnya menggunakan **Apache Spark**, mengorkestrasinya dengan **Apache Airflow**, menyimpannya ke **ClickHouse Warehouse**, dan memvisualisasikannya melalui **Metabase**. 

Arsitektur ini dirancang sebagai materi pendamping praktikum/pelatihan *Big Data* yang mengadopsi prinsip-prinsip dari buku referensi *"Mining of Massive Datasets"* (MMDS).

## 🏗️ Arsitektur Sistem

Aliran data (*Data Flow*) dalam sistem ini dirancang menggunakan pendekatan *Micro-batching*:

1. **Data Source:** Wikipedia Recent Changes API.
2. **Ingestion & Data Lake:** Python `requests` menyedot 500 riwayat suntingan terbaru setiap 2 menit dan menyimpannya secara lokal dalam format `.parquet`.
3. **Data Processing:** Apache Spark (PySpark) membaca *file* Parquet, melakukan agregasi (kalkulasi *Heavy Hitters*, deteksi aktivitas *Bot* vs Manusia, dan penghitungan volume *bytes*).
4. **Data Warehouse:** Hasil agregasi dimuat (Di-*load* / *Truncate-Insert*) ke dalam tabel analitik di **ClickHouse**.
5. **Orchestration:** Seluruh proses di atas dijadwalkan dan diawasi sepenuhnya oleh **Apache Airflow**.
6. **Data Visualization:** **Metabase** terhubung langsung ke ClickHouse untuk menyajikan metrik *real-time dashboard*.

## 🛠️ Tech Stack

* **Orchestration:** Apache Airflow 2.9
* **Processing Engine:** Apache Spark (PySpark 3.5)
* **Data Warehouse:** ClickHouse (Column-oriented OLAP)
* **Business Intelligence:** Metabase
* **Infrastructure:** Docker & Docker Compose
* **Language:** Python 3.11

## 📂 Struktur Proyek

```text
wikipedia-realtime-pipeline/
├── dags/
│   ├── scripts/
│   │   ├── fetch_wikipedia_stream.py   # Script ekstraksi dari API ke Data Lake
│   │   ├── process_wikipedia_spark.py  # Script PySpark untuk agregasi & Load ke ClickHouse
│   │   └── wikipedia_pipeline.py       # Definisi DAG Airflow
├── data_lake/                          # Folder lokal penampungan file .parquet
├── docker-compose.yml                  # Konfigurasi infrastruktur service
├── Dockerfile                          # Custom Airflow image (dengan instalasi Java JRE)
├── requirements.txt                    # Dependensi pustaka Python
├── .gitignore                          
└── README.md
```
## 🚀 Cara Menjalankan (Lokal)

### Prasyarat
Pastikan Anda sudah menginstal [Docker](https://docs.docker.com/get-docker/) dan [Docker Compose](https://docs.docker.com/compose/install/) di sistem Anda.

### Langkah-langkah
1. **Clone Repository:**
   ```bash
   git clone [https://github.com/USERNAME_ANDA/wikipedia-realtime-pipeline.git](https://github.com/USERNAME_ANDA/wikipedia-realtime-pipeline.git)
   cd wikipedia-realtime-pipeline
   ```

2. **Bangun Custom Image (Instalasi Java & Dependensi Python):**
   ```bash
   docker-compose build
   ```

3. **Jalankan Seluruh Layanan:**
   ```bash
   docker-compose up -d
   ```

4. **Inisialisasi Airflow:**
   * Buka Airflow UI (cek tabel akses di bawah).
   * Nyalakan (Toggle *Unpause*) DAG `wikipedia_realtime_stream`.
   * Airflow akan mulai menarik data setiap 2 menit secara otomatis.

## 🔐 Akses Layanan & Port

Seluruh layanan berjalan di atas *container* Docker. Anda dapat mengakses antarmuka penggunanya melalui *browser* di localhost Anda:

| Layanan | URL Akses | Username | Password |
| :--- | :--- | :--- | :--- |
| **Apache Airflow** | `http://localhost:8080` | `admin` | `admin` |
| **Metabase** | `http://localhost:3000` | *(Buat akun baru saat setup awal)* | - |
| **ClickHouse (TCP)**| `localhost:9000` | `admin` | `rahasia` |

## 📊 Metrik yang Dianalisis

Visualisasi *dashboard* di Metabase menjawab pertanyaan bisnis berikut secara *real-time*:
* **Top 30 Heavy Hitters:** Artikel apa yang saat ini paling sering disunting atau sedang mengalami perang suntingan (*edit war*)?
* **Volume Data:** Seberapa banyak *bytes* karakter teks yang ditambahkan/dihapus?
* **Bot vs Human Edits:** Seberapa dominan peran *script* otonom (bot) dalam merawat artikel Wikipedia hari ini?

---
*Dibuat untuk keperluan Pelatihan Data Engineering & Big Data Analytics.*


**Catatan Sebelum Menekan Commit:**
1. Jangan lupa untuk mengganti tulisan `USERNAME_ANDA` di bagian **Langkah-langkah** (langkah nomor 1) dengan *username* GitHub Anda yang sesungguhnya.
2. Anda bisa langsung melakukan `git add README.md`, `git commit -m "docs: add comprehensive README"`, lalu `git push`!
