# Kubestronaut CKA Mastery Program — Lộ Trình Học & Thực Hành Trọn Gói

Chào mừng bạn đến với hệ thống ôn luyện **Certified Kubernetes Administrator (CKA)** được thiết kế theo tư duy **Kubestronaut**: **hiểu đúng bản chất vận hành bên dưới → thực hành chuẩn kỹ thuật → rèn phản xạ phòng thi → vượt mốc 66% với biên độ an toàn 80%+**.

Bạn **không cần tự tìm đường đi** nữa. Toàn bộ kiến thức bản chất, bài lab thực hành, kịch bản phá-sửa cụm (Break & Fix), sổ theo dõi và đề thi thử chuẩn cấu trúc 2025/2026 đã được thiết kế sẵn ngay trong thư mục này. Việc của bạn là **đi đúng thứ tự từ Module `00` đến Module `06`**.

---

## 1. Kiến Trúc Hệ Thống Học Tập Trong Thư Mục Này

| Thứ tự | Thư mục / Tệp | Trọng số thi | Vai trò trong lộ trình |
| :---: | :--- | :---: | :--- |
| **GĐ 0** | [00-lab-setup/README.md](./00-lab-setup/README.md) | Nền tảng | Dựng môi trường Lab chuẩn (`kubeadm` thật + `kind` / Killercoda) & cài Speed Toolkit |
| **M01** | [01-cluster-architecture/](./01-cluster-architecture/01-theory-deep-dive.md) | **25%** | Kiến trúc Control Plane, PKI, RBAC, `kubeadm` upgrade, `etcd` backup/restore, Helm, Kustomize, CRDs |
| **M02** | [02-workloads-scheduling/](./02-workloads-scheduling/01-theory-deep-dive.md) | **15%** | Scheduler pipeline, Deployments, ConfigMap/Secret, HPA, Affinity, Taints, PriorityClass, Static Pods |
| **M03** | [03-storage/](./03-storage/01-theory-deep-dive.md) | **10%** | CSI, PV/PVC binding, StorageClass, `WaitForFirstConsumer`, ReclaimPolicy, Volume Expansion |
| **M04** | [04-services-networking/](./04-services-networking/01-theory-deep-dive.md) | **20%** | CNI, kube-proxy, CoreDNS, Services, NetworkPolicy (AND/OR), Ingress & **Gateway API** |
| **M05** | [05-troubleshooting/](./05-troubleshooting/01-theory-deep-dive.md) | **30%** | Tư duy chẩn đoán 4 tầng & 10 bài thực chiến tự phá – tự sửa cụm (Break & Fix) |
| **M06** | [06-mock-exam-drill/mock-exam-01.md](./06-mock-exam-drill/mock-exam-01.md) | **100%** | Đề thi thử 16 câu chuẩn form 2025/2026 (bấm giờ 120 phút) + Bộ chấm điểm & lời giải |
| **Hỗ trợ** | [CKA-ROADMAP.md](./CKA-ROADMAP.md) · [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) · [CHEATSHEET.md](./CHEATSHEET.md) · [ERROR-LOG.md](./ERROR-LOG.md) | Xuyên suốt | Tổng quan chiến lược thi PSI, bảng tick tiến độ, sổ lệnh thực chiến & nhật ký lỗi sai |

---

## 2. Quy Tắc Học Của Một Kubestronaut (Bắt Buộc Tuân Thủ)

1. **Hiểu trước, gõ sau:** Mỗi module đều có 2 file: `01-theory-deep-dive.md` (bản chất) và `02-hands-on-labs.md` (thực hành). Đọc kỹ phần bản chất để hiểu *vì sao Kubernetes hoạt động như vậy*, rồi mới mở lab.
2. **Cấm đọc đáp án trước khi thử:** Đáp án trong các bài lab đều được gói trong thẻ thu gọn. Hãy tự làm bằng CLI và tài liệu chính thức (`kubernetes.io/docs`, `gateway.networking.k8s.io`, `helm.sh/docs`) trước khi đối chiếu.
3. **Luôn tự Verify:** Không bao giờ coi là "xong" khi mới thấy `created` hoặc `configured`. Bắt buộc chạy lệnh kiểm chứng thực tế (`get`, `describe`, `curl`, `nslookup`, `auth can-i`).
4. **Quy tắc 24 giờ cho lỗi sai:** Câu nào sai hoặc làm quá thời gian mục tiêu → ghi ngay vào [ERROR-LOG.md](./ERROR-LOG.md) → hôm sau xóa đi **tự gõ lại từ đầu không nhìn tài liệu**.
5. **Đặt ngưỡng qua bài:** Chỉ đánh dấu `[x]` trong [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) khi bạn tự làm trơn tru dưới thời gian quy định.

---

## 3. Lịch Trình Thực Thi Từng Ngày (Kế Hoạch 30 Ngày Chuẩn Chỉnh)

*(Mỗi ngày cần **2 – 2.5 giờ** tập trung. Nếu bạn học 1–1.5 giờ/ngày, kéo giãn thành 6 tuần theo đúng thứ tự dưới đây.)*

### Tuần 1: Nền Tảng Lab + Workloads + Storage + Bắt Đầu Cluster Architecture
| Ngày | Việc cần làm | Tài liệu mở | Đầu ra bắt buộc |
| :---: | :--- | :--- | :--- |
| **Ngày 1** | Dựng môi trường Lab + cài Speed Toolkit + làm 1 bài test chẩn đoán đầu vào | [00-lab-setup/README.md](./00-lab-setup/README.md) | Cụm lab chạy ổn, thuộc tay `k`, `$do`, `vim` YAML |
| **Ngày 2** | Học sâu bản chất Workloads & Scheduling (Scheduler pipeline, Pod lifecycle, HPA, Affinity, Taints) | [02-workloads-scheduling/01-theory-deep-dive.md](./02-workloads-scheduling/01-theory-deep-dive.md) | Giải thích được cơ chế lập lịch, probes, HPA, Static Pod |
| **Ngày 3** | Thực hành toàn bộ 7 bài Lab Workloads & Scheduling | [02-workloads-scheduling/02-hands-on-labs.md](./02-workloads-scheduling/02-hands-on-labs.md) | Pass 7/7 lab + tick [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) |
| **Ngày 4** | Học bản chất Storage (CSI, PV/PVC Binding, `WaitForFirstConsumer`, Expansion) + làm 5 bài Lab Storage | [03-storage/01-theory-deep-dive.md](./03-storage/01-theory-deep-dive.md) & [02-hands-on-labs.md](./03-storage/02-hands-on-labs.md) | Pass 5/5 lab Storage |
| **Ngày 5** | Học bản chất Kiến trúc Cụm: Control Plane, Request Lifecycle, PKI/Certs, RBAC | [01-cluster-architecture/01-theory-deep-dive.md](./01-cluster-architecture/01-theory-deep-dive.md) (Phần 1–3) | Vẽ lại được luồng 1 lệnh `kubectl apply` đi qua Control Plane |
| **Ngày 6** | Thực hành RBAC, `kubeadm` upgrade (Control Plane + Worker) trên cụm thật | [01-cluster-architecture/02-hands-on-labs.md](./01-cluster-architecture/02-hands-on-labs.md) (Lab 1–3) | Upgrade thành công cụm không lỗi |
| **Ngày 7** | Thực hành `etcd` Backup & Restore + ôn lại các lỗi Tuần 1 trong Error Log | [01-cluster-architecture/02-hands-on-labs.md](./01-cluster-architecture/02-hands-on-labs.md) (Lab 4) | Restore `etcd` thành công dưới 8 phút |

### Tuần 2: Hoàn Tất Cluster Arch (Helm/Kustomize/CRD) + Làm Chủ Services & Networking
| Ngày | Việc cần làm | Tài liệu mở | Đầu ra bắt buộc |
| :---: | :--- | :--- | :--- |
| **Ngày 8** | Học bản chất & thực hành Helm, Kustomize, CRDs/Operators | [01-cluster-architecture/01-theory-deep-dive.md](./01-cluster-architecture/01-theory-deep-dive.md) (Phần 4–6) & [02-hands-on-labs.md](./01-cluster-architecture/02-hands-on-labs.md) (Lab 5–8) | Pass 8/8 lab Module 01 |
| **Ngày 9** | Học bản chất Networking: Pod Network Model, CNI, kube-proxy, Services, CoreDNS | [04-services-networking/01-theory-deep-dive.md](./04-services-networking/01-theory-deep-dive.md) (Phần 1–3) | Hiểu rõ đường đi gói tin Pod→Service→Endpoint→Pod |
| **Ngày 10** | Học bản chất NetworkPolicy (AND/OR), Ingress & **Gateway API** | [04-services-networking/01-theory-deep-dive.md](./04-services-networking/01-theory-deep-dive.md) (Phần 4–6) | Phân biệt chuẩn Ingress vs Gateway API, nắm chắc AND/OR |
| **Ngày 11** | Thực hành Lab Services, CoreDNS, NetworkPolicy (Lab 1–4) | [04-services-networking/02-hands-on-labs.md](./04-services-networking/02-hands-on-labs.md) | Viết NetPol chuẩn không bị bẫy DNS/AND-OR |
| **Ngày 12** | Thực hành Lab Ingress & Gateway API + Migrate Ingress → Gateway API (Lab 5–8) | [04-services-networking/02-hands-on-labs.md](./04-services-networking/02-hands-on-labs.md) | Pass 8/8 lab Networking |
| **Ngày 13** | Làm lại từ đầu toàn bộ các câu từng sai ở Tuần 1–2 trong Error Log | [ERROR-LOG.md](./ERROR-LOG.md) | 0 lỗi tồn đọng chưa sửa được |
| **Ngày 14** | Nghỉ phục hồi nhẹ + đọc lướt [CHEATSHEET.md](./CHEATSHEET.md) & tập tra cứu Docs < 15 giây | [CHEATSHEET.md](./CHEATSHEET.md) | Tìm đúng mẫu YAML trên docs chính thức < 15s |

### Tuần 3: Đột Phá Troubleshooting (30% Điểm) — Tư Duy Phá & Sửa Cụm
| Ngày | Việc cần làm | Tài liệu mở | Đầu ra bắt buộc |
| :---: | :--- | :--- | :--- |
| **Ngày 15** | Nắm vững Khung Tư Duy Troubleshooting 4 Tầng (Pod → Net → Node → Control Plane) | [05-troubleshooting/01-theory-deep-dive.md](./05-troubleshooting/01-theory-deep-dive.md) | Thuộc luồng phản xạ cho mọi triệu chứng lỗi |
| **Ngày 16** | Break & Fix Tầng 1 & Tầng 2 (Pod, Probe, OOM, Service, DNS, NetPol — Kịch bản 1–4) | [05-troubleshooting/02-break-and-fix-labs.md](./05-troubleshooting/02-break-and-fix-labs.md) | Tự phát hiện nguyên nhân gốc trong < 4 phút/ca |
| **Ngày 17** | Break & Fix Tầng 3: Worker Node `NotReady` (`kubelet`, `containerd`, `kubelet.conf`, swap — Kịch bản 5–7) | [05-troubleshooting/02-break-and-fix-labs.md](./05-troubleshooting/02-break-and-fix-labs.md) | Khôi phục Worker Node lên `Ready` thuần thục |
| **Ngày 18** | Break & Fix Tầng 4: Control Plane sập (`apiserver`, `scheduler`, `etcd`, `crictl` — Kịch bản 8–10) | [05-troubleshooting/02-break-and-fix-labs.md](./05-troubleshooting/02-break-and-fix-labs.md) | Sửa cụm khi `kubectl` chết hẳn trong < 6 phút |
| **Ngày 19** | Tổng duyệt lại 10 kịch bản Break & Fix ngẫu nhiên (bấm giờ) | [05-troubleshooting/02-break-and-fix-labs.md](./05-troubleshooting/02-break-and-fix-labs.md) | Pass 10/10 kịch bản |
| **Ngày 20** | **Thi thử Mock Exam #01** (bấm đúng 120 phút, chỉ mở Docs được phép) | [06-mock-exam-drill/mock-exam-01.md](./06-mock-exam-drill/mock-exam-01.md) | Tự chấm điểm với [mock-exam-01-solutions.md](./06-mock-exam-drill/mock-exam-01-solutions.md) |
| **Ngày 21** | Phân tích toàn bộ Mock Exam #01, cập nhật Error Log & làm lại các câu mất điểm | [ERROR-LOG.md](./ERROR-LOG.md) | Hiểu triệt để 16/16 câu |

### Tuần 4: Tăng Tốc Thực Chiến — Killer.sh & Chốt Lịch Thi Thật
| Ngày | Việc cần làm | Tài liệu mở | Đầu ra bắt buộc |
| :---: | :--- | :--- | :--- |
| **Ngày 22–23** | Luyện đề bổ sung theo domain yếu (Killercoda / KodeKloud / Mock #01 vòng 2) | [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) | Đạt ≥ 85% ở bài tập từng domain |
| **Ngày 24** | **Killer.sh — Phiên 1** (bấm đúng 120 phút như thi thật) | Killer.sh (Linux Foundation) | Mốc tốt lần 1: ≥ 55–65%; ghi toàn bộ câu sai vào Error Log |
| **Ngày 25** | Tận dụng 34h còn lại của Phiên 1: đọc giải thích chính thức, làm lại toàn bộ câu sai/chậm | [ERROR-LOG.md](./ERROR-LOG.md) | Tự làm lại 100% câu hỏi Killer.sh không cần xem đáp án |
| **Ngày 26–27** | Vá triệt để lỗ hổng còn lại (đặc biệt Gateway API, Helm/Kustomize, `etcd`, `kubeadm`, Control Plane) | Các module tương ứng | Tích xanh ≥ 95% bảng [PROGRESS-TRACKER.md](./PROGRESS-TRACKER.md) |
| **Ngày 28** | **Killer.sh — Phiên 2** (bấm đúng 120 phút tổng duyệt) | Killer.sh (Linux Foundation) | **Đạt ≥ 85%–90%** và còn dư ≥ 15 phút |
| **Ngày 29** | Kiểm tra PSI Secure Browser, giấy tờ, dọn bàn thi; chỉ đọc lại Error Log & Cheat Sheet | [CKA-ROADMAP.md](./CKA-ROADMAP.md) (Giai đoạn 4) | Hoàn tất 100% Checklist trước ngày thi, ngủ đủ 8 tiếng |
| **Ngày 30** | **NGÀY THI CKA** — Áp dụng chiến thuật 2 lượt, luôn check `context`, luôn verify | — | **PASS CKA!** |
