# Hướng Dẫn Sử Dụng Killercoda & Killer.sh Để Luyện Thi CKA (Chuẩn Kubestronaut)

**Killercoda** và **Killer.sh** đều được sáng lập bởi **Kim Wuestkamp** (đối tác chính thức của Linux Foundation / CNCF cho môi trường thi thử CKA/CKAD/CKS). Đây là 2 vũ khí quan trọng nhất giúp bạn đỗ CKA.

---

## 1. Phân Biệt Killercoda vs Killer.sh

| Tiêu chí | Killercoda (`killercoda.com`) | Killer.sh (`killer.sh`) |
| :--- | :--- | :--- |
| **Mục đích** | **Luyện tập hàng ngày theo từng chuyên đề / kịch bản nhỏ** | **Thi thử tổng lực 120 phút (Simulator thi thật)** |
| **Chi phí** | **100% Miễn phí** (chỉ cần tài khoản GitHub/Google) | Tặng kèm **2 phiên miễn phí** khi đăng ký thi CKA trên Linux Foundation (hoặc mua riêng) |
| **Thời lượng môi trường** | ~60 phút mỗi bài (hết giờ có thể khởi động lại ngay) | Mỗi phiên kéo dài **36 giờ** liên tục |
| **Độ khó** | Bám sát từng kỹ năng cụ thể, có gợi ý và bài giải | **Khó hơn đề thi thật ~20–30%**, câu hỏi dài và phức tạp |
| **Môi trường cụm** | Cụm `kubeadm` thật (thường 2 node: `controlplane` + `node01`) trên trình duyệt | Cụm đa node, giao diện máy ảo remote desktop giống hệt phòng thi PSI |

---

## 2. Top Kịch Bản Killercoda CKA Bắt Buộc Phải Làm

Truy cập: **[killercoda.com/killer-shell-cka](https://killercoda.com/killer-shell-cka)**

### Nhóm 1: Cluster Architecture & Quản Trị Hệ Thống (Trọng số cao)
1. **Cluster Upgrade (Kubeadm):** Nâng cấp Control Plane và Worker Node từ minor version này sang minor version tiếp theo.
2. **etcd Backup and Restore:** Snapshot etcd, restore ra thư mục mới, chỉnh sửa `etcd.yaml` static pod.
3. **RBAC:** Tạo ServiceAccount, Role, RoleBinding và xác minh quyền với `kubectl auth can-i`.
4. **Kubelet Troubleshooting:** Sửa lỗi worker node khi kubelet không chạy được do cấu hình sai.

### Nhóm 2: Services & Networking
1. **NetworkPolicy:** Viết policy chặn Ingress/Egress, phân biệt logic AND vs OR, mở port DNS 53.
2. **Ingress & CoreDNS:** Tạo Ingress rules, cấu hình backend và debug DNS resolution.
3. **Gateway API (Nâng cao):** Cấu hình `Gateway` và `HTTPRoute` theo chuẩn mới.

### Nhóm 3: Troubleshooting (30% điểm thi CKA)
1. **Control Plane Failure:** Sửa lỗi khi kube-apiserver hoặc kube-scheduler bị crash (kiểm tra `/etc/kubernetes/manifests/` và `crictl`).
2. **Broken Pod / Application:** Pod kẹt Pending, CrashLoopBackOff, OOMKilled, sai probes.

### Nhóm 4: Playground Tự Do (Free Sandbox)
- **Kubernetes Playground (2 Node):** `killercoda.com/playgrounds/scenario/kubernetes`
  - Dùng để tự thực hành các bài lab trong thư mục `01-cluster-architecture`, `02-workloads-scheduling`, `03-storage`, `04-services-networking`, `05-troubleshooting` mà không cần cài đặt nặng máy.

---

## 3. Quy Trình Sử Dụng Killercoda Hiệu Quả (Tránh Học Vẹt)

1. **Bước 1: Không bấm "Show Solution" ngay.** Đọc đề, mở terminal và tự làm bằng lệnh imperative hoặc tra `kubernetes.io/docs`.
2. **Bước 2: Tự Verify.** Kiểm tra trạng thái tài nguyên (`kubectl get`, `kubectl describe`, `curl`, `nslookup`).
3. **Bước 3: Đối chiếu đáp án.** So sánh cách làm của bạn với giải pháp của Killercoda xem họ dùng cờ/lệnh nào tối ưu tốc độ hơn.
4. **Bước 4: Ghi vào [ERROR-LOG.md](./ERROR-LOG.md).** Nếu bị sai hoặc làm quá 6 phút/bài, ghi lại lý do và hôm sau làm lại từ đầu.

