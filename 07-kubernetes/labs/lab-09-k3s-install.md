# LAB 09 — Install dan Konfigurasi K3s

## Tujuan
Menginstal K3s (Kubernetes ringan) sebagai cluster single-node, memverifikasi komponennya, dan mempelajari operasi dasar kubectl.

## Prasyarat
- Lab 01 (environment siap) selesai
- Minimum 2 CPU, 2GB RAM, 20GB disk
- Akses sudo
- Koneksi internet

## Estimasi Waktu
30 menit

---

## Mengapa K3s?

K3s adalah distribusi Kubernetes yang dioptimalkan oleh Rancher:
- **Ringan**: Binary tunggal < 100MB
- **Mudah install**: Satu command
- **Production-ready**: Digunakan di edge, IoT, dan lab
- **Kompatibel penuh**: Semua manifest K8s standar berjalan di K3s

---

## Langkah 1: Persiapan Sistem

```bash
# Cek spesifikasi sistem
echo "=== Sistem Info ==="
echo "CPU Cores: $(nproc)"
echo "RAM: $(free -h | grep Mem | awk '{print $2}')"
echo "Disk Free: $(df -h / | tail -1 | awk '{print $4}')"
echo "OS: $(lsb_release -ds)"
echo "Kernel: $(uname -r)"

# Pastikan hostname terset
hostnamectl set-hostname k3s-master
hostname

# Disable swap (rekomendasi K8s)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verifikasi swap off
free -h    # Baris Swap harus 0
```

---

## Langkah 2: Install K3s

```bash
# Install K3s (server/master)
curl -sfL https://get.k3s.io | sh -

# Tunggu hingga selesai (sekitar 1-2 menit)
# Output yang diharapkan:
# [INFO] Installing k3s to /usr/local/bin/k3s
# [INFO] Creating /usr/local/lib/systemd/system/k3s.service
# [INFO] Starting k3s
```

Verifikasi instalasi:

```bash
# Cek status service K3s
sudo systemctl status k3s

# Cek versi
k3s --version
kubectl version --short 2>/dev/null || kubectl version

# Cek nodes (gunakan sudo karena kubeconfig di /etc/rancher/k3s/)
sudo k3s kubectl get nodes
```

---

## Langkah 3: Konfigurasi kubectl untuk User Biasa

```bash
# Buat direktori .kube
mkdir -p ~/.kube

# Salin kubeconfig dari K3s
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# Set environment variable
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc

# Test tanpa sudo
kubectl get nodes
kubectl get nodes -o wide
```

Output yang diharapkan:
```
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   2m    v1.29.x+k3s1
```

---

## Langkah 4: Jelajahi Komponen Cluster

```bash
echo "=== Cluster Info ==="
kubectl cluster-info

echo ""
echo "=== Nodes ==="
kubectl get nodes -o wide

echo ""
echo "=== Semua Namespace ==="
kubectl get namespaces

echo ""
echo "=== System Pods ==="
kubectl get pods -n kube-system

echo ""
echo "=== Semua Pods di Semua Namespace ==="
kubectl get pods -A

echo ""
echo "=== Storage Classes ==="
kubectl get storageclass

echo ""
echo "=== Services di kube-system ==="
kubectl get services -n kube-system
```

### Komponen K3s yang Berjalan

```bash
# Lihat semua pod sistem
kubectl get pods -n kube-system -o wide
```

| Pod | Fungsi |
|---|---|
| `coredns-*` | DNS internal cluster |
| `local-path-provisioner-*` | Storage provisioner lokal |
| `metrics-server-*` | Metrics untuk HPA dan kubectl top |
| `traefik-*` | Ingress controller bawaan K3s |

---

## Langkah 5: Setup kubectl Aliases dan Autocomplete

```bash
# Install bash completion
sudo apt install -y bash-completion

# Aktifkan kubectl autocomplete
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Tambahkan alias
cat >> ~/.bashrc << 'EOF'
# kubectl aliases
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgi='kubectl get ingress'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias ke='kubectl exec -it'
alias ka='kubectl apply -f'
alias kd='kubectl delete -f'

# kubectl dengan namespace
export KUBE_NAMESPACE=default
alias kns='kubectl config set-context --current --namespace'

# Aktifkan autocomplete untuk alias k
complete -F __start_kubectl k
EOF

source ~/.bashrc

# Test alias
k get nodes
kgpa
```

---

## Langkah 6: Buat Namespace untuk Lab

```bash
# Buat namespace terpisah untuk lab
kubectl create namespace devops-lab

# Set sebagai namespace default untuk sesi ini
kubectl config set-context --current --namespace=devops-lab

# Verifikasi
kubectl config view --minify | grep namespace
kubectl get pods    # Sekarang default ke namespace devops-lab
```

---

## Langkah 7: Test Cluster dengan Pod Sederhana

```bash
# Jalankan pod test
kubectl run test-pod --image=nginx:alpine --restart=Never -n devops-lab

# Tunggu pod Running
kubectl wait --for=condition=Ready pod/test-pod -n devops-lab --timeout=60s

# Cek pod
kubectl get pod test-pod -n devops-lab
kubectl describe pod test-pod -n devops-lab

# Test akses dari dalam pod
kubectl exec -it test-pod -n devops-lab -- sh
# Di dalam pod:
# wget -qO- http://localhost
# cat /etc/hostname
# exit

# Hapus pod test
kubectl delete pod test-pod -n devops-lab
```

---

## Langkah 8: Test Persistent Storage

```bash
# K3s menggunakan local-path provisioner secara default
kubectl get storageclass

# Test buat PVC
kubectl apply -n devops-lab -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: local-path
EOF

# Cek PVC terbuat
kubectl get pvc -n devops-lab
# STATUS harus Bound setelah pod yang menggunakan PVC ini dibuat

# Cleanup
kubectl delete pvc test-pvc -n devops-lab
```

---

## Langkah 9: Lihat Resource Usage

```bash
# Cek resource usage nodes (butuh metrics-server — sudah ada di K3s)
kubectl top nodes

# Cek resource usage pods
kubectl top pods -A

# Jika metrics belum tersedia, tunggu beberapa menit
```

---

## Langkah 10: Install kubectl Plugin Manager (krew) — Opsional

```bash
# Install krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc

# Install plugin berguna
kubectl krew install ctx ns neat tree

# Contoh penggunaan:
# kubectl ctx          — switch context
# kubectl ns           — switch namespace
# kubectl get pod | kubectl neat  — output lebih bersih
# kubectl tree deployment flask-app  — lihat ownership tree
```

---

## Verifikasi Akhir

```bash
echo "=== Verifikasi Lab 09 ==="
echo -n "K3s service berjalan: "
  systemctl is-active k3s | grep -q active && echo "✅" || echo "❌"
echo -n "Node Ready: "
  kubectl get nodes | grep -q "Ready" && echo "✅" || echo "❌"
echo -n "kubectl terkonfigurasi: "
  kubectl cluster-info &>/dev/null && echo "✅" || echo "❌"
echo -n "Namespace devops-lab ada: "
  kubectl get namespace devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "CoreDNS berjalan: "
  kubectl get pods -n kube-system | grep -q "coredns.*Running" && echo "✅" || echo "❌"
echo -n "Traefik Ingress Controller berjalan: "
  kubectl get pods -n kube-system | grep -q "traefik.*Running" && echo "✅" || echo "❌"
echo -n "Metrics server berjalan: "
  kubectl get pods -n kube-system | grep -q "metrics-server.*Running" && echo "✅" || echo "❌"

echo ""
echo "=== Informasi Cluster ==="
kubectl cluster-info
echo ""
kubectl get nodes -o wide
```

---

## Troubleshooting

### Node NotReady
```bash
# Cek log K3s
sudo journalctl -u k3s -f

# Restart K3s
sudo systemctl restart k3s
```

### kubectl: connection refused
```bash
# Pastikan kubeconfig benar
cat ~/.kube/config

# Cek API server berjalan
sudo ss -tlnp | grep 6443
```

### Pod tidak bisa start (ImagePullBackOff)
```bash
# Cek detail error
kubectl describe pod POD_NAME

# Cek koneksi internet dari node
curl -I https://registry-1.docker.io
```

---

## Checkpoint ✅

- [ ] K3s berhasil terinstall dan service berjalan
- [ ] Node berstatus `Ready`
- [ ] `kubectl get nodes` berjalan tanpa sudo
- [ ] Namespace `devops-lab` terbuat
- [ ] Pod test berhasil dijalankan dan dihapus
- [ ] Memahami semua pod sistem di namespace `kube-system`
- [ ] Alias kubectl sudah terkonfigurasi
