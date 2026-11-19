# Lộ Trình Ôn Luyện & Thi CKA Chuẩn Chỉnh (Bản Cập Nhật Mới Nhất)

Tài liệu này là bản lộ trình tổng thể cho việc ôn luyện và thi chứng chỉ **Certified Kubernetes Administrator (CKA)** theo chương trình cập nhật mới nhất (2025/2026), thiết kế chuyên biệt cho người **đang trong giai đoạn luyện đề thi** để vừa chắc bản chất kiến thức, vừa đạt tốc độ cao trong phòng thi.

> **Các tài liệu đi kèm trong thư mục này:**
> - [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) — Bảng tick tiến độ chi tiết từng đầu mục kiến thức & theo dõi điểm Mock Test / Killer.sh.
> - [CHEATSHEET.md](./CHEATSHEET.md) — Sổ tay lệnh thực chiến, mẫu YAML cốt lõi và luồng lệnh Troubleshooting 4 tầng.
> - [ERROR-LOG.md](./ERROR-LOG.md) — Mẫu nhật ký lỗi sai bắt buộc phải ghi chép khi luyện đề để tránh học vẹt.

---

## 1. Tổng Quan Kỳ Thi & Cấu Trúc Điểm CKA

| Thông số | Chi tiết chính thức |
| :--- | :--- |
| **Hình thức** | 100% thực hành trên terminal (Performance-based), ~15–20 câu trên nhiều cụm Kubernetes |
| **Thời lượng** | **120 phút** (2 giờ) |
| **Điểm đạt (Passing Score)** | **66%** (Mục tiêu luyện thi an toàn: **80%+**) |
| **Chính sách thi lại** | **1 lần thi lại miễn phí** (1 free retake) trong thời hạn hiệu lực đăng ký |
| **Hiệu lực chứng chỉ** | **2 năm** |
| **Tài liệu được phép mở** | `kubernetes.io/docs`, `kubernetes.io/blog`, `helm.sh/docs`, `gateway.networking.k8s.io`, `github.com/kubernetes` (mở trong Firefox của máy ảo PSI, **không** dùng Google/bookmark cá nhân) |

### Phân Bổ Trọng Số 5 Domain

| Domain | Trọng số | Trọng tâm bắt buộc (gồm các phần cập nhật mới) |
| :--- | :---: | :--- |
| **1. Troubleshooting** | **30%** | Sửa lỗi Control Plane, Worker Node (`kubelet`/`containerd`), Networking, Service/DNS, ứng dụng/Pod |
| **2. Cluster Architecture, Installation & Configuration** | **25%** | `kubeadm` (init/join/upgrade), `etcd` backup/restore, RBAC, **Helm**, **Kustomize**, **CRDs & Operators** |
| **3. Services & Networking** | **20%** | Services, CoreDNS, NetworkPolicy, Ingress, **Gateway API** (`GatewayClass`, `Gateway`, `HTTPRoute`), CNI |
| **4. Workloads & Scheduling** | **15%** | Deployment rollout/rollback, ConfigMap/Secret, Resource Limits, **HPA**, Affinity, Taints/Tolerations, PriorityClass |
| **5. Storage** | **10%** | StorageClass, Dynamic Provisioning, PV/PVC, Access Modes, Reclaim Policy, Volume Expansion |

**Lưu ý quan trọng:** Đề thi CKA mới đánh mạnh vào **Gateway API**, **Helm**, **Kustomize**, **CRDs**, **HPA/Autoscaling** và **Troubleshooting đa tầng**. Nếu bạn đang luyện theo bộ đề cũ trước 2025, bắt buộc phải bổ sung đủ các mảng này.

---

## 2. Tổng Quan Lộ Trình 6 Tuần

```text
[Giai đoạn 0: Ngày 1-2]   Chẩn đoán năng lực & Setup Lab kubeadm + Speed Toolkit
          │
          ▼
[Giai đoạn 1: Tuần 1-2]   Workloads (15%) + Storage (10%) + Cluster Arch (25%) = 50% điểm
          │
          ▼
[Giai đoạn 2: Tuần 3]     Services & Networking (20%) + Troubleshooting (30%)  = 50% điểm
          │
          ▼
[Giai đoạn 3: Tuần 4-5]   Luyện đề 3 vòng + Error Log + Killer.sh (Phiên 1 & Phiên 2)
          │
          ▼
[Giai đoạn 4: Tuần 6]     Tổng ôn lỗ hổng + Chuẩn bị môi trường PSI + Ngày thi
```

*(Có thể co gọn thành 4 tuần hoặc giãn ra 8 tuần tùy quỹ thời gian mỗi ngày, nhưng giữ nguyên thứ tự các giai đoạn.)*

---

## 3. Chi Tiết Từng Giai Đoạn Ôn Luyện

### Giai đoạn 0 (Ngày 1–2): Chẩn Đoán Năng Lực & Xây "Vũ Khí Tốc Độ"

1. **Làm 1 bài Mock Test chẩn đoán (không mở Google):** Chấm điểm và ghi rõ bạn đang yếu nhất ở domain nào vào [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md).
2. **Thiết lập môi trường Lab đúng chuẩn:**
   - Bắt buộc có ít nhất 1 cụm **`kubeadm` thật** (1 control-plane + 1–2 worker nodes trên VM/Multipass/Vagrant hoặc Killercoda) để thực hành upgrade, etcd backup/restore và tự phá/sửa control plane, worker node.
   - `minikube` / `kind` chỉ dùng tốt cho Workloads, Storage, RBAC, Helm, Kustomize, Gateway API; **không thay thế được `kubeadm`** cho các bài quản trị hệ thống.
3. **Thuần thục bộ thao tác tăng tốc Terminal (xem chi tiết trong [CHEATSHEET.md](./CHEATSHEET.md)):**
   - Thiết lập `alias k=kubectl`, bash autocomplete, `export do="--dry-run=client -o yaml"`, `export now="--force --grace-period=0"`.
   - Cấu hình `~/.vimrc` chuẩn YAML (`set ts=2 sw=2 et`).
   - Thành thạo sinh YAML bằng imperative commands + chỉnh sửa nhanh trong `vim`.
   - Thành thạo `kubectl explain <resource>.spec --recursive` và lọc dữ liệu với `--sort-by`, `jsonpath`, `custom-columns`.

---

### Giai đoạn 1 (Tuần 1–2): Làm Chủ 50% Điểm Nền Tảng

#### A. Workloads & Scheduling (15%) — Ngày 1 đến 3 (Tuần 1)
- **Kiến thức bản chất cần nắm chắc:**
  - Deployment: `rollout status`, `rollout history`, `rollout undo`, `RollingUpdate` (`maxSurge`, `maxUnavailable`) vs `Recreate`.
  - ConfigMap & Secret: nạp qua `env`, `envFrom` và mount `volume` (hiểu rõ volume tự cập nhật sau chu kỳ sync, còn biến môi trường `env` không tự cập nhật khi không restart Pod).
  - Resource Management & Scaling: `requests` vs `limits`, **LimitRange**, **ResourceQuota**, **HPA (`HorizontalPodAutoscaler`)** (hiểu cách hoạt động với Metrics Server, target CPU/Memory, `minReplicas`/`maxReplicas`).
  - Scheduling nâng cao: `nodeSelector`, `nodeAffinity` (`requiredDuringScheduling...` vs `preferredDuringScheduling...`), `podAffinity`/`podAntiAffinity`, **Taints & Tolerations** (`NoSchedule`, `PreferNoSchedule`, `NoExecute`), **PriorityClass** (`value`, `preemptionPolicy`).
  - Kiến trúc Pod đặc biệt: **Static Pods** (`/etc/kubernetes/manifests`, kiểm tra `staticPodPath` trong `/var/lib/kubelet/config.yaml`), **DaemonSets**, **InitContainers** và **Sidecar Containers** (mẫu chia sẻ `emptyDir` đọc log).
- **Bài tập tự kiểm chứng (tự làm không nhìn đáp án):**
  1. Tạo Deployment có `requests`/`limits`, cấu hình HPA scale 1 → 4 replicas theo CPU 50%.
  2. Tạo Pod chỉ được schedule lên node có taint `env=prod:NoSchedule` và label `disk=ssd`.
  3. Tạo Static Pod trực tiếp trên worker node và xác nhận mirror pod xuất hiện trên control plane.
  4. Tạo Pod 2 container (app + sidecar busybox) chia sẻ volume `emptyDir`, sidecar đọc file log do app ghi ra.

#### B. Storage (10%) — Ngày 4 đến 5 (Tuần 1)
- **Kiến thức bản chất cần nắm chắc:**
  - Phân biệt rõ vòng đời `emptyDir`, `hostPath`, `PersistentVolume (PV)`, `PersistentVolumeClaim (PVC)`, `StorageClass (SC)`.
  - Điều kiện Bind giữa PV & PVC: khớp `storageClassName`, khớp `accessModes` (`RWO`, `ROX`, `RWX`), `PV capacity >= PVC request`.
  - Dynamic Provisioning: `provisioner`, `reclaimPolicy` (`Delete` vs `Retain`), `volumeBindingMode` (`Immediate` vs `WaitForFirstConsumer` — lý do PVC giữ trạng thái `Pending` đến khi có Pod sử dụng).
  - Volume Expansion: `allowVolumeExpansion: true` trên StorageClass và tăng `spec.resources.requests.storage` trên PVC.
- **Bài tập tự kiểm chứng:**
  1. Đổi default StorageClass bằng annotation `storageclass.kubernetes.io/is-default-class: "true"`.
  2. Tạo PV + PVC bind thủ công, mount vào Pod, ghi dữ liệu test, xóa Pod tạo lại xác nhận dữ liệu còn nguyên.
  3. Tạo PVC động với `WaitForFirstConsumer` và quan sát trạng thái chuyển từ `Pending` → `Bound` khi Pod được schedule.

#### C. Cluster Architecture, Installation & Configuration (25%) — Tuần 2 (7 ngày)
1. **RBAC (Ngày 1):**
   - Phân biệt `Role` vs `ClusterRole`, `RoleBinding` vs `ClusterRoleBinding`.
   - Hiểu trường hợp kết hợp `ClusterRole` + `RoleBinding` (dùng bộ quyền dùng chung nhưng giới hạn trong 1 namespace).
   - Luôn verify bằng `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa> -n <ns>`.
2. **`kubeadm` Init, Join & Upgrade (Ngày 2–3):**
   - Chuẩn bị OS trên node: tắt swap (`swapoff -a`), nạp kernel modules (`overlay`, `br_netfilter`), bật `sysctl` (`net.ipv4.ip_forward = 1`, `net.bridge.bridge-nf-call-iptables = 1`), cấu hình `containerd` (`SystemdCgroup = true`).
   - Thuần thục quy trình nâng cấp Cluster từ minor `v1.X` lên `v1.(X+1)`:
     - Control Plane đầu tiên: `drain` → đổi apt repo sang `v1.(X+1)` → upgrade `kubeadm` → `kubeadm upgrade plan` → `kubeadm upgrade apply v1.(X+1).y` → upgrade `kubelet` & `kubectl` → `systemctl daemon-reload && systemctl restart kubelet` → `uncordon`.
     - Worker Node / Control Plane phụ: `drain` → đổi apt repo → upgrade `kubeadm` → `kubeadm upgrade node` → upgrade `kubelet` & `kubectl` → `daemon-reload && restart kubelet` → `uncordon`.
3. **Backup & Restore `etcd` (Ngày 4):**
   - Lấy đúng 4 tham số từ `/etc/kubernetes/manifests/etcd.yaml`: `--listen-client-urls`, `--trusted-ca-file`, `--cert-file`, `--key-file`.
   - Chạy backup bằng `ETCDCTL_API=3 etcdctl snapshot save` và kiểm tra bằng `snapshot status`.
   - Chạy restore bằng `etcdutl snapshot restore` (hoặc `ETCDCTL_API=3 etcdctl snapshot restore`) sang `--data-dir` mới → cập nhật `hostPath.path` trong `/etc/kubernetes/manifests/etcd.yaml` → chờ `etcd` và `kube-apiserver` khởi động lại → kiểm chứng tài nguyên đã phục hồi.
4. **Helm, Kustomize & CRDs/Operators (Ngày 5–7):**
   - **Helm:** `repo add/update`, `search repo`, `show values`, `template`, `install`, `upgrade`, `list -A`, `history`, `rollback`, `uninstall`.
   - **Kustomize:** Tổ chức `base/` & `overlays/`, viết `kustomization.yaml` (`resources`, `namespace`, `namePrefix`, `commonLabels`, `images`, `patches`, `configMapGenerator`), preview bằng `kubectl kustomize` và áp dụng bằng `kubectl apply -k`.
   - **CRDs & Operators:** Liệt kê CRD (`kubectl get crd`), kiểm tra schema bằng `kubectl explain`, đọc/tạo Custom Resource và debug Operator pod.

---

### Giai đoạn 2 (Tuần 3): Services & Networking (20%) + Troubleshooting (30%)

Tuần này chiếm **50% tổng số điểm** — ưu tiên thực hành phá và sửa thực tế.

#### A. Services & Networking (20%) — Ngày 1 đến 3
1. **Services & Endpoints:**
   - `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`, Headless (`clusterIP: None`).
   - Kiểm tra `kubectl get ep,endpointslices` ngay khi Service không nhận traffic; đối chiếu `selector` ↔ `Pod labels` và `targetPort` ↔ `containerPort`.
2. **CoreDNS:**
   - Nắm chắc định dạng FQDN: `<svc>.<ns>.svc.cluster.local` và Pod DNS `<ip-dashed>.<ns>.pod.cluster.local`.
   - Kiểm tra ConfigMap `coredns` trong namespace `kube-system`.
   - Test DNS nhanh bằng pod tạm (`busybox:1.36` / `curlimages/curl`).
3. **NetworkPolicy (Cẩn thận bẫy AND vs OR):**
   - Viết thành thạo Default Deny Ingress / Default Deny Egress.
   - Ghi nhớ quy tắc cú pháp:
     - **AND:** `namespaceSelector` và `podSelector` nằm chung dưới **1 gạch đầu dòng `-`**.
     - **OR:** `namespaceSelector` và `podSelector` nằm ở **2 gạch đầu dòng `-` riêng biệt**.
   - Khi áp dụng Egress NetworkPolicy, nhớ cho phép UDP/TCP port `53` tới CoreDNS nếu Pod cần resolve tên miền.
4. **Ingress & Gateway API (Trọng tâm bắt buộc):**
   - **Ingress:** `ingressClassName`, `host`, `pathType` (`Prefix`, `Exact`), `tls`.
   - **Gateway API:** Nắm chắc 3 đối tượng cốt lõi:
     - `GatewayClass`
     - `Gateway` (`gatewayClassName`, `listeners`, `protocol`, `port`, `hostname`, `tls.certificateRefs`, `allowedRoutes`)
     - `HTTPRoute` (`parentRefs`, `hostnames`, `rules`, `matches`, `filters`, `backendRefs`, `weight` để chia tỷ lệ traffic)
   - **Bài tập bắt buộc:** Thực hành **migrate một Ingress (có TLS)** sang `Gateway` + `HTTPRoute` chỉ sử dụng tài liệu tại `gateway.networking.k8s.io`.
5. **CNI Plugin:**
   - Biết cài đặt CNI (Calico/Flannel) sau `kubeadm init`, kiểm tra Pod CIDR không lệch với cấu hình CNI.

#### B. Troubleshooting 4 Tầng (30% — Quan Trọng Nhất) — Ngày 4 đến 7

Luôn đi theo quy trình chẩn đoán có hệ thống 4 tầng (chi tiết lệnh xem trong [CHEATSHEET.md](./CHEATSHEET.md)):

| Tầng sự cố | Triệu chứng nhận diện | Hướng điều tra cốt lõi |
| :--- | :--- | :--- |
| **Tầng 1: Pod / App** | `Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, `CreateContainerConfigError`, `0/1 Ready` | `describe pod` → `logs -p` → `get events --sort-by='.lastTimestamp'` → kiểm tra `requests/limits`, `pvc`, `taints`, `probes`, `cm/secret`, `command/args` |
| **Tầng 2: Service / Net** | Lỗi kết nối tới Service, Ingress, Gateway hoặc DNS | `get ep,endpointslices` → test `Pod IP` → test `Service IP` → test `DNS` → soi `NetworkPolicy` → soi logs `coredns` & `kube-proxy` |
| **Tầng 3: Worker Node** | Node `NotReady`, Pod treo `ContainerCreating` hoặc `Terminating` | `ssh <node>` → `systemctl status kubelet containerd` → `journalctl -u kubelet -xeu` → kiểm tra `/var/lib/kubelet/config.yaml`, `/etc/kubernetes/kubelet.conf`, ổ đĩa (`df -h`), swap (`free -m`) |
| **Tầng 4: Control Plane** | `kubectl` báo `connection refused` hoặc timeout | `ssh <control-plane>` → `crictl ps -a` & `crictl logs` → `/var/log/pods/` → soát `/etc/kubernetes/manifests/*.yaml` (sai cờ, sai path cert, sai port `etcd`/API, lỗi cú pháp YAML) |

**8 kịch bản tự phá & tự sửa bắt buộc trên cụm `kubeadm`:**
1. Sửa sai đường dẫn certificate hoặc sai cờ `--etcd-servers` trong `/etc/kubernetes/manifests/kube-apiserver.yaml`.
2. Sửa sai image hoặc cú pháp YAML trong `kube-scheduler.yaml` / `kube-controller-manager.yaml`.
3. Dừng dịch vụ `kubelet` hoặc sửa sai cổng API trong `/etc/kubernetes/kubelet.conf` trên Worker Node.
4. Xóa hoặc cấu hình lệch Pod CIDR của CNI khiến CoreDNS không chạy / Pod không ping nhau được.
5. Tạo Service lệch selector label hoặc sai `targetPort`.
6. Áp NetworkPolicy chặn nhầm traffic hoặc quên mở cổng 53 DNS ở Egress.
7. Tạo PVC `Pending` do lệch `storageClassName` hoặc `accessModes`.
8. Tạo Pod bị `OOMKilled` hoặc `Pending` do thiếu CPU / node bị taint.

---

### Giai đoạn 3 (Tuần 4–5): Luyện Đề 3 Vòng & Chinh Phục Killer.sh

Áp dụng kỷ luật **3 vòng** để đảm bảo luyện đề xong là nắm chắc bản chất, không học thuộc lòng:

#### Vòng 1 (Tuần 4 — Ngày 1 đến 5): Luyện Đề Theo Chủ Đề + Nhật Ký Lỗi (Error Log)
- Mỗi câu làm sai, làm quá 8 phút, hoặc phải xem gợi ý → **ghi ngay vào [ERROR-LOG.md](./ERROR-LOG.md)**.
- Không chuyển bài vội: đóng đáp án, xóa tài nguyên vừa tạo, và **tự làm lại sau 24 giờ** đến khi hoàn thành gọn trong < 5 phút.
- **Kỷ luật Docs:** Tuyệt đối **tắt Google / ChatGPT** khi làm bài. Chỉ dùng thanh tìm kiếm nội bộ của `kubernetes.io/docs`, `gateway.networking.k8s.io`, `helm.sh/docs`. Mục tiêu: tìm ra đúng mẫu YAML trong **< 15 giây**.

#### Vòng 2 (Cuối Tuần 4): Killer.sh — Phiên 1
- Khi đăng ký thi CKA, bạn có **2 phiên giả lập Killer.sh miễn phí** (mỗi phiên mở 36 giờ, chung bộ câu hỏi, độ khó cao hơn đề thi thật ~20–30%).
- **Cách chạy Phiên 1:**
  1. Bấm đúng **120 phút** làm nghiêm túc như thi thật (đạt **55%–65%** lần đầu đã là tín hiệu rất tốt).
  2. Ghi điểm và phân tích chi tiết vào [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) và [ERROR-LOG.md](./ERROR-LOG.md).
  3. Dùng 34 tiếng còn lại của phiên để đọc kỹ lời giải chính thức, hiểu bản chất từng câu, học cách tối ưu tốc độ và tự gõ lại toàn bộ các câu sai/chậm.

#### Vòng 3 (Tuần 5): Vá Lỗ Hổng & Killer.sh — Phiên 2
- **Ngày 1–4:** Ôn tập trung đúng những phần bị mất điểm ở Phiên 1 + làm lại toàn bộ sổ Error Log + luyện thêm đề cập nhật 2025/2026 (Gateway API, Helm, Kustomize, HPA, Troubleshooting Control Plane).
- **Ngày 5–6 (cách ngày thi 4–5 ngày):** Mở **Killer.sh Phiên 2**, bấm đúng **120 phút** làm lại toàn bộ từ đầu.
  - **Mục tiêu bắt buộc:** Đạt **≥ 85%–90%** và còn dư tối thiểu **15–20 phút** soát bài.

---

### Giai đoạn 4 (Tuần 6): Chuẩn Bị Môi Trường PSI & Chiến Thuật Phòng Thi

#### Checklist 48 giờ trước giờ thi
- [ ] Giấy tờ tùy thân (CCCD bản có tiếng Anh hoặc Hộ chiếu) — họ tên **khớp chính xác từng chữ** với hồ sơ Linux Foundation.
- [ ] Bàn làm việc trống hoàn toàn (chỉ máy tính, 1 màn hình, cốc nước trong suốt không nhãn; gỡ giấy note/bảng viết xung quanh; phòng kín, yên tĩnh, đủ ánh sáng).
- [ ] Cài đặt và chạy System Check cho **PSI Secure Browser** trước ít nhất 24h; tắt toàn bộ phần mềm chạy nền (VPN, Docker Desktop, Slack, Discord, ứng dụng quay màn hình).
- [ ] 1 ngày trước thi: **không học nhồi kiến thức mới**, chỉ đọc lại [ERROR-LOG.md](./ERROR-LOG.md), [CHEATSHEET.md](./CHEATSHEET.md) và ngủ đủ 7–8 tiếng.

#### Chiến thuật sống còn trong 120 phút phòng thi
1. **Quy tắc #1 — Context & SSH:** Đầu mỗi câu hỏi **luôn chạy lệnh `kubectl config use-context ...`** đề cho. Nếu câu hỏi yêu cầu `ssh <node>`, làm xong **nhớ `exit`** để quay lại máy chủ thi trước khi qua câu mới.
2. **Chiến thuật 2 Lượt (Two-Pass Strategy):**
   - **Lượt 1 (85–90 phút đầu):** Đi lần lượt từng câu. Câu rõ hướng giải → làm dứt điểm & verify trong 3–6 phút. Câu quá nặng hoặc quá **7 phút chưa ra hướng** → **Flag (đánh dấu) và bỏ qua ngay**! Không để 1 câu khó đốt mất 25 phút của 4 câu dễ.
   - **Lượt 2 (30–35 phút cuối):** Quay lại xử lý các câu đã Flag (ưu tiên câu trọng số % cao trước) với tâm lý vững vì đã cầm chắc ≥ 65–70% điểm từ Lượt 1.
3. **Verify thực tế ngay sau mỗi câu:**
   - Pod/Deployment → chờ `Running` & `Ready` (không dừng ở `created`).
   - Service/Ingress/Gateway/NetworkPolicy → test bằng `curl` hoặc `nslookup`.
   - File kết quả (ví dụ `/opt/result.txt`) → `cat` kiểm tra đúng đường dẫn, đúng định dạng đề yêu cầu.
   - Sửa file hệ thống (`/etc/kubernetes/manifests/...`) → luôn sao lưu trước (`cp file.yaml /tmp/file.yaml.bak`).
4. **Thao tác PSI Remote Desktop:** Copy/Paste trong terminal Linux dùng `Ctrl+Shift+C` / `Ctrl+Shift+V` (hoặc chuột phải). Khi dán YAML vào `vim`, gõ `:set paste` trước hoặc dùng `cat <<'EOF' > file.yaml` để không bị vỡ thụt dòng.

---

## 4. Tiêu Chí "Đủ Chín Để Đặt Lịch Thi" (Exam Readiness Gate)

Chỉ chốt ngày thi khi bạn đạt tối thiểu **9/10 tiêu chí** sau:

- [ ] **Killer.sh Phiên 2 đạt ≥ 85%** trong đúng 120 phút.
- [ ] **Cluster Upgrade (`kubeadm`)** trọn vẹn (Control Plane + Worker) dưới 12 phút không vấp.
- [ ] **`etcd` Backup & Restore** (kèm sửa `etcd.yaml` và verify) dưới 8 phút.
- [ ] **RBAC** (`SA` + `Role/ClusterRole` + `Binding` + verify `auth can-i`) dưới 4 phút.
- [ ] Thành thạo **Helm** (`install/upgrade/template/rollback`) và **Kustomize** (`base/overlays`, `apply -k`).
- [ ] Viết & verify được **Gateway API** (`Gateway` + `HTTPRoute`) và migrate từ `Ingress` sang `Gateway API`.
- [ ] Viết chuẩn **NetworkPolicy** (nắm chắc AND/OR và cổng 53 DNS).
- [ ] Tự sửa được **Control Plane sập** (khi `kubectl` chết) bằng `crictl` + `/etc/kubernetes/manifests` + `/var/log/pods`.
- [ ] Tự sửa được **Worker Node `NotReady`** bằng `systemctl` + `journalctl -u kubelet` + `kubelet.conf`.
- [ ] Tra đúng mẫu YAML trên `kubernetes.io/docs`, `helm.sh/docs`, `gateway.networking.k8s.io` trong **< 15 giây**.

