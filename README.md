# DevOps Training Kit

## Overview

Training DevOps lengkap dari Linux hingga Kubernetes, dirancang untuk praktisi IT yang ingin memahami dan menerapkan budaya serta toolchain DevOps secara menyeluruh.

## Target Peserta

| Role | Level |
|---|---|
| Junior DevOps Engineer | Pemula → Menengah |
| Sysadmin | Pemula → Menengah |
| Infrastructure Engineer | Menengah |
| Software Developer | Pemula |

## Prasyarat

- Familiar dengan command line dasar
- Akses ke mesin Linux (VM, WSL, atau cloud instance)
- Koneksi internet

## Struktur Modul

| No | Modul | Lab |
|---|---|---|
| 01 | [Pengenalan DevOps](./01-introduction/module.md) | [Lab 01 — Setup Environment](./01-introduction/labs/lab-01-setup-environment.md) |
| 02 | [Linux Fundamental](./02-linux-fundamental/module.md) | [Lab 02 — User Management](./02-linux-fundamental/labs/lab-02-user-management.md) · [Lab 03 — Permission](./02-linux-fundamental/labs/lab-03-permission.md) |
| 03 | [Networking](./03-networking/module.md) | [Lab 04 — NGINX Reverse Proxy](./03-networking/labs/lab-04-nginx-reverse-proxy.md) |
| 04 | [Git](./04-git/module.md) | [Lab 05 — Git Workflow](./04-git/labs/lab-05-git-workflow.md) |
| 05 | [Docker](./05-docker/module.md) | [Lab 06 — Dockerfile](./05-docker/labs/lab-06-dockerfile.md) · [Lab 07 — Docker Compose](./05-docker/labs/lab-07-docker-compose.md) |
| 06 | [CI/CD](./06-cicd/module.md) | [Lab 08 — GitHub Actions](./06-cicd/labs/lab-08-github-actions.md) |
| 07 | [Kubernetes](./07-kubernetes/module.md) | [Lab 09 — Install K3s](./07-kubernetes/labs/lab-09-k3s-install.md) · [Lab 10 — Deployment](./07-kubernetes/labs/lab-10-deployment.md) |
| 08 | [Monitoring](./08-monitoring/module.md) | [Lab 11 — Monitoring Stack](./08-monitoring/labs/lab-11-monitoring-stack.md) |
| 09 | [DevSecOps](./09-devsecops/module.md) | [Lab 12 — Container Scan](./09-devsecops/labs/lab-12-container-scan.md) |

## Estimasi Waktu

Total durasi pelatihan: **±40 jam**

| Modul | Teori | Praktik |
|---|---|---|
| 01 Pengenalan DevOps | 2 jam | 1 jam |
| 02 Linux Fundamental | 3 jam | 3 jam |
| 03 Networking | 2 jam | 2 jam |
| 04 Git | 2 jam | 2 jam |
| 05 Docker | 3 jam | 4 jam |
| 06 CI/CD | 3 jam | 3 jam |
| 07 Kubernetes | 4 jam | 4 jam |
| 08 Monitoring | 2 jam | 2 jam |
| 09 DevSecOps | 2 jam | 2 jam |

## Cara Menggunakan Modul Ini

1. Baca `module.md` di setiap folder untuk memahami konsep teori.
2. Kerjakan setiap **Lab** secara berurutan di mesin Linux Anda.
3. Setiap lab memiliki **Tujuan**, **Langkah**, dan **Verifikasi** — pastikan verifikasi berhasil sebelum lanjut.
4. Tandai progress Anda di setiap lab.

---

> Dikembangkan untuk kebutuhan internal training. Diperbarui: Mei 2026.
