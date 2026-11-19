# Module 05: 10 Kịch Bản Thực Chiến "Tự Phá – Tự Sửa" (Break & Fix Labs — 30%)

> **Cách luyện cực hiệu quả:** Chạy đoạn **"Lệnh Tạo Lỗi (Break)"**, sau đó **đóng mục đáp án lại**, giả định bạn vừa vào phòng thi gặp đúng sự cố đó, và áp dụng đúng **Tư duy 4 Tầng** để tìm ra nguyên nhân gốc + sửa dứt điểm + verify!
>
> *Lưu ý: Kịch bản 5–10 thực hiện trên cụm `kubeadm` thật (Killercoda / VM). Luôn sao lưu file hệ thống trước khi phá!*

---

## Kịch bản 1 (Tầng 1): Pod Kẹt `CreateContainerConfigError` & `CrashLoopBackOff`
**Mục tiêu thời gian:** 4 phút

**Lệnh Tạo Lỗi:**
```bash
k create ns dbg
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  namespace: dbg
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo $APP_GREETING && sleep 3600"]
      env:
        - name: APP_GREETING
          valueFrom:
            configMapKeyRef:
              name: greeting-cm
              key: msg
EOF
```
**Nhiệm vụ:** Đưa `broken-app` lên trạng thái `Running` (`1/1`) và in ra dòng `"Hello CKA"` trong logs.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 1</summary>

1. Chạy `k get pod broken-app -n dbg` → `CreateContainerConfigError`.
2. Chạy `k describe pod broken-app -n dbg` → Events báo: `configmap "greeting-cm" not found`.
3. Tạo ConfigMap còn thiếu:
```bash
k create cm greeting-cm -n dbg --from-literal=msg="Hello CKA"
# Verify
k get pod broken-app -n dbg     # Running 1/1
k logs broken-app -n dbg        # Hello CKA
```
</details>

---

## Kịch bản 2 (Tầng 1): Pod `Running` Nhưng `OOMKilled` Liên Tục
**Mục tiêu thời gian:** 4 phút

**Lệnh Tạo Lỗi:**
```bash
cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mem-hog
  namespace: dbg
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mem-hog
  template:
    metadata:
      labels:
        app: mem-hog
    spec:
      containers:
        - name: stress
          image: polinux/stress
          command: ["stress"]
          args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
          resources:
            limits:
              memory: "32Mi"
EOF
```
**Nhiệm vụ:** Chẩn đoán nguyên nhân crash và sửa Deployment sao cho Pod chạy ổn định không còn restart.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 2</summary>

1. `k describe pod -n dbg -l app=mem-hog` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137` (ứng dụng cần `100M` nhưng limit chỉ `32Mi`).
2. Tăng `limits.memory` lên `256Mi`:
```bash
k set resources deploy/mem-hog -n dbg --limits=memory=256Mi --requests=memory=128Mi
k rollout status deploy/mem-hog -n dbg
k get pods -n dbg -l app=mem-hog
```
</details>

---

## Kịch bản 3 (Tầng 1–2): Pod `0/1 Ready` & Service Không Có Endpoints
**Mục tiêu thời gian:** 5 phút

**Lệnh Tạo Lỗi:**
```bash
cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portal
  namespace: dbg
spec:
  replicas: 2
  selector:
    matchLabels:
      app: portal
  template:
    metadata:
      labels:
        app: portal
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz-nonexistent
              port: 8080
            periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: portal-svc
  namespace: dbg
spec:
  selector:
    app: portal-wrong
  ports:
    - port: 80
      targetPort: 8080
EOF
```
**Nhiệm vụ:** Tìm ra **cả 3 lỗi** và sửa sao cho `portal-svc` trả về trang mặc định của Nginx khi gọi `curl http://portal-svc.dbg`.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 3</summary>

**3 lỗi bị cài sẵn:**
1. `readinessProbe` của Deployment gọi sai path (`/healthz-nonexistent` → đổi thành `/`) và sai port (`8080` → đổi thành `80`).
2. `Service.spec.selector` sai (`portal-wrong` → đổi thành `portal`).
3. `Service.spec.ports[0].targetPort` sai (`8080` → đổi thành `80`).

```bash
k patch deploy portal -n dbg --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"},
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":80}
]'
k patch svc portal-svc -n dbg -p '{"spec":{"selector":{"app":"portal"},"ports":[{"port":80,"targetPort":80}]}}'

# === VERIFY ===
k get pods,ep -n dbg
k run curl-chk -n dbg --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://portal-svc.dbg | head -n 5
```
</details>

---

## Kịch bản 4 (Tầng 2): Pod Không Phân Giải Được DNS Trong Cluster
**Mục tiêu thời gian:** 5 phút

**Lệnh Tạo Lỗi:**
```bash
k scale deploy coredns -n kube-system --replicas=0
```
**Nhiệm vụ:** Giả định bạn không biết ai vừa sửa gì — chỉ biết các Pod báo `bad address` khi gọi Service theo tên. Hãy tự lần ra nguyên nhân và khôi phục.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 4</summary>

```bash
# Chẩn đoán
k get pods,ep -n kube-system -l k8s-app=kube-dns    # Không có pod nào, endpoint kube-dns rỗng!
k get deploy coredns -n kube-system                # READY 0/0

# Khôi phục
k scale deploy coredns -n kube-system --replicas=2
k rollout status deploy coredns -n kube-system

# Verify
k run dns-ok --rm -it --restart=Never --image=busybox:1.36 -- nslookup kubernetes.default
```
</details>

---

## Kịch bản 5 (Tầng 3 — Cụm `kubeadm`): Worker Node `NotReady` Do `kubelet` Chết
**Mục tiêu thời gian:** 5 phút

**Lệnh Tạo Lỗi (chạy trên `node01`):**
```bash
ssh node01 "sudo systemctl stop kubelet && sudo systemctl disable kubelet"
```
**Nhiệm vụ:** Sau ~40 giây `k get nodes` báo `node01` là `NotReady`. Hãy chẩn đoán và khôi phục (đảm bảo tự chạy lại sau khi reboot!).

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 5</summary>

```bash
ssh node01
sudo systemctl status kubelet --no-pager      # inactive (dead), disabled
sudo systemctl enable --now kubelet
sudo systemctl status kubelet --no-pager      # active (running), enabled
exit

# Verify
k get nodes                                   # node01: Ready
```
</details>

---

## Kịch bản 6 (Tầng 3 — Cụm `kubeadm`): Worker Node `NotReady` Do Sai `kubelet.conf`
**Mục tiêu thời gian:** 6 phút

**Lệnh Tạo Lỗi (chạy trên `node01`):**
```bash
ssh node01 "sudo cp /etc/kubernetes/kubelet.conf /tmp/kubelet.conf.bak && sudo sed -i 's/:6443/:6444/g' /etc/kubernetes/kubelet.conf && sudo systemctl restart kubelet"
```
**Nhiệm vụ:** `node01` vẫn thấy `kubelet` đang chạy (`active`) hoặc restart liên tục nhưng trạng thái node trên Control Plane vẫn `NotReady`. Tìm ra nguyên nhân bằng `journalctl` và sửa.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 6</summary>

```bash
ssh node01
sudo journalctl -u kubelet -n 30 --no-pager
# Sẽ thấy lỗi rõ ràng: "connection refused" tới port 6444!
sudo grep server: /etc/kubernetes/kubelet.conf
sudo sed -i 's/:6444/:6443/g' /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
exit

# Verify
k get nodes
```
</details>

---

## Kịch bản 7 (Tầng 3 — Cụm `kubeadm`): Worker Node Bị `SchedulingDisabled` & Taint Ẩn
**Mục tiêu thời gian:** 3 phút

**Lệnh Tạo Lỗi:**
```bash
k cordon node01
```
**Nhiệm vụ:** Pod mới tạo không chịu lên `node01`. Kiểm tra và mở lại quyền lập lịch cho `node01`.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 7</summary>

```bash
k get nodes            # STATUS: Ready,SchedulingDisabled
k uncordon node01
k get nodes            # STATUS: Ready
```
</details>

---

## Kịch bản 8 (Tầng 4 — Cụm `kubeadm`): `kube-scheduler` Chết — Mọi Pod Mới Kẹt `Pending` Vĩnh Viễn
**Mục tiêu thời gian:** 6 phút

**Lệnh Tạo Lỗi (chạy trên `controlplane`):**
```bash
sudo cp /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/kube-scheduler.yaml.bak
sudo sed -i 's/kube-scheduler:/kube-schedulerr:/g' /etc/kubernetes/manifests/kube-scheduler.yaml
```
**Nhiệm vụ:** Tạo thử Pod `test-sched` (`nginx:1.27`) → thấy kẹt `Pending` mà **`describe pod` không có event nào**! Tìm và sửa lỗi.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 8</summary>

```bash
k get pods -n kube-system | grep scheduler    # ImagePullBackOff / ErrImagePull!
k describe pod -n kube-system -l component=kube-scheduler
# Phát hiện image gõ nhầm `kube-schedulerr` trong static pod manifest

sudo sed -i 's/kube-schedulerr:/kube-scheduler:/g' /etc/kubernetes/manifests/kube-scheduler.yaml
# Chờ ~15s kubelet tự restart static pod

# Verify
k get pods -n kube-system | grep scheduler    # Running
k get pod test-sched                          # Running
```
</details>

---

## Kịch bản 9 (Tầng 4 — Cụm `kubeadm`): `kube-apiserver` Sập Hoàn Toàn (Sai Port `etcd`)
**Mục tiêu thời gian:** 7 phút

**Lệnh Tạo Lỗi (chạy trên `controlplane`):**
```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak
sudo sed -i 's/2379/2380/g' /etc/kubernetes/manifests/kube-apiserver.yaml
```
**Nhiệm vụ:** Sau ~20 giây, `kubectl get nodes` báo `connection refused`! Dùng công cụ tầng OS để tìm chính xác nguyên nhân và khôi phục Control Plane.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 9</summary>

```bash
# Lúc này kubectl CHẾT, dùng crictl trên controlplane:
sudo crictl ps -a | grep kube-apiserver
# Lấy CONTAINER ID mới nhất của kube-apiserver và xem logs:
sudo crictl logs <container-id> | tail -n 30
# Hoặc: grep -i etcd /etc/kubernetes/manifests/kube-apiserver.yaml
# Thấy --etcd-servers=https://127.0.0.1:2380 (sai! 2380 là peer port, client port phải là 2379)

sudo sed -i 's/2380/2379/g' /etc/kubernetes/manifests/kube-apiserver.yaml
# Chờ 20-30s cho kubelet khởi động lại kube-apiserver

# Verify
sudo crictl ps | grep kube-apiserver
k get nodes
```
</details>

---

## Kịch bản 10 (Tầng 4 — Cụm `kubeadm`): `kube-apiserver` Sập Do Sai Đường Dẫn Certificate
**Mục tiêu thời gian:** 7 phút

**Lệnh Tạo Lỗi (chạy trên `controlplane`):**
```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak
sudo sed -i 's/apiserver-etcd-client.crt/apiserver-etcd-client-wrong.crt/g' /etc/kubernetes/manifests/kube-apiserver.yaml
```
**Nhiệm vụ:** `kubectl` không phản hồi. Chẩn đoán bằng `crictl logs` / `/var/log/pods/` và khôi phục cụm.

<details>
<summary>Xem luồng chẩn đoán & cách sửa Kịch bản 10</summary>

```bash
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs <container-id> | tail -n 20
# Log báo: open /etc/kubernetes/pki/apiserver-etcd-client-wrong.crt: no such file or directory
ls -la /etc/kubernetes/pki/apiserver-etcd-client*

sudo sed -i 's/apiserver-etcd-client-wrong.crt/apiserver-etcd-client.crt/g' /etc/kubernetes/manifests/kube-apiserver.yaml

# Verify
k get nodes
k get pods -n kube-system
```
</details>

