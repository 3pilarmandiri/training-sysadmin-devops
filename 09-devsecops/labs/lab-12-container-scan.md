# LAB 12 — DevSecOps: Container Security Scan

## Tujuan
Melakukan security assessment pada Docker image menggunakan Trivy, menganalisis vulnerabilities, memperbaiki Dockerfile yang tidak aman, dan mengintegrasikan scan ke pipeline CI/CD.

## Prasyarat
- Docker terinstall (Lab 01)
- Git dan akun GitHub (Lab 05)
- Lab 06 (Dockerfile) selesai

## Estimasi Waktu
60 menit

---

## Skenario

Anda adalah Security Engineer yang diminta mengaudit container image yang digunakan tim. Anda harus:
1. Scan dan analisis vulnerabilities
2. Perbaiki Dockerfile yang tidak aman
3. Scan ulang dan verifikasi perbaikan
4. Integrasikan scan ke pipeline CI/CD

---

## Langkah 1: Install Trivy

```bash
# Install Trivy via script
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | \
  sudo sh -s -- -b /usr/local/bin

# Verifikasi
trivy --version

# Update database CVE
trivy image --download-db-only
```

---

## Langkah 2: Scan Image "Berbahaya" (Sengaja Dibuat Insecure)

```bash
mkdir -p ~/devops-labs/security
cd ~/devops-labs/security

# Buat Dockerfile yang SENGAJA tidak aman (untuk dipelajari)
cat > Dockerfile.insecure << 'EOF'
# ⚠️ JANGAN GUNAKAN INI DI PRODUCTION — ini contoh yang SALAH!

# Masalah 1: Base image terlalu lama (banyak CVE)
FROM python:3.8

# Masalah 2: Berjalan sebagai root
# (tidak ada USER directive)

# Masalah 3: Secret hardcoded
ENV DB_PASSWORD=SuperSecret123
ENV API_KEY=abc123xyz-production-key
ENV ADMIN_TOKEN=hardcoded-admin-token

WORKDIR /app

COPY requirements-insecure.txt requirements.txt

# Masalah 4: Versi package tidak di-pin
RUN pip install flask requests

# Masalah 5: Package rentan di-install
RUN pip install pillow==8.0.0    # CVE tersedia

COPY app.py .

EXPOSE 8080

# Masalah 6: Jalankan sebagai root
CMD ["python", "app.py"]
EOF

cat > requirements-insecure.txt << 'EOF'
flask==2.0.0
requests==2.25.0
pillow==8.0.0
pyyaml==5.3.1
EOF

cat > app.py << 'EOF'
from flask import Flask, jsonify, request
import os

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({"status": "running"})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
EOF

# Build image insecure
docker build -t myapp:insecure -f Dockerfile.insecure .
```

---

## Langkah 3: Scan Image Insecure

```bash
echo "=== SCAN IMAGE INSECURE ==="
echo "Ini akan memakan waktu beberapa menit..."

# Scan dasar — tampilkan semua severity
trivy image myapp:insecure 2>/dev/null | tail -50

echo ""
echo "=== RINGKASAN VULNERABILITIES ==="
trivy image --format json myapp:insecure 2>/dev/null | \
  python3 << 'PYEOF'
import json, sys

data = json.load(sys.stdin)
results = data.get("Results", [])

total = {"CRITICAL": 0, "HIGH": 0, "MEDIUM": 0, "LOW": 0, "UNKNOWN": 0}

for result in results:
    vulns = result.get("Vulnerabilities", []) or []
    for v in vulns:
        sev = v.get("Severity", "UNKNOWN")
        total[sev] = total.get(sev, 0) + 1

print(f"Hasil scan: myapp:insecure")
print(f"{'='*40}")
for sev, count in total.items():
    emoji = {"CRITICAL": "🔴", "HIGH": "🟠", "MEDIUM": "🟡", "LOW": "🟢", "UNKNOWN": "⚪"}
    print(f"{emoji.get(sev, '')} {sev}: {count}")
print(f"{'='*40}")
print(f"Total vulnerabilities: {sum(total.values())}")
PYEOF
```

---

## Langkah 4: Lihat Detail Vulnerabilities Kritis

```bash
# Filter hanya CRITICAL
echo "=== CRITICAL VULNERABILITIES ==="
trivy image --severity CRITICAL --format table myapp:insecure 2>/dev/null

echo ""
echo "=== HIGH VULNERABILITIES ==="
trivy image --severity HIGH --format table myapp:insecure 2>/dev/null | head -40

# Generate report HTML
trivy image --format template \
  --template "@/usr/local/share/trivy/templates/html.tpl" \
  -o trivy-report-insecure.html \
  myapp:insecure 2>/dev/null || \
  trivy image --format json -o trivy-report-insecure.json myapp:insecure 2>/dev/null

echo ""
echo "Laporan tersimpan: trivy-report-insecure.json"
```

---

## Langkah 5: Scan Image "Aman" (Bandingkan)

```bash
# Scan image Alpine yang sudah terupdate
echo "=== SCAN IMAGE ALPINE (UPDATED) ==="
trivy image --severity HIGH,CRITICAL python:3.12-alpine 2>/dev/null | tail -20

echo ""
# Bandingkan distroless
echo "=== SCAN DISTROLESS IMAGE ==="
trivy image --severity HIGH,CRITICAL gcr.io/distroless/python3 2>/dev/null | tail -10
```

---

## Langkah 6: Buat Dockerfile yang Aman

```bash
cat > Dockerfile.secure << 'EOF'
# ✅ Dockerfile yang AMAN — ikuti ini untuk production

# Fix 1: Gunakan base image terbaru, versi spesifik
FROM python:3.12.3-slim-bookworm

# Fix 2: Update package untuk patch keamanan
RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

WORKDIR /app

# Fix 3: Pin versi semua dependencies
COPY requirements-secure.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

# Fix 4: Buat user non-root
RUN groupadd -r appuser --gid=1001 && \
    useradd -r -g appuser --uid=1001 --no-create-home appuser

# Fix 5: Set ownership
RUN chown -R appuser:appuser /app

# Fix 6: Jalankan sebagai non-root
USER appuser

# Fix 7: Expose port, bukan publish
EXPOSE 8080

# Fix 8: Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Fix 9: Gunakan form exec (bukan shell)
ENTRYPOINT ["python3"]
CMD ["app.py"]

# Fix 10: Metadata untuk traceability
LABEL org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.description="Secure Flask App" \
      security.rootless="true" \
      security.scan.required="true"
EOF

cat > requirements-secure.txt << 'EOF'
flask==3.0.3
requests==2.31.0
pillow==10.3.0
pyyaml==6.0.1
gunicorn==22.0.0
EOF

# Build image secure
docker build -t myapp:secure -f Dockerfile.secure .
```

---

## Langkah 7: Scan Image yang Sudah Diperbaiki

```bash
echo "=== SCAN IMAGE SECURE ==="
trivy image --severity HIGH,CRITICAL myapp:secure 2>/dev/null

echo ""
echo "=== PERBANDINGAN RINGKASAN ==="

# Hitung CVE insecure
INSECURE_CRIT=$(trivy image --severity CRITICAL --format json myapp:insecure 2>/dev/null | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print(sum(len((r.get('Vulnerabilities') or [])) for r in d.get('Results', [])))")

SECURE_CRIT=$(trivy image --severity CRITICAL --format json myapp:secure 2>/dev/null | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print(sum(len((r.get('Vulnerabilities') or [])) for r in d.get('Results', [])))")

echo "┌─────────────────────────────┬────────────┐"
echo "│ Image                       │ CRITICAL   │"
echo "├─────────────────────────────┼────────────┤"
echo "│ myapp:insecure              │ $INSECURE_CRIT          │"
echo "│ myapp:secure                │ $SECURE_CRIT          │"
echo "└─────────────────────────────┴────────────┘"
```

---

## Langkah 8: Scan Secrets dan Misconfiguration

```bash
# Scan misconfigurations di Dockerfile
echo "=== MISCONFIGURATION SCAN — Dockerfile Insecure ==="
trivy config --severity HIGH,CRITICAL Dockerfile.insecure 2>/dev/null || \
  trivy fs --security-checks config Dockerfile.insecure 2>/dev/null

echo ""
echo "=== MISCONFIGURATION SCAN — Dockerfile Secure ==="
trivy config --severity HIGH,CRITICAL Dockerfile.secure 2>/dev/null

echo ""
# Scan untuk exposed secrets di image filesystem
echo "=== SECRET SCAN di image ==="
trivy image --security-checks secret myapp:insecure 2>/dev/null | grep -A3 "Secrets"
```

---

## Langkah 9: Scan Filesystem dan Repository

```bash
# Scan directory untuk kerentanan dan secrets
echo "=== SCAN FILESYSTEM ==="
trivy fs --severity HIGH,CRITICAL . 2>/dev/null

# Cari file yang berisi pattern secret
echo ""
echo "=== MENCARI SECRET DI FILE ==="
cat > detect-secrets.sh << 'EOF'
#!/bin/bash
echo "Mencari pola secret yang umum..."
patterns=(
    "password\s*=\s*['\"][^'\"]{6,}"
    "api_key\s*=\s*['\"][^'\"]{6,}"
    "secret\s*=\s*['\"][^'\"]{6,}"
    "token\s*=\s*['\"][^'\"]{6,}"
    "AWS_SECRET"
    "BEGIN RSA PRIVATE KEY"
    "BEGIN EC PRIVATE KEY"
)

for pattern in "${patterns[@]}"; do
    echo "--- Pattern: $pattern ---"
    grep -rn --include="*.py" --include="*.yaml" --include="*.yml" \
         --include="*.env" --include="*.txt" \
         -i "$pattern" . 2>/dev/null | grep -v ".git"
done
EOF

chmod +x detect-secrets.sh
./detect-secrets.sh
```

---

## Langkah 10: Buat Security Gate di Script

```bash
cat > security-gate.sh << 'EOF'
#!/bin/bash
# Security gate: gagalkan build jika ada CRITICAL CVE

IMAGE="${1:-myapp:secure}"
MAX_CRITICAL=0
MAX_HIGH=5

echo "=== Security Gate untuk: $IMAGE ==="

# Scan dan simpan hasil
REPORT=$(trivy image --format json --quiet "$IMAGE" 2>/dev/null)

# Hitung CVE per severity
CRITICAL=$(echo "$REPORT" | python3 -c "
import json, sys
d = json.load(sys.stdin)
total = sum(len([v for v in (r.get('Vulnerabilities') or []) if v.get('Severity') == 'CRITICAL'])
           for r in d.get('Results', []))
print(total)
" 2>/dev/null || echo "0")

HIGH=$(echo "$REPORT" | python3 -c "
import json, sys
d = json.load(sys.stdin)
total = sum(len([v for v in (r.get('Vulnerabilities') or []) if v.get('Severity') == 'HIGH'])
           for r in d.get('Results', []))
print(total)
" 2>/dev/null || echo "0")

echo "CRITICAL: $CRITICAL (max: $MAX_CRITICAL)"
echo "HIGH: $HIGH (max: $MAX_HIGH)"

# Evaluasi
PASSED=true
if [ "$CRITICAL" -gt "$MAX_CRITICAL" ]; then
    echo "❌ GAGAL: $CRITICAL CRITICAL vulnerabilities ditemukan (max: $MAX_CRITICAL)"
    PASSED=false
fi

if [ "$HIGH" -gt "$MAX_HIGH" ]; then
    echo "❌ GAGAL: $HIGH HIGH vulnerabilities ditemukan (max: $MAX_HIGH)"
    PASSED=false
fi

if [ "$PASSED" = true ]; then
    echo "✅ Security gate LULUS!"
    exit 0
else
    echo "🚨 Security gate GAGAL — jangan deploy image ini!"
    exit 1
fi
EOF

chmod +x security-gate.sh

echo "=== Test Security Gate dengan image INSECURE ==="
./security-gate.sh myapp:insecure || echo ""

echo ""
echo "=== Test Security Gate dengan image SECURE ==="
./security-gate.sh myapp:secure && echo ""
```

---

## Langkah 11: GitHub Actions Security Pipeline

```bash
mkdir -p .github/workflows

cat > .github/workflows/security.yml << 'EOF'
name: Security Scan Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    # Scan harian jam 08:00 WIB (00:00 UTC)
    - cron: "0 0 * * *"

jobs:
  # ─── Dependency Vulnerability Scan ───────────
  dependency-scan:
    name: Dependency Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dan scan dengan Safety
        run: |
          pip install safety
          safety check -r requirements-secure.txt \
            --json -o safety-report.json || true

      - name: Upload hasil scan
        uses: actions/upload-artifact@v4
        with:
          name: dependency-scan-report
          path: safety-report.json

  # ─── SAST Scan ────────────────────────────────
  sast:
    name: SAST — Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Scan dengan Bandit
        run: |
          pip install bandit
          bandit -r . -f json -o bandit-report.json \
            --exclude ./.git,./venv || true
          bandit -r . --severity-level medium

  # ─── Container Image Scan ─────────────────────
  container-scan:
    name: Container Image Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build -t myapp:${{ github.sha }} -f Dockerfile.secure .

      - name: Scan dengan Trivy — tabel
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --severity HIGH,CRITICAL \
            myapp:${{ github.sha }}

      - name: Scan dengan Trivy — SARIF
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 0           # Jangan fail dulu, upload dulu

      - name: Upload ke GitHub Security Tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

      - name: Security Gate — fail jika CRITICAL
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          severity: CRITICAL
          exit-code: 1           # Fail jika ada CRITICAL

  # ─── Secret Detection ─────────────────────────
  secret-scan:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0         # Scan seluruh history

      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified
EOF
```

---

## Langkah 12: SBOM (Software Bill of Materials)

SBOM adalah inventaris lengkap semua komponen di dalam software/image:

```bash
# Generate SBOM dalam format CycloneDX
trivy image --format cyclonedx \
  -o sbom-myapp.json \
  myapp:secure 2>/dev/null

echo "=== Isi SBOM (ringkasan) ==="
python3 << 'EOF'
import json

with open('sbom-myapp.json') as f:
    sbom = json.load(f)

components = sbom.get('components', [])
print(f"Total komponen: {len(components)}")
print(f"\n{'='*50}")
print(f"{'Nama':<30} {'Versi':<20} {'Tipe'}")
print(f"{'='*50}")
for comp in components[:20]:  # Tampilkan 20 pertama
    print(f"{comp.get('name', ''):<30} {comp.get('version', ''):<20} {comp.get('type', '')}")
if len(components) > 20:
    print(f"... dan {len(components)-20} komponen lainnya")
EOF
```

---

## Verifikasi Akhir

```bash
echo "=== Verifikasi Lab 12 ==="
echo -n "Trivy terinstall: "
  trivy --version &>/dev/null && echo "✅" || echo "❌"
echo -n "Image insecure ter-scan: "
  [ -f trivy-report-insecure.json ] && echo "✅" || echo "⏭️ (skip jika tidak ada)"
echo -n "Dockerfile.secure dibuat: "
  [ -f Dockerfile.secure ] && echo "✅" || echo "❌"
echo -n "Image secure ter-build: "
  docker image inspect myapp:secure &>/dev/null && echo "✅" || echo "❌"
echo -n "Security gate berjalan: "
  [ -f security-gate.sh ] && bash security-gate.sh myapp:secure 2>/dev/null && echo "✅" || echo "⚠️"
echo -n "GitHub Actions workflow terbuat: "
  [ -f .github/workflows/security.yml ] && echo "✅" || echo "❌"
echo -n "SBOM terbuat: "
  [ -f sbom-myapp.json ] && echo "✅" || echo "❌"
```

---

## Perbandingan Sebelum vs Sesudah

```bash
echo "=== LAPORAN AKHIR PERBANDINGAN ==="
echo ""
echo "BEFORE (Dockerfile.insecure):"
echo "  - Base: python:3.8 (EOL, banyak CVE)"
echo "  - Berjalan sebagai: root"
echo "  - Secret: hardcoded di ENV"
echo "  - Dependencies: tidak di-pin, versi lama"
echo ""
echo "AFTER (Dockerfile.secure):"
echo "  - Base: python:3.12.3-slim-bookworm (terbaru)"
echo "  - Berjalan sebagai: appuser (UID 1001)"
echo "  - Secret: diberikan via env runtime"
echo "  - Dependencies: di-pin ke versi aman"
echo "  - Healthcheck: ada"
echo "  - Labels: traceability lengkap"
```

---

## Cleanup

```bash
docker rmi myapp:insecure myapp:secure 2>/dev/null
echo "Cleanup selesai"
```

---

## Checkpoint ✅

- [ ] Trivy berhasil terinstall dan database CVE diunduh
- [ ] Image insecure berhasil discan dan hasilnya dianalisis
- [ ] Memahami severity level (CRITICAL, HIGH, MEDIUM, LOW)
- [ ] Dockerfile.secure berhasil dibuat dengan perbaikan
- [ ] Image secure menunjukkan jumlah CVE yang lebih sedikit
- [ ] Security gate script berhasil membedakan image aman dan tidak aman
- [ ] GitHub Actions security workflow terbuat
- [ ] SBOM berhasil di-generate
- [ ] Memahami perbedaan SAST, DAST, dan Container Scan
