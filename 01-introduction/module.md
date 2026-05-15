# BAB 1 — Pengenalan DevOps

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Menjelaskan konsep dan filosofi DevOps
- Membedakan DevOps dengan model pengembangan tradisional (Waterfall, Agile)
- Memahami pipeline CI/CD secara konseptual
- Mengenal ekosistem tools DevOps modern

---

## 1.1 Apa itu DevOps?

**DevOps** adalah gabungan kata **Development** dan **Operations**. DevOps bukan sekadar toolset, melainkan sebuah **budaya dan praktik** yang menjembatani tim pengembang perangkat lunak (Dev) dengan tim operasional infrastruktur (Ops) agar dapat bekerja secara kolaboratif dan efisien.

### Masalah Sebelum DevOps ("Wall of Confusion")

```
   DEV TEAM              OPS TEAM
  ──────────            ──────────
  "It works on         "Doesn't work
  my machine!"          in production!"
        ↑                     ↑
        └──────── GAP ────────┘
```

Tim Dev fokus pada kecepatan fitur baru → Tim Ops fokus pada stabilitas sistem → Konflik terus-menerus.

### Solusi: DevOps

DevOps menjawab masalah ini dengan:
- **Shared responsibility** — Dev dan Ops memiliki tanggung jawab bersama atas kualitas dan kestabilan sistem
- **Automation** — Otomasi proses build, test, dan deploy
- **Continuous feedback** — Monitoring berkelanjutan untuk deteksi masalah lebih awal
- **Collaboration** — Komunikasi dan kerja sama antar tim

---

## 1.2 Prinsip CALMS

DevOps dibangun di atas lima pilar yang dikenal sebagai **CALMS**:

| Pilar | Penjelasan |
|---|---|
| **C**ulture | Membangun budaya kolaborasi, kepercayaan, dan tanggung jawab bersama |
| **A**utomation | Otomasi tugas repetitif: build, test, deploy, provisioning |
| **L**ean | Menghilangkan pemborosan proses, fokus pada nilai yang diterima pelanggan |
| **M**easurement | Mengukur semua aspek: performa, frekuensi deploy, MTTR, dll |
| **S**haring | Berbagi pengetahuan, tools, dan lessons learned antar tim |

---

## 1.3 DevOps vs Waterfall vs Agile

```
WATERFALL:
  Requirements → Design → Development → Testing → Deployment → Maintenance
  (Linier, satu arah, siklus panjang berbulan-bulan)

AGILE:
  Sprint 1 (2 minggu) → Sprint 2 → Sprint 3 → ...
  (Iteratif, tapi Dev dan Ops masih terpisah)

DEVOPS:
  Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → (loop)
  (Kolaboratif, otomatis, loop pendek harian/mingguan)
```

---

## 1.4 CI/CD — Continuous Integration & Continuous Delivery

### Continuous Integration (CI)
Praktik di mana developer secara rutin menggabungkan (merge) kode ke repository bersama, kemudian secara otomatis menjalankan **build** dan **test**.

```
Developer Push Code
        ↓
   Git Repository
        ↓
   CI Server (GitHub Actions, Jenkins, GitLab CI)
        ↓
  ┌─────────────────────────────────┐
  │ 1. Build (compile / package)    │
  │ 2. Unit Test                    │
  │ 3. Static Code Analysis (lint)  │
  │ 4. Security Scan                │
  └─────────────────────────────────┘
        ↓
   Laporan: ✅ PASS / ❌ FAIL
```

**Tujuan CI:** Deteksi bug lebih awal, hindari "integration hell".

### Continuous Delivery (CD)
Kode yang lolos CI secara otomatis disiapkan untuk rilis ke environment (staging/production).

```
CI Pipeline Berhasil
        ↓
   Artifact (Docker Image / JAR / binary)
        ↓
  Deploy ke Staging → Smoke Test
        ↓
  Deploy ke Production (manual approval atau otomatis)
```

### Perbedaan CD — Delivery vs Deployment

| | Continuous Delivery | Continuous Deployment |
|---|---|---|
| Deploy ke staging | Otomatis | Otomatis |
| Deploy ke production | **Manual approval** | **Otomatis penuh** |

---

## 1.5 Ekosistem Tools DevOps

```
PLAN          CODE         BUILD        TEST          RELEASE
──────────    ──────────   ──────────   ──────────    ──────────
Jira          Git          Maven        JUnit         GitHub
Confluence    GitHub       Gradle       Pytest        GitLab
Trello        GitLab       npm          Selenium      Jenkins

DEPLOY        OPERATE      MONITOR      SECURITY
──────────    ──────────   ──────────   ──────────
Ansible       Docker       Prometheus   Trivy
Terraform     Kubernetes   Grafana      SonarQube
Helm          Linux        ELK Stack    Vault
ArgoCD        Nginx        Datadog      OWASP ZAP
```

---

## 1.6 Metrik DevOps (DORA Metrics)

**DORA (DevOps Research and Assessment)** mendefinisikan empat metrik kunci performa tim DevOps:

| Metrik | Definisi | Target Elite |
|---|---|---|
| **Deployment Frequency** | Seberapa sering deploy ke production | Beberapa kali per hari |
| **Lead Time for Changes** | Waktu dari commit hingga production | < 1 jam |
| **Change Failure Rate** | % deploy yang menyebabkan kegagalan | < 5% |
| **MTTR** (Mean Time to Restore) | Waktu rata-rata pemulihan insiden | < 1 jam |

---

## 1.7 Cloud Native

**Cloud Native** adalah pendekatan membangun aplikasi yang dirancang khusus untuk lingkungan cloud:

- **Microservices** — Aplikasi dipecah menjadi layanan-layanan kecil yang independen
- **Containers** — Setiap service dikemas dalam container (Docker)
- **Orchestration** — Container dikelola oleh orkestrator (Kubernetes)
- **Dynamic management** — Scaling, self-healing, load balancing otomatis
- **DevOps practices** — CI/CD, observability, infrastructure as code

---

## Ringkasan

| Konsep | Inti |
|---|---|
| DevOps | Budaya kolaborasi Dev + Ops dengan otomasi |
| CI | Integrasi kode otomatis + test otomatis |
| CD | Rilis otomatis ke environment target |
| Cloud Native | Aplikasi dirancang untuk cloud dengan container & microservices |
| DORA Metrics | Ukur performa tim DevOps secara objektif |

## Referensi

- [The DevOps Handbook](https://itrevolution.com/the-devops-handbook/) — Gene Kim et al.
- [DORA State of DevOps Report](https://dora.dev/)
- [CNCF Cloud Native Landscape](https://landscape.cncf.io/)
