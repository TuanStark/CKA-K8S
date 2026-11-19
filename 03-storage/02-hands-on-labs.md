# Module 03: Hands-On Labs — Storage (10%)

> **Hướng dẫn:** Tự làm theo thời gian mục tiêu, xác nhận trạng thái `Bound` và dữ liệu ghi/đọc thực tế bên trong Pod trước khi mở đáp án.

---

## Lab 3.1: Static Provisioning — Tạo PV, PVC & Mount Vào Pod
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
1. Tạo namespace `storage-lab`.
2. Tạo PersistentVolume `app-pv`:
   - `capacity`: `1Gi`
   - `accessModes`: `ReadWriteOnce`
   - `persistentVolumeReclaimPolicy`: `Retain`
   - `storageClassName`: `manual`
   - `hostPath.path`: `/mnt/data-cka`
3. Tạo PersistentVolumeClaim `app-pvc` trong namespace `storage-lab` yêu cầu `500Mi`, `ReadWriteOnce`, `storageClassName: manual`.
4. Tạo Pod `writer-pod` (`nginx:1.27`) trong `storage-lab` mount `app-pvc` vào `/usr/share/nginx/html`. Ghi `"CKA Storage OK"` vào `index.html`, xóa Pod và tạo lại Pod mới mount cùng PVC để chứng minh dữ liệu còn nguyên.

<details>
<summary>Xem đáp án & lệnh verify Lab 3.1</summary>

```bash
k create ns storage-lab

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data-cka
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: storage-lab
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer-pod
  namespace: storage-lab
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: app-pvc
EOF

# === VERIFY ===
k get pv,pvc -n storage-lab        # STATUS: Bound
k wait --for=condition=Ready pod/writer-pod -n storage-lab --timeout=60s
k exec writer-pod -n storage-lab -- sh -c 'echo "CKA Storage OK" > /usr/share/nginx/html/index.html'
k exec writer-pod -n storage-lab -- curl -s localhost
```
</details>

---

## Lab 3.2: Tạo StorageClass Với `WaitForFirstConsumer` & Đặt Làm Mặc Định
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo StorageClass tên `local-fast`:
   - `provisioner`: `kubernetes.io/no-provisioner`
   - `volumeBindingMode`: `WaitForFirstConsumer`
   - `reclaimPolicy`: `Retain`
   - `allowVolumeExpansion`: `true`
2. Đặt `local-fast` làm **default StorageClass** duy nhất trong cluster (nhớ gỡ annotation default khỏi các StorageClass khác nếu đang có!).

<details>
<summary>Xem đáp án & lệnh verify Lab 3.2</summary>

```bash
# Kiểm tra SC nào đang là default để gỡ nếu cần
k get sc

cat <<'EOF' | k apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
EOF

# Nếu có SC cũ tên <old-sc> đang là default, gỡ bằng:
# k patch sc <old-sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# === VERIFY ===
k get sc     # Chỉ duy nhất local-fast có (default)
```
</details>

---

## Lab 3.3: Bẫy `Released` PV & Khôi Phục Tái Sử Dụng
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
Từ Lab 3.1:
1. Xóa Pod `writer-pod` và xóa PVC `app-pvc` trong `storage-lab`.
2. Kiểm tra trạng thái PV `app-pv` → quan sát trạng thái `Released`.
3. Tạo PVC mới `app-pvc-2` (cùng cấu hình `manual`, `500Mi`, `RWO`) → quan sát thấy nó bị `Pending`.
4. Xử lý `app-pv` để `app-pvc-2` tự động `Bound` vào `app-pv` mà **không xóa PV**.

<details>
<summary>Xem đáp án & lệnh verify Lab 3.3</summary>

```bash
k delete pod writer-pod -n storage-lab $now
k delete pvc app-pvc -n storage-lab
k get pv app-pv     # STATUS: Released

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc-2
  namespace: storage-lab
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF

k get pvc app-pvc-2 -n storage-lab    # STATUS: Pending

# Gỡ spec.claimRef trên PV để đưa PV từ Released -> Available -> tự Bound vào app-pvc-2
k patch pv app-pv --type json -p='[{"op": "remove", "path": "/spec/claimRef"}]'

# === VERIFY ===
k get pv app-pv
k get pvc app-pvc-2 -n storage-lab    # STATUS: Bound
```
</details>

---

## Lab 3.4: Sắp Xếp & Trích Xuất PV Theo Dung Lượng (Câu Hay Gặp Trong Đề Thi)
**Mục tiêu thời gian:** 3 phút

**Đề bài:**
Tạo thêm 2 PV nhỏ (`pv-small` 100Mi, `pv-large` 5Gi), sau đó xuất danh sách tất cả các PV được sắp xếp tăng dần theo **dung lượng (`capacity.storage`)** ra file `/tmp/pvs-by-size.txt`.

<details>
<summary>Xem đáp án & lệnh verify Lab 3.4</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-small
spec:
  capacity:
    storage: 100Mi
  accessModes: [ReadWriteOnce]
  hostPath:
    path: /mnt/small
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-large
spec:
  capacity:
    storage: 5Gi
  accessModes: [ReadWriteOnce]
  hostPath:
    path: /mnt/large
EOF

k get pv --sort-by='.spec.capacity.storage' > /tmp/pvs-by-size.txt

# === VERIFY ===
cat /tmp/pvs-by-size.txt
```
</details>

---

## Lab 3.5: Chẩn Đoán PVC `Pending` & Mở Rộng Dung Lượng
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
Cho YAML lỗi sau — hãy áp dụng vào cụm, tìm ra tất cả lỗi sai khiến `broken-pvc` bị `Pending` không bind được vào `pv-large` và sửa lại cho `Bound`:
- `pv-large` có `capacity: 5Gi`, `accessModes: [ReadWriteOnce]`, `storageClassName` để trống (`""`).
- `broken-pvc` yêu cầu `storage: 2Gi`, `accessModes: [ReadWriteMany]`, `storageClassName: local-fast`.

*(Lưu ý: Một số trường của PVC là immutable — khi cần sửa `accessModes` hoặc `storageClassName`, bạn phải xóa PVC và tạo lại!)*

<details>
<summary>Xem đáp án & lệnh verify Lab 3.5</summary>

**Nguyên nhân lỗi:**
1. Lệch `accessModes`: PVC yêu cầu `ReadWriteMany` nhưng `pv-large` chỉ có `ReadWriteOnce`.
2. Lệch `storageClassName`: PVC ghi `local-fast` nhưng `pv-large` có `storageClassName: ""`.

```bash
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fixed-pvc
  namespace: storage-lab
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF

# === VERIFY ===
k get pvc fixed-pvc -n storage-lab    # STATUS: Bound vào pv-large
```
</details>

