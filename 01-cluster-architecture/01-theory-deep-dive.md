# Module 01: Cluster Architecture, Installation & Configuration (25%) — Hiểu Sâu Bản Chất

> **Tại sao module này cực kỳ quan trọng?** Nó chiếm **25% điểm trực tiếp** và là **nền tảng gốc để giải 30% điểm Troubleshooting**. Nếu bạn không hiểu cách `kube-apiserver`, `etcd`, `kubelet`, chứng chỉ PKI và `kubeadm` liên kết với nhau, bạn sẽ hoàn toàn mù phương hướng khi gặp câu sửa cụm bị sập.

---

## 1. Bản Chất Kiến Trúc Kubernetes & Vòng Lặp Điều Hòa (Reconciliation Loop)

Kubernetes không phải là một hệ thống chạy lệnh mệnh lệnh (imperative) kiểu "hãy khởi động container này ngay". Bản chất của nó là **hệ thống trạng thái khai báo (Declarative State Machine)** xoay quanh **1 nguồn chân lý duy nhất là `etcd`**.

```text
                       ┌─────────────────────────────────────────────┐
                       │               CONTROL PLANE                 │
  kubectl / Helm ─────►│  kube-apiserver  ◄─────────►  etcd (DB)     │
  (HTTPS :6443)        │    ▲      ▲     ▲                           │
                       │    │      │     └────► kube-scheduler       │
                       │    │      └──────────► kube-controller-mgr  │
                       └────┼────────────────────────────────────────┘
                            │ (Watch & Report Status)
       ┌────────────────────┴────────────────────┐
       ▼                                         ▼
┌─────────────────────┐                   ┌─────────────────────┐
│    WORKER NODE 1    │                   │    WORKER NODE 2    │
│  kubelet ◄──► CRI   │                   │  kubelet ◄──► CRI   │
│  kube-proxy  (CNI)  │                   │  kube-proxy  (CNI)  │
└─────────────────────┘                   └─────────────────────┘
```

### Luồng đi thực tế của một lệnh `kubectl apply -f deploy.yaml`
1. **`kube-apiserver` tiếp nhận request** và cho đi qua 4 cửa gác theo đúng thứ tự:
   - **Authentication** (Bạn là ai? — X.509 cert / ServiceAccount token).
   - **Authorization** (Bạn có quyền không? — **RBAC**).
   - **Admission Controllers** (Mutating → Validating, ví dụ gán default StorageClass, kiểm tra ResourceQuota/LimitRange).
   - **Persist vào `etcd`**: Chỉ duy nhất `kube-apiserver` nói chuyện trực tiếp với `etcd`!
2. **`kube-controller-manager`** (chứa Deployment controller, ReplicaSet controller, Node controller...) đang *watch* API server thấy có Deployment mới → tạo ReplicaSet → tạo các đối tượng Pod (chưa có `spec.nodeName`).
3. **`kube-scheduler`** thấy Pod chưa có `nodeName` → chạy thuật toán lọc (Filtering) & chấm điểm (Scoring) → ghi `spec.nodeName = worker-1` ngược lại vào `kube-apiserver`.
4. **`kubelet` trên `worker-1`** thấy có Pod được gán tên node của mình → gọi **CRI (`containerd`)** kéo image & tạo container, gọi **CNI** cấp IP cho Pod, gọi **CSI** gắn volume → cập nhật `Pod.status` lên `kube-apiserver`.

> **Tư duy Kubestronaut:** Trên cụm cài bằng `kubeadm`, chính `kubelet` trên Control Plane chạy `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` dưới dạng **Static Pods** từ `/etc/kubernetes/manifests/*.yaml`. Vì vậy, khi bạn sửa 1 file trong `/etc/kubernetes/manifests/`, **`kubelet` tự động phát hiện và khởi động lại Static Pod đó** mà không cần (và không thể) dùng `kubectl apply`!

---

## 2. Hệ Thống Chứng Chỉ (PKI) & Kubeconfig

Mọi thành phần trong Kubernetes đều xác thực nhau qua **mTLS** bằng các chứng chỉ nằm ở `/etc/kubernetes/pki/`:
- **CA gốc của cluster:** `/etc/kubernetes/pki/ca.crt` & `ca.key`.
- **CA riêng cho `etcd`:** `/etc/kubernetes/pki/etcd/ca.crt` & `ca.key` (cần cho `etcdctl`!).
- **Các file kubeconfig quan trọng:**
  - `/etc/kubernetes/admin.conf` — Quyền `cluster-admin` (file được copy sang `~/.kube/config`).
  - `/etc/kubernetes/kubelet.conf` — Identity của `kubelet` khi nói chuyện với `apiserver` (rất hay bị phá trong đề Troubleshooting!).
- **Kiểm tra hạn chứng chỉ (hay gặp khi debug):**
  ```bash
  kubeadm certs check-expiration
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -E "Subject:|Not After|DNS:|IP Address:"
  ```

---

## 3. RBAC (Role-Based Access Control) — Bản Chất & Bẫy Thi

RBAC trả lời câu hỏi: **Ai (`Subjects`) + Được làm gì (`Verbs`) + Trên tài nguyên nào (`Resources` & `apiGroups`) + Trong phạm vi nào (`Namespace` hay toàn Cluster)?**

| Đối tượng | Phạm vi áp dụng | Dùng khi nào? |
| :--- | :--- | :--- |
| `Role` + `RoleBinding` | Chỉ 1 Namespace | Cấp quyền cho user/SA thao tác trên Pods/Deployments/Secrets trong 1 namespace cụ thể |
| `ClusterRole` + `ClusterRoleBinding` | Toàn bộ Cluster | Cấp quyền cho tài nguyên cluster-scoped (`nodes`, `pv`, `namespaces`) hoặc đọc Pods ở **tất cả** namespaces |
| **`ClusterRole` + `RoleBinding`** | **Chỉ 1 Namespace** | **Mẹo kinh điển:** Tái sử dụng 1 bộ quyền mẫu (`ClusterRole`) nhưng giới hạn hiệu lực trong đúng 1 namespace bằng `RoleBinding`! |

### Bẫy hay gặp trong đề thi RBAC:
- **Sai `apiGroups`:** Core resources (`pods`, `services`, `secrets`, `configmaps`, `nodes`, `pvc`, `pv`) thuộc `apiGroups: [""]`. Còn `deployments`, `daemonsets`, `replicasets`, `statefulsets` thuộc `apiGroups: ["apps"]`; `jobs`, `cronjobs` thuộc `batch`; `ingresses`, `networkpolicies` thuộc `networking.k8s.io`. Quên? Chạy `kubectl api-resources`!
- **Quên gắn `serviceAccountName` vào Pod:** Đề yêu cầu Pod dùng ServiceAccount vừa tạo — phải thêm `spec.serviceAccountName: <sa>` vào Pod/Deployment!
- **Luôn kiểm chứng bằng `kubectl auth can-i`:**
  ```bash
  kubectl auth can-i create deployments --as=system:serviceaccount:dev:ci-sa -n dev
  ```

---

## 4. Bản Chất `kubeadm` Upgrade & `etcd` Backup/Restore

### A. Tại sao `kubeadm upgrade` phải theo đúng thứ tự?
Kubernetes có chính sách **Version Skew Policy**:
- `kube-apiserver` phải luôn có phiên bản **lớn hơn hoặc bằng** `kube-controller-manager`, `kube-scheduler`, và **`kubelet`** (chênh lệch tối đa 2–3 minor version, và **không bao giờ được để `kubelet` cao hơn `apiserver`**!).
- Do đó, thứ tự bắt buộc luôn là: **Control Plane trước (nâng `kubeadm` → `kubeadm upgrade apply` để nâng các Static Pods control plane → nâng `kubelet`/`kubectl`) → sau đó mới tới từng Worker Node (`kubeadm` → `kubeadm upgrade node` → `kubelet`/`kubectl`)**.
- Chỉ nâng **từng minor version một** (ví dụ `1.30 → 1.31`, không nhảy cóc `1.30 → 1.32`).

### B. Bản chất `etcd` Backup & Restore
- `etcd` là kho dữ liệu key-value phân tán theo giao thức đồng thuận **Raft** (yêu cầu số lượng node lẻ 1, 3, 5 để giữ quorum `N/2 + 1`).
- **Backup** (`etcdctl snapshot save`) kết nối qua cổng client HTTPS `2379` của `etcd` đang chạy → bắt buộc cần 4 cờ: `--endpoints`, `--cacert`, `--cert`, `--key` (luôn đọc trực tiếp từ `/etc/kubernetes/manifests/etcd.yaml` chứ không đoán!).
- **Restore** (`etcdutl snapshot restore` hoặc `etcdctl snapshot restore`) là thao tác **offline trên đĩa cứng** — nó giải nén snapshot ra một thư mục `--data-dir` **mới** (ví dụ `/var/lib/etcd-restore`) với metadata cụm sạch.
- **Bước quyết định mà 50% thí sinh quên:** Sau khi restore xong ra thư mục mới, `etcd` vẫn đang đọc thư mục cũ `/var/lib/etcd`! Bạn **phải sửa `hostPath.path`** của volume `etcd-data` trong `/etc/kubernetes/manifests/etcd.yaml` trỏ sang `/var/lib/etcd-restore`, chờ `kubelet` restart static pod `etcd` và `kube-apiserver` đồng bộ lại.

---

## 5. Hệ Sinh Thái Mới Trong Đề Thi CKA 2025/2026: Helm, Kustomize & CRDs

### A. Helm (Trình quản lý gói cho Kubernetes)
- **Chart** = gói template YAML (giống gói `.deb`/`.rpm`).
- **Values** (`values.yaml`) = các biến cấu hình để tùy biến Chart mà không sửa trực tiếp template.
- **Release** = 1 phiên bản cài đặt cụ thể của Chart vào 1 namespace (lưu lịch sử revision dưới dạng Secret `sh.helm.release.v1...` trong namespace đó).
- **Kỹ năng thi bắt buộc:** Thêm/cập nhật repo, tìm phiên bản chart, xem `values`, render thử bằng `helm template` (rất hay gặp câu *"render chart ra file YAML mà không cài"* hoặc *"cài chart vào namespace X với giá trị Y"*), `upgrade`, `rollback`, `uninstall`.

### B. Kustomize (Tùy biến YAML thuần khai báo, tích hợp sẵn trong `kubectl`)
- Khác với Helm (dùng template Go `{{ .Values... }}`), Kustomize dùng **YAML thuần hợp lệ** với cấu trúc:
  - `base/`: chứa tài nguyên gốc dùng chung + `kustomization.yaml`.
  - `overlays/dev/` và `overlays/prod/`: kế thừa `base/` và ghi đè khác biệt (namespace, prefix, replicas, image tag, patches, configMapGenerator).
- **Lệnh thi bắt buộc:** `kubectl kustomize <dir>` (chỉ in YAML kết quả ra stdout — dùng kiểm tra hoặc redirect ra file) và `kubectl apply -k <dir>` (áp dụng trực tiếp vào cluster).

### C. CRDs (Custom Resource Definitions) & Operators
- **CRD** mở rộng API của Kubernetes bằng cách đăng ký thêm `kind` mới (ví dụ `Certificate`, `Gateway`, `HTTPRoute`, `Application`, `Prometheus`).
- **Custom Resource (CR)** là một đối tượng cụ thể tạo từ CRD đó.
- **Operator** = **CRD + Custom Controller** (một Pod chạy vòng lặp điều hòa riêng để quản lý vòng đời ứng dụng phức tạp như Database, Cert-Manager, Istio).
- **Trong đề thi:** Bạn thường được yêu cầu kiểm tra các CRD đang cài (`k get crd`), xem tài liệu trường của một CRD (`k explain <crd>`), liệt kê/lọc các Custom Resource, hoặc debug vì sao Operator không tạo tài nguyên như mong muốn.

---

**Đã hiểu chắc bản chất Module 01?** Chuyển sang thực hành ngay tại [02-hands-on-labs.md](./02-hands-on-labs.md).

