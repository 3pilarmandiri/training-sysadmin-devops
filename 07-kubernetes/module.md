# BAB 7 — Kubernetes

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami arsitektur dan komponen Kubernetes
- Membuat dan mengelola Pod, Deployment, dan Service
- Mengkonfigurasi Ingress untuk routing HTTP
- Mengelola konfigurasi dengan ConfigMap dan Secret
- Melakukan rolling update dan rollback

---

## 7.1 Mengapa Kubernetes?

Docker Compose baik untuk satu host, tapi untuk production skala besar dibutuhkan orkestrator:

| Kebutuhan | Docker Compose | Kubernetes |
|---|---|---|
| Multi-host | ❌ | ✅ |
| Auto-healing | ❌ | ✅ (restart container yang crash) |
| Auto-scaling | Manual | ✅ (HPA) |
| Rolling update | Manual | ✅ |
| Load balancing | ❌ (butuh NGINX) | ✅ (built-in) |
| Secret management | File .env | ✅ (Kubernetes Secrets) |
| Resource limits | Terbatas | ✅ (requests/limits) |
| Service discovery | Hostname | ✅ (DNS internal) |

---

## 7.2 Arsitektur Kubernetes

```
┌──────────────────────────────────────────────────────┐
│                   CONTROL PLANE                       │
│                                                       │
│  ┌──────────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ API Server   │  │  etcd    │  │   Scheduler    │  │
│  │ (kube-api)   │  │(database)│  │  (penempatan   │  │
│  │              │  │          │  │   pod)         │  │
│  └──────┬───────┘  └──────────┘  └────────────────┘  │
│         │                                             │
│  ┌──────▼───────────────────────────────────────────┐ │
│  │         Controller Manager                        │ │
│  │  (Deployment, ReplicaSet, Node controllers, dll)  │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────┘
                       │ API calls
       ┌───────────────┼──────────────────────┐
       ▼               ▼                      ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
│  Worker     │  │  Worker     │  │  Worker Node    │
│  Node 1     │  │  Node 2     │  │  3              │
│             │  │             │  │                 │
│  kubelet    │  │  kubelet    │  │  kubelet        │
│  kube-proxy │  │  kube-proxy │  │  kube-proxy     │
│             │  │             │  │                 │
│  ┌────────┐ │  │  ┌────────┐ │  │  ┌────────────┐ │
│  │  Pod   │ │  │  │  Pod   │ │  │  │   Pod      │ │
│  │ Pod    │ │  │  │        │ │  │  │            │ │
│  └────────┘ │  │  └────────┘ │  │  └────────────┘ │
└─────────────┘  └─────────────┘  └─────────────────┘
```

### Komponen Control Plane

| Komponen | Fungsi |
|---|---|
| **API Server** | Gateway semua operasi K8s. Semua `kubectl` command → API Server |
| **etcd** | Database distributed key-value untuk menyimpan state cluster |
| **Scheduler** | Memutuskan node mana yang akan menjalankan pod baru |
| **Controller Manager** | Menjalankan controller loops (Deployment, ReplicaSet, dll) |

### Komponen Worker Node

| Komponen | Fungsi |
|---|---|
| **kubelet** | Agent di setiap node; memastikan container berjalan sesuai spec |
| **kube-proxy** | Mengelola iptables/ipvs untuk network routing ke Service |
| **Container Runtime** | Docker, containerd, CRI-O |

---

## 7.3 Objek-Objek Kubernetes

### Pod

Unit terkecil di K8s. Satu Pod berisi satu atau lebih container yang berbagi network dan storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
    - name: app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"         # 0.1 CPU core
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /health
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 30
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      env:
        - name: APP_ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

### Deployment

Mengelola Pod replicas dan rolling update:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: default
  labels:
    app: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Boleh ada 1 pod extra saat update
      maxUnavailable: 0     # Tidak ada pod yang down saat update
  template:
    metadata:
      labels:
        app: flask-app
        version: "1.0.0"
    spec:
      containers:
        - name: flask-app
          image: ghcr.io/user/flask-app:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          envFrom:
            - configMapRef:
                name: flask-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flask-secrets
                  key: db-password
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
```

### Service

Expose deployment sebagai endpoint yang stabil (IP/DNS tidak berubah meskipun pod restart):

```yaml
# ClusterIP — hanya dalam cluster
apiVersion: v1
kind: Service
metadata:
  name: flask-app-svc
spec:
  type: ClusterIP
  selector:
    app: flask-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
---
# NodePort — expose ke luar via port di setiap node
apiVersion: v1
kind: Service
metadata:
  name: flask-app-nodeport
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080       # Port 30000-32767
---
# LoadBalancer — provisioning cloud LB (AWS ELB, GCP LB)
apiVersion: v1
kind: Service
metadata:
  name: flask-app-lb
spec:
  type: LoadBalancer
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 8080
```

### Ingress

Routing HTTP/HTTPS berdasarkan host dan path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: flask-app-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

---

## 7.4 ConfigMap dan Secret

### ConfigMap — Konfigurasi Non-Sensitif

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  MAX_WORKERS: "4"
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://app:8080;
      }
    }
```

### Secret — Data Sensitif

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: flask-secrets
type: Opaque
data:
  # Nilai harus di-encode base64
  # echo -n "SuperSecret" | base64
  db-password: U3VwZXJTZWNyZXQ=
  api-key: bXlhcGlrZXkxMjM=
stringData:
  # stringData otomatis di-encode
  redis-url: redis://cache:6379
```

```bash
# Buat secret dari command line
kubectl create secret generic flask-secrets \
  --from-literal=db-password=SuperSecret \
  --from-literal=api-key=myapikey123

# Buat secret dari file
kubectl create secret generic tls-certs \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# Buat secret TLS
kubectl create secret tls app-tls-secret \
  --cert=./cert.crt \
  --key=./cert.key
```

---

## 7.5 Persistent Volume

```yaml
# PersistentVolume (admin provisioning)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/postgresql

---
# PersistentVolumeClaim (developer request)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

---

## 7.6 Perintah kubectl Penting

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# Namespace
kubectl get namespaces
kubectl create namespace dev
kubectl config set-context --current --namespace=dev

# Pod
kubectl get pods
kubectl get pods -A                      # Semua namespace
kubectl get pods -l app=flask-app        # Filter by label
kubectl describe pod flask-app-xxx       # Detail pod
kubectl logs flask-app-xxx               # Log pod
kubectl logs -f flask-app-xxx            # Follow log
kubectl logs flask-app-xxx -c sidecar    # Container tertentu
kubectl exec -it flask-app-xxx -- bash   # Masuk ke pod
kubectl delete pod flask-app-xxx         # Hapus pod (akan restart oleh Deployment)

# Deployment
kubectl get deployments
kubectl describe deployment flask-app
kubectl rollout status deployment/flask-app    # Status rolling update
kubectl rollout history deployment/flask-app   # History update
kubectl rollout undo deployment/flask-app      # Rollback
kubectl rollout undo deployment/flask-app --to-revision=2  # Rollback ke versi tertentu
kubectl scale deployment flask-app --replicas=5            # Scale manual

# Apply/Delete resources
kubectl apply -f deployment.yaml          # Create/update
kubectl apply -f ./manifests/            # Apply direktori
kubectl delete -f deployment.yaml
kubectl delete deployment flask-app

# Service & Ingress
kubectl get services
kubectl get ingress
kubectl describe ingress flask-ingress

# ConfigMap & Secret
kubectl get configmaps
kubectl get secrets
kubectl describe configmap flask-config
kubectl get secret flask-secrets -o jsonpath='{.data.db-password}' | base64 -d

# Resource usage
kubectl top nodes
kubectl top pods

# Troubleshooting
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod failing-pod-xxx    # Cek "Events:" di bawah
kubectl port-forward pod/flask-app-xxx 8080:8080  # Forward port lokal
kubectl port-forward svc/flask-app-svc 8080:80
```

---

## 7.7 Rolling Update dan Rollback

```bash
# Update image ke versi baru
kubectl set image deployment/flask-app \
  flask-app=ghcr.io/user/flask-app:2.0.0

# Atau edit deployment langsung
kubectl edit deployment flask-app

# Monitor proses rolling update
kubectl rollout status deployment/flask-app
# Waiting for deployment "flask-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "flask-app" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "flask-app" successfully rolled out

# Annotate untuk history yang informatif
kubectl annotate deployment flask-app \
  kubernetes.io/change-cause="Update ke v2.0.0: fitur baru login"

# Lihat history
kubectl rollout history deployment/flask-app
# REVISION  CHANGE-CAUSE
# 1         Deploy awal v1.0.0
# 2         Update ke v2.0.0: fitur baru login

# Rollback ke versi sebelumnya
kubectl rollout undo deployment/flask-app

# Rollback ke revision tertentu
kubectl rollout undo deployment/flask-app --to-revision=1
```

---

## 7.8 Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Buat HPA
kubectl autoscale deployment flask-app --cpu-percent=70 --min=2 --max=10

# Monitor HPA
kubectl get hpa
kubectl describe hpa flask-app-hpa
```

---

## Ringkasan Objek K8s

| Objek | Fungsi |
|---|---|
| Pod | Unit terkecil, container berjalan |
| Deployment | Kelola replikasi dan update Pod |
| Service | Endpoint stabil untuk Pod |
| Ingress | Routing HTTP berdasarkan host/path |
| ConfigMap | Konfigurasi non-sensitif |
| Secret | Data sensitif (terenkripsi) |
| PersistentVolume | Storage persisten |
| Namespace | Isolasi logical resource |
| HPA | Auto-scaling berdasarkan metrik |
