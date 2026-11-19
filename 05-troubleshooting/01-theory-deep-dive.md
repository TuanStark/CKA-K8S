# Module 05: Troubleshooting (30%) — Tư Duy Chẩn Đoán 4 Tầng Của Kubestronaut

> **Tại sao module này quyết định ĐỖ hay TRƯỢT CKA?** Troubleshooting chiếm **30% tổng số điểm** — lớn nhất trong cả 5 domain! Sự khác biệt giữa người trượt (55%) và người đỗ chắc (85%+) nằm ở chỗ: **người đỗ có quy trình điều tra khoa học trong 60 giây đầu tiên, không bao giờ đoán mò hay sửa bừa**.

---

## 1. Nguyên Tắc Vàng Khi Gặp Sự Cố Trong Phòng Thi

1. **Sao lưu trước khi sửa file hệ thống:** Trước khi chạm vào bất kỳ file nào trong `/etc/kubernetes/manifests/` hoặc `/var/lib/kubelet/`, luôn chạy `cp <file> /tmp/<file>.bak`.
2. **Đọc lỗi từ dưới lên trên (Root Cause, không phải triệu chứng):** Đừng nhìn `CrashLoopBackOff` rồi vội xóa Pod — hãy xem `describe`, `logs --previous`, và `events` để biết chính xác container chết vì mã lỗi gì (`OOMKilled`, exit code `1`, thiếu file config, sai lệnh...).
3. **Định vị đúng Tầng sự cố trước khi gõ lệnh:**

```text
                       `kubectl get nodes` có chạy được không?
                         │                               │
                        KHÔNG                            CÓ
                         │                               │
                         ▼                               ▼
            ┌────────────────────────┐      Node có bị `NotReady` không?
            │ TẦNG 4: CONTROL PLANE  │        │                     │
            │ (apiserver / etcd /    │       CÓ                   KHÔNG
            │  kubelet trên CP node) │        │                     │
            └────────────────────────┘        ▼                     ▼
                                   ┌───────────────────┐   Pod có `Running & Ready`?
                                   │ TẦNG 3: WORKER    │     │                  │
                                   │ (kubelet / CRI /  │   KHÔNG               CÓ (nhưng
                                   │  kubelet.conf /   │     │              không gọi được)
                                   │  swap / disk)     │     ▼                  │
                                   └───────────────────┘ ┌──────────────┐       ▼
                                                         │ TẦNG 1: POD  │ ┌──────────────────┐
                                                         │ (Scheduler / │ │ TẦNG 2: NETWORK  │
                                                         │  Image / App │ │ (EP / Port / DNS │
                                                         │  Probe / PVC)│ │  NetPol / CNI)   │
                                                         └──────────────┘ └──────────────────┘
```

---

## 2. Bản Đồ Triệu Chứng → Nguyên Nhân Gốc → Lệnh Chẩn Đoán

### Tầng 1: Ứng Dụng / Pod
| Trạng thái Pod | 95% Nguyên nhân gốc | Lệnh kiểm tra đầu tiên |
| :--- | :--- | :--- |
| **`Pending`** | Thiếu CPU/RAM, node bị Taint, sai `nodeSelector`/`Affinity`, PVC `Pending`, hoặc **`kube-scheduler` đang chết!** | `k describe pod <p>` (xem mục `Events:`); nếu không có event nào → kiểm tra `k get pods -n kube-system` xem `kube-scheduler` còn sống không! |
| **`ImagePullBackOff`** | Sai tên image/tag, registry không tồn tại, hoặc thiếu `imagePullSecrets` | `k describe pod <p>` → xem dòng `Failed to pull image` |
| **`CreateContainerConfigError`** | Tham chiếu tới `ConfigMap` hoặc `Secret` không tồn tại (hoặc sai key) | `k describe pod <p>` |
| **`CrashLoopBackOff`** | Lệnh `command/args` sai, app crash ngay khi khởi động, thiếu biến môi trường, hoặc **`livenessProbe` sai** | `k logs <p> -c <c> --previous` và `k describe pod <p>` |
| **`OOMKilled` (Exit 137)** | Container ngốn RAM vượt quá `resources.limits.memory` | `k describe pod <p>` → `Last State: Terminated, Reason: OOMKilled` → tăng `limits.memory` |
| **`Running` nhưng `0/1 Ready`** | **`readinessProbe` đang fail** (sai path HTTP, sai port, hoặc app chưa sẵn sàng) | `k describe pod <p>` → xem `Readiness probe failed` |

### Tầng 2: Service, DNS & Networking
Đi tuần tự đúng **4 bước cô lập điểm nghẽn**:
1. **Kiểm tra Endpoints:** `k get ep,endpointslices <svc> -n <ns>` → Trống? Kiểm tra `Service.spec.selector` vs `Pod.metadata.labels` và `readinessProbe`.
2. **Test trực tiếp Pod IP:** `curl http://<pod-ip>:<container-port>` từ pod tạm → Fail? Lỗi nằm ở app trong container hoặc NetworkPolicy.
3. **Test Service IP:** `curl http://<svc-cluster-ip>:<svc-port>` → Pod IP ok nhưng Service IP fail? Kiểm tra `targetPort` trên Service hoặc `kube-proxy` pod.
4. **Test tên DNS:** `nslookup <svc>.<ns>.svc.cluster.local` → Service IP ok nhưng DNS fail? Kiểm tra Pod `coredns` trong `kube-system`, ConfigMap `coredns`, hoặc Egress NetworkPolicy chặn cổng `53`.

### Tầng 3: Worker Node (`NotReady`)
SSH ngay vào Worker Node bị `NotReady` (`ssh <node>`) và chạy chuỗi 5 lệnh:
1. `systemctl status kubelet containerd --no-pager` — Dịch vụ nào đang `inactive`/`failed`?
2. `journalctl -u kubelet -n 50 --no-pager` — Đọc đúng dòng lỗi màu đỏ!
3. Kiểm tra 2 file cấu hình hay bị sửa hỏng nhất trong đề thi:
   - `/etc/kubernetes/kubelet.conf` (hay bị sửa sai cổng `server: https://<ip>:6443` hoặc sai đường dẫn cert!)
   - `/var/lib/kubelet/config.yaml` và `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` (sai cờ khởi động hoặc sai `cgroupDriver`).
4. Kiểm tra chứng chỉ hết hạn: `openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates`.
5. Kiểm tra tài nguyên node: `df -h` (đầy đĩa?), `free -m` (swap có bị bật lại không? → `swapoff -a`).

### Tầng 4: Control Plane Sập (Khi `kubectl` Chết Hoàn Toàn!)
Khi `kubectl` trả về `The connection to the server <ip>:6443 was refused`:
> **Tuyệt đối nhớ:** Lúc này `kubectl` vô dụng! Bạn phải làm việc **trực tiếp trên OS của Control Plane node** với **`crictl`**, **`journalctl`** và **`/etc/kubernetes/manifests/`**!

Chuỗi điều tra chuẩn Kubestronaut:
1. Kiểm tra `kubelet` & `containerd` trên Control Plane còn sống không: `systemctl status kubelet containerd`.
2. Liệt kê tất cả container (kể cả đã crash): `crictl ps -a`.
   - Tìm container `kube-apiserver` hoặc `etcd` ở trạng thái `Exited`.
3. Đọc log container vừa chết: `crictl logs <container-id>` (hoặc xem file trong `/var/log/pods/kube-system_kube-apiserver.../` và `/var/log/containers/`).
4. Mở `/etc/kubernetes/manifests/kube-apiserver.yaml` (hoặc `etcd.yaml`, `kube-scheduler.yaml`, `kube-controller-manager.yaml`) soi kỹ:
   - Sai tên cờ (ví dụ `--etcd-server` thiếu chữ `s`, hoặc cờ bị xóa ở bản K8s mới).
   - Sai port `etcd` (phải là `2379`, không phải `2380` — `2380` là peer port!).
   - Sai đường dẫn cert trong `/etc/kubernetes/pki/...` hoặc sai `volumeMounts` / `hostPath`.
   - Lỗi thụt dòng YAML khiến `kubelet` không parse được manifest (lỗi này thấy rõ ngay trong `journalctl -u kubelet`!).
5. Sửa xong lưu file → `kubelet` tự động khởi động lại Static Pod sau 5–20 giây (nếu không tự lên, chạy `systemctl restart kubelet` hoặc di chuyển file ra `/tmp` rồi đưa lại vào `/etc/kubernetes/manifests/`).

---

**Đã nắm chắc bản đồ chẩn đoán 4 tầng?** Bắt tay vào thực chiến **10 kịch bản Tự Phá – Tự Sửa** tại [02-break-and-fix-labs.md](./02-break-and-fix-labs.md).

