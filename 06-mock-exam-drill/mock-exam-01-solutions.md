# Mock Exam #01 — Lời Giải Chuẩn Kubestronaut & Bộ Chấm Điểm (100%)

> **Cách chấm điểm trung thực:** Chỉ tính trọn điểm một câu khi bạn **hoàn thành đủ tất cả các ý** và **pass toàn bộ lệnh Verify** tương ứng. Nếu thiếu 1 ý hoặc sai tên namespace/resource → tính 0 điểm (giống cơ chế chấm tự động của CKA!).

---

## Câu 1 (5%) — RBAC & ServiceAccount
```bash
k create ns finance
k create sa report-sa -n finance
k create role report-role -n finance --verb=get,list --resource=pods,services,configmaps
k create rolebinding report-rb -n finance --role=report-role --serviceaccount=finance:report-sa

k run report-runner -n finance --image=busybox:1.36 $do -- sleep 3600 > /tmp/q1-pod.yaml
sed -i '/containers:/i \  serviceAccountName: report-sa' /tmp/q1-pod.yaml
k apply -f /tmp/q1-pod.yaml

# === VERIFY (Đạt 5/5 khi tất cả khớp) ===
k auth can-i get pods --as=system:serviceaccount:finance:report-sa -n finance       # yes
k auth can-i list configmaps --as=system:serviceaccount:finance:report-sa -n finance # yes
k auth can-i delete pods --as=system:serviceaccount:finance:report-sa -n finance    # no
k get pod report-runner -n finance -o jsonpath='{.spec.serviceAccountName}{"\n"}'   # report-sa
```

---

## Câu 2 (7%) — Nâng Cấp Worker Node Bằng `kubeadm`
```bash
# 1. Drain từ Control Plane
k drain node01 --ignore-daemonsets --delete-emptydir-data

# 2. SSH sang node01
ssh node01
sudo -i
# Cập nhật repo minor nếu cần:
# sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
apt-get update
TARGET_PKG=$(apt-cache madison kubeadm | head -n 1 | awk '{print $3}')

apt-mark unhold kubeadm
apt-get install -y "kubeadm=${TARGET_PKG}"
apt-mark hold kubeadm

kubeadm upgrade node

apt-mark unhold kubelet kubectl
apt-get install -y "kubelet=${TARGET_PKG}" "kubectl=${TARGET_PKG}"
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
exit

# 3. Uncordon
k uncordon node01

# === VERIFY ===
k get nodes -o wide   # node01: Ready, VERSION khớp Control Plane
```

---

## Câu 3 (7%) — Backup & Restore `etcd`
```bash
# 1. Backup
ETCDCTL_API=3 etcdctl snapshot save /opt/exam-etcd-snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. Kiểm tra snapshot
etcdutl snapshot status /opt/exam-etcd-snap.db -w table

# 3. Tạo pod kiểm chứng
k run canary-delete-me --image=nginx:1.27

# 4. Restore sang thư mục mới & cập nhật etcd.yaml
etcdutl snapshot restore /opt/exam-etcd-snap.db --data-dir=/var/lib/etcd-exam-restore
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd-exam.yaml.bak
# Cập nhật hostPath của volume etcd-data -> /var/lib/etcd-exam-restore
sed -i 's|path: /var/lib/etcd.*|path: /var/lib/etcd-exam-restore|' /etc/kubernetes/manifests/etcd.yaml

# === VERIFY (sau ~30s) ===
k get pod canary-delete-me   # Phải báo NotFound!
```

---

## Câu 4 (6%) — Helm & Kustomize
```bash
# 1. Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install exam-web bitnami/nginx -n helm-exam --create-namespace \
  --set replicaCount=2 --set service.type=ClusterIP

# 2. Kustomize
mkdir -p /tmp/exam-kust/base /tmp/exam-kust/overlays/staging

cat <<'EOF' > /tmp/exam-kust/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: nginx:1.26
EOF

cat <<'EOF' > /tmp/exam-kust/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF

cat <<'EOF' > /tmp/exam-kust/overlays/staging/replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
EOF

cat <<'EOF' > /tmp/exam-kust/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
namePrefix: stg-
resources:
  - ../../base
images:
  - name: nginx
    newTag: "1.27"
patches:
  - path: replicas.yaml
EOF

k create ns staging
k apply -k /tmp/exam-kust/overlays/staging

# === VERIFY ===
helm list -n helm-exam
k get deploy stg-api -n staging -o custom-columns='NAME:.metadata.name,REP:.spec.replicas,IMG:.spec.template.spec.containers[0].image'
```

---

## Câu 5 (5%) — HPA (`autoscaling/v2`)
```bash
k create ns autoscale
cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
  namespace: autoscale
spec:
  replicas: 1
  selector:
    matchLabels:
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-hpa
  namespace: autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 45
EOF

# === VERIFY ===
k get hpa checkout-hpa -n autoscale
```

---

## Câu 6 (5%) — Taints, Tolerations & NodeAffinity
```bash
k label node node01 tier=backend
k taint node node01 tier=backend:NoSchedule

cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-worker
  template:
    metadata:
      labels:
        app: order-worker
    spec:
      tolerations:
        - key: "tier"
          operator: "Equal"
          value: "backend"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: tier
                    operator: In
                    values:
                      - backend
      containers:
        - name: worker
          image: nginx:1.27
EOF

# === VERIFY ===
k get pods -l app=order-worker -o wide    # Cả 2 pod đều chạy trên node01
```

---

## Câu 7 (5%) — PriorityClass & Sidecar Pod
```bash
cat <<'EOF' | k apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: mission-critical
value: 500000
globalDefault: false
description: "Mission critical workloads"
---
apiVersion: v1
kind: Pod
metadata:
  name: audit-logger
spec:
  priorityClassName: mission-critical
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo tx-ok >> /var/log/app/tx.log; sleep 5; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "tail -n+1 -F /var/log/app/tx.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
EOF

# === VERIFY ===
k get pod audit-logger           # READY 2/2
k logs audit-logger -c sidecar   # In ra "tx-ok"
```

---

## Câu 8 (5%) — StorageClass & PV/PVC Binding
```bash
cat <<'EOF' | k apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exam-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: exam-pv
spec:
  capacity:
    storage: 2Gi
  accessModes: [ReadWriteOnce]
  storageClassName: exam-sc
  hostPath:
    path: /mnt/exam-vol
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exam-pvc
spec:
  storageClassName: exam-sc
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-consumer
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: exam-pvc
EOF

# === VERIFY ===
k get pv exam-pv && k get pvc exam-pvc && k get pod storage-consumer  # Bound + Running
```

---

## Câu 9 (5%) — Sắp Xếp PV Theo Dung Lượng
```bash
k get pv --sort-by='.spec.capacity.storage' > /opt/pv-sorted.txt

# === VERIFY ===
cat /opt/pv-sorted.txt
```

---

## Câu 10 (6%) — NetworkPolicy (AND Logic + Egress DNS)
```bash
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress-lock
  namespace: secure-db
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              zone: app
          podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-lock
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              zone: db
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 80
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF

# === VERIFY ===
k exec backend -n secure-app -- wget -qO- --timeout=2 http://pg-svc.secure-db   # OK
k exec intruder -n secure-app -- wget -qO- --timeout=2 http://pg-svc.secure-db  # TIMEOUT
```

---

## Câu 11 (8%) — Gateway API (`Gateway` HTTPS + `HTTPRoute` Traffic Splitting)
```bash
cat <<'EOF' | k apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: portal-gw
  namespace: web-gw
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "portal.cka.local"
      tls:
        mode: Terminate
        certificateRefs:
          - name: portal-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: portal-route
  namespace: web-gw
spec:
  parentRefs:
    - name: portal-gw
  hostnames:
    - "portal.cka.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v2
      backendRefs:
        - name: v2-svc
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: v1-svc
          port: 80
          weight: 70
        - name: v2-svc
          port: 80
          weight: 30
EOF

# === VERIFY ===
k get gateway,httproute -n web-gw
k describe httproute portal-route -n web-gw
```

---

## Câu 12 (6%) — NodePort Service & CoreDNS Verification
```bash
k create ns net-exam
k create deploy edge-web -n net-exam --image=nginx:1.27 --replicas=2 --port=80
k expose deploy edge-web -n net-exam --name=edge-svc --type=NodePort --port=8080 --target-port=80 $do > /tmp/edge-svc.yaml
sed -i '/targetPort: 80/a \    nodePort: 30180' /tmp/edge-svc.yaml
k apply -f /tmp/edge-svc.yaml

k run dns-q12 -n net-exam --rm -i --restart=Never --image=busybox:1.36 \
  -- nslookup edge-svc.net-exam.svc.cluster.local > /opt/edge-dns.txt

# === VERIFY ===
k get svc,ep -n net-exam
cat /opt/edge-dns.txt
```

---

## Câu 13 (6%) — Sửa Pod `OOMKilled` & Thiếu ConfigMap
```bash
# 1. Tăng limits.memory cho worker-oom
k set resources deploy/worker-oom -n trouble-1 --limits=memory=256Mi --requests=memory=128Mi

# 2. Tạo ConfigMap bị thiếu cho cfg-broken
k create cm env-cfg -n trouble-1 --from-literal=MODE=active

# === VERIFY ===
k get pods -n trouble-1     # Tất cả Running & Ready 1/1
```

---

## Câu 14 (6%) — Sửa Service & ReadinessProbe
```bash
k patch deploy api-broken -n trouble-2 --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"},
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":80}
]'
k patch svc api-broken-svc -n trouble-2 -p '{"spec":{"selector":{"app":"api-broken"},"ports":[{"port":80,"targetPort":80}]}}'

# === VERIFY ===
k get pods,ep -n trouble-2
k run test-q14 -n trouble-2 --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://api-broken-svc.trouble-2 | head -n 5
```

---

## Câu 15 (8%) — Khôi Phục Worker Node `NotReady`
```bash
ssh node01
sudo -i
journalctl -u kubelet -n 30 --no-pager
cp /etc/kubernetes/kubelet.conf /tmp/kubelet.conf.bak
sed -i 's/:6444/:6443/g' /etc/kubernetes/kubelet.conf
systemctl enable --now kubelet
systemctl restart kubelet
systemctl status kubelet --no-pager
exit

# === VERIFY ===
k get nodes   # node01: Ready
```

---

## Câu 16 (10%) — Khôi Phục Control Plane Khi `kubectl` Chết
```bash
# Trên Control Plane (sudo -i):
crictl ps -a | grep kube-apiserver
crictl logs <exited-container-id> | tail -n 30
cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak

# Sửa lỗi cổng etcd (2380 -> 2379) hoặc đường dẫn cert bị sai trong manifest
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# Chờ 20-30s cho kubelet khởi động lại static pod kube-apiserver
crictl ps | grep kube-apiserver

# === VERIFY ===
k get nodes
k get pods -n kube-system
```

