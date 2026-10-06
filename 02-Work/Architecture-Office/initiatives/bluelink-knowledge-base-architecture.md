1. apa yang ada di dalamnya?
   ini contoh template doc inputannya
siapa yang men-trigger pembuatannya?
- arstek yang telah selesai merancang system akan mentriger dokumen di project servicenya 
- ketika deploy jenkins akan triger mencopy semua doc yang ada di project tersebut ke repo ini https://git.bluebird.id/tools/bluelink-knowledgebase nanti di grouping berdasarkan masing masing tim, contoh 
  - mrg (./services/mrg/{nama-service})
itu dari tool apa di Bluebird?
- clickup ini card PRD inputnya
  
Update Meta-Spec: Manual atau Auto?
- jenkins yang langsung copy ke central repo ci/cd
- ketika deploy jenkins akan triger mencopy semua doc yang ada di project tersebut ke repo ini https://git.bluebird.id/tools/bluelink-knowledgebase nanti di grouping berdasarkan masing masing tim, contoh 
  - mrg (./services/MRG/{nama-service})
5. untuk rancangan document didalam repo aku mau ini dari 
   - /arc:feature "nama" --card FEAT-123
     - ini akan menggenerate docs ke folder ini :
	       - mrg (./docs/feature/{kode_prd}-nama feature.md)
	       - jika ada senggolan dependencis dll yang mengarah perubahan spec repo tersebut maka meta-specs juga di update didalam folder ini: ./docs/meta-spec
     - isinya dengan format :
       
   ```
   ---

category: architecture

date: '2026-04-10'

source: adr

status: published

summary: Arsitektur pencarian dokumen di Bluelink menggunakan Neo4j sebagai navigation

  layer, diikuti targeted vault read — menggantikan full-text scan yang tidak scalable

  untuk jutaan file.

tags:

- architecture

- ai

- infra

title: Bluelink Graph-First Search Architecture

---

  

**## Konteks**

  

Sistem knowledge base Bluebird membutuhkan pencarian dokumen yang scalable. Pendekatan full-text search (FTS) klasik melakukan scan seluruh dokumen — tidak efisien saat jumlah file mencapai ribuan hingga jutaan.

  

****Masalah utama FTS di skala besar:****

- Scan 10.000 file × rata-rata 5KB = 50MB per query

- Latency naik linear seiring jumlah file

- Tidak memanfaatkan relasi antar dokumen

  

**## Isi Utama**

  

**### Pendekatan: Graph-First RAG**

  

```

Query user

    ↓

[1] Ekstrak keywords dari query

    ↓

[2] Neo4j — cari Document nodes yang relevan

    via CONTAINS pada title/summary + relasi Category/Section

    ↓

[3a] Jika ketemu → baca hanya file yang relevan dari vault (targeted read)

[3b] Jika tidak ketemu → fallback ke FTS vault (full scan, terbatas)

    ↓

[4] Re-rank hasil: graph_score × 3 + keyword_density

    ↓

Kembalikan top-N hasil ke user

```

  

**### Kenapa Neo4j sebagai navigation layer?**

  

| | FTS Langsung | Graph-First |

|---|---|---|

| 1.000 file | ~5MB scan | ~50KB targeted |

| 100.000 file | ~500MB scan | ~50KB targeted |

| 1.000.000 file | ~5GB scan | ~50KB targeted |

  

Neo4j query complexity: ****O(log n)**** via index — tidak terpengaruh jumlah file.

  

**### Syarat agar graph-first efektif**

  

Setiap dokumen ****wajib**** diindex ke Neo4j via `enrich_neo4j.py` dengan:

- `summary` terisi di frontmatter (dipakai sebagai teks pencarian di graph)

- `tags` atau `category` terisi (untuk link ke Category node)

- `status: published`

  

**### Komponen yang terlibat**

  

| Komponen | Peran |

|---|---|

| `markdown-vault-mcp` | Storage dokumen (filesystem) |

| `neo4j` | Graph index — Document, Category, Section nodes |

| `mcp-gateway` | Aggregator + implementasi graph-first search |

| `enrich_neo4j.py` | Sync vault → Neo4j (jalankan manual atau terjadwal) |

  

**## Keputusan / Hasil**

  

****Keputusan:**** Implementasi graph-first sebagai strategi default `bluelink_search`.

  

****Alasan:****

- FTS tidak scalable untuk target jutaan dokumen

- Neo4j sudah ada di infrastruktur — tidak perlu tambah komponen baru

- Graph relationship memberi konteks tambahan yang FTS tidak punya

  

****Trade-off yang diterima:****

- Dokumen baru harus di-enrich dulu ke Neo4j sebelum bisa ditemukan via graph

- Jika enrich belum jalan, fallback ke FTS tetap berfungsi

  

**## Referensi**

  

- `gateway/main.py` — implementasi `bluelink_search` handler

- `scripts/enrich_neo4j.py` — pipeline sync vault → Neo4j

- `guides/_template.md` — template standar dokumen
   ```


1. inputan terkait feature bisa dari clickup atau yang lain atau sedang brainstorm. namun ketika arsitek menjalankan comand ini:
   -  /arc:feature "nama" --card FEAT-123 akan generate hasil brainstrom.
     
    saya lupa pada template adr ada development status tiap feature [development, staging, regress, production ]
     Siapa yang isi content ADR-nya?
     - ketika menerima comand ini :/arc:feature "nama" --card FEAT-123 
     - claude code akan menyimpulkan dari brainstorm dan membca prd dari product dan telah di setujui oleh arsitek.
       
1.  dari meta specs ada disana ada timnya 
	```
	title: "n2cservice"

type: service-documentation

reader: enera

team: MRG
	```
Jenkins script: siapa yang buat?
- nanti kita buat sh file dokument untuk mengupdate jadi tinggal sisipin di jenkins file 