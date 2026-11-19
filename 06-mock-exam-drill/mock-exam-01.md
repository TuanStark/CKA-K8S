# Mock Exam #01 — Đề Thi Thử Chuẩn Form CKA (Cập Nhật 2025/2026)

## Quy Định Làm Bài (Bắt Buộc Nghiêm Túc Như Phòng Thi PSI)
- **Thời lượng:** Đúng **120 phút** (bấm đồng hồ đếm ngược, không dừng giữa chừng).
- **Tổng số câu:** **16 câu** — Tổng điểm: **100%** — Điểm đạt: **≥ 66%** (Mục tiêu Kubestronaut: **≥ 85%**).
- **Tài liệu được phép mở:** CHỈ `kubernetes.io/docs`, `gateway.networking.k8s.io`, `helm.sh/docs`. **Cấm Google, cấm ChatGPT, cấm mở `CHEATSHEET.md` hay file đáp án.**
- **Chiến thuật khuyến nghị:** Áp dụng **Two-Pass Strategy** — câu nào quá 7 phút chưa thông → ghi chú lại để bỏ qua và quay lại ở lượt 2.

| Domain | Câu hỏi | Tổng điểm |
| :--- | :--- | :---: |
| **Cluster Architecture, Install & Config** | Câu 1 – Câu 4 | **25%** |
| **Workloads & Scheduling** | Câu 5 – Câu 7 | **15%** |
| **Storage** | Câu 8 – Câu 9 | **10%** |
| **Services & Networking** | Câu 10 – Câu 12 | **20%** |
| **Troubleshooting** | Câu 13 – Câu 16 | **30%** |

---

## PHẦN 1: CLUSTER ARCHITECTURE, INSTALLATION & CONFIGURATION (25%)

### Câu 1 (5%) — RBAC & ServiceAccount
Trong namespace `finance`:
1. Tạo ServiceAccount tên `report-sa`.
2. Tạo Role tên `report-role` chỉ cho phép `get`, `list` trên tài nguyên `pods`, `services` và `configmaps`.
3. Gắn `report-role` cho `report-sa` qua RoleBinding tên `report-rb`.
4. Tạo Pod tên `report-runner` dùng image `busybox:1.36` (`sleep 3600`) chạy dưới quyền `report-sa`.

---

### Câu 2 (7%) — Nâng Cấp Worker Node Bằng `kubeadm`
Cụm hiện có Control Plane đã được nâng cấp sẵn lên phiên bản mục tiêu `v1.32.x`, còn worker node `node01` vẫn đang ở `v1.31.x`.
1. Drain an toàn `node01` (bỏ qua DaemonSets và cho phép xóa emptyDir).
2. SSH vào `node01`, nâng cấp `kubeadm`, chạy `kubeadm upgrade node`, sau đó nâng cấp `kubelet` và `kubectl` lên đúng phiên bản khớp với Control Plane.
3. Khởi động lại `kubelet`, uncordon `node01` và đảm bảo `node01` trở lại trạng thái `Ready`.

*(Nếu bạn làm bài trên cụm lab 1 node hoặc đã ở bản mới nhất, hãy viết đầy đủ kịch bản lệnh chuẩn ra file `/tmp/q2-upgrade.sh` để tự chấm theo rubric.)*

---

### Câu 3 (7%) — Backup & Restore `etcd`
1. Chụp snapshot `etcd` của cụm hiện tại và lưu vào `/opt/exam-etcd-snap.db`.
2. Xác minh tính hợp lệ của snapshot.
3. Tạo một Pod kiểm chứng tên `canary-delete-me` (`nginx:1.27`) trong namespace `default`.
4. Thực hiện **restore** snapshot `/opt/exam-etcd-snap.db` ra thư mục `/var/lib/etcd-exam-restore`, cập nhật manifest `etcd` và xác nhận `canary-delete-me` đã biến mất sau khi cụm khôi phục.

---

### Câu 4 (6%) — Helm & Kustomize (Curriculum Mới)
1. **Helm:** Cài đặt chart `bitnami/nginx` (hoặc chart nginx có sẵn trong repo của bạn) với tên release `exam-web` trong namespace `helm-exam`, cấu hình `replicaCount=2` và `service.type=ClusterIP`.
2. **Kustomize:** Tạo cấu trúc Kustomize tại `/tmp/exam-kust` gồm:
   - `base/`: Deployment `api` (`nginx:1.26`, `1` replica) + `kustomization.yaml`.
   - `overlays/staging/`: đặt `namespace: staging`, `namePrefix: stg-`, nâng image `nginx` lên tag `1.27` và patch `replicas: 2`.
   - Tạo namespace `staging` và áp dụng overlay `staging` bằng `kubectl apply -k`.

---

## PHẦN 2: WORKLOADS & SCHEDULING (15%)

### Câu 5 (5%) — HPA (HorizontalPodAutoscaler v2)
Trong namespace `autoscale`:
1. Tạo Deployment `checkout-api` (`nginx:1.27`, `1` replica) với `resources.requests`: `cpu: 100m`, `memory: 128Mi` và `resources.limits`: `cpu: 250m`, `memory: 256Mi`.
2. Tạo HPA (`autoscaling/v2`) tên `checkout-hpa` cho `checkout-api`:
   - `minReplicas: 1`, `maxReplicas: 5`
   - Target CPU averageUtilization: `60%`
   - `behavior.scaleDown.stabilizationWindowSeconds: 45`.

---

### Câu 6 (5%) — Scheduling Với Taint, Toleration & NodeAffinity
1. Gắn label `tier=backend` và taint `tier=backend:NoSchedule` lên một worker node (ví dụ `node01`).
2. Tạo Deployment `order-worker` (`2` replicas, image `nginx:1.27`) trong namespace `default` sao cho:
   - Các Pod **bắt buộc** chỉ được lập lịch lên các node có label `tier=backend` (`requiredDuringSchedulingIgnoredDuringExecution`).
   - Có Toleration tương ứng với `tier=backend:NoSchedule`.

---

### Câu 7 (5%) — PriorityClass & Sidecar Log Pod
1. Tạo `PriorityClass` tên `mission-critical` có `value: 500000`, `globalDefault: false`.
2. Tạo Pod tên `audit-logger` trong namespace `default` sử dụng `priorityClassName: mission-critical`, có volume `emptyDir` tên `shared-logs` mount vào `/var/log/app` ở cả 2 container:
   - Container `app` (`busybox:1.36`): ghi `"tx-ok"` mỗi 5 giây vào `/var/log/app/tx.log`.
   - Container `sidecar` (`busybox:1.36`): chạy `tail -n+1 -F /var/log/app/tx.log`.

---

## PHẦN 3: STORAGE (10%)

### Câu 8 (5%) — StorageClass & PV/PVC Binding
1. Tạo StorageClass tên `exam-sc` với `provisioner: kubernetes.io/no-provisioner`, `volumeBindingMode: WaitForFirstConsumer`, `reclaimPolicy: Retain`, `allowVolumeExpansion: true`.
2. Tạo PersistentVolume `exam-pv` (`2Gi`, `ReadWriteOnce`, `storageClassName: exam-sc`, `hostPath.path: /mnt/exam-vol`) và PersistentVolumeClaim `exam-pvc` trong namespace `default` yêu cầu `1Gi`, `ReadWriteOnce`, `storageClassName: exam-sc`.
3. Tạo Pod `storage-consumer` (`nginx:1.27`) mount `exam-pvc` vào `/data` để kích hoạt binding (`WaitForFirstConsumer` → `Bound`).

---

### Câu 9 (5%) — Trích Xuất PV Theo Dung Lượng
Sắp xếp tất cả các PersistentVolume hiện có trong cụm theo **dung lượng tăng dần** (`.spec.capacity.storage`) và ghi toàn bộ output của lệnh `kubectl get pv` ra file `/opt/pv-sorted.txt`.

---

## PHẦN 4: SERVICES & NETWORKING (20%)

### Câu 10 (6%) — NetworkPolicy (AND Logic + Egress DNS)
Trong cụm có 2 namespace:
- Namespace `secure-db` (label `zone=db`) chứa Pod `postgres` (`nginx:1.27`, label `app=postgres`, port `80`) và Service `pg-svc`.
- Namespace `secure-app` (label `zone=app`) chứa Pod `backend` (`busybox:1.36`, label `app=backend`, `sleep 3600`) và Pod `intruder` (`busybox:1.36`, label `app=intruder`, `sleep 3600`).

**Yêu cầu:**
1. Tạo NetworkPolicy `db-ingress-lock` trong `secure-db` áp lên `app=postgres`: chỉ cho phép Ingress TCP cổng `80` từ các Pod có label `app=backend` **VÀ** nằm trong namespace `zone=app`.
2. Tạo NetworkPolicy `backend-egress-lock` trong `secure-app` áp lên `app=backend`: chỉ cho phép Egress tới `app=postgres` (namespace `zone=db`, TCP 80) **VÀ** CoreDNS (UDP/TCP 53).

---

### Câu 11 (8%) — Migrate Ingress Sang Gateway API (Trọng Tâm Mới)
Trong namespace `web-gw`, hiện có 2 Service: `v1-svc` (port `80`) và `v2-svc` (port `80`), cùng TLS Secret `portal-tls` cho tên miền `portal.cka.local` và `GatewayClass` tên `nginx`.
Hãy tạo cấu hình **Gateway API (`gateway.networking.k8s.io/v1`)** thay thế Ingress cũ:
1. `Gateway` tên `portal-gw` trong `web-gw`:
   - `gatewayClassName: nginx`
   - Listener `https`: protocol `HTTPS`, port `443`, hostname `portal.cka.local`, `tls.mode: Terminate`, `certificateRefs` trỏ tới `portal-tls`.
2. `HTTPRoute` tên `portal-route` trong `web-gw`:
   - Gắn `parentRefs` tới `portal-gw`, `hostnames: ["portal.cka.local"]`.
   - Rule 1: Request có path `PathPrefix /v2` → đẩy 100% tới `v2-svc:80`.
   - Rule 2: Request có path `PathPrefix /` → chia lưu lượng **70%** tới `v1-svc:80` và **30%** tới `v2-svc:80`.

---

### Câu 12 (6%) — NodePort Service & CoreDNS Verification
1. Trong namespace `net-exam`, tạo Deployment `edge-web` (`nginx:1.27`, `2` replicas, port `80`).
2. Expose thành Service `edge-svc` loại `NodePort` với `port: 8080`, `targetPort: 80`, `nodePort: 30180`.
3. Chạy truy vấn `nslookup` tới FQDN của `edge-svc` từ một Pod tạm và ghi kết quả ra `/opt/edge-dns.txt`.

---

## PHẦN 5: TROUBLESHOOTING (30%)

### Câu 13 (6%) — Sửa Pod `OOMKilled` & Thiếu ConfigMap
Trong namespace `trouble-1`:
- Một Deployment `worker-oom` bị crash liên tục vì `limits.memory: 16Mi` trong khi container cần `80Mi` RAM (Exit Code 137).
- Một Pod `cfg-broken` kẹt `CreateContainerConfigError` vì tham chiếu ConfigMap `env-cfg` (key `MODE`) chưa được tạo.
Hãy chẩn đoán và khôi phục cả hai về trạng thái `Running` (`1/1 Ready`), với `MODE=active`.

---

### Câu 14 (6%) — Sửa Service Không Trả Về Traffic
Trong namespace `trouble-2`, Deployment `api-broken` (image `nginx:1.27`, port `80`) đang bị `0/2 Ready` do `readinessProbe` cấu hình sai đường dẫn `/wrong-health` và sai cổng `8080`. Đồng thời Service `api-broken-svc` cấu hình sai `selector` (`app=api-typo`) và sai `targetPort` (`8080`).
Hãy sửa toàn bộ lỗi để `curl http://api-broken-svc.trouble-2` trả về HTTP 200.

---

### Câu 15 (8%) — Khôi Phục Worker Node `NotReady`
Worker node `node01` đang ở trạng thái `NotReady`. Khi SSH vào `node01`, bạn phát hiện dịch vụ `kubelet` liên tục lỗi kết nối do `/etc/kubernetes/kubelet.conf` bị cấu hình sai cổng API (`6444` thay vì `6443`) và dịch vụ `kubelet` chưa được bật tự khởi động (`disabled`).
Hãy tìm lỗi bằng `journalctl`, sửa cấu hình, bật `enable --now kubelet` và xác nhận `node01` trở lại `Ready`.

---

### Câu 16 (10%) — Khôi Phục Control Plane Khi `kubectl` Không Phản Hồi
Trên Control Plane node, `kubectl get nodes` báo `The connection to the server <ip>:6443 was refused`.
Hãy sử dụng `crictl ps -a`, `crictl logs` và `/etc/kubernetes/manifests/kube-apiserver.yaml` để phát hiện lỗi cấu hình (sai cờ `--etcd-servers` cổng `2380` hoặc sai đường dẫn cert), khôi phục `kube-apiserver` hoạt động bình thường và xác nhận toàn bộ các Pod trong `kube-system` ở trạng thái `Running`.

---

**Hết 120 phút?** Mở [mock-exam-01-solutions.md](./mock-exam-01-solutions.md) để tự chấm điểm chi tiết từng câu, ghi điểm vào [PROGRESS-TRACKER.md](../PROGRESS-TRACKER.md) và ghi mọi câu sai/chậm vào [ERROR-LOG.md](../ERROR-LOG.md)!

