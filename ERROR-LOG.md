# CKA Error Log — Sổ Nhật Ký Lỗi Sai Khi Luyện Đề

> **Nguyên tắc vàng:** Mỗi khi làm sai 1 câu, làm quá 8 phút, hoặc phải mở gợi ý — hãy ghi ngay vào bảng dưới đây. Sau **24 giờ**, đóng đáp án và **tự làm lại câu đó từ đầu** cho đến khi tích được cột `Đã làm lại trơn tru`.

| # | Ngày | Nguồn đề / Câu số | Domain | Lỗi mắc phải (Cụ thể) | Nguyên nhân gốc rễ (Quên cú pháp / Hiểu sai / Đọc thiếu đề) | Cách làm đúng & Lệnh Verify | Đã làm lại sau 24h? |
| :-: | :--- | :--- | :--- | :--- | :--- | :--- | :-: |
| 01 | *(Ví dụ)* | Killer.sh #04 | Networking | NetworkPolicy không cho Pod gọi DNS | Áp Egress policy nhưng quên mở port 53 UDP/TCP | Thêm egress rule port 53 UDP/TCP tới `kube-system`; verify bằng `nslookup` | [ ] |
| 02 | *(Ví dụ)* | Lab #12 | Cluster Arch | `etcd` không lên sau khi restore | Quên đổi `hostPath.path` trong `/etc/kubernetes/manifests/etcd.yaml` sang `--data-dir` mới | Sửa `etcd-data` hostPath → chờ `crictl ps` thấy `etcd` Running → `k get pods -A` | [ ] |
| 03 | | | | | | | [ ] |
| 04 | | | | | | | [ ] |
| 05 | | | | | | | [ ] |

---

## Tổng Kết Lỗi Hay Lặp Lại (Rà Soát Cuối Mỗi Tuần)

Ghi lại 3–5 lỗi khiến bạn mất thời gian hoặc mất điểm nhiều nhất trong tuần để tập trung sửa:

- **Tuần 1:**
  - ...
- **Tuần 2:**
  - ...
- **Tuần 3:**
  - ...
- **Tuần 4:**
  - ...
- **Tuần 5 (Trước ngày thi):**
  - ...

