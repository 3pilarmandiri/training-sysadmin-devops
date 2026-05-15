# BAB 9 — DevSecOps

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami filosofi "Security as Code" dan "Shift Left Security"
- Melakukan vulnerability scan pada container image dengan Trivy
- Menganalisis dan memperbaiki celah keamanan di Dockerfile
- Mengelola secrets dengan aman menggunakan Vault atau alternatifnya
- Mengintegrasikan security scan ke dalam pipeline CI/CD

---

## 9.1 Apa itu DevSecOps?

**DevSecOps** adalah evolusi dari DevOps yang mengintegrasikan praktik **keamanan (Security)** secara otomatis ke seluruh tahap SDLC (Software Development Lifecycle), bukan hanya di akhir (sebelum rilis).

### DevOps vs DevSecOps

```
DEVOPS (tanpa security terintegrasi):
Plan → Code → Build → Test → Release → Deploy → Monitor
                                                    ↑
                                       Security audit DI SINI saja
                                       (terlambat, mahal diperbaiki)

DEVSECOPS (security di setiap tahap):
Plan → Code → Build → Test → Release → Deploy → Monitor
  ↓      ↓      ↓       ↓       ↓         ↓        ↓
Threat  SAST  Image   DAST   Signing   RBAC    Runtime
Model   Scan  Scan    Scan   & Policy  Config  Protection
```

### Prinsip "Shift Left Security"

"Shift Left" berarti memindahkan pemeriksaan keamanan **seawal mungkin** dalam proses pengembangan:

| Waktu Temukan | Biaya Perbaikan | Metode |
|---|---|---|
| Saat coding | Sangat murah | IDE plugin, pre-commit hook |
| Saat CI | Murah | SAST, dependency scan |
| Saat testing | Sedang | DAST, integration test |
| Saat production | Sangat mahal | Incident response |

---

## 9.2 Jenis-Jenis Security Scan

### SAST — Static Application Security Testing

Menganalisis **source code** tanpa menjalankan program.

```bash
# Tools SAST populer:

# Python — bandit
pip install bandit
bandit -r src/ -f json -o bandit-report.json

# JavaScript — npm audit
npm audit
npm audit fix

# Semua bahasa — Semgrep
semgrep --config=auto src/

# Secret detection — detect-secrets / trufflehog
detect-secrets scan . --all-files
trufflehog git file://. --only-verified
```

### DAST — Dynamic Application Security Testing

Menganalisis **aplikasi yang sedang berjalan**:

```bash
# OWASP ZAP
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://your-app-url \
  -r zap-report.html

# Nikto
nikto -h http://your-app-url
```

### Dependency/SCA Scan

Memeriksa **dependensi third-party** untuk kerentanan:

```bash
# Python
pip install safety
safety check -r requirements.txt

# JavaScript
npm audit
npx audit-ci --critical

# OWASP Dependency Check
docker run --rm \
  -v $(pwd):/src \
  owasp/dependency-check:latest \
  --project "MyApp" \
  --scan /src
```

### Container Image Scan

Memeriksa kerentanan di **Docker image**:

```bash
# Trivy (paling populer)
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest

# Grype (Anchore)
grype nginx:latest

# Snyk
snyk container test nginx:latest
```

---

## 9.3 Trivy — Container Security Scanner

Trivy adalah scanner keamanan open-source yang dapat memeriksa:
- Container images
- Filesystem
- Git repository
- Kubernetes cluster
- SBOM (Software Bill of Materials)

### Cara Kerja Trivy

```
Docker Image
    │
    ▼
┌──────────────────────────────────────────┐
│                 TRIVY                    │
│                                          │
│  1. Unpack image layers                  │
│  2. Detect OS (Alpine, Debian, dll)      │
│  3. List packages (apk, apt, npm, pip)   │
│  4. Compare dengan CVE database          │
│     (NVD, GitHub Advisory, dll)          │
│  5. Output: List CVE + severity          │
└──────────────────────────────────────────┘
    │
    ▼
Report: CVE-XXXX-YYYY (CRITICAL) — libssl 1.0.2k
        CVE-XXXX-ZZZZ (HIGH)    — python 3.8.10
```

### Severity Level CVE

| Level | CVSS Score | Tindakan |
|---|---|---|
| CRITICAL | 9.0 – 10.0 | Perbaiki segera, jangan deploy |
| HIGH | 7.0 – 8.9 | Perbaiki dalam sprint ini |
| MEDIUM | 4.0 – 6.9 | Rencanakan perbaikan |
| LOW | 0.1 – 3.9 | Terima atau monitor |
| UNKNOWN | — | Investigasi lebih lanjut |

### Format Output Trivy

```bash
# Table (default)
trivy image nginx:alpine

# JSON (untuk CI/CD)
trivy image -f json -o results.json nginx:alpine

# SARIF (untuk GitHub Security tab)
trivy image -f sarif -o results.sarif nginx:alpine

# Template kustom
trivy image --format template \
  --template "@html.tpl" \
  -o report.html nginx:alpine
```

---

## 9.4 Dockerfile Security Best Practices

### 1. Gunakan Base Image Minimal

```dockerfile
# BURUK — terlalu besar, banyak celah keamanan
FROM ubuntu:22.04

# LEBIH BAIK
FROM python:3.12-slim

# TERBAIK — distroless (tidak ada shell, package manager)
FROM gcr.io/distroless/python3-debian12
```

### 2. Jalankan sebagai Non-Root User

```dockerfile
# BURUK
FROM python:3.12-slim
WORKDIR /app
COPY . .
CMD ["python", "app.py"]
# Container berjalan sebagai root! ← Berbahaya

# BAIK
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
CMD ["python", "app.py"]
```

### 3. Jangan Simpan Secret di Image

```dockerfile
# SANGAT BURUK
ENV DB_PASSWORD=secret123
RUN echo "password" > /app/secret.txt

# BURUK (masih tersimpan di layer, bisa dilihat dengan docker history)
RUN export SECRET=xxx && do-something && unset SECRET

# BENAR — secret hanya ada saat runtime
# docker run -e DB_PASSWORD=secret123 myapp
# Atau gunakan Docker Secrets / Kubernetes Secrets
```

### 4. Pin Versi Image dan Package

```dockerfile
# BURUK — tidak diketahui versi apa yang akan dipakai
FROM nginx
FROM python:latest
RUN pip install flask

# BAIK — versi terpinned
FROM nginx:1.25.3-alpine
FROM python:3.12.3-slim-bookworm
RUN pip install flask==3.0.3
```

### 5. Multi-Stage Build untuk Minimalkan Attack Surface

```dockerfile
# Stage build: ada semua tools
FROM python:3.12 AS builder
WORKDIR /build
RUN pip install pyinstaller
COPY . .
RUN pyinstaller --onefile app.py

# Stage production: hanya binary final
FROM debian:12-slim
COPY --from=builder /build/dist/app /usr/local/bin/app
USER nobody
ENTRYPOINT ["/usr/local/bin/app"]
# Tidak ada Python, pip, atau source code!
```

### 6. Scan di Setiap Build

```dockerfile
# Dockerfile dengan HEALTHCHECK
FROM python:3.12-slim

# Security labels
LABEL org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.created="2024-01-01" \
      security.scan.required="true"

# Update packages untuk patch security holes
RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# ...
```

---

## 9.5 Secret Management

### Masalah: Secret Tersebar di Mana-Mana

```
Lokasi berbahaya untuk menyimpan secret:
❌ Hardcode di source code: DB_PASSWORD = "secret123"
❌ Environment variable tidak terenkripsi di Dockerfile
❌ File .env yang di-commit ke Git
❌ Config file di server tanpa enkripsi
❌ Secret dikirim via Slack/email
```

### Solusi 1: GitHub/GitLab Secrets (untuk CI/CD)

```yaml
# Simpan di: Repository → Settings → Secrets
# Akses di pipeline:
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### Solusi 2: Kubernetes Secrets

```bash
# Buat secret
kubectl create secret generic app-secrets \
  --from-literal=db-password=SuperSecret \
  --from-literal=api-key=MyApiKey123

# Gunakan di Pod
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
EOF
```

### Solusi 3: HashiCorp Vault

```bash
# Jalankan Vault (dev mode untuk lab)
docker run -d \
  --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  hashicorp/vault:latest

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN=root

# Simpan secret
vault kv put secret/myapp \
  db-password=SuperSecret \
  api-key=MyApiKey123

# Baca secret
vault kv get secret/myapp
vault kv get -field=db-password secret/myapp

# Integrasi di aplikasi (Python)
# pip install hvac
```

```python
import hvac

client = hvac.Client(url='http://vault:8200', token='root')
secret = client.secrets.kv.v2.read_secret_version(path='myapp')
db_password = secret['data']['data']['db-password']
```

### Solusi 4: Enkripsi Secrets di Git (SOPS)

```bash
# SOPS (Secrets OPerationS) — enkripsi file secret sebelum commit
pip install sops

# Enkripsi dengan AWS KMS atau GPG
sops --encrypt --kms arn:aws:kms:... secrets.yaml > secrets.enc.yaml

# Dekripsi saat digunakan
sops --decrypt secrets.enc.yaml
```

---

## 9.6 Kubernetes Security

### RBAC — Role-Based Access Control

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flask-app-sa
  namespace: devops-lab

---
# Role — hak akses dalam namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: flask-app-role
  namespace: devops-lab
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  # Tidak ada akses ke secrets!

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flask-app-binding
  namespace: devops-lab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: flask-app-role
subjects:
  - kind: ServiceAccount
    name: flask-app-sa
    namespace: devops-lab
```

### Network Policy

```yaml
# Batasi traffic masuk/keluar Pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: flask-app-netpol
  namespace: devops-lab
spec:
  podSelector:
    matchLabels:
      app: flask-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Hanya terima traffic dari ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              app: traefik
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Hanya bisa akses database dan redis
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # DNS (wajib)
    - ports:
        - protocol: UDP
          port: 53
```

### Pod Security Standards

```yaml
# Enforce pod security di namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: devops-lab
  labels:
    pod-security.kubernetes.io/enforce: restricted   # Paling ketat
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 9.7 Integrasi Security ke CI/CD Pipeline

```yaml
# .github/workflows/security.yml
name: Security Pipeline

on: [push, pull_request]

jobs:
  # ─── SAST ─────────────────────────────────────
  sast:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan dengan Bandit (Python SAST)
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-results.json || true

      - name: Detect secrets dengan trufflehog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}

      - name: Upload SAST results
        uses: actions/upload-artifact@v4
        with:
          name: sast-results
          path: bandit-results.json

  # ─── Dependency Scan ──────────────────────────
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan Python dependencies
        run: |
          pip install safety
          safety check -r requirements.txt --json -o safety-report.json || true

  # ─── Container Scan ───────────────────────────
  container-scan:
    name: Container Image Scan
    runs-on: ubuntu-latest
    needs: [sast]
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan dengan Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 1           # Fail jika ada HIGH/CRITICAL

      - name: Upload ke GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

---

## Ringkasan

| Kategori | Tools | Integrasi |
|---|---|---|
| SAST | Bandit, Semgrep, SonarQube | Pre-commit, CI/CD |
| Dependency | Safety, npm audit, Snyk | CI/CD, weekly schedule |
| Container Image | Trivy, Grype, Snyk Container | CI/CD build stage |
| DAST | OWASP ZAP, Nikto | Post-deploy staging |
| Secret Detection | detect-secrets, trufflehog | Pre-commit, CI/CD |
| Secret Management | Vault, K8s Secrets, AWS SSM | Runtime |
| K8s Security | RBAC, NetworkPolicy, PSS | Cluster config |
