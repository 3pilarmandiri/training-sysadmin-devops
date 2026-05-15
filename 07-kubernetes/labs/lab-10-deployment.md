# LAB 10 — Kubernetes Deployment Lengkap

## Tujuan
Deploy aplikasi Flask API ke Kubernetes dengan Deployment, Service, ConfigMap, Secret, Ingress, dan melakukan rolling update serta rollback.

## Prasyarat
- Lab 09 selesai (K3s berjalan, namespace devops-lab tersedia)
- Gambar Docker flask-app dari Lab 06 (atau gunakan image public)

## Estimasi Waktu
60 menit

---

## Skenario

Anda akan men-deploy aplikasi Flask API ke Kubernetes cluster dengan konfigurasi production-ready:

```
Ingress (traefik)
    │
    └── Service (flask-app-svc:80)
            │
            └── Deployment (flask-app, 2 replicas)
                    │
                    ├── ConfigMap (flask-config)
                    ├── Secret (flask-secrets)
                    └── PersistentVolumeClaim (logs-pvc)
```

---

## Langkah 1: Siapkan Direktori Manifests

```bash
mkdir -p ~/devops-labs/kubernetes/flask-deploy
cd ~/devops-labs/kubernetes/flask-deploy
mkdir -p manifests
```

---

## Langkah 2: Buat ConfigMap

```bash
cat > manifests/01-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
  namespace: devops-lab
  labels:
    app: flask-app
    component: config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "8080"
  MAX_WORKERS: "2"
  # File konfigurasi bisa disimpan sebagai value
  app.ini: |
    [server]
    host = 0.0.0.0
    port = 8080
    workers = 2
    
    [logging]
    level = info
    format = json
EOF

kubectl apply -f manifests/01-configmap.yaml
kubectl get configmap -n devops-lab
kubectl describe configmap flask-config -n devops-lab
```

---

## Langkah 3: Buat Secret

```bash
cat > manifests/02-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: flask-secrets
  namespace: devops-lab
  labels:
    app: flask-app
    component: secrets
type: Opaque
stringData:
  db-password: "Training@2024!"
  api-key: "lab-api-key-xyz123"
  jwt-secret: "super-secret-jwt-key"
EOF

kubectl apply -f manifests/02-secret.yaml
kubectl get secret -n devops-lab

# Verifikasi isi secret (decoded)
kubectl get secret flask-secrets -n devops-lab \
  -o jsonpath='{.data.db-password}' | base64 -d
echo ""
```

---

## Langkah 4: Buat Deployment

```bash
cat > manifests/03-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: devops-lab
  labels:
    app: flask-app
    version: "1.0.0"
  annotations:
    kubernetes.io/change-cause: "Deploy awal versi 1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: flask-app
        version: "1.0.0"
    spec:
      # Graceful shutdown
      terminationGracePeriodSeconds: 30

      containers:
        - name: flask-app
          # Gunakan image public untuk lab ini
          image: kennethreitz/httpbin:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          
          # Environment dari ConfigMap
          envFrom:
            - configMapRef:
                name: flask-config
          
          # Environment dari Secret
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flask-secrets
                  key: db-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: flask-secrets
                  key: api-key
          
          # Resource requests & limits
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /get
              port: 80
            initialDelaySeconds: 20
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /get
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
          
          # Startup probe (untuk slow startup)
          startupProbe:
            httpGet:
              path: /get
              port: 80
            failureThreshold: 30
            periodSeconds: 5
EOF

kubectl apply -f manifests/03-deployment.yaml

# Monitor deployment
kubectl rollout status deployment/flask-app -n devops-lab

# Cek pod
kubectl get pods -n devops-lab -l app=flask-app
kubectl get pods -n devops-lab -l app=flask-app -o wide
```

Output yang diharapkan:
```
NAME                         READY   STATUS    RESTARTS   AGE
flask-app-xxxx-yyy           1/1     Running   0          30s
flask-app-xxxx-zzz           1/1     Running   0          30s
```

---

## Langkah 5: Buat Service

```bash
cat > manifests/04-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: flask-app-svc
  namespace: devops-lab
  labels:
    app: flask-app
spec:
  type: ClusterIP
  selector:
    app: flask-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
EOF

kubectl apply -f manifests/04-service.yaml

# Verifikasi service
kubectl get service -n devops-lab
kubectl describe service flask-app-svc -n devops-lab

# Test akses via service (dari dalam cluster)
kubectl run curl-test --image=curlimages/curl:latest \
  --rm -it --restart=Never -n devops-lab \
  -- curl -s http://flask-app-svc/get | head -20
```

---

## Langkah 6: Buat Ingress

```bash
cat > manifests/05-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-ingress
  namespace: devops-lab
  labels:
    app: flask-app
  annotations:
    # Traefik annotations (K3s default ingress)
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: flask.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flask-app-svc
                port:
                  number: 80
EOF

kubectl apply -f manifests/05-ingress.yaml

# Cek Ingress
kubectl get ingress -n devops-lab
kubectl describe ingress flask-ingress -n devops-lab

# Tambahkan host ke /etc/hosts
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "Menambahkan $NODE_IP flask.local ke /etc/hosts"
echo "$NODE_IP flask.local" | sudo tee -a /etc/hosts

# Test via Ingress
curl -s http://flask.local/get | python3 -m json.tool | head -20
```

---

## Langkah 7: Inspect Detail Pod

```bash
# Lihat environment variables dalam pod
POD_NAME=$(kubectl get pods -n devops-lab -l app=flask-app -o jsonpath='{.items[0].metadata.name}')
echo "Pod: $POD_NAME"

# Lihat env vars (termasuk dari ConfigMap dan Secret)
kubectl exec -n devops-lab $POD_NAME -- env | sort

# Verifikasi secret masuk sebagai env
kubectl exec -n devops-lab $POD_NAME -- sh -c 'echo "DB_PASSWORD: $DB_PASSWORD"'

# Cek resource yang digunakan
kubectl top pod -n devops-lab
```

---

## Langkah 8: Rolling Update

```bash
# Simulasi update ke "versi 2" dengan image berbeda
kubectl set image deployment/flask-app \
  flask-app=nginxdemos/hello:latest \
  -n devops-lab

# Annotate untuk history
kubectl annotate deployment/flask-app \
  kubernetes.io/change-cause="Update ke v2.0.0: ganti image ke nginxdemos/hello" \
  -n devops-lab

# Monitor rolling update secara real-time
kubectl rollout status deployment/flask-app -n devops-lab -w

# Selama update, lihat pod baru muncul dan lama terminating
kubectl get pods -n devops-lab -w &
WATCH_PID=$!
sleep 15
kill $WATCH_PID 2>/dev/null

# Lihat history
kubectl rollout history deployment/flask-app -n devops-lab

# Test endpoint setelah update
curl -s http://flask.local/ | head -20
```

---

## Langkah 9: Rollback

```bash
# Lihat revisi yang tersedia
kubectl rollout history deployment/flask-app -n devops-lab

# Rollback ke versi sebelumnya (revision 1)
kubectl rollout undo deployment/flask-app -n devops-lab

# Atau rollback ke revision tertentu
# kubectl rollout undo deployment/flask-app --to-revision=1 -n devops-lab

# Monitor rollback
kubectl rollout status deployment/flask-app -n devops-lab

# Verifikasi image kembali ke versi lama
kubectl get deployment flask-app -n devops-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
```

---

## Langkah 10: Scale Deployment

```bash
# Scale up manual
kubectl scale deployment flask-app --replicas=4 -n devops-lab

# Lihat pod baru muncul
kubectl get pods -n devops-lab -l app=flask-app

# Scale down
kubectl scale deployment flask-app --replicas=2 -n devops-lab

# Verifikasi
kubectl get pods -n devops-lab

# Atau edit langsung
kubectl edit deployment flask-app -n devops-lab
# Ubah spec.replicas ke nilai yang diinginkan
```

---

## Langkah 11: Debug Pod yang Bermasalah

```bash
# Simulasikan pod yang gagal dengan image tidak ada
kubectl run bad-pod \
  --image=imagetidakada/tidakexist:latest \
  -n devops-lab

# Lihat status
kubectl get pod bad-pod -n devops-lab

# Describe untuk melihat error
kubectl describe pod bad-pod -n devops-lab
# Lihat bagian Events: ErrImagePull / ImagePullBackOff

# Lihat log (meskipun container gagal start)
kubectl logs bad-pod -n devops-lab 2>&1 || echo "Container belum bisa start"

# Hapus pod bermasalah
kubectl delete pod bad-pod -n devops-lab

# Lihat events untuk troubleshooting
kubectl get events -n devops-lab --sort-by='.lastTimestamp' | tail -20
```

---

## Langkah 12: Apply Semua Sekaligus

Praktikkan juga cara deploy semua manifest sekaligus:

```bash
# Hapus resource yang ada
kubectl delete -f manifests/ -n devops-lab 2>/dev/null || true

# Tunggu semua terhapus
sleep 5

# Apply semua manifest sekaligus
kubectl apply -f manifests/

# Monitor
kubectl get all -n devops-lab
```

---

## Langkah 13: Buat Horizontal Pod Autoscaler

```bash
cat > manifests/06-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-app-hpa
  namespace: devops-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
EOF

kubectl apply -f manifests/06-hpa.yaml

# Cek HPA
kubectl get hpa -n devops-lab
kubectl describe hpa flask-app-hpa -n devops-lab
```

---

## Verifikasi Akhir

```bash
echo "=== Verifikasi Lab 10 ==="
echo -n "Deployment ada: "
  kubectl get deployment flask-app -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "Pod Running (min 2): "
  RUNNING=$(kubectl get pods -n devops-lab -l app=flask-app --field-selector=status.phase=Running --no-headers | wc -l)
  [ "$RUNNING" -ge 2 ] && echo "✅ ($RUNNING pods)" || echo "❌ ($RUNNING pods)"
echo -n "Service ada: "
  kubectl get service flask-app-svc -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "Ingress ada: "
  kubectl get ingress flask-ingress -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "ConfigMap ada: "
  kubectl get configmap flask-config -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "Secret ada: "
  kubectl get secret flask-secrets -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "HPA ada: "
  kubectl get hpa flask-app-hpa -n devops-lab &>/dev/null && echo "✅" || echo "❌"
echo -n "Ingress accessible: "
  curl -sf http://flask.local/ &>/dev/null && echo "✅" || echo "❌"

echo ""
echo "=== Semua Resources ==="
kubectl get all -n devops-lab
```

---

## Cleanup Lab

```bash
# Hapus semua resource di namespace devops-lab
kubectl delete -f manifests/ -n devops-lab

# Atau hapus seluruh namespace (semua resource terhapus)
# kubectl delete namespace devops-lab

# Hapus entry /etc/hosts
sudo sed -i '/flask.local/d' /etc/hosts
```

---

## Checkpoint ✅

- [ ] ConfigMap terbuat dan ter-mount di pod sebagai env vars
- [ ] Secret terbuat dan ter-mount sebagai env vars sensitif
- [ ] Deployment dengan 2 replika berjalan dan healthy
- [ ] Service berhasil route traffic ke pod
- [ ] Ingress berhasil route traffic via domain `flask.local`
- [ ] Rolling update berhasil dilakukan tanpa downtime
- [ ] Rollback ke versi sebelumnya berhasil
- [ ] Scale up/down berhasil
- [ ] HPA terbuat
- [ ] Memahami cara debug pod yang bermasalah
