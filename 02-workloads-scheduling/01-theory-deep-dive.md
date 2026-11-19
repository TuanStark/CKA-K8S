# Module 02: Workloads & Scheduling (15%) — Hiểu Sâu Bản Chất

> **Mục tiêu bản chất:** Hiểu cách `kube-scheduler` ra quyết định đặt Pod lên Node nào, cơ chế quản lý tài nguyên (CPU throttling vs Memory OOMKill), tự động mở rộng (HPA), cách truyền cấu hình động/tĩnh và các dạng Pod đặc biệt (Static Pod, DaemonSet, Sidecar/InitContainer).

---

## 1. Kiến Trúc Đường Ống Lập Lịch (`kube-scheduler` Pipeline)

Khi một Pod mới xuất hiện chưa có `spec.nodeName`, `kube-scheduler` chạy qua **2 pha cốt lõi**:

```text
Unscheduled Pod
       │
       ▼
┌─────────────────────────────────────────┐
│ PHA 1: FILTERING (Lọc cứng — Đạt/Loại)  │  Loại các Node không đủ CPU/RAM, bị Taint
│  • Taints & Tolerations                 │  mà Pod không tolerate, sai nodeSelector /
│  • nodeSelector / required NodeAffinity │  required NodeAffinity, hoặc đang Unschedulable
│  • Resource requests fit                │
└──────────────────┬──────────────────────┘
                   │ (Còn các Node đạt chuẩn)
                   ▼
┌─────────────────────────────────────────┐
│ PHA 2: SCORING (Chấm điểm 0–100)        │  Ưu tiên Node còn nhiều tài nguyên trống,
│  • preferred NodeAffinity (weight)      │  khớp preferred Affinity / AntiAffinity
│  • PodTopologySpread / ResourceBalance  │
└──────────────────┬──────────────────────┘
                   │
                   ▼
         Ghi spec.nodeName (Bind)
```

> **Tư duy Kubestronaut:** Nếu **không có Node nào vượt qua Pha 1**, Pod sẽ kẹt ở trạng thái `Pending` kèm event `FailedScheduling`! Muốn biết vì sao `Pending`, chỉ cần chạy `kubectl describe pod <name>` — Scheduler luôn ghi chính xác số node bị loại vì từng lý do (ví dụ: `0/2 nodes are available: 1 node(s) had untolerated taint, 1 Insufficient memory`).

---

## 2. Phân Biệt Taints/Tolerations vs NodeAffinity (Hay Bị Nhầm!)

Đây là cặp khái niệm **nhiều người học CKA nhầm lẫn nhất**:

| Cơ chế | Đặt ở đâu? | Bản chất thực sự | Ví dụ thực tế |
| :--- | :--- | :--- | :--- |
| **Taints & Tolerations** | Taint gắn trên **Node**; Toleration gắn trên **Pod** | **Đẩy Pod ra khỏi Node** (Node *từ chối* mọi Pod trừ khi Pod có "giấy thông hành" Toleration). **Lưu ý:** Có Toleration **KHÔNG đảm bảo** Pod sẽ chắc chắn vào Node đó — nó vẫn có thể chạy trên Node bình thường khác! | Control Plane node có taint `node-role.kubernetes.io/control-plane:NoSchedule` để tránh app chạy nhầm lên Control Plane |
| **NodeSelector / NodeAffinity** | Label gắn trên **Node**; Affinity gắn trên **Pod** | **Hút Pod về phía Node** (Pod chỉ muốn chạy trên các Node có nhãn khớp điều kiện). | Pod GPU chỉ chạy trên các node có label `gpu=nvidia` |
| **Kết hợp cả hai** | Taint + Label trên Node; Toleration + NodeAffinity trên Pod | **Độc quyền hoàn toàn (Dedicated Node):** Chỉ Pod A mới được vào Node X, và Pod A cũng chỉ chạy trên Node X! | Nhóm node riêng cho workload thanh toán `payments` |

### 3 Effect của Taint cần thuộc:
- `NoSchedule`: Pod mới không tolerated sẽ không được schedule; Pod đang chạy giữ nguyên.
- `PreferNoSchedule`: Cố tránh nhưng nếu hết chỗ vẫn schedule.
- `NoExecute`: Pod mới không được vào **VÀ** Pod đang chạy không có toleration sẽ bị đuổi (evict) ngay (hoặc sau `tolerationSeconds`).

---

## 3. Requests, Limits, QoS, LimitRange, ResourceQuota & HPA

### Bản chất CPU vs Memory (Cực kỳ quan trọng khi debug!)
- **CPU là tài nguyên nén được (compressible):** Nếu container vượt `limits.cpu`, Linux CFS sẽ **bóp băng thông (throttling)** làm app chạy chậm hơn — **KHÔNG giết container**.
- **Memory là tài nguyên KHÔNG nén được (incompressible):** Nếu container vượt `limits.memory`, Linux kernel OOM killer sẽ **giết ngay container** → bạn sẽ thấy trạng thái `OOMKilled` (exit code `137`)!
- **`kube-scheduler` chỉ nhìn `requests` (không nhìn `limits`)** khi tính xem Node còn đủ chỗ chứa Pod hay không.

### LimitRange vs ResourceQuota
- **`LimitRange` (cấp Pod/Container trong Namespace):** Thiết lập giá trị `default` (limits), `defaultRequest` (requests), `min`, `max` cho **từng Container/Pod**.
- **`ResourceQuota` (cấp tổng Namespace):** Chặn tổng tài nguyên (ví dụ tổng `requests.cpu <= 4`, tổng số `pods <= 10`) của **cả Namespace**. *Bẫy thi:* Khi Namespace có `ResourceQuota` cho CPU/Memory nhưng không có `LimitRange`, mọi Pod tạo ra **không khai báo `requests`/`limits`** sẽ bị API Server từ chối tạo!

### HPA (`HorizontalPodAutoscaler`) — Trọng tâm mới CKA 2025/2026
- HPA hỏi **Metrics Server** định kỳ (mặc định 15s) để lấy mức sử dụng CPU/Memory thực tế của các Pod thuộc Deployment/StatefulSet.
- Công thức cơ bản:
  `desiredReplicas = ceil[ currentReplicas * ( currentMetricValue / desiredMetricValue ) ]`
- **Điều kiện bắt buộc để HPA hoạt động theo `%`:**
  1. **Metrics Server** phải đang chạy (`kubectl top pods` ra kết quả).
  2. Container trong Deployment **bắt buộc phải khai báo `resources.requests.cpu` (hoặc `memory`)** — nếu không có `requests`, HPA không có mẫu số để tính `%` và sẽ hiện `<unknown>/50%`!

---

## 4. Deployment Rollout, Probes, ConfigMap/Secret & PriorityClass

### A. Deployment Strategy & Rollback
- Deployment quản lý **ReplicaSet**; mỗi lần thay đổi `spec.template` (image, env, volume...), Deployment tạo ReplicaSet mới và dịch chuyển Pod theo `maxSurge` & `maxUnavailable`.
- Lệnh sống còn: `k rollout status`, `k rollout history`, `k rollout undo deploy/<name> --to-revision=<n>`.

### B. 3 Loại Probes (Bản chất khác nhau hoàn toàn!)
- **`readinessProbe`:** Kiểm tra container đã **sẵn sàng nhận traffic** chưa. Nếu fail → gỡ Pod IP khỏi **Service Endpoints** (KHÔNG restart container!).
- **`livenessProbe`:** Kiểm tra tiến trình bên trong có bị treo/deadlock không. Nếu fail → `kubelet` **kill & restart container**.
- **`startupProbe`:** Dùng cho app khởi động chậm; vô hiệu hóa liveness/readiness cho đến khi app boot xong lần đầu.

### C. ConfigMap & Secret — Hành vi cập nhật
- Nạp qua **`env` / `envFrom`**: Chỉ đọc 1 lần lúc container start. Sửa ConfigMap/Secret **không tự cập nhật** vào container cho đến khi restart Pod.
- Nạp qua **`volumeMount`** (không dùng `subPath`): `kubelet` tự đồng bộ file mới sau vài chục giây. *(Lưu ý: nếu dùng `subPath`, file cũng **không** tự cập nhật!)*

### D. PriorityClass
- Gán độ ưu tiên số nguyên (`value`) cho Pod (`spec.priorityClassName`).
- Khi cụm hết tài nguyên, Pod có độ ưu tiên cao hơn có thể **đuổi (preempt)** Pod độ ưu tiên thấp hơn (trừ khi `preemptionPolicy: Never`).

### E. Static Pods vs DaemonSets
- **Static Pod:** Được quản lý trực tiếp bởi `kubelet` trên 1 node cụ thể qua file YAML trong `staticPodPath` (mặc định `/etc/kubernetes/manifests`). API Server chỉ thấy 1 **mirror pod** có hậu tố `-<nodename>` và **không thể xóa bằng `kubectl delete pod`** (muốn xóa phải xóa file YAML trên node đó!).
- **DaemonSet:** Đảm bảo chạy đúng 1 bản sao Pod trên mỗi Node (hoặc nhóm Node khớp selector), thường dùng cho log collector, monitoring agent, CNI plugin, `kube-proxy`.

---

**Đã nắm chắc bản chất Module 02?** Bắt tay vào thực hành 7 bài lab tại [02-hands-on-labs.md](./02-hands-on-labs.md).

