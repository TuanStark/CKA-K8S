# CKA Battle-Tested Cheat Sheet — Lệnh Thực Chiến & Mẫu Cốt Lõi

> Tài liệu dùng để **ôn luyện và rèn phản xạ** trước kỳ thi (trong phòng thi không được mang ghi chú cá nhân — hãy thuộc các lệnh cốt lõi và luyện tra nhanh YAML dài trên `kubernetes.io/docs`, `gateway.networking.k8s.io`, `helm.sh/docs`).

---

## 1. Thiết Lập Đầu Phiên (Speed Toolkit)

```bash
# Kiểm tra trước xem môi trường đã có sẵn alias chưa
alias

# Thiết lập nhanh (nếu chưa có)
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Cấu hình Vim chuẩn cho YAML (~/.vimrc)
cat <<'EOF' >> ~/.vimrc
set tabstop=2
set shiftwidth=2
set expandtab
EOF
```

**Chuyển context & namespace an toàn:**
```bash
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<ns>
```

---

## 2. Lệnh Imperative Sinh YAML Cực Nhanh

```bash
# Pod, Deployment, Job, CronJob
k run nginx --image=nginx:1.27 --port=80 $do > pod.yaml
k create deploy web --image=nginx:1.27 --replicas=3 $do > deploy.yaml
k create job myjob --image=busybox:1.36 -- /bin/sh -c "date; sleep 5" $do > job.yaml
k create cronjob mycj --image=busybox:1.36 --schedule="*/5 * * * *" -- date $do > cj.yaml

# ConfigMap & Secret
k create cm app-cm --from-literal=ENV=prod --from-file=app.conf $do > cm.yaml
k create secret generic db-sec --from-literal=PASS=s3cr3t $do > sec.yaml
k create secret tls web-tls --cert=tls.crt --key=tls.key $do > tls.yaml

# Service & Ingress
k expose deploy web --name=web-svc --port=80 --target-port=8080 --type=ClusterIP $do > svc.yaml
k create svc nodeport web-np --tcp=80:8080 --node-port=30080 $do > np.yaml
k create ingress web-ing --class=nginx --rule="app.example.com/api*=web-svc:80" $do > ing.yaml

# HPA (Autoscaling)
k autoscale deploy web --cpu-percent=50 --min=1 --max=5 $do > hpa.yaml

# RBAC
k create sa dev-sa -n dev
k create role dev-role --verb=get,list,watch --resource=pods,deployments -n dev $do > role.yaml
k create rolebinding dev-rb --role=dev-role --serviceaccount=dev:dev-sa -n dev $do > rb.yaml
k create clusterrole pv-reader --verb=get,list --resource=persistentvolumes $do > cr.yaml
k create clusterrolebinding pv-rb --clusterrole=pv-reader --serviceaccount=dev:dev-sa $do > crb.yaml

# Verify RBAC
k auth can-i list pods --as=system:serviceaccount:dev:dev-sa -n dev

# Tra nhanh cấu trúc trường YAML ngay tại terminal
k explain pod.spec.affinity.nodeAffinity --recursive | less

# Sắp xếp & trích xuất bằng jsonpath / custom-columns
k get pods -A --sort-by='.metadata.creationTimestamp'
k get pv --sort-by='.spec.capacity.storage'
k get nodes -o custom-columns='NAME:.metadata.name,IP:.status.addresses[?(@.type=="InternalIP")].address,TAINTS:.spec.taints[*].key'
```

---

## 3. Quy Trình `kubeadm` Upgrade (Control Plane & Worker)

> **Lưu ý:** Luôn đọc kỹ đề xem yêu cầu nâng cấp từ bản nào lên bản nào (chỉ nâng 1 minor version mỗi lần).

```bash
# === BƯỚC 1: Từ máy quản trị, drain node cần nâng cấp ===
k drain <node-name> --ignore-daemonsets --delete-emptydir-data

# === BƯỚC 2: SSH vào node và cập nhật apt repo đúng minor mới (ví dụ v1.32) ===
ssh <node-name>
sudo -i
# Sửa phiên bản minor trong /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-cache madison kubeadm | head -n 5

# === BƯỚC 3: Nâng cấp kubeadm ===
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.32.x-1.1
apt-mark hold kubeadm
kubeadm version

# === BƯỚC 4: Chạy kubeadm upgrade ===
# Trên Control Plane ĐẦU TIÊN:
kubeadm upgrade plan
kubeadm upgrade apply v1.32.x

# Trên Worker Node (hoặc Control Plane phụ):
kubeadm upgrade node

# === BƯỚC 5: Nâng cấp kubelet & kubectl ===
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.32.x-1.1 kubectl=1.32.x-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet --no-pager
exit

# === BƯỚC 6: Uncordon và xác nhận ===
k uncordon <node-name>
k get nodes -o wide
```

---

## 4. `etcd` Backup & Restore

```bash
# 1. Lấy đường dẫn cert & endpoints từ static pod manifest
grep -E 'listen-client-urls|trusted-ca-file|cert-file|key-file|data-dir' /etc/kubernetes/manifests/etcd.yaml

# 2. Backup snapshot
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Kiểm tra file snapshot hợp lệ
etcdutl snapshot status /opt/etcd-backup.db -w table
# (Hoặc: ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db -w table)

# 3. Restore snapshot sang thư mục mới
etcdutl snapshot restore /opt/etcd-backup.db --data-dir=/var/lib/etcd-restore
# (Nếu máy không cài etcdutl thì dùng: ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db --data-dir=/var/lib/etcd-restore)

# 4. Cập nhật hostPath của volume `etcd-data` trong /etc/kubernetes/manifests/etcd.yaml
# Sao lưu trước:
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
# Sửa path: /var/lib/etcd  --->  /var/lib/etcd-restore

# 5. Chờ Static Pod etcd & kube-apiserver khởi động lại và kiểm chứng
crictl ps | grep -E 'etcd|kube-apiserver'
k get nodes
k get pods -A
```

---

## 5. Helm, Kustomize & CRDs

```bash
# === HELM ===
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx --versions
helm show values bitnami/nginx > values.yaml
helm template my-rel bitnami/nginx -f values.yaml          # Xem YAML trước khi cài
helm install my-rel bitnami/nginx -n web --create-namespace -f values.yaml --set replicaCount=2
helm upgrade my-rel bitnami/nginx -n web --set replicaCount=3
helm list -A
helm history my-rel -n web
helm rollback my-rel 1 -n web
helm uninstall my-rel -n web

# === KUSTOMIZE ===
kubectl kustomize overlays/prod/          # Preview YAML kết quả
kubectl apply -k overlays/prod/           # Áp dụng trực tiếp

# === CRDs ===
kubectl get crd
kubectl api-resources | grep <keyword>
kubectl explain <crd-kind>.spec
```

---

## 6. Networking: NetworkPolicy & Gateway API

### Bẫy kinh điển NetworkPolicy (AND vs OR)

```yaml
# === AND: Pod phải có label role=frontend VÀ nằm trong namespace env=prod ===
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: prod
        podSelector:
          matchLabels:
            role: frontend

# === OR: Pod thuộc namespace env=prod HOẶC Pod có label role=frontend (cùng namespace) ===
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: prod
      - podSelector:
          matchLabels:
            role: frontend
```

### Mở cổng DNS khi viết Egress NetworkPolicy

```yaml
egress:
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
```

### Mẫu Gateway API (`Gateway` + `HTTPRoute` + Traffic Splitting)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "app.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: web-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
    - name: web-gateway
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-v1-svc
          port: 80
          weight: 80
        - name: api-v2-svc
          port: 80
          weight: 20
```

---

## 7. Luồng Lệnh Troubleshooting 4 Tầng

### Tầng 1: Pod / Application
```bash
k get pods -A -o wide
k describe pod <pod> -n <ns>
k logs <pod> -n <ns> -c <container>
k logs <pod> -n <ns> -c <container> --previous
k get events -n <ns> --sort-by='.lastTimestamp'
```

### Tầng 2: Service / DNS / Network
```bash
k get svc,ep,endpointslices -n <ns>
# Test kết nối nhanh bằng pod tạm
k run net-dbg --rm -it --restart=Never --image=busybox:1.36 -- sh
# Bên trong pod:
#   nslookup <svc>.<ns>.svc.cluster.local
#   wget -qO- --timeout=3 http://<pod-ip>:<port>
#   wget -qO- --timeout=3 http://<svc>.<ns>:<port>
k get netpol -A
k logs -n kube-system -l k8s-app=kube-dns
```

### Tầng 3: Worker Node (`NotReady`)
```bash
ssh <worker-node>
sudo -i
systemctl status kubelet containerd --no-pager
journalctl -u kubelet -n 50 --no-pager
# Kiểm tra config & chứng chỉ & tài nguyên
cat /var/lib/kubelet/config.yaml
cat /etc/kubernetes/kubelet.conf
df -h
free -m    # Swap phải bằng 0 (swapoff -a)
```

### Tầng 4: Control Plane (`kubectl` chết hoàn toàn)
```bash
ssh <control-plane-node>
sudo -i
# Xem các container static pod kể cả đã chết
crictl ps -a
crictl logs <container-id>
# Xem log trực tiếp trên đĩa
ls -lt /var/log/pods/kube-system_*
tail -n 50 /var/log/containers/kube-apiserver*
# Kiểm tra kỹ YAML trong staticPodPath
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Kiểm tra kubelet có đang chạy để đọc manifests không
systemctl status kubelet --no-pager
journalctl -u kubelet -n 50 --no-pager
```

