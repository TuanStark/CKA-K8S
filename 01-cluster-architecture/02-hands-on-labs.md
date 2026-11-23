# Module 01: Hands-On Labs — Cluster Architecture, Installation & Configuration (25%)

> **Hướng dẫn thực hành:** Tự làm từng bài theo đúng **Thời gian mục tiêu**, chỉ tra cứu `kubernetes.io/docs` và `helm.sh/docs`. Chạy phần **Lệnh Verify** trước khi mở đáp án.

---

## Lab 1.1: Thiết Kế RBAC Chuẩn & Kiểm Chứng Bằng `auth can-i`
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo namespace `payments`.
2. Tạo ServiceAccount tên `auditor-sa` trong namespace `payments`.
3. Tạo Role tên `pod-log-reader` trong namespace `payments` chỉ cho phép các quyền `get`, `list`, `watch` đối với tài nguyên `pods` và `pods/log`.
4. Gắn Role trên cho `auditor-sa` thông qua RoleBinding tên `auditor-binding`.
5. Tạo Pod tên `audit-box` dùng image `busybox:1.36` (`sleep 3600`) chạy bằng ServiceAccount `auditor-sa` trong namespace `payments`.

**Verify:** `auditor-sa` phải trả `yes` cho `get pods`, `get pods/log` trong `payments`, và trả `no` cho `delete pods` trong `payments` hoặc `get pods` trong `default`.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.1</summary>

```bash
k create ns payments
k create sa auditor-sa -n payments
k create role pod-log-reader -n payments --verb=get,list,watch --resource=pods,pods/log
k create rolebinding auditor-binding -n payments --role=pod-log-reader --serviceaccount=payments:auditor-sa

k run audit-box -n payments --image=busybox:1.36 $do -- sleep 3600 > audit-box.yaml
# Thêm serviceAccountName: auditor-sa vào dưới spec:
sed -i '/containers:/i \  serviceAccountName: auditor-sa' audit-box.yaml
k apply -f audit-box.yaml

# === VERIFY ===
k auth can-i get pods --as=system:serviceaccount:payments:auditor-sa -n payments          # yes
k auth can-i get pods/log --as=system:serviceaccount:payments:auditor-sa -n payments      # yes
k auth can-i delete pods --as=system:serviceaccount:payments:auditor-sa -n payments       # no
k auth can-i get pods --as=system:serviceaccount:payments:auditor-sa -n default           # no
k get pod audit-box -n payments -o jsonpath='{.spec.serviceAccountName}{"\n"}'            # auditor-sa
```
</details>

---

## Lab 1.2: Tái Sử Dụng `ClusterRole` Với `RoleBinding` (Bẫy Scope RBAC)
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo 2 namespace: `team-a` và `team-b`.
2. Tạo 1 `ClusterRole` tên `deploy-manager` có quyền `get`, `list`, `create`, `update` trên `deployments` (nhớ đúng `apiGroups`!).
3. Tạo ServiceAccount `deployer` trong `team-a`.
4. Dùng **`RoleBinding`** tên `team-a-deploy-rb` trong `team-a` để cấp `ClusterRole/deploy-manager` cho `team-a:deployer` (chỉ có tác dụng trong `team-a`, không có quyền trong `team-b`).

<details>
<summary>Xem đáp án & lệnh verify Lab 1.2</summary>

```bash
k create ns team-a
k create ns team-b
k create sa deployer -n team-a
k create clusterrole deploy-manager --verb=get,list,create,update --resource=deployments.apps
k create rolebinding team-a-deploy-rb -n team-a --clusterrole=deploy-manager --serviceaccount=team-a:deployer

# === VERIFY ===
k auth can-i create deployments --as=system:serviceaccount:team-a:deployer -n team-a   # yes
k auth can-i create deployments --as=system:serviceaccount:team-a:deployer -n team-b   # no
k auth can-i delete deployments --as=system:serviceaccount:team-a:deployer -n team-a   # no
```
</details>

---

## Lab 1.3: Nâng Cấp Cluster Bằng `kubeadm` (Thực Hành Trên Cụm `kubeadm` Thật)
**Mục tiêu thời gian:** 12 phút

**Đề bài:**
Giả sử cụm hiện đang chạy Kubernetes `v1.31.x` gồm 1 Control Plane (`controlplane`) và 1 Worker (`node01`). Hãy nâng cấp toàn bộ cụm lên bản vá/minor tiếp theo (ví dụ `v1.32.x`), đảm bảo không làm gián đoạn workload vượt mức cần thiết và cả 2 node sau khi nâng cấp đều ở trạng thái `Ready` với đúng phiên bản mới.

<details>
<summary>Xem quy trình chuẩn & lệnh verify Lab 1.3</summary>

```bash
# === PHẦN 1: CONTROL PLANE ===
k drain controlplane --ignore-daemonsets --delete-emptydir-data

# Trên controlplane (sudo -i):
# 1. Cập nhật repo apt sang minor mới (nếu đổi minor version)
sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
apt-get update
TARGET_PKG=$(apt-cache madison kubeadm | head -n 1 | awk '{print $3}')
TARGET_VER=$(echo $TARGET_PKG | cut -d'-' -f1)

# 2. Upgrade kubeadm -> plan -> apply
apt-mark unhold kubeadm && apt-get install -y "kubeadm=${TARGET_PKG}" && apt-mark hold kubeadm
kubeadm upgrade plan
kubeadm upgrade apply "v${TARGET_VER}" -y

# 3. Upgrade kubelet & kubectl -> restart
apt-mark unhold kubelet kubectl && apt-get install -y "kubelet=${TARGET_PKG}" "kubectl=${TARGET_PKG}" && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet

# Quay lại uncordon controlplane
k uncordon controlplane

# === PHẦN 2: WORKER NODE (node01) ===
k drain node01 --ignore-daemonsets --delete-emptydir-data

ssh node01
sudo -i
sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
apt-get update
TARGET_PKG=$(apt-cache madison kubeadm | head -n 1 | awk '{print $3}')

apt-mark unhold kubeadm && apt-get install -y "kubeadm=${TARGET_PKG}" && apt-mark hold kubeadm
kubeadm upgrade node
apt-mark unhold kubelet kubectl && apt-get install -y "kubelet=${TARGET_PKG}" "kubectl=${TARGET_PKG}" && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
exit

k uncordon node01

# === VERIFY ===
k get nodes -o wide
```
</details>

---

## Lab 1.4: Backup & Restore `etcd` Hoàn Chỉnh (Thực Hành Trên Cụm `kubeadm` Thật)
**Mục tiêu thời gian:** 8 phút

**Đề bài:**
1. Tạo 1 Pod tên `before-backup` (`nginx:1.27`) trong namespace `default`.
2. Backup dữ liệu `etcd` hiện tại ra file `/opt/cka-etcd-backup.db`.
3. Xóa pod `before-backup` và tạo 1 pod mới tên `after-backup` (`nginx:1.27`).
4. Thực hiện **Restore** cụm về đúng thời điểm snapshot `/opt/cka-etcd-backup.db` bằng thư mục mới `/var/lib/etcd-from-backup`.
5. Xác nhận `before-backup` đã quay trở lại và `after-backup` biến mất.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.4</summary>

```bash
# 1. Tạo dữ liệu trước khi backup
k run before-backup --image=nginx:1.27

# 2. Soi cert & endpoint từ manifest etcd
grep -E 'listen-client-urls|trusted-ca-file|cert-file|key-file' /etc/kubernetes/manifests/etcd.yaml

# Backup snapshot
ETCDCTL_API=3 etcdctl snapshot save /opt/cka-etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Kiểm tra snapshot
etcdutl snapshot status /opt/cka-etcd-backup.db -w table

# 3. Thay đổi trạng thái cụm để kiểm chứng sau restore
k delete pod before-backup $now
k run after-backup --image=nginx:1.27

# 4. Restore snapshot sang thư mục mới (Những gi trị này có trong cat /etc/kubernetes/manifests/etcd.yaml)
etcdutl snapshot restore /opt/cka-etcd-backup.db \
  --name=controlplane \
  --initial-cluster=controlplane=https://172.30.1.2:2380 \
  --initial-advertise-peer-urls=https://172.30.1.2:2380 \
  --data-dir=/var/lib/etcd-from-backup

# Sao lưu manifest và đổi hostPath của volume etcd-data
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
sed -i 's|path: /var/lib/etcd$|path: /var/lib/etcd-from-backup|' /etc/kubernetes/manifests/etcd.yaml

# 5. Chờ 20-40s cho etcd & apiserver khởi động lại rồi verify
crictl ps | grep -E 'etcd|kube-apiserver'
k get pods
# Kết quả mong đợi: có `before-backup`, KHÔNG có `after-backup`!
```
</details>

---

## Lab 1.5: Helm — Quản Lý Release, Tùy Biến Values & Rollback
**Mục tiêu thời gian:** 6 phút

**Đề bài:**
1. Thêm Helm repo `bitnami` (`https://charts.bitnami.com/bitnami`) hoặc dùng bất kỳ repo công khai nào (ví dụ `ingress-nginx` / `prometheus-community`).
2. Tạo namespace `helm-lab`.
3. Render trước chart `bitnami/nginx` với `replicaCount=2` ra file `/tmp/nginx-rendered.yaml` **mà chưa cài lên cluster**.
4. Cài đặt release tên `shop-web` từ chart `bitnami/nginx` vào namespace `helm-lab` với `replicaCount=2` và `service.type=ClusterIP`.
5. Nâng cấp release `shop-web` lên `replicaCount=3`, kiểm tra lịch sử revision, sau đó **rollback** về revision `1`.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.5</summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Render template không cài
helm template shop-web bitnami/nginx --set replicaCount=2 > /tmp/nginx-rendered.yaml
head -n 25 /tmp/nginx-rendered.yaml

# Cài đặt
helm install shop-web bitnami/nginx -n helm-lab --create-namespace \
  --set replicaCount=2 --set service.type=ClusterIP

# Upgrade & kiểm tra history
helm upgrade shop-web bitnami/nginx -n helm-lab \
  --set replicaCount=3 --set service.type=ClusterIP
helm history shop-web -n helm-lab

# Rollback về revision 1
helm rollback shop-web 1 -n helm-lab

# === VERIFY ===
helm list -n helm-lab
k get deploy -n helm-lab    # READY 2/2
```
</details>

---

## Lab 1.6: Kustomize — Base & Overlays (Dev vs Prod)
**Mục tiêu thời gian:** 7 phút

**Đề bài:**
Tạo cấu trúc thư mục Kustomize trong `/tmp/kustomize-lab`:
1. `base/`: chứa Deployment `web` (1 replica, image `nginx:1.26`) và `kustomization.yaml`.
2. `overlays/prod/`: kế thừa `../../base`, thiết lập:
   - `namespace: prod-app`
   - `namePrefix: prod-`
   - Đổi image `nginx` sang tag `1.27`
   - Patch tăng `replicas` từ `1` lên `3`
3. Tạo namespace `prod-app`, preview kết quả bằng `kubectl kustomize` rồi áp dụng bằng `kubectl apply -k`.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.6</summary>

```bash
mkdir -p /tmp/kustomize-lab/base /tmp/kustomize-lab/overlays/prod

cat <<'EOF' > /tmp/kustomize-lab/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
EOF

cat <<'EOF' > /tmp/kustomize-lab/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF

cat <<'EOF' > /tmp/kustomize-lab/overlays/prod/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
EOF

cat <<'EOF' > /tmp/kustomize-lab/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-app
namePrefix: prod-
resources:
  - ../../base
images:
  - name: nginx
    newTag: "1.27"
patches:
  - path: replica-patch.yaml
EOF

k create ns prod-app
k kustomize /tmp/kustomize-lab/overlays/prod
k apply -k /tmp/kustomize-lab/overlays/prod

# === VERIFY ===
k get deploy prod-web -n prod-app -o custom-columns='NAME:.metadata.name,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image'
# Mong đợi: prod-web   3   nginx:1.27
```
</details>

---

## Lab 1.7: Làm Việc Với CRD (Custom Resource Definition)
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
1. Liệt kê toàn bộ CRD hiện có trong cluster ra file `/tmp/crds.txt`.
2. Cài đặt một CRD tùy chỉnh tên `backups.ops.example.com` (kind: `Backup`, shortName: `bk`, phạm vi `Namespaced`, có trường `spec.schedule` dạng string và `spec.retentionDays` dạng integer).
3. Dùng `kubectl explain backup.spec` để xác nhận API Server đã nhận diện schema mới.
4. Tạo 1 đối tượng `Backup` tên `nightly-db` với `schedule: "0 2 * * *"` và `retentionDays: 7`, rồi kiểm tra bằng shortName `k get bk`.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.7</summary>

```bash
k get crd > /tmp/crds.txt

cat <<'EOF' | k apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.ops.example.com
spec:
  group: ops.example.com
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
      - bk
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                schedule:
                  type: string
                retentionDays:
                  type: integer
EOF

k explain backup.spec

cat <<'EOF' | k apply -f -
apiVersion: ops.example.com/v1alpha1
kind: Backup
metadata:
  name: nightly-db
spec:
  schedule: "0 2 * * *"
  retentionDays: 7
EOF

# === VERIFY ===
k get bk nightly-db -o yaml
```
</details>

---

## Lab 1.8: Trích Xuất Thông Tin Cụm Bằng JSONPath & Custom-Columns
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Ghi danh sách tất cả các node cùng `InternalIP` (mỗi node 1 dòng dạng `<node-name> <ip>`) ra file `/tmp/node-ips.txt`.
2. Tìm tất cả các Pod đang chạy trên toàn cluster, sắp xếp tăng dần theo **thời gian tạo** và xuất tên namespace + tên pod ra `/tmp/pods-sorted.txt`.

<details>
<summary>Xem đáp án & lệnh verify Lab 1.8</summary>

```bash
k get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}' > /tmp/node-ips.txt
cat /tmp/node-ips.txt

k get pods -A --sort-by='.metadata.creationTimestamp' --no-headers -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name' > /tmp/pods-sorted.txt
head -n 10 /tmp/pods-sorted.txt
```
</details>

