# Module 02: Hands-On Labs — Workloads & Scheduling (15%)

> **Hướng dẫn:** Bấm giờ từng bài lab, chỉ mở `kubernetes.io/docs` khi cần tra YAML, tự chạy lệnh Verify rồi mới đối chiếu đáp án.

---

## Lab 2.1: Deployment Rolling Update, Record & Rollback
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo namespace `workloads`.
2. Tạo Deployment tên `frontend` gồm `3` replicas dùng image `nginx:1.25` trong namespace `workloads`.
3. Cập nhật image của `frontend` lên `nginx:1.27` và cấu hình chiến lược `RollingUpdate` với `maxUnavailable: 1`, `maxSurge: 1`.
4. Kiểm tra lịch sử rollout, sau đó **rollback** về đúng revision đầu tiên (`nginx:1.25`).

<details>
<summary>Xem đáp án & lệnh verify Lab 2.1</summary>

```bash
k create ns workloads
k create deploy frontend -n workloads --image=nginx:1.25 --replicas=3

# Cập nhật strategy và image
k patch deploy frontend -n workloads -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'
k set image deploy/frontend nginx=nginx:1.27 -n workloads
k rollout status deploy/frontend -n workloads

# Xem lịch sử & rollback
k rollout history deploy/frontend -n workloads
k rollout undo deploy/frontend -n workloads --to-revision=1
k rollout status deploy/frontend -n workloads

# === VERIFY ===
k get deploy frontend -n workloads -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'  # nginx:1.25
```
</details>

---

## Lab 2.2: Dedicated Node — Kết Hợp Taint + Toleration + NodeAffinity
**Mục tiêu thời gian:** 6 phút

**Đề bài:**
1. Chọn 1 worker node (ví dụ `node01`, nếu cụm chỉ có 1 node thì dùng node đó). Gắn label `workload=gpu` và taint `dedicated=gpu:NoSchedule`.
2. Tạo Pod `regular-pod` (`nginx:1.27`) không có toleration → xác nhận nó **không** được schedule lên `node01` (nếu chỉ có 1 node nó sẽ ở trạng thái `Pending`).
3. Tạo Pod `gpu-pod` (`nginx:1.27`) có cả **Toleration** khớp `dedicated=gpu:NoSchedule` và **`requiredDuringSchedulingIgnoredDuringExecution` NodeAffinity** yêu cầu label `workload=gpu`.
4. Xác nhận `gpu-pod` chạy đúng trên `node01`. *(Sau khi xong lab nhớ gỡ taint/label để không ảnh hưởng các lab sau!)*

<details>
<summary>Xem đáp án & lệnh verify Lab 2.2</summary>

```bash
NODE=$(k get nodes --no-headers | awk '!/control-plane/{print $1; exit}')
[ -z "$NODE" ] && NODE=$(k get nodes --no-headers | awk '{print $1; exit}')

k label node "$NODE" workload=gpu
k taint node "$NODE" dedicated=gpu:NoSchedule

k run regular-pod --image=nginx:1.27

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values:
                  - gpu
  containers:
    - name: app
      image: nginx:1.27
EOF

# === VERIFY ===
k get pods -o wide

# === DỌN DẸP SAU LAB ===
k delete pod regular-pod gpu-pod $now
k taint node "$NODE" dedicated=gpu:NoSchedule-
k label node "$NODE" workload-
```
</details>

---

## Lab 2.3: ConfigMap & Secret (Cả `env` Và `volumeMount` + ReadOnly)
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
1. Trong namespace `workloads`, tạo ConfigMap `app-cfg` với `APP_MODE=production` và `LOG_LEVEL=info`.
2. Tạo Secret `db-creds` với `DB_USER=admin` và `DB_PASS=Sup3rS3cr3t`.
3. Tạo Pod `cfg-pod` (`busybox:1.36`, chạy `sleep 3600`):
   - Nạp toàn bộ key của `app-cfg` thành biến môi trường (`envFrom`).
   - Mount Secret `db-creds` vào đường dẫn `/etc/db-creds` ở chế độ `readOnly: true`.

<details>
<summary>Xem đáp án & lệnh verify Lab 2.3</summary>

```bash
k create cm app-cfg -n workloads --from-literal=APP_MODE=production --from-literal=LOG_LEVEL=info
k create secret generic db-creds -n workloads --from-literal=DB_USER=admin --from-literal=DB_PASS=Sup3rS3cr3t

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cfg-pod
  namespace: workloads
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      envFrom:
        - configMapRef:
            name: app-cfg
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/db-creds
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-creds
EOF

# === VERIFY ===
k exec cfg-pod -n workloads -- sh -c 'env | grep -E "APP_MODE|LOG_LEVEL"; echo "---"; cat /etc/db-creds/DB_USER; echo ""'
```
</details>

---

## Lab 2.4: HPA (HorizontalPodAutoscaler) Chuẩn CKA 2025/2026
**Mục tiêu thời gian:** 6 phút

**Đề bài:**
1. Trong namespace `workloads`, tạo Deployment `autoscale-web` (`nginx:1.27`, 1 replica) có `resources.requests`: `cpu: 50m`, `memory: 64Mi` và `resources.limits`: `cpu: 200m`, `memory: 128Mi`.
2. Tạo HPA tên `autoscale-web-hpa` (dùng `autoscaling/v2`) theo dõi `autoscale-web`:
   - `minReplicas: 1`, `maxReplicas: 4`
   - Mục tiêu trung bình CPU utilization = `50%`
   - Cấu hình `behavior.scaleDown.stabilizationWindowSeconds: 30` (hay gặp trong đề mới!).

<details>
<summary>Xem đáp án & lệnh verify Lab 2.4</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: autoscale-web
  namespace: workloads
spec:
  replicas: 1
  selector:
    matchLabels:
      app: autoscale-web
  template:
    metadata:
      labels:
        app: autoscale-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: autoscale-web-hpa
  namespace: workloads
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: autoscale-web
  minReplicas: 1
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
EOF

# === VERIFY ===
k get hpa -n workloads
k describe hpa autoscale-web-hpa -n workloads
```
</details>

---

## Lab 2.5: Multi-Container Pod (InitContainer + Sidecar Log Shipper)
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
Tạo Pod tên `log-app` trong namespace `workloads` chia sẻ một volume `emptyDir` tên `log-vol`:
1. `initContainer` tên `init-setup` (`busybox:1.36`): ghi dòng `"App initialized"` vào `/var/log/app/init.log`.
2. Container chính `writer` (`busybox:1.36`): mỗi 5 giây ghi timestamp vào `/var/log/app/app.log`.
3. Container phụ (sidecar) `reader` (`busybox:1.36`): chạy `tail -n+1 -f /var/log/app/app.log` để đẩy log ra stdout.

<details>
<summary>Xem đáp án & lệnh verify Lab 2.5</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-app
  namespace: workloads
spec:
  initContainers:
    - name: init-setup
      image: busybox:1.36
      command: ["sh", "-c", "echo 'App initialized' > /var/log/app/init.log"]
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /var/log/app/app.log; sleep 5; done"]
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -n+1 -F /var/log/app/app.log"]
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app
  volumes:
    - name: log-vol
      emptyDir: {}
EOF

# === VERIFY ===
k get pod log-app -n workloads             # READY 2/2
k logs log-app -n workloads -c reader      # Thấy các dòng date chạy liên tục
```
</details>

---

## Lab 2.6: PriorityClass & Gắn Độ Ưu Tiên Cho Deployment
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Kiểm tra các `PriorityClass` hiện có trong cluster và tìm giá trị cao nhất của các PriorityClass do người dùng tạo (bỏ qua `system-cluster-critical` và `system-node-critical`).
2. Tạo một `PriorityClass` mới tên `high-priority-app` với `value: 100000` và `description: "Critical business workloads"`.
3. Cập nhật Deployment `frontend` trong namespace `workloads` để dùng `priorityClassName: high-priority-app`.

<details>
<summary>Xem đáp án & lệnh verify Lab 2.6</summary>

```bash
k get priorityclasses

cat <<'EOF' | k apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-app
value: 100000
globalDefault: false
description: "Critical business workloads"
EOF

k patch deploy frontend -n workloads -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority-app"}}}}'

# === VERIFY ===
k get pods -n workloads -l app=frontend -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName'
```
</details>

---

## Lab 2.7: Static Pod Trực Tiếp Trên Node
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
1. SSH vào 1 node (ví dụ `controlplane` hoặc `node01`). Tìm chính xác đường dẫn `staticPodPath` trong file cấu hình của `kubelet`.
2. Tạo một Static Pod tên `static-web` chạy image `nginx:1.27` tại thư mục đó.
3. Quay lại control plane xác nhận Pod `static-web-<nodename>` đang `Running`.
4. Thử xóa bằng `k delete pod static-web-<nodename>` → quan sát chuyện gì xảy ra, sau đó xóa đúng cách bằng việc xóa file YAML trên node.

<details>
<summary>Xem đáp án & lệnh verify Lab 2.7</summary>

```bash
# Kiểm tra staticPodPath trên node
grep staticPodPath /var/lib/kubelet/config.yaml
# Mặc định: /etc/kubernetes/manifests

k run static-web --image=nginx:1.27 $do > /etc/kubernetes/manifests/static-web.yaml

# === VERIFY ===
k get pods -A | grep static-web

# Dọn dẹp đúng cách (xóa file manifest)
rm -f /etc/kubernetes/manifests/static-web.yaml
```
</details>

