# LAB 08 — GitHub Actions CI/CD Pipeline

## Tujuan
Membuat GitHub Actions workflow yang lengkap untuk aplikasi Flask: lint → test → build Docker image → push ke registry → notify.

## Prasyarat
- Lab 05 (Git) dan Lab 06 (Docker) selesai
- Akun GitHub aktif
- Repository GitHub (buat baru jika belum ada)

## Estimasi Waktu
60 menit

---

## Skenario

Anda akan membuat pipeline CI/CD untuk proyek Flask API dari Lab 06. Pipeline akan otomatis berjalan setiap kali ada push ke branch `main` atau `develop`, dan saat Pull Request dibuat.

---

## Langkah 1: Siapkan Repository

```bash
# Gunakan proyek Flask dari Lab 06 atau buat baru
mkdir -p ~/devops-labs/cicd/flask-cicd
cd ~/devops-labs/cicd/flask-cicd

git init
git branch -M main
```

Buat struktur proyek:

```bash
mkdir -p src tests .github/workflows

# Aplikasi Flask
cat > src/app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

VERSION = os.getenv("APP_VERSION", "1.0.0")

@app.route("/")
def index():
    return jsonify({"app": "Flask CI/CD Demo", "version": VERSION, "status": "running"})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

@app.route("/add/<int:a>/<int:b>")
def add(a, b):
    return jsonify({"result": a + b, "operation": f"{a} + {b}"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
EOF

# Unit tests
cat > tests/test_app.py << 'EOF'
import pytest
import sys
sys.path.insert(0, 'src')

from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_index(client):
    response = client.get('/')
    assert response.status_code == 200
    data = response.get_json()
    assert data['status'] == 'running'
    assert 'version' in data

def test_health(client):
    response = client.get('/health')
    assert response.status_code == 200
    assert response.get_json()['status'] == 'healthy'

def test_add(client):
    response = client.get('/add/3/4')
    assert response.status_code == 200
    assert response.get_json()['result'] == 7

def test_add_negative(client):
    response = client.get('/add/-5/10')
    assert response.status_code == 200
    assert response.get_json()['result'] == 5
EOF

# Requirements
cat > requirements.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
EOF

cat > requirements-dev.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
pytest==8.2.0
pytest-cov==5.0.0
flake8==7.0.0
black==24.3.0
isort==5.13.2
EOF

# Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "src.app:app"]
EOF

# .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
.env
venv/
.coverage
htmlcov/
.pytest_cache/
dist/
*.egg-info/
EOF

# Setup.cfg untuk konfigurasi tools
cat > setup.cfg << 'EOF'
[flake8]
max-line-length = 100
exclude = venv,.git,__pycache__

[isort]
profile = black
multi_line_output = 3
EOF
```

---

## Langkah 2: Commit Kode Awal

```bash
git add .
git commit -m "feat: initial Flask CI/CD demo project"
```

---

## Langkah 3: Buat Workflow CI Lengkap

```bash
cat > .github/workflows/ci.yml << 'EOF'
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

env:
  PYTHON_VERSION: "3.12"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── Job 1: Lint & Format Check ──────────────
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest

    steps:
      - name: Checkout kode
        uses: actions/checkout@v4

      - name: Setup Python ${{ env.PYTHON_VERSION }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
          cache-dependency-path: requirements-dev.txt

      - name: Install dev dependencies
        run: pip install -r requirements-dev.txt

      - name: Lint dengan flake8
        run: |
          echo "::group::Flake8 Output"
          flake8 src/ tests/ --statistics
          echo "::endgroup::"

      - name: Cek format dengan black
        run: black --check src/ tests/

      - name: Cek urutan import dengan isort
        run: isort --check-only src/ tests/

  # ─── Job 2: Unit Test ────────────────────────
  test:
    name: Unit Test
    runs-on: ubuntu-latest
    needs: lint

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
          cache-dependency-path: requirements-dev.txt

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Jalankan pytest dengan coverage
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v \
            --tb=short

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage.xml
          retention-days: 7

  # ─── Job 3: Build Docker Image ───────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login ke GitHub Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            APP_VERSION=${{ github.ref_name }}-${{ github.run_number }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Output image info
        run: |
          echo "## Docker Image Info" >> $GITHUB_STEP_SUMMARY
          echo "| Key | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|-----|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| Tags | ${{ steps.meta.outputs.tags }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Digest | ${{ steps.build.outputs.digest }} |" >> $GITHUB_STEP_SUMMARY

  # ─── Job 4: Smoke Test Image ─────────────────
  smoke-test:
    name: Smoke Test Image
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - name: Login ke Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Jalankan container dan test
        run: |
          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-$(git rev-parse --short HEAD)"
          
          echo "Pulling image: $IMAGE"
          docker pull "$IMAGE"
          
          echo "Menjalankan container..."
          docker run -d --name smoke-test -p 8080:8080 "$IMAGE"
          
          echo "Menunggu startup..."
          sleep 10
          
          echo "Test health endpoint..."
          for i in {1..5}; do
            if curl -sf http://localhost:8080/health; then
              echo "✅ Health check berhasil"
              break
            fi
            echo "Retry $i..."
            sleep 3
          done
          
          echo "Test endpoint lain..."
          curl -sf http://localhost:8080/ | grep -q "running"
          curl -sf http://localhost:8080/add/5/3 | grep -q "8"
          
          echo "✅ Smoke test berhasil"

      - name: Cleanup
        if: always()
        run: |
          docker stop smoke-test 2>/dev/null || true
          docker rm smoke-test 2>/dev/null || true

  # ─── Job 5: Notifikasi ───────────────────────
  notify:
    name: Notifikasi
    runs-on: ubuntu-latest
    needs: [lint, test, build]
    if: always() && github.ref == 'refs/heads/main'

    steps:
      - name: Summary Pipeline
        run: |
          echo "## 🚀 Pipeline Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Job | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|-----|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| Lint | ${{ needs.lint.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Test | ${{ needs.test.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Build | ${{ needs.build.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "Commit: ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "Branch: ${{ github.ref_name }}" >> $GITHUB_STEP_SUMMARY
          echo "Triggered by: ${{ github.actor }}" >> $GITHUB_STEP_SUMMARY
EOF
```

---

## Langkah 4: Buat Workflow CD (Deploy)

```bash
cat > .github/workflows/cd.yml << 'EOF'
name: CD Pipeline

on:
  workflow_run:
    workflows: [CI Pipeline]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy ke Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Deploy ke staging
        run: |
          echo "🚀 Deploying ke Staging..."
          echo "Image: ghcr.io/${{ github.repository }}:latest"
          echo ""
          echo "Simulasi deploy command:"
          echo "  ssh ubuntu@staging.example.com"
          echo "  docker pull ghcr.io/${{ github.repository }}:latest"
          echo "  docker compose up -d --no-deps app"
          echo ""
          echo "✅ Deploy staging berhasil (simulasi)"

      - name: Update summary
        run: |
          echo "## ✅ Staging Deployment" >> $GITHUB_STEP_SUMMARY
          echo "Environment: staging" >> $GITHUB_STEP_SUMMARY
          echo "URL: https://staging.example.com" >> $GITHUB_STEP_SUMMARY
EOF
```

---

## Langkah 5: Buat Workflow Manual Deployment

```bash
cat > .github/workflows/deploy-manual.yml << 'EOF'
name: Manual Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options:
          - staging
          - production
      image-tag:
        description: Docker image tag (contoh: sha-abc123)
        required: false
        default: latest
      dry-run:
        description: Dry run (tidak benar-benar deploy)
        type: boolean
        default: true

jobs:
  deploy:
    name: Deploy ${{ inputs.image-tag }} ke ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - name: Validasi input
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Image tag: ${{ inputs.image-tag }}"
          echo "Dry run: ${{ inputs.dry-run }}"
          echo "Triggered by: ${{ github.actor }}"

      - name: Deploy (atau dry run)
        run: |
          if [ "${{ inputs.dry-run }}" == "true" ]; then
            echo "🔍 DRY RUN — tidak ada perubahan yang dilakukan"
            echo "Command yang akan dijalankan:"
            echo "  docker pull ghcr.io/${{ github.repository }}:${{ inputs.image-tag }}"
            echo "  docker compose up -d --no-deps app"
          else
            echo "🚀 Deploying ke ${{ inputs.environment }}..."
            echo "✅ Deploy berhasil"
          fi
EOF
```

---

## Langkah 6: Push ke GitHub

```bash
# Commit semua workflow file
git add .github/
git commit -m "ci: tambah GitHub Actions workflow CI/CD lengkap"

# Push ke GitHub (pastikan sudah ada remote)
git remote add origin https://github.com/USERNAME/flask-cicd.git
# Atau via SSH:
# git remote add origin git@github.com:USERNAME/flask-cicd.git

git push -u origin main
```

---

## Langkah 7: Monitor Pipeline di GitHub

1. Buka repository di GitHub
2. Klik tab **Actions**
3. Lihat workflow **CI Pipeline** berjalan
4. Klik job untuk melihat log detail setiap step
5. Setelah selesai, cek **Summary** untuk ringkasan

---

## Langkah 8: Simulasikan Failure

```bash
# Buat kode yang punya lint error
cat >> src/app.py << 'EOF'


def bad_function(x,y,z):
    return x+y+z  # flake8 akan protes: E231, E501
EOF

git add src/app.py
git commit -m "test: kode dengan lint error (intentional)"
git push
```

Buka GitHub Actions — pipeline harus **gagal** di stage Lint. Ini adalah behavior yang diinginkan!

```bash
# Fix lint error
cat > src/app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

VERSION = os.getenv("APP_VERSION", "1.0.0")


@app.route("/")
def index():
    return jsonify({"app": "Flask CI/CD Demo", "version": VERSION, "status": "running"})


@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200


@app.route("/add/<int:a>/<int:b>")
def add(a, b):
    return jsonify({"result": a + b, "operation": f"{a} + {b}"})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
EOF

git add src/app.py
git commit -m "fix: perbaiki lint error"
git push
```

---

## Langkah 9: Pull Request Workflow

```bash
# Buat branch fitur
git checkout -b feature/add-multiply

# Tambah fitur baru
cat >> src/app.py << 'EOF'


@app.route("/multiply/<int:a>/<int:b>")
def multiply(a, b):
    return jsonify({"result": a * b, "operation": f"{a} * {b}"})
EOF

# Tambah test
cat >> tests/test_app.py << 'EOF'


def test_multiply(client):
    response = client.get('/multiply/4/5')
    assert response.status_code == 200
    assert response.get_json()['result'] == 20
EOF

git add .
git commit -m "feat: tambah endpoint multiply"
git push -u origin feature/add-multiply
```

Buat Pull Request di GitHub — pipeline akan otomatis jalan untuk PR ini. CI akan memvalidasi kode sebelum merge diizinkan.

---

## Langkah 10: Tambah Branch Protection

Di GitHub:
1. Repository → Settings → Branches → Add rule
2. Branch name pattern: `main`
3. Centang:
   - ✅ Require status checks to pass before merging
   - ✅ Require branches to be up to date before merging
   - Status checks: `lint`, `test`, `build`
   - ✅ Require pull request reviews before merging

Sekarang tidak ada yang bisa push langsung ke `main` tanpa melewati CI!

---

## Verifikasi Akhir

Jalankan secara lokal untuk memastikan semua langkah CI bisa direproduksi:

```bash
cd ~/devops-labs/cicd/flask-cicd

# Install dev deps
pip install -r requirements-dev.txt

echo "=== Langkah 1: Lint ==="
flake8 src/ tests/ && echo "✅ Lint OK" || echo "❌ Lint gagal"

echo ""
echo "=== Langkah 2: Format Check ==="
black --check src/ tests/ && echo "✅ Format OK" || echo "❌ Format perlu diperbaiki"

echo ""
echo "=== Langkah 3: Unit Test ==="
pytest tests/ --cov=src --cov-report=term-missing -v

echo ""
echo "=== Langkah 4: Build Docker ==="
docker build -t flask-cicd:local .
echo "✅ Build OK"

echo ""
echo "=== Langkah 5: Smoke Test ==="
CID=$(docker run -d -p 8080:8080 flask-cicd:local)
sleep 5
curl -sf http://localhost:8080/health && echo "✅ Smoke test OK" || echo "❌ Smoke test gagal"
docker stop $CID && docker rm $CID
docker rmi flask-cicd:local
```

---

## Checkpoint ✅

- [ ] Workflow CI berjalan otomatis saat push
- [ ] Pipeline gagal saat ada lint error (dan berhasil setelah diperbaiki)
- [ ] Docker image berhasil di-build dan di-push ke GHCR
- [ ] Smoke test memverifikasi image berjalan dengan benar
- [ ] Pipeline PR berjalan untuk Pull Request
- [ ] Branch protection rules aktif di `main`
- [ ] Summary pipeline tampil di GitHub Actions UI
