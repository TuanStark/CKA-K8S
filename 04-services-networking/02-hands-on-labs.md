# Module 04: Hands-On Labs — Services & Networking (20%)

> **Hướng dẫn:** Luôn test kết nối thực tế bằng `curl` / `wget` / `nslookup` từ một Pod tạm trong cụm để verify trước khi mở đáp án.

---

## Lab 4.1: Service ClusterIP, NodePort & Kiểm Tra EndpointSlices
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo namespace `net-lab`.
2. Tạo Deployment `web-app` (`nginx:1.27`, `2` replicas, port `80`) trong `net-lab`.
3. Expose `web-app` thành Service `web-svc` dạng `NodePort` với `port: 8080`, `targetPort: 80`, `nodePort: 30088`.
4. Kiểm tra `Endpoints` và `EndpointSlices` của `web-svc`, sau đó test kết nối bằng cả **DNS nội bộ** (`web-svc.net-lab.svc.cluster.local:8080`) và **`<NodeIP>:30088`**.

<details>
<summary>Xem đáp án & lệnh verify Lab 4.1</summary>

```bash
k create ns net-lab
k create deploy web-app -n net-lab --image=nginx:1.27 --replicas=2 --port=80
k expose deploy web-app -n net-lab --name=web-svc --type=NodePort --port=8080 --target-port=80 $do > web-svc.yaml
# Thêm nodePort: 30088 vào ports[0]
sed -i '/targetPort: 80/a \    nodePort: 30088' web-svc.yaml
k apply -f web-svc.yaml

# === VERIFY ===
k get svc,ep,endpointslices -n net-lab
k run tester -n net-lab --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://web-svc.net-lab.svc.cluster.local:8080 | head -n 5
NODE_IP=$(k get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl -s "http://${NODE_IP}:30088" | head -n 5
```
</details>

---

## Lab 4.2: CoreDNS — Kiểm Tra Phân Giải Tên Miền Service & Pod
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
1. Tạo 1 Pod tên `dns-target` (`nginx:1.27`) trong namespace `net-lab`, expose qua Service `dns-svc` cổng `80`.
2. Dùng một Pod tạm (`busybox:1.36`), chạy `nslookup` cho cả **Service FQDN** và **Pod FQDN** (`<ip-dashed>.net-lab.pod.cluster.local`), lưu output ra `/tmp/dns-verify.txt`.

<details>
<summary>Xem đáp án & lệnh verify Lab 4.2</summary>

```bash
k run dns-target -n net-lab --image=nginx:1.27 --port=80 --expose --service-overrides='{"metadata":{"name":"dns-svc"}}'
# Hoặc tách 2 lệnh rõ ràng:
# k run dns-target -n net-lab --image=nginx:1.27 --port=80
# k expose pod dns-target -n net-lab --name=dns-svc --port=80

k wait --for=condition=Ready pod/dns-target -n net-lab --timeout=60s
POD_IP=$(k get pod dns-target -n net-lab -o jsonpath='{.status.podIP}')
POD_DNS="${POD_IP//./-}.net-lab.pod.cluster.local"

{
  k run dns-chk -n net-lab --rm -i --restart=Never --image=busybox:1.36 -- nslookup dns-svc.net-lab.svc.cluster.local
  k run dns-chk2 -n net-lab --rm -i --restart=Never --image=busybox:1.36 -- nslookup "${POD_DNS}"
} > /tmp/dns-verify.txt

cat /tmp/dns-verify.txt
```
</details>

---

## Lab 4.3: NetworkPolicy — Default Deny & Bẫy AND vs OR
**Mục tiêu thời gian:** 7 phút

**Đề bài:**
Chuẩn bị môi trường:
- Namespace `prod` (label `env=prod`) chứa Pod `db` (`nginx:1.27`, label `role=db`, port `80`) và Service `db-svc`.
- Namespace `app` (label `env=app`) chứa 2 Pod: `api` (`busybox:1.36`, label `role=api`) và `rogue` (`busybox:1.36`, label `role=rogue`).
- Namespace `default` chứa Pod `fake-api` (`busybox:1.36`, label `role=api`).

**Yêu cầu:**
Tạo NetworkPolicy `allow-only-app-api` trong namespace `prod` áp lên Pod `role=db`:
- Chặn tất cả Ingress mặc định.
- **CHỈ** cho phép traffic TCP cổng `80` đến từ các Pod có label `role=api` **VÀ** nằm trong namespace có label `env=app`.
- **Verify:** `api` (trong `app`) gọi `db-svc.prod` thành công; cả `rogue` (trong `app`) và `fake-api` (trong `default`) đều phải **timeout**! *(Lưu ý: cần cụm có CNI hỗ trợ NetworkPolicy như Calico/Cilium.)*

<details>
<summary>Xem đáp án & lệnh verify Lab 4.3</summary>

```bash
k create ns prod && k label ns prod env=prod
k create ns app && k label ns app env=app

k run db -n prod --image=nginx:1.27 --labels="role=db" --port=80
k expose pod db -n prod --name=db-svc --port=80

k run api -n app --image=busybox:1.36 --labels="role=api" -- sleep 3600
k run rogue -n app --image=busybox:1.36 --labels="role=rogue" -- sleep 3600
k run fake-api -n default --image=busybox:1.36 --labels="role=api" -- sleep 3600

cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-app-api
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: app
          podSelector:
            matchLabels:
              role: api
      ports:
        - protocol: TCP
          port: 80
EOF

# === VERIFY ===
k exec api -n app -- wget -qO- --timeout=2 http://db-svc.prod          # THÀNH CÔNG
k exec rogue -n app -- wget -qO- --timeout=2 http://db-svc.prod        # TIMEOUT (bị chặn)
k exec fake-api -n default -- wget -qO- --timeout=2 http://db-svc.prod # TIMEOUT (bị chặn)
```
</details>

---

## Lab 4.4: NetworkPolicy — Egress Policy Kèm Mở Cổng DNS (53)
**Mục tiêu thời gian:** 6 phút

**Đề bài:**
Trong namespace `app` (từ Lab 4.3), tạo NetworkPolicy `api-egress-only-db` áp lên Pod `role=api`:
- Chỉ cho phép `api` gửi Egress traffic tới:
  1. Pod `role=db` trong namespace `env=prod` ở cổng TCP `80`.
  2. CoreDNS (UDP và TCP cổng `53`) để phân giải được tên miền `db-svc.prod`.
- Mọi kết nối Egress khác (ví dụ gọi ra `web-svc.net-lab`) phải bị chặn!

<details>
<summary>Xem đáp án & lệnh verify Lab 4.4</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-only-db
  namespace: app
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              env: prod
          podSelector:
            matchLabels:
              role: db
      ports:
        - protocol: TCP
          port: 80
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF

# === VERIFY ===
k exec api -n app -- wget -qO- --timeout=2 http://db-svc.prod             # THÀNH CÔNG
k exec api -n app -- wget -qO- --timeout=2 http://web-svc.net-lab:8080    # TIMEOUT (bị chặn)
```
</details>

---

## Lab 4.5: Ingress Với Nhiều Path & TLS Secret
**Mục tiêu thời gian:** 5 phút

**Đề bài:**
Trong namespace `net-lab`:
1. Tạo 1 self-signed TLS cert (`openssl req -x509...`) cho host `shop.example.com` và lưu vào TLS Secret tên `shop-tls`.
2. Tạo Ingress tên `shop-ingress` dùng `ingressClassName: nginx`:
   - Host: `shop.example.com`, bật TLS dùng secret `shop-tls`.
   - Path `/web` (`Prefix`) → Service `web-svc` cổng `8080`.
   - Path `/dns` (`Exact`) → Service `dns-svc` cổng `80`.

<details>
<summary>Xem đáp án & lệnh verify Lab 4.5</summary>

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/shop.key -out /tmp/shop.crt -subj "/CN=shop.example.com"
k create secret tls shop-tls -n net-lab --cert=/tmp/shop.crt --key=/tmp/shop.key

cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: net-lab
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 8080
          - path: /dns
            pathType: Exact
            backend:
              service:
                name: dns-svc
                port:
                  number: 80
EOF

# === VERIFY ===
k describe ingress shop-ingress -n net-lab
```
</details>

---

## Lab 4.6: Gateway API — Cài CRD, Tạo `Gateway` HTTPS & `HTTPRoute` (Trọng Tâm Mới!)
**Mục tiêu thời gian:** 7 phút

*(Lưu ý trên cụm lab tự dựng: nếu cụm chưa có Gateway API CRDs, cài nhanh bằng lệnh: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml`)*

**Đề bài:**
Trong namespace `net-lab`, thực hành **Migrate Ingress `shop-ingress` (Lab 4.5) sang Gateway API**:
1. Giả sử `GatewayClass` tên `nginx` đã tồn tại (tạo nhanh 1 `GatewayClass` tên `nginx` nếu cụm lab chưa có).
2. Tạo `Gateway` tên `shop-gateway` sử dụng `gatewayClassName: nginx`, cấu hình listener `https`:
   - `protocol: HTTPS`, `port: 443`, `hostname: "shop.example.com"`
   - `tls.mode: Terminate` trỏ tới Secret `shop-tls`.
3. Tạo `HTTPRoute` tên `shop-route` gắn với `parentRefs: shop-gateway`, `hostnames: ["shop.example.com"]`:
   - Route 1: `PathPrefix` `/web` → `web-svc:8080`.
   - Route 2: `Exact` `/dns` → `dns-svc:80`.

<details>
<summary>Xem đáp án & lệnh verify Lab 4.6</summary>

```bash
# Cài Gateway API CRDs (chỉ cần chạy 1 lần trên cụm lab tự dựng)
# k apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

cat <<'EOF' | k apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: example.net/nginx-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shop-gateway
  namespace: net-lab
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "shop.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: shop-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
  namespace: net-lab
spec:
  parentRefs:
    - name: shop-gateway
  hostnames:
    - "shop.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /web
      backendRefs:
        - name: web-svc
          port: 8080
    - matches:
        - path:
            type: Exact
            value: /dns
      backendRefs:
        - name: dns-svc
          port: 80
EOF

# === VERIFY ===
k get gatewayclass,gateway,httproute -n net-lab
k describe httproute shop-route -n net-lab
```
</details>

---

## Lab 4.7: Gateway API — Canary Traffic Splitting (80% / 20%) & Header Matching
**Mục tiêu thời gian:** 6 phút

**Đề bài:**
Trong namespace `net-lab`, tạo `HTTPRoute` tên `canary-route` trỏ tới `shop-gateway` cho hostname `api.example.com`:
- Rule 1: Request có HTTP header `X-Canary: true` → đẩy **100%** sang `dns-svc:80`.
- Rule 2 (mặc định `PathPrefix /`): Chia traffic **80%** sang `web-svc:8080` và **20%** sang `dns-svc:80`.

<details>
<summary>Xem đáp án & lệnh verify Lab 4.7</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
  namespace: net-lab
spec:
  parentRefs:
    - name: shop-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - headers:
            - type: Exact
              name: X-Canary
              value: "true"
      backendRefs:
        - name: dns-svc
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web-svc
          port: 8080
          weight: 80
        - name: dns-svc
          port: 80
          weight: 20
EOF

# === VERIFY ===
k get httproute canary-route -n net-lab -o yaml
```
</details>

---

## Lab 4.8: Headless Service & StatefulSet DNS
**Mục tiêu thời gian:** 4 phút

**Đề bài:**
Trong namespace `net-lab`, tạo một Headless Service `db-headless` (`clusterIP: None`, selector `app=web-app`, port `80`). Dùng `nslookup db-headless.net-lab.svc.cluster.local` để quan sát sự khác biệt so với `ClusterIP` thường (trả về trực tiếp IP của cả 2 Pod thay vì 1 Virtual IP!).

<details>
<summary>Xem đáp án & lệnh verify Lab 4.8</summary>

```bash
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Service
metadata:
  name: db-headless
  namespace: net-lab
spec:
  clusterIP: None
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
EOF

# === VERIFY ===
k run hl-chk -n net-lab --rm -i --restart=Never --image=busybox:1.36 -- nslookup db-headless.net-lab.svc.cluster.local
# Kết quả: Trả về 2 dòng Address là IP thật của 2 pod thuộc deploy web-app!
```
</details>

