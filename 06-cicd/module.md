# BAB 6 — CI/CD Pipeline

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami konsep dan tahapan CI/CD pipeline
- Membuat GitHub Actions workflow untuk build, test, dan deploy
- Mengimplementasikan quality gate (lint, unit test, security scan)
- Mengelola secrets dan environment di pipeline

---

## 6.1 Apa itu CI/CD Pipeline?

Pipeline CI/CD adalah rantai otomasi yang membawa kode dari developer ke production secara cepat dan aman.

```
                    CI/CD Pipeline
─────────────────────────────────────────────────────────────
Developer  →  Source Control  →  CI  →  CD  →  Production
─────────────────────────────────────────────────────────────

git push
    │
    ▼
GitHub/GitLab
    │
    ▼
┌──────────────────────────────────────────────────┐
│                 CI Pipeline                       │
│  ┌──────┐  ┌───────┐  ┌──────┐  ┌────────────┐  │
│  │Lint  │→ │ Test  │→ │Build │→ │Security    │  │
│  │Code  │  │(unit) │  │Image │  │Scan        │  │
│  └──────┘  └───────┘  └──────┘  └────────────┘  │
└──────────────────────────────────────────────────┘
    │ (jika semua lulus)
    ▼
┌──────────────────────────────────────────────────┐
│                 CD Pipeline                       │
│  ┌─────────┐  ┌──────────┐  ┌────────────────┐  │
│  │Deploy   │→ │Smoke     │→ │Deploy          │  │
│  │Staging  │  │Test      │  │Production      │  │
│  └─────────┘  └──────────┘  └────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 6.2 GitHub Actions — Konsep Dasar

### Komponen Utama

```yaml
# Hierarki komponen:
Workflow
└── Job (berjalan di runner)
    └── Step (langkah dalam job)
        ├── Action (reusable unit)
        └── Shell command (run: ...)
```

| Komponen | Penjelasan |
|---|---|
| **Workflow** | File YAML di `.github/workflows/`. Satu repo bisa banyak workflow. |
| **Event/Trigger** | Yang memicu workflow: `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| **Job** | Unit kerja yang berjalan di satu runner. Bisa paralel atau sequential. |
| **Runner** | Mesin virtual yang menjalankan job (GitHub-hosted atau self-hosted) |
| **Step** | Satu langkah dalam job: action atau shell command |
| **Action** | Kode reusable yang bisa dipanggil di step (`uses:`) |

### Contoh Workflow Lengkap

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# Trigger
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:          # Manual trigger dari UI

# Environment variable global
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── Job 1: Lint ──────────────────────────────
  lint:
    name: Code Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout kode
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Install linter
        run: pip install flake8 black isort

      - name: Jalankan flake8
        run: flake8 src/ --max-line-length=100

      - name: Cek format kode (black)
        run: black --check src/

  # ─── Job 2: Test ──────────────────────────────
  test:
    name: Unit Test
    runs-on: ubuntu-latest
    needs: lint                 # Hanya jalan jika lint berhasil

    services:                   # Service container untuk test
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt pytest pytest-cov

      - name: Jalankan unit test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            -v

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          fail_ci_if_error: false

  # ─── Job 3: Build & Push Image ────────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login ke GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata untuk Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build dan push image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            APP_VERSION=${{ github.ref_name }}
            BUILD_DATE=${{ github.event.repository.updated_at }}

  # ─── Job 4: Security Scan ─────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - name: Scan image dengan Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build.outputs.image-tag }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload hasil scan
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  # ─── Job 5: Deploy ke Staging ─────────────────
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: [build, security]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy ke staging server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/myapp
            docker compose pull
            docker compose up -d --no-deps app
            docker compose exec app curl -f http://localhost:8080/health

  # ─── Job 6: Deploy ke Production ──────────────
  deploy-production:
    name: Deploy Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com

    steps:
      - name: Deploy ke production
        run: echo "Deploy ke production (manual approval required)"
```

---

## 6.3 Konsep Penting GitHub Actions

### Secrets dan Variables

```yaml
# Gunakan secrets untuk data sensitif
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  API_KEY: ${{ secrets.API_KEY }}
  
# GitHub otomatis menyediakan GITHUB_TOKEN
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  
# Gunakan vars untuk data non-sensitif
  APP_ENV: ${{ vars.APP_ENV }}
```

**Cara menambahkan:**
- Repo → Settings → Secrets and variables → Actions → New repository secret

### Konteks (Context)

```yaml
# Context yang tersedia
${{ github.actor }}          # Username yang memicu event
${{ github.ref }}            # Branch/tag: refs/heads/main
${{ github.ref_name }}       # Hanya nama: main
${{ github.sha }}            # Full commit SHA
${{ github.event_name }}     # push, pull_request, dll
${{ github.repository }}     # owner/repo-name
${{ runner.os }}             # Linux, Windows, macOS
${{ job.status }}            # success, failure, cancelled
```

### Matrix Strategy — Test Banyak Versi

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: python --version
```

### Conditional Steps

```yaml
steps:
  - name: Hanya saat push ke main
    if: github.ref == 'refs/heads/main'
    run: echo "Deploy production!"

  - name: Hanya saat gagal
    if: failure()
    run: |
      curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
        -d '{"text": "❌ Pipeline gagal di ${{ github.repository }}"}'

  - name: Selalu jalan (termasuk saat gagal)
    if: always()
    run: echo "Cleanup resources"
```

### Artifact — Berbagi File Antar Job

```yaml
jobs:
  build:
    steps:
      - name: Build aplikasi
        run: python setup.py bdist_wheel

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist-package
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: dist-package
          path: dist/

      - name: Deploy
        run: ls dist/
```

---

## 6.4 Reusable Workflow

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      SSH_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy ${{ inputs.image-tag }} ke ${{ inputs.environment }}
        run: echo "Deploying..."
```

```yaml
# .github/workflows/main.yml — Panggil reusable workflow
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image-tag: sha-${{ github.sha }}
    secrets:
      SSH_KEY: ${{ secrets.STAGING_SSH_KEY }}
```

---

## 6.5 Self-Hosted Runner

```bash
# Setup runner di server sendiri
# 1. GitHub → Settings → Actions → Runners → New self-hosted runner
# 2. Ikuti instruksi instalasi

# Install dan konfigurasi
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.316.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.316.0/actions-runner-linux-x64-2.316.0.tar.gz
tar xzf actions-runner-linux-x64-2.316.0.tar.gz
./config.sh --url https://github.com/OWNER/REPO --token TOKEN

# Install sebagai service
sudo ./svc.sh install
sudo ./svc.sh start
```

```yaml
# Menggunakan self-hosted runner
jobs:
  build:
    runs-on: self-hosted    # atau label kustom
    # runs-on: [self-hosted, linux, x64]
```

---

## Ringkasan

| Komponen | Penjelasan |
|---|---|
| Workflow | File `.yml` di `.github/workflows/` |
| Trigger | `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| Job | Unit kerja di runner; bisa paralel dengan `needs` |
| Step | Satu langkah: `uses:` (action) atau `run:` (shell) |
| Secrets | Data sensitif tersimpan aman di GitHub Settings |
| Matrix | Jalankan job di berbagai kombinasi (versi OS, bahasa) |
| Artifact | Berbagi file antar job |
| Environment | Stage deployment dengan protection rules dan URL |
