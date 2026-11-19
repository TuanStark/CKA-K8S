# CKA Progress Tracker — Theo Dõi Tiến Độ & Điểm Luyện Đề

Đánh dấu `[x]` chỉ khi bạn **tự tay làm lại được không nhìn đáp án** và **biết cách verify kết quả**.

---

## 1. Tiến Độ Theo 5 Domain Chính Thức

### Domain 1: Troubleshooting (30%)
- [ ] Debug Pod `Pending` (taints, nodeSelector/affinity, thiếu CPU/Memory, PVC `Pending`)
- [ ] Debug Pod `ImagePullBackOff` / `ErrImagePull` / `CreateContainerConfigError`
- [ ] Debug Pod `CrashLoopBackOff` / `OOMKilled` (`kubectl logs -p`, `describe`)
- [ ] Debug Readiness / Liveness / Startup Probes cấu hình sai
- [ ] Debug Service không nhận traffic (`get ep/endpointslices`, selector, `targetPort`)
- [ ] Debug CoreDNS & phân giải tên miền trong cluster
- [ ] Debug NetworkPolicy chặn nhầm traffic / chặn cổng 53 DNS
- [ ] Debug Worker Node `NotReady` (`systemctl`, `journalctl -u kubelet`, `containerd`, `kubelet.conf`, swap/disk)
- [ ] Debug Control Plane chết khi `kubectl` không chạy (`crictl ps -a`, `crictl logs`, `/var/log/pods`, `/etc/kubernetes/manifests/*.yaml`)
- [ ] Giám sát tài nguyên cụm & ứng dụng (`kubectl top nodes`, `kubectl top pods --sort-by=cpu/memory`)

### Domain 2: Cluster Architecture, Installation & Configuration (25%)
- [ ] Quản lý RBAC (`ServiceAccount`, `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`)
- [ ] Kiểm chứng quyền RBAC bằng `kubectl auth can-i --as=...`
- [ ] Chuẩn bị node & cài cụm bằng `kubeadm init` / `kubeadm join` + cài CNI
- [ ] Nâng cấp Control Plane bằng `kubeadm` (`drain` → `kubeadm upgrade plan/apply` → `kubelet`/`kubectl` → `uncordon`)
- [ ] Nâng cấp Worker Node bằng `kubeadm` (`drain` → `kubeadm upgrade node` → `kubelet`/`kubectl` → `uncordon`)
- [ ] Backup `etcd` bằng `etcdctl snapshot save` & kiểm tra bằng `snapshot status`
- [ ] Restore `etcd` bằng `etcdutl`/`etcdctl snapshot restore` + cập nhật `etcd.yaml` + verify
- [ ] **Helm:** `repo add/update`, `search`, `show values`, `template`, `install`, `upgrade`, `rollback`, `uninstall`
- [ ] **Kustomize:** `base` & `overlays`, `kustomization.yaml`, `patches`, `configMapGenerator`, `kubectl apply -k`
- [ ] **CRDs & Operators:** Liệt kê CRD, `kubectl explain`, kiểm tra Custom Resources, debug Operator

### Domain 3: Services & Networking (20%)
- [ ] Tạo & cấu hình `ClusterIP`, `NodePort`, `ExternalName`, Headless Service
- [ ] Kiểm tra & chỉnh sửa CoreDNS ConfigMap (`kube-system`)
- [ ] Viết NetworkPolicy: Default Deny Ingress / Egress
- [ ] Viết NetworkPolicy phân biệt chuẩn logic **AND** vs **OR** (`namespaceSelector` + `podSelector`)
- [ ] Cấu hình **Ingress** (`IngressClass`, path `Prefix`/`Exact`, TLS Secret)
- [ ] Cấu hình **Gateway API**: `GatewayClass`, `Gateway` (HTTP/HTTPS, listeners, TLS)
- [ ] Cấu hình **Gateway API**: `HTTPRoute` (`parentRefs`, `matches`, `backendRefs`, traffic splitting/weights)
- [ ] Chuyển đổi (migrate) cấu hình từ `Ingress` sang `Gateway` + `HTTPRoute`
- [ ] Cài đặt & kiểm tra CNI (Calico / Flannel) & Pod CIDR

### Domain 4: Workloads & Scheduling (15%)
- [ ] Deployment rollout, scale, `rollout history`, `rollout undo`, `RollingUpdate` vs `Recreate`
- [ ] ConfigMap & Secret (nạp qua `env`, `envFrom`, và mount thành `volume`)
- [ ] Resource `requests` & `limits`, `LimitRange`, `ResourceQuota`
- [ ] **HPA (`HorizontalPodAutoscaler`)** theo CPU / Memory với Metrics Server
- [ ] `nodeSelector`, `nodeAffinity`, `podAffinity`, `podAntiAffinity`
- [ ] `Taints` & `Tolerations` (`NoSchedule`, `PreferNoSchedule`, `NoExecute`)
- [ ] `PriorityClass` (ưu tiên lập lịch & preemption)
- [ ] Static Pods (`/etc/kubernetes/manifests`), DaemonSets, Jobs & CronJobs
- [ ] Multi-container Pods (InitContainers & Sidecar chia sẻ `emptyDir`)

### Domain 5: Storage (10%)
- [ ] Volumes cơ bản: `emptyDir`, `hostPath`
- [ ] Tạo PV & PVC thủ công, hiểu rõ quy tắc Bind (`storageClassName`, `accessModes`, `capacity`)
- [ ] `StorageClass` & Dynamic Provisioning (`reclaimPolicy`, `volumeBindingMode: WaitForFirstConsumer`)
- [ ] Đặt StorageClass mặc định (`is-default-class`)
- [ ] Mở rộng dung lượng PVC (`allowVolumeExpansion: true`)

---

## 2. Bảng Theo Dõi Điểm Mock Test & Killer.sh

| Ngày | Bài thi / Nguồn đề | Thời gian làm | Điểm (%) | Số câu Flag / Chưa kịp | Domain mất điểm nhiều nhất | Hành động khắc phục |
| :--- | :--- | :---: | :---: | :---: | :--- | :--- |
| YYYY-MM-DD | Mock Chẩn đoán (GĐ 0) | /120m | % | | | |
| YYYY-MM-DD | Mock Practice #1 | /120m | % | | | |
| YYYY-MM-DD | Mock Practice #2 | /120m | % | | | |
| YYYY-MM-DD | Mock Practice #3 | /120m | % | | | |
| YYYY-MM-DD | **Killer.sh — Phiên 1** | /120m | % | | | *(Mục tiêu: ≥ 55–65%)* |
| YYYY-MM-DD | **Killer.sh — Phiên 2** | /120m | % | | | *(Mục tiêu: ≥ 85–90%)* |

---

## 3. Checklist Chuẩn Bị Trước Ngày Thi

- [ ] Đạt ≥ 9/10 tiêu chí ở mục **Exam Readiness Gate** trong [CKA-ROADMAP.md](./CKA-ROADMAP.md)
- [ ] Đã đọc lại toàn bộ sổ [ERROR-LOG.md](./ERROR-LOG.md) và tự làm lại các câu từng sai
- [ ] Đã kiểm tra giấy tờ tùy thân (CCCD tiếng Anh / Hộ chiếu) khớp tên Linux Foundation
- [ ] Đã test hệ thống PSI Secure Browser thành công trước ngày thi ≥ 24h
- [ ] Đã dọn phòng & bàn thi đúng quy định giám thị

