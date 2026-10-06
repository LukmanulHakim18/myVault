---
title: "Service Spec Generation Prompt"
type: prompt-template
tags: [template, prompt, meta-spec, mrg]
created: 2026-04-08
updated: 2026-04-08
---

# Service Spec Generation Prompt

Template prompt untuk generate dokumentasi service spec standar MRG.
Referensi output: [[02-Work/Teams/MRG/02-services/authservice/README|Auth Service Spec]]

---

## 📋 Cara Penggunaan

1. Copy seluruh blok prompt di bawah ke chat Claude baru
2. Isi semua placeholder `{{...}}`
3. Pastikan Claude memiliki akses Desktop Commander dan Obsidian MCP
4. Claude akan tampilkan preview — konfirmasi "lanjut" sebelum ditulis ke Obsidian

---

## 🤖 Prompt Template


```
Saya ingin kamu membuat service spec documentation standar MRG.

---

### INPUT DATA

**Nama Service**: {{nama_service}}
**Folder source code lokal**: {{path_folder}}
**Output Obsidian path**: 02-Work/Teams/MRG/02-services/{{nama_folder_obsidian}}/

**Informasi tambahan (kosongkan jika tidak tahu):**
- Owner / PIC      : {{nama_pic}}
- Email kontak     : {{email_pic}}
- Tier             : {{P0 / P1 / P2 / P3}}
- SLO Availability : {{99.9% / 99.5% / dll}}
- SLO Latency p99  : {{<200ms / <500ms / dll}}
- Runbook link     : {{link atau kosongkan}}

---

### LANGKAH 1 — BACA SOURCE CODE

Baca file-file berikut dari `{{path_folder}}` menggunakan Desktop Commander:

**A. API Contracts** — folder `contract/`
- `*.proto` → semua gRPC service definition, method list, interceptors (di comment), message types
- `*.yaml` → REST endpoint mapping (HTTP method + path per gRPC method)
- Jika tidak ada `.yaml`, baca `*.swagger.json` sebagai alternatif

**B. Dependencies** — folder `repository/repoiface/`
- Baca **semua file `.go`** di folder ini
- Setiap file = satu interface dependency
- Ekstrak: nama interface, nama method, parameter, return type
- Nama file menunjukkan nama dependency (contoh: `redis.go` → Redis, `user.go` → User Service)

**C. Deployment** — folder `k8s/`
- `deployment.yaml` → namespace, replicas, resource limits, health check config, env vars, secrets yang di-mount
- `service.yaml` → port configuration
- `huawei-application.yaml` → cluster name, ArgoCD project/repo

**D. CI/CD** — file `Jenkinsfile`
- VERSION_PREFIX → versi service
- Setiap stage `Deploy to *` → branch name, environment (dev/stg/regress/prod), NAMESPACE_HUAWEI

**E. Configuration** — file `config/default.go`
- Semua key di `defaultConfig` map → nama config, default value, kategori

**F. Usecase** — folder `usecase/`
- List nama file (tanpa `_test.go`) → representasi business logic / fitur yang ada

**G. Module & Dependencies** — file `go.mod`
- Module name → repository URL
- Go version
- Semua `require` block → library name + versi

---

### LANGKAH 2 — EKSTRAK DAN KELOMPOKKAN DATA

Setelah membaca semua file, kelompokkan data sebagai berikut:

**API:**
- Tabel gRPC methods: nama method, HTTP method, REST path, interceptors, request type, response type
- Tandai method yang di-comment (`//`) sebagai "deprecated/disabled"

**Dependencies:**
- Internal services: nama service, protocol (gRPC/HTTP/PubSub), Go interface methods-nya
- Infrastructure: Redis, Kafka, PubSub, dll → beserta method interface-nya
- Library versions: dari go.mod

**Config:**
- Kelompokkan config key per kategori: App, JWT/Token, Redis, OTP, FDS, User Service, Notification, Security, Legacy

**Deployment:**
- Tabel: environment → branch → namespace → cluster
- Secrets yang di-mount
- Health check (readiness + liveness)

**Usecase / Business Logic:**
- Kelompokkan usecase file per flow/kategori

---

### LANGKAH 3 — TAMPILKAN PREVIEW

Tampilkan ringkasan sebelum menulis:
- List file Obsidian yang akan dibuat
- Data yang berhasil diekstrak per sumber file
- Field yang tidak ditemukan di source code (akan diisi `_TODO: isi manual_`)

**Tunggu konfirmasi "lanjut" sebelum menulis ke Obsidian.**

---

### LANGKAH 4 — TULIS KE OBSIDIAN

Buat file-file berikut di path `02-Work/Teams/MRG/02-services/{{nama_folder_obsidian}}/`:
```

#### FILE 1: `README.md` — Main Service Spec

Frontmatter wajib:
```yaml
title: "{{Nama Service}}"
type: service-documentation
team: MRG
status: production
owner: {{email_pic}}
tier: {{P0/P1/P2/P3}}
version: {{VERSION_PREFIX dari Jenkinsfile}}
go_version: {{dari go.mod}}
grpc_port: {{dari service.yaml}}
rest_port: {{dari service.yaml}}
slo_availability: "{{nilai}}"
slo_latency_p99: "{{nilai}}"
repository: {{module name dari go.mod}}
tags: [mrg, service, {{nama_service}}, documentation]
created: {{tanggal hari ini}}
updated: {{tanggal hari ini}}
```

Sections (urutan tetap):
1. **Overview** — deskripsi fungsi utama (dari nama usecase files + interface methods)
2. **Service Identity** — tabel: service name, repo, team, owner, contact, tier, status, version, go version
3. **SLO** — tabel: availability, latency p99, error rate
4. **Tech Stack** — tabel: language, protocol, database, cache, message queue, monitoring, security, container, CI/CD
5. **Deployment** — tabel K8s (namespace/env, cluster, cloud, replicas, resource limits, ports) + tabel CI/CD branch→env + health check YAML snippet
6. **Konsep Utama** — 3-5 poin arsitektur utama (derive dari interface + usecase)
7. **Dependencies** — tabel internal services + tabel client libraries (nama, versi dari go.mod, purpose) + tabel infrastructure
8. **API Contracts** — tabel per kategori: method, HTTP method, REST path, interceptors, description
9. **Business Flows** — ringkasan flow utama dari usecase files, link ke `flows.md`
10. **Configuration** — tabel env vars per kategori (dari config/default.go)
11. **Security Measures** — rate limiting, token security, K8s secrets
12. **Runbook & Incident** — tabel: PIC, runbook link, APM service name, common issues
13. **Project Structure** — tree folder top-level
14. **Related Documentation** — internal links ke file lain

---

#### FILE 2: `api-reference.md` — API Detail

Sections:
- Server configuration (ports table dari service.yaml)
- Per kategori gRPC method:
  - Protobuf signature lengkap (`rpc Method(Request) returns (Response)`)
  - HTTP mapping (method + path dari .yaml)
  - Interceptors
  - Request message fields (dari .proto)
  - Response message fields (dari .proto)
- Common message types di bagian akhir

---

#### FILE 3: `dependencies.md` — Dependency Map

Sections:
- Mermaid graph TD: upstream (internal services + infra) + downstream consumers
- Tabel detail upstream: service name, protocol, library, version, purpose
- Go interface per dependency (dari repoiface/*.go) — tampilkan method signatures lengkap
- Configuration per dependency: env var host + port (dari config/default.go)
- Downstream consumers table (jika diketahui)

---

#### FILE 4: `flows.md` — Business Flows

Buat hanya jika ada flow yang jelas dari usecase files.
Kelompokkan usecase files menjadi flows (contoh: login flow, registration flow, dll).

Per flow:
- Mermaid sequence diagram (actor: Client, Service, dependencies yang terlibat)
- Step-by-step penjelasan
- Key points (security, timing, edge cases dari interface methods)

Flow comparison matrix di bagian akhir.

---

### ATURAN FORMAT

- Bahasa Indonesia untuk penjelasan naratif, English untuk code dan technical terms
- Mermaid diagram harus valid syntax
- Method yang di-comment di proto → tandai sebagai `~~deprecated~~` di tabel
- Internal links: `[[filename|Display Name]]`
- Jika data tidak tersedia: gunakan `_TODO: isi manual_`
- Footer tiap file:
  ```
  *Last Updated*: {{tanggal hari ini}}
  *Generated from*: Repository analysis — {{path_folder}}
  ```
```

---

## 📝 Placeholder Reference

| Placeholder | Contoh |
|-------------|--------|
| `{{nama_service}}` | `authservice`, `userservice` |
| `{{path_folder}}` | `D:\code\go\mybb-ms\authservice` |
| `{{nama_folder_obsidian}}` | `authservice`, `userservice` |
| `{{nama_pic}}` | `Alfian Maulana` |
| `{{email_pic}}` | `alfian.maulana@bluebirdgroup.com` |
| `{{Tier}}` | `P0`, `P1`, `P2`, `P3` |
| `{{SLO Availability}}` | `99.9%`, `99.5%` |
| `{{SLO Latency p99}}` | `<200ms`, `<500ms` |

---

## 🗂️ Struktur Output

```
02-Work/Teams/MRG/02-services/{{nama_service}}/
├── README.md          ← Main spec (wajib)
├── api-reference.md   ← Full gRPC + REST reference (wajib)
├── dependencies.md    ← Dependency graph + interfaces (wajib)
└── flows.md           ← Business flows + sequence diagrams (opsional)
```

---

## ✅ Checklist Quality

- [ ] Frontmatter: semua field terisi, tidak ada yang kosong (pakai `_TODO_` jika belum)
- [ ] API tabel: semua gRPC methods ada, termasuk yang disabled/deprecated
- [ ] REST path mapping: semua method ada HTTP method + path-nya
- [ ] Dependencies: semua file repoiface terwakili
- [ ] Go interface signatures: lengkap di `dependencies.md`
- [ ] Config: semua key dari `config/default.go` terdokumentasi
- [ ] CI/CD: semua branch → environment mapping ada
- [ ] K8s secrets terdokumentasi
- [ ] Mermaid diagram valid
- [ ] Common issues / runbook stub ada
- [ ] Last Updated = tanggal hari ini

---

## 🔄 Changelog

| Versi | Tanggal | Perubahan |
|-------|---------|-----------|
| 1.0 | 2026-04-08 | Initial version |
| 1.1 | 2026-04-08 | Perbaikan sumber data: tambah contract/yaml, repoiface, config/default.go, usecase |

---

*Referensi output*: [[02-Work/Teams/MRG/02-services/authservice/README|Auth Service Spec]]
