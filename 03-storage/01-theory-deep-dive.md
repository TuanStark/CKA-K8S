# Module 03: Storage (10%) — Hiểu Sâu Bản Chất

> **Mục tiêu bản chất:** Storage tuy chỉ chiếm **10% (1–2 câu thi)** nhưng là **10% cực kỳ dễ ăn trọn điểm tuyệt đối** nếu bạn nắm chắc cơ chế tách biệt giữa hạ tầng lưu trữ (`PV` / `StorageClass`) và nhu cầu của ứng dụng (`PVC`), cùng 2 cái bẫy kinh điển: `WaitForFirstConsumer` và `ReclaimPolicy`.

---

## 1. Kiến Trúc Lưu Trữ Trong Kubernetes

```text
                       ┌──────────────────┐
                       │   StorageClass   │  (Khuôn mẫu cấp phát động:
                       │  provisioner +   │   provisioner, bindingMode,
                       │  reclaimPolicy   │   allowVolumeExpansion)
                       └────────┬─────────┘
                                │ (Dynamic Provisioning)
                                ▼
┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
│     Pod      │──────►│       PVC        │◄─────►│        PV        │
│ (volumeMount)│       │  (Namespaced)    │ Bind  │ (Cluster-scoped) │
└──────────────┘       └──────────────────┘ 1 - 1 └──────────────────┘
```

### Phân biệt 4 loại Volume cốt lõi:
1. **`emptyDir`:** Tạo mới khi Pod được gán lên Node, **mất sạch khi Pod bị xóa** (nhưng *sống sót* nếu container bị crash & restart trong cùng Pod!). Rất hay dùng cho Sidecar chia sẻ file/log.
2. **`hostPath`:** Mount thẳng một thư mục từ filesystem của Worker Node vào Pod (chỉ dùng cho tác vụ hệ thống hoặc bài lab 1 node — không di động giữa các node).
3. **`PersistentVolume (PV)`:** Tài nguyên cấp **Cluster** (không có Namespace!) đại diện cho một phân vùng đĩa thật (NFS, Ceph, EBS, local disk...).
4. **`PersistentVolumeClaim (PVC)`:** Phiếu yêu cầu dung lượng cấp **Namespace** của ứng dụng. Pod chỉ gọi tên PVC, không gọi trực tiếp PV.

---

## 2. Cơ Chế Binding 1-1 Giữa PV và PVC (Rất Hay Bị Bẫy!)

Để một **PVC** bind được vào một **PV** (Static Provisioning), **BẮT BUỘC khớp cả 4 điều kiện**:
1. **`storageClassName` khớp hoàn toàn** (nếu PV không đặt `storageClassName` mà cụm có default StorageClass, PVC không ghi `storageClassName` sẽ tự lấy default SC → **không khớp PV**! Khi muốn bind thủ công không dùng SC, đặt `storageClassName: ""` ở cả hai bên).
2. **`accessModes` khớp:**
   - `ReadWriteOnce` (`RWO`): Mount đọc/ghi bởi **1 Node** tại một thời điểm.
   - `ReadOnlyMany` (`ROX`): Mount chỉ đọc bởi **nhiều Node**.
   - `ReadWriteMany` (`RWX`): Mount đọc/ghi đồng thời bởi **nhiều Node** (ví dụ NFS, CephFS).
   - `ReadWriteOncePod` (`RWOP`): Chỉ duy nhất **1 Pod** trong toàn cluster được mount đọc/ghi.
3. **Dung lượng:** `PV capacity >= PVC request`. (Nếu có nhiều PV đạt chuẩn, Kubernetes ưu tiên PV có dung lượng **nhỏ nhất nhưng vẫn đủ**).
4. **Label Selector** (nếu PVC có khai báo `spec.selector`).

> **Lưu ý quan trọng:** Quan hệ PV ↔ PVC là **1-1 độc quyền**. Nếu PVC yêu cầu `1Gi` mà chỉ có PV `10Gi` khả dụng, PVC sẽ bind chiếm trọn cả PV `10Gi` đó!

---

## 3. StorageClass, Dynamic Provisioning & 2 Bẫy Kinh Điển

### Bẫy #1: `volumeBindingMode: WaitForFirstConsumer`
- Với `Immediate`: Ngay khi bạn tạo PVC, provisioner lập tức tạo PV ở 1 zone bất kỳ → có nguy cơ Pod sau đó được schedule ở zone khác và không mount được đĩa!
- Với **`WaitForFirstConsumer`**: Khi bạn vừa tạo PVC, nó sẽ **cố tình giữ trạng thái `Pending`**! Đừng hoảng tưởng lỗi — nó chỉ chờ cho đến khi có **Pod đầu tiên sử dụng PVC đó được schedule lên 1 Node**, lúc đó PV mới được tạo đúng vị trí và PVC tự chuyển sang `Bound`.

### Bẫy #2: `reclaimPolicy` (`Delete` vs `Retain`)
Khi bạn **xóa PVC**, chuyện gì xảy ra với PV và dữ liệu bên trong?
- **`Delete`** (mặc định của Dynamic Provisioning): Tự động xóa luôn PV và ổ đĩa vật lý bên dưới.
- **`Retain`** (mặc định của Static PV): Giữ nguyên PV và dữ liệu, chuyển trạng thái PV sang **`Released`** (chứ không về `Available`!). Một PVC mới **không thể bind lại** vào PV đang `Released` cho đến khi admin kiểm tra dữ liệu và gỡ trường `spec.claimRef` trên PV.

### Volume Expansion (Mở rộng dung lượng PVC)
- Yêu cầu StorageClass có `allowVolumeExpansion: true`.
- Chỉnh sửa trực tiếp `spec.resources.requests.storage` trên PVC tăng lên (chỉ tăng, không giảm).

---

**Đã hiểu chắc bản chất Storage?** Thực hành ngay 5 bài lab tại [02-hands-on-labs.md](./02-hands-on-labs.md).

