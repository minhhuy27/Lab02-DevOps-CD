# Lab02-DevOps-CD — Istio Service Mesh

Repo này chứa các manifest Istio dùng cho Spring Petclinic (namespace `dev`). Mục tiêu: bật mTLS, giới hạn truy cập giữa các service, cấu hình retry và cung cấp bộ test/evidence.

## Cấu trúc thư mục `istio/`
```
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
```
kubectl apply -n dev -f istio/mtls
kubectl apply -n dev -f istio/authorization-policy
kubectl apply -n dev -f istio/virtual-service
# (tuỳ chọn) kubectl apply -n dev -f istio/test/pod-curl-plain.yaml
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

## Kiểm thử AuthorizationPolicy

- Từ api-gateway: curl tới customers/visits/vets → 200.
    # Customers
      ```
    kubectl exec -n dev deploy/api-gateway -- curl -s -o /dev/null -w "%{http_code}\n" http://customers-service:8081/owners
      ```
    # Visits 
      ```
    kubectl exec -n dev deploy/api-gateway -- curl -i "http://visits-service:8082/pets/visits?petId=1" 
      ```
    # Vets
      ```
    kubectl exec -n dev deploy/api-gateway -- curl -s -o /dev/null -w "%{http_code}\n" http://vets-service:8083/vets
      ```

- Từ pod không được phép (curl-plain): curl tới customers/visits/vets → bị chặn.
    # Tạo pod test
      ```
    kubectl run curltest -n dev --image=curlimages/curl:8.5.0 --restart=Never --command -- sleep 3600
    kubectl wait --for=condition=ready pod/curltest -n dev --timeout=180s
      ```
    # Vào pod để test
      ```
    kubectl exec -n dev -it curltest -- sh
      ```
    # ví dụ:
      ```
    curl -v http://customers-service.dev.svc.cluster.local:8081/owners        
    curl -v "http://visits-service.dev.svc.cluster.local:8082/pets/visits?petId=1"  
    curl -v http://vets-service.dev.svc.cluster.local:8083/vets
      ```
- Curl giữa các microservice không được phép: (từ vets-service đến customers-service)

      ```                
    kubectl exec -n dev deploy/vets-service -- curl -v http://customers-service:8081/owners
      ```


## Kiểm thử Retry (VirtualService)
Ví dụ `istio/virtual-service/visits-vs.yaml`:
- Tạo lỗi 5xx thông qua endpoint lỗi:
      ```
    kubectl exec -n dev deploy/api-gateway -- curl -i "http://visits-service:8082/oops"
      ```
- Xem log:
      ```
    kubectl logs -f -l app=visits-service -n dev -c istio-proxy
      ```
## Quan sát Kiali
- Port-forward Kiali: `kubectl -n istio-system port-forward svc/kiali 20001:20001`, mở http://localhost:20001, chọn namespace `dev`.
- Replay last 1m để xem traffic, khóa mTLS và tỷ lệ lỗi.

## Chẩn đoán nhanh
- Liệt kê cấu hình: `kubectl get authorizationpolicy,peerauthentication,virtualservice,destinationrule -n dev`
- Kiểm tra selector AP khớp label `app` của pod.
- Đặt tên port HTTP; bỏ/điều chỉnh deny-all nếu chặn discovery không mong muốn.
- Log proxy: `kubectl logs -n dev deploy/api-gateway -c istio-proxy | head`
