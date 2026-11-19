# Module 04: Services & Networking (20%) — Hiểu Sâu Bản Chất

> **Mục tiêu bản chất:** Hiểu chính xác một gói tin đi từ Pod A → DNS → Service → `kube-proxy` → Pod B (hoặc từ bên ngoài → Ingress / **Gateway API** → Service → Pod) như thế nào. Hiểu rõ đường đi gói tin giúp bạn làm chủ cả 20% Networking lẫn phần khó nhất của 30% Troubleshooting!

---

## 1. Mô Hình Mạng Kubernetes & Vai Trò CNI

Kubernetes áp đặt **3 nguyên tắc vàng về mạng** (và giao cho **CNI plugin** như Calico, Flannel, Cilium hiện thực):
1. **Mọi Pod có một địa chỉ IP duy nhất** trong toàn cụm (tất cả container bên trong cùng 1 Pod dùng chung network namespace → gọi nhau qua `localhost`).
2. **Mọi Pod trên bất kỳ Node nào đều giao tiếp trực tiếp được với Pod trên Node khác** mà không cần NAT.
3. **Agent trên Node (`kubelet`) có thể gọi tới mọi Pod trên Node đó.**

> **Ghi nhớ:** Nếu cụm mới khởi tạo bằng `kubeadm init` nhưng **chưa cài CNI**, Node sẽ ở trạng thái `NotReady` và Pod `coredns` sẽ bị kẹt ở `Pending` (vì chưa có mạng cho Pod).

---

## 2. Bản Chất Services, EndpointSlices & `kube-proxy`

Pod là đối tượng ngắn hạn (ephemeral) — chết đi sống lại sẽ đổi IP. **Service** cung cấp một **Virtual IP (ClusterIP) cố định** và tên DNS cố định:

```text
Pod Client ──(1. DNS query)──► CoreDNS ──► trả về ClusterIP (ví dụ 10.96.10.20)
    │
    └──(2. Gửi gói tin tới 10.96.10.20:80)──► kube-proxy (iptables / IPVS / eBPF)
                                                   │ (DNAT + Load Balance ngẫu nhiên)
                                                   ▼
                                         EndpointSlice (10.244.1.5:8080, 10.244.2.8:8080)
                                                   │
                                                   ▼
                                              Pod Backend
```

### 4 Loại Service Cốt Lõi
| Loại Service | Bản chất | Dùng khi nào? |
| :--- | :--- | :--- |
| **`ClusterIP`** *(mặc định)* | IP ảo chỉ truy cập được **bên trong cụm** | Giao tiếp nội bộ giữa các microservice |
| **`NodePort`** | Mở 1 cổng cố định (dải `30000–32767`) trên **TẤT CẢ các Node** → đẩy về ClusterIP → Pod | Expose nhanh ra ngoài khi không có Cloud LoadBalancer |
| **`LoadBalancer`** | Tự động cấp IP public (Cloud LB / MetalLB) trỏ tới NodePort → ClusterIP | Production trên Cloud |
| **Headless (`clusterIP: None`)** | **Không cấp ClusterIP**, DNS trả thẳng danh sách **IP của từng Pod** | StatefulSet (Database cluster cần kết nối đích danh từng Pod) |

> **Phản xạ Kubestronaut:** Khi Service không hoạt động, chạy ngay `kubectl get ep,endpointslices <svc> -n <ns>`.
> - Nếu `Endpoints` **TRỐNG (`<none>`)**: 99% là **`Service.spec.selector` không khớp `Pod.metadata.labels`**, hoặc Pod đang fail **`readinessProbe`**!
> - Nếu `Endpoints` **CÓ IP** nhưng vẫn connection refused: 99% là **sai `targetPort`** (không khớp cổng app thực sự lắng nghe trong container) hoặc bị **NetworkPolicy** chặn!

---

## 3. CoreDNS — Hệ Thống Phân Giải Tên Miền Trong Cluster

Mọi Pod khi tạo ra đều được `kubelet` tự ghi `/etc/resolv.conf` trỏ `nameserver` tới ClusterIP của Service `kube-dns` trong `kube-system` và thiết lập `search`:
```text
search <pod-ns>.svc.cluster.local svc.cluster.local cluster.local
```
Nhờ đó:
- Gọi Service **cùng namespace**: chỉ cần `<svc-name>`.
- Gọi Service **khác namespace**: `<svc-name>.<target-ns>` hoặc FQDN đầy đủ `<svc-name>.<target-ns>.svc.cluster.local`.
- Gọi trực tiếp 1 Pod (IP `10.244.1.5` ở namespace `dev`): `10-244-1-5.dev.pod.cluster.local`.

---

## 4. NetworkPolicy — Tường Lửa Tầng L3/L4 & 3 Bẫy Kinh Điển

Mặc định trong Kubernetes: **Mọi Pod đều nói chuyện được với mọi Pod khác** (Allow All).
Ngay khi bạn áp một `NetworkPolicy` có `podSelector` khớp Pod X (và liệt kê `policyTypes: [Ingress]` hoặc `[Egress]`):
- **Tất cả traffic chiều đó tới/từ Pod X lập tức bị CHẶN (Default Deny)**, trừ những luồng được cho phép tường minh trong `ingress:` / `egress:`!
- **NetworkPolicy có tính cộng gộp (Additive — không có luật Deny):** Nếu nhiều NetworkPolicy cùng chọn 1 Pod, chỉ cần **1 luật cho phép** là gói tin được đi qua.
- **Lưu ý CNI:** Flannel thuần **không** thực thi NetworkPolicy; các cụm thi CKA luôn cài **Calico** hoặc **Cilium**.

### 3 Cái Bẫy Chết Người Của NetworkPolicy (Học Thuộc Lòng!)
1. **Bẫy AND vs OR:**
   - **AND** (cùng 1 mục `-` trong mảng `from`/`to`): Pod phải khớp CẢ `namespaceSelector` VÀ `podSelector`.
   - **OR** (2 mục `-` tách biệt): Khớp `namespaceSelector` HOẶC khớp `podSelector` (trong cùng namespace của Policy).
2. **Bẫy quên mở DNS cổng 53 ở Egress:** Khi bạn viết `Egress` policy chỉ cho Pod gọi tới DB cổng `5432`, Pod sẽ **không phân giải được tên miền `db-svc`** trừ khi bạn mở thêm Egress TCP/UDP port `53` tới CoreDNS!
3. **Bẫy thiếu `policyTypes`:** Luôn khai báo rõ `policyTypes: [Ingress]` hoặc `[Egress]` hoặc cả hai, đặc biệt khi viết Default Deny (không có block `ingress:`/`egress:` bên dưới).

---

## 5. Ingress vs Gateway API (Trọng Tâm Mới Nhất Của CKA 2025/2026)

### A. Ingress (Kiến trúc cũ)
- Chỉ có 1 tài nguyên duy nhất (`Ingress`) nhồi nhét chung cả TLS, Hostname, Path routing, và các tính năng nâng cao phải dựa vào **annotations riêng biệt không chuẩn hóa** của từng hãng (`nginx.ingress.kubernetes.io/...`).
- Các trường bắt buộc nhớ: `spec.ingressClassName`, `rules[].host`, `http.paths[].pathType` (`Prefix` hoặc `Exact`), `backend.service.name/port`, `tls[].secretName`.

### B. Gateway API (Chuẩn hiện đại — Chắc chắn có trong đề thi CKA mới!)
Gateway API giải quyết điểm yếu của Ingress bằng **mô hình phân tách 3 vai trò (Role-Oriented Design)**:

```text
[Infrastructure Provider]  ──►  GatewayClass  (Định nghĩa loại controller: nginx, envoy, cilium...)
                                      ▲
[Cluster Operator]         ──►     Gateway    (Khai báo cổng lắng nghe, HTTP/HTTPS, TLS certs, allowedRoutes)
                                      ▲ (parentRefs)
[Application Developer]    ──►    HTTPRoute   (Định nghĩa hostnames, path/header matching, filters, traffic splitting weights)
                                      │ (backendRefs)
                                      ▼
                                   Service ──► Pods
```

### So sánh nhanh để Migrate từ Ingress → Gateway API trong phòng thi:
| Khái niệm bên Ingress | Tương đương bên Gateway API (`gateway.networking.k8s.io/v1`) |
| :--- | :--- |
| `ingressClassName: nginx` | `Gateway.spec.gatewayClassName: nginx` |
| `spec.tls[].secretName` | `Gateway.spec.listeners[].tls.certificateRefs` (`mode: Terminate`) |
| `spec.rules[].host` | `HTTPRoute.spec.hostnames` (và tùy chọn ở `Gateway.listeners[].hostname`) |
| `pathType: Prefix` + `path: /api` | `HTTPRoute.spec.rules[].matches[].path: { type: PathPrefix, value: /api }` |
| `backend.service` | `HTTPRoute.spec.rules[].backendRefs` (hỗ trợ thêm `weight:` cho Canary!) |
| Cross-namespace backend | Cần đối tượng `ReferenceGrant` ở namespace đích |

> **Mẹo tra Docs phòng thi:** Trong phòng thi PSI, bạn được mở trực tiếp **`gateway.networking.k8s.io`**. Hãy nhớ 2 mục: **Guides → TLS** (cho `Gateway` HTTPS) và **Guides → Traffic splitting / HTTP routing** (cho `HTTPRoute`).

---

**Đã hiểu chắc bản chất Networking?** Thực hành ngay 8 bài lab chuẩn thi tại [02-hands-on-labs.md](./02-hands-on-labs.md).

