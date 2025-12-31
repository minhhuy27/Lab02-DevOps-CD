# Lab02-DevOps-CD — Istio Service Mesh

Repo này chứa các manifest Istio dùng cho Spring Petclinic (namespace `dev`). Mục tiêu: bật mTLS, giới hạn truy cập giữa các service, cấu hình retry và cung cấp bộ test/evidence.

## Cấu trúc thư mục `istio/`

```text
istio/
├─ mtls/
│  ├─ dev-peerauth.yaml          # PeerAuthentication mTLS STRICT toàn namespace dev
│  ├─ default-mtls.yaml          # DestinationRule wildcard, TLS ISTIO_MUTUAL
│  ├─ discovery-server-dr.yaml   # DestinationRule riêng cho discovery-server
│  └─ gateway-peerauth.yaml      # PeerAuth cho ingress gateway (permissive)
├─ authorization-policy/
│  ├─ customers-allow-gw.yaml    # Chỉ cho phép api-gateway gọi customers-service
│  ├─ vets-allow-gw.yaml         # Chỉ cho phép api-gateway gọi vets-service
│  ├─ visits-allow-gw.yaml       # Chỉ cho phép api-gateway gọi visits-service
│  └─ discovery-allow-all.yaml   # Cho phép gọi discovery-server
├─ virtual-service/
│  ├─ customers-vs.yaml          # Route + retry policy cho customers-service
│  ├─ vets-vs.yaml               # Route + retry policy cho vets-service
│  └─ visits-vs.yaml             # Route + retry policy cho visits-service
└─ test/
   └─ pod-curl-plain.yaml        # Pod test không sidecar (chứng minh mTLS chặn plaintext)

```

## Triển khai nhanh

```bash
# 1. Apply mTLS policies
kubectl apply -f istio/mtls/

# 2. Apply Authorization policies
kubectl apply -f istio/authorization-policy/

# 3. Apply Virtual Services (Retry)
kubectl apply -f istio/virtual-service/

```

## Kiểm thử mTLS
- Pod không sidecar (plaintext):
  ```
  kubectl apply -f istio/test/pod-curl-plain.yaml
  kubectl wait -n dev --for=condition=Ready pod/curl-plain
  kubectl exec -n dev curl-plain -- curl -v http://customers-service:8081/owners
  ```
  Kỳ vọng: reset/403/503 (bị chặn).
- Pod có sidecar (api-gateway):
  ```
  kubectl exec -n dev deploy/api-gateway -- curl -v http://customers-service:8081/owners
  ```
  Kỳ vọng: 200 OK.
- Kiali: mở graph, biểu tượng khóa trên các edge chứng tỏ mTLS bật.


## Kiểm thử Authorization (Zero Trust)

### 1. Test traffic hợp lệ (Allowed)

Chứng minh `api-gateway` có quyền truy cập vào các service con (do có Principal đúng).

```bash
# Từ api-gateway gọi visits-service (Mong đợi: 200 OK / JSON response)
kubectl exec -n dev deploy/api-gateway -- curl -v "http://visits-service:8082/pets/visits?petId=1"

# Từ api-gateway gọi customers-service (Mong đợi: 200 OK)
kubectl exec -n dev deploy/api-gateway -- curl -v http://customers-service:8081/owners

# Từ api-gateway gọi vets-service (Mong đợi: 200 OK)
kubectl exec -n dev deploy/api-gateway -- curl -s -o /dev/null -w "%{http_code}\n" http://vets-service:8083/vets
```

### 2. Test traffic bất hợp pháp (Denied)

Chứng minh các truy cập từ nguồn khác (kể cả trong cùng namespace) đều bị chặn.

* **Từ pod `curl-plain` (Không có Sidecar/mTLS):**
```bash
# Triển khai pod test
kubectl run curltest -n dev --image=curlimages/curl:8.5.0 --restart=Never --command -- sleep 3600
kubectl wait --for=condition=ready pod/curltest -n dev --timeout=180s

# Vào pod test
kubectl exec -n dev -it curltest -- sh

```
```bash
# Chạy lệnh (Mong đợi: Connection Reset / Bị chặn bởi mTLS STRICT)
curl -v http://customers-service.dev.svc.cluster.local:8081/owners        
curl -v "http://visits-service.dev.svc.cluster.local:8082/pets/visits?petId=1"  
curl -v http://vets-service.dev.svc.cluster.local:8083/vets

```


* **Từ service khác gọi chéo (Lateral Movement):**
(Ví dụ: Hacker chiếm quyền `vets-service` và cố gắng truy cập dữ liệu khách hàng)
```bash
# Mong đợi: 403 Forbidden (RBAC Access Denied)
kubectl exec -n dev deploy/vets-service -- curl -v http://customers-service:8081/owners

```



## Kiểm thử Retry (VirtualService)

Ví dụ với `istio/virtual-service/visits-vs.yaml` (Policy: Retry 3 lần nếu gặp lỗi 5xx):

1. **Tạo lỗi 5xx thông qua endpoint lỗi:**
```bash
kubectl exec -n dev deploy/api-gateway -- curl -i "http://visits-service:8082/oops"

```


2. **Xem log để kiểm chứng (Phải thấy log retry 3-4 lần):**
```bash
kubectl logs -f -l app=visits-service -n dev -c istio-proxy

```

## Quan sát Kiali

1. **Port-forward Kiali:**
```bash
kubectl -n istio-system port-forward svc/kiali 20001:20001

```

2. **Truy cập Dashboard:**
* Mở trình duyệt: `http://localhost:20001`
* Chọn namespace: `dev`
* Trong mục **Graph**, chọn **Display** -> Tích vào **Security** để xem biểu tượng ổ khóa (mTLS).


## Chẩn đoán nhanh
- Liệt kê cấu hình: `kubectl get authorizationpolicy,peerauthentication,virtualservice,destinationrule -n dev`
- Kiểm tra selector AP khớp label `app` của pod.
- Đặt tên port HTTP; bỏ/điều chỉnh deny-all nếu chặn discovery không mong muốn.
- Log proxy: `kubectl logs -n dev deploy/api-gateway -c istio-proxy | head`




```

```
