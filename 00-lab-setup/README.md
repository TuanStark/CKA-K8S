# Module 00: Thiết Lập Môi Trường Lab & Bộ Công Cụ Tốc Độ (Speed Toolkit)

> **Mục tiêu Module 00:** Có sẵn môi trường thực hành giống phòng thi thật nhất (đặc biệt có cụm **`kubeadm` thật** để upgrade, backup/restore `etcd`, và phá/sửa Control Plane & Worker Node), đồng thời thiết lập phản xạ gõ terminal cực nhanh.

---

## 1. Kiến Trúc Môi Trường Lab Chuẩn Để Luyện CKA

Một sai lầm phổ biến là chỉ luyện trên `minikube` hoặc Docker Desktop. Để đạt 100% kiến thức CKA, bạn nên kết hợp **2 lớp môi trường**:

| Lớp môi trường | Dùng để luyện gì? | Công cụ đề xuất |
| :--- | :--- | :--- |
| **Lớp 1: Cụm `kubeadm` thật (2–3 node Linux)** | `kubeadm init/join/upgrade`, `etcd` backup/restore, Static Pods, phá & sửa `kubelet`/`containerd`/`kube-apiserver`, CNI | **Killercoda** (miễn phí trên trình duyệt — có sẵn cụm `kubeadm` 2 node!) HOẶC **Vagrant / Multipass / VMware / VirtualBox** trên máy local (Ubuntu 22.04/24.04) |
| **Lớp 2: Cụm nhanh trên máy local (tùy chọn)** | Luyện nhanh Workloads, Storage, RBAC, Helm, Kustomize, Gateway API, NetworkPolicy khi offline | **`kind`** (Kubernetes in Docker) + cài đặt `helm`, `kubectl` |

> **Khuyên dùng từ Kubestronaut:**
> - Nếu máy bạn ≥ 16GB RAM: dựng cụm `kubeadm` 2 node (1 control-plane 2 vCPU/2GB RAM + 1 worker 2 vCPU/2GB RAM) bằng **Multipass** hoặc **Vagrant** để tự do phá và chụp snapshot VM trong 5 giây.
> - Nếu máy RAM thấp hoặc muốn thực hành ngay lập tức: dùng **Killercoda Kubernetes Playground** (`killercoda.com/playgrounds/scenario/kubernetes`) — miễn phí, có sẵn cụm `kubeadm` thật 2 node (`controlplane` + `node01`), reset sạch mỗi phiên.

---

## 2. Hướng Dẫn Dựng Cụm `kubeadm` Chuẩn Trên Ubuntu 22.04 / 24.04 (Chạy Trên Cả Control Plane & Worker)

Đây đồng thời là kiến thức thuộc **Domain Cluster Architecture (25%)** — hãy hiểu từng dòng lệnh chứ đừng chỉ copy-paste!

### Bước 1: Chuẩn bị OS, Kernel & Containerd (Chạy trên TẤT CẢ các node)

```bash
# 1. Tắt swap (kubelet mặc định từ chối chạy nếu swap đang bật)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Nạp kernel modules phục vụ container networking & overlay filesystem
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 3. Bật IP forwarding & bridge netfilter cho iptables thấy traffic bridge
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4. Cài containerd và bật SystemdCgroup = true (bắt buộc khớp với kubelet)
sudo apt-get update
sudo apt-get install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Bước 2: Cài `kubelet`, `kubeadm`, `kubectl` (Chạy trên TẤT CẢ các node)

```bash
# Thêm repo chính thức pkgs.k8s.io (chọn minor version, ví dụ v1.31 để sau luyện upgrade lên v1.32)
K8S_MINOR="v1.31"
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL "https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key" | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Bước 3: Khởi tạo Control Plane & Join Worker

```bash
# === CHỈ CHẠY TRÊN CONTROL PLANE ===
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Cài CNI (ví dụ Calico) để node chuyển từ NotReady -> Ready
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml

# Lấy lệnh join nếu quên:
kubeadm token create --print-join-command

# === CHẠY LỆNH JOIN TRÊN WORKER NODE ===
# sudo kubeadm join <cp-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Bước 4: Cài Thêm Công Cụ Bắt Buộc Cho CKA 2025/2026

```bash
# 1. Cài Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 2. Cài etcdctl / etcdutl (nếu chưa có trên control plane)
# Kiểm tra: which etcdctl etcdutl
# Hoặc: sudo apt-get install -y etcd-client

# 3. Cài Metrics Server (cần cho bài Lab HPA & kubectl top)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# Lưu ý trên cụm lab self-signed cert: cần thêm cờ --kubelet-insecure-tls vào Deployment metrics-server trong kube-system
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## 3. Thiết Lập Speed Toolkit (Chạy & Luyện Thành Phản Xạ Cơ Bắp)

Thêm vào `~/.bashrc` trên máy lab của bạn và luyện gõ thuộc lòng trong 30 giây (để vào phòng thi PSI thiết lập ngay nếu chưa có sẵn):

```bash
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
```

Cấu hình `~/.vimrc` cho việc sửa YAML nhanh, không bao giờ lỗi tab/indent:
```vim
set tabstop=2
set shiftwidth=2
set expandtab
set number
```

---

## 4. Bài Tập Khởi Động 10 Phút (Kiểm Tra Phản Xạ Cơ Bản)

Hãy bấm giờ 10 phút và tự thực hiện 5 thao tác sau trên terminal **không tra Google**:

1. Sinh file `pod.yaml` cho pod tên `warmup`, image `nginx:1.27`, port `80`, label `tier=web` bằng 1 lệnh duy nhất với `$do`.
2. Tạo Deployment `api` gồm 3 replica, image `nginx:1.27`, sau đó expose thành Service `api-svc` cổng `8080` trỏ tới container port `80`.
3. Xóa nhanh pod `warmup` trong < 1 giây bằng biến `$now`.
4. Dùng `kubectl explain` tìm chính xác tên trường để cấu hình `readinessProbe` dạng `httpGet` trong Pod.
5. Liệt kê tất cả node cùng `InternalIP` và `OS-IMAGE` bằng `-o custom-columns`.

<details>
<summary>Xem đáp án tham khảo Bài Tập Khởi Động</summary>

```bash
# 1
k run warmup --image=nginx:1.27 --port=80 --labels="tier=web" $do > pod.yaml
k apply -f pod.yaml

# 2
k create deploy api --image=nginx:1.27 --replicas=3
k expose deploy api --name=api-svc --port=8080 --target-port=80

# 3
k delete pod warmup $now

# 4
k explain pod.spec.containers.readinessProbe.httpGet

# 5
k get nodes -o custom-columns='NAME:.metadata.name,IP:.status.addresses[?(@.type=="InternalIP")].address,OS:.status.nodeInfo.osImage'
```
</details>

---

**Hoàn tất Module 00?** Chuyển sang [Module 02: Workloads & Scheduling](../02-workloads-scheduling/01-theory-deep-dive.md) (nếu đi theo lịch 30 ngày từ dễ đến khó) hoặc [Module 01: Cluster Architecture](../01-cluster-architecture/01-theory-deep-dive.md).

