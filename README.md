## 📌 Deskripsi Proyek

Proyek ini bertujuan untuk menganalisis ekosistem olahraga global (khususnya Olimpiade dan event terkait) dalam rentang tahun **2000–2026** menggunakan pendekatan **Knowledge Graph** dan **Graph Data Science (Neo4j)**.

Alih-alih hanya melihat siapa juara berdasarkan medali, proyek ini menjawab pertanyaan *"Siapa juara bertahan?"* dari perspektif:

* Struktur jaringan olahraga
* Keterhubungan antar entitas
* Pengaruh dan dominasi dalam graph

---

## 🧠 Latar Belakang

Olimpiade memiliki sejarah panjang sejak **776 SM di Olympia, Yunani** dan terus berkembang hingga era modern melalui inisiatif **Baron Pierre de Coubertin**.

Namun, pertanyaan menarik muncul:

> Apakah "juara" hanya ditentukan oleh medali, atau bisa dilihat dari dominasi dalam jaringan global?

---

## 🔍 Metodologi

### 1. 📥 Pengumpulan Data

#### 🔹 Wikidata (Data Utama)

Digunakan untuk mengambil:

* Event olahraga
* Cabang olahraga (cabor)
* Lokasi
* Tahun

```sparql
SELECT ?event ?eventLabel ?caborLabel ?lokasiLabel ?tahun
WHERE {
  ?event wdt:P31 wd:Q16510064 .
  ?event wdt:P641 ?cabor .
  ?event wdt:P17 ?lokasi .
  ?event wdt:P580 ?tanggal .

  BIND(YEAR(?tanggal) AS ?tahun)
  FILTER(?tahun >= 2000 && ?tahun <= 2026)

  SERVICE wikibase:label {
    bd:serviceParam wikibase:language "en".
  }
}
ORDER BY DESC(?tahun)
LIMIT 5000
```

---

#### 🔹 DBpedia (Data Pelengkap)

Digunakan untuk memperkaya informasi cabang olahraga.

```sparql
SELECT ?namaOlimpiade (GROUP_CONCAT(DISTINCT ?lokasi; separator=", ") AS ?lokasiLengkap) ?tahun
WHERE {
  ?event a dbo:SportsEvent ;
         dbo:location ?loc ;
         rdfs:label ?namaOlimpiade ;
         dbo:startDate ?startDate .

  ?loc rdfs:label ?lokasi .
  
  BIND(YEAR(?startDate) AS ?tahun)
  
  FILTER (lang(?namaOlimpiade) = "en")
  FILTER (lang(?lokasi) = "en")
  FILTER (?tahun >= 2000 && ?tahun <= 2026)
}
GROUP BY ?namaOlimpiade ?tahun
ORDER BY ?tahun
```

---

### 2. 🔗 Integrasi Data (Python - Pandas)

Langkah yang dilakukan:

* **Schema Alignment**
* **Concatenation**
* **Data Cleaning** (menghapus QID)

```python
import pandas as pd

df_total = pd.concat([df_wiki, df_dbpedia], axis=0, ignore_index=True)
df_clean = df_total[~df_total['nama_event'].str.contains('^Q\d+$', na=False)]
```

---

### 3. 📊 Statistik Dataset

| Sumber Data  | Jumlah Cabor | Status          |
| ------------ | ------------ | --------------- |
| Wikidata     | 75           | Perlu Pengayaan |
| DBpedia      | 11           | Pelengkap       |
| **Gabungan** | **86**       | Siap Analisis   |

---

## 🕸️ Graph Modeling (Neo4j)

### 🔹 Struktur Graph

* **Node:**

  * Event
  * Cabor
  * Lokasi
* **Relasi:**

  * `CABANG_DARI`
  * `DISELENGGARAKAN_DI`

---

### 🔹 Import Data

```cypher
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/polydeuces30/ETS-Graf/refs/heads/main/df_concat.csv' AS row

MERGE (e:Event {nama: row.nama_event})
SET e.tahun = toInteger(row.tahun)

MERGE (c:Cabor {nama: row.cabang_olahraga})
MERGE (l:Lokasi {nama: row.lokasi})

MERGE (e)-[:CABANG_DARI]->(c)
MERGE (e)-[:DISELENGGARAKAN_DI]->(l);
```

---

### 🔹 Graph Projection

```cypher
CALL gds.graph.project(
'olahragaGraph',
['Event', 'Cabor', 'Lokasi'],
['CABANG_DARI', 'DISELENGGARAKAN_DI']
);
```

---

## ⚙️ Analisis Graph

### 1. 🧩 Community Detection (Louvain)

* Mengidentifikasi komunitas dalam jaringan
* Ditemukan **Super Cluster (ID: 596)** dengan **851 entitas**
* Dominasi: **Figure Skating**

---

### 2. ⭐ Centrality (PageRank)

* Mengukur pengaruh node
* **Amerika Serikat (US)** memiliki skor tertinggi: **8.947**

👉 Menjadi pusat jaringan olahraga global

---

### 3. 🔗 Similarity (Node Similarity)

* Mengidentifikasi event dengan struktur identik
* Banyak event memiliki skor **1.0 (identik)**

👉 Contoh:

* Event Compak Sporting Ukraina
* Biathlon World Cup

---

## 📈 Insight Utama

### 🥇 1. Skating sebagai "Juara Bertahan"

* Komunitas terbesar dan paling padat
* Konsistensi tinggi dalam jaringan

### 🌍 2. Amerika Serikat sebagai Pusat Dunia

* Node paling berpengaruh
* Hub global olahraga

### 🔁 3. Stabilitas Event Global

* Event besar memiliki struktur tetap
* Konsistensi lokasi & cabang olahraga

---

## 🎯 Kesimpulan

"Juara bertahan" dalam konteks graph bukan hanya tentang kemenangan, tetapi tentang:

* **Koneksi**
* **Konsistensi**
* **Pengaruh dalam jaringan**

👉 Olahraga adalah sistem terstruktur yang memiliki pusat kekuatan yang bertahan dari waktu ke waktu.


## 🔗 Tautan

* 📦 Repository: [https://github.com/polydeuces30/ETS-Graf](https://github.com/polydeuces30/ETS-Graf)
* 📈 Visualisasi: Neo4j Browser
