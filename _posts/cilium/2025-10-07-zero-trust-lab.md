---
title: Cilium Zero-Trust Lab
date: 2025-10-07 18:00:00 +0900
categories: [Cilium]
tags: [Cilium]
---

> **서문**
> 본 문서는 Cloudbro 오픈 프로젝트 **“Cilium 기반 멀티클러스터 Zero-Trust & Observability 아키텍처 설계”**에 직접 참여하여 수행한 설계·검증 결과를 바탕으로 재구성한 실습 가이드입니다. 전체 아키텍처, 소스 코드, GKE 멀티클러스터 구축 가이드, 본 랩의 워크로드·매니페스트는 레포지토리에서 확인하실 수 있습니다: [cloudbro-draupnir/draupnir](https://github.com/cloudbro-draupnir/draupnir)
> 
> **진행 순서 안내**
> 1. 먼저 상기 레포지토리의 **README**를 확인하시기 바랍니다. 
> 2. 안내에 따라 **GKE 멀티클러스터를 선 구축**하십시오. 
> 3. 레포지토리를 **클론 후 워크로드·매니페스트**를 준비하십시오. 
> 4. 준비가 완료되면 **본 가이드의 실습 절차를 순차적으로 진행**하십시오.

## **🧩 Preparation**

### **1. Hubble / Hubble UI**

```bash
# Hubble Relay 포트포워딩
cilium hubble port-forward -n kube-system
```

```bash
kubectl get svc -n kube-system --context $CLUSTER1 | grep hubble

# 예시 출력
hubble-metrics                                       ClusterIP      None             <none>           9965/TCP                       42h
hubble-peer                                          ClusterIP      34.118.233.33    <none>           443/TCP                        44h
hubble-relay                                         ClusterIP      34.118.231.13    <none>           80/TCP                         43h
hubble-ui                                            LoadBalancer   34.118.233.250   104.198.125.51   80:32346/TCP                   43h

# Hubble UI 접속
# http://104.198.125.51
```

### **2. Grafana 연결**

```bash
# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --context $CLUSTER1

# Grafana 접속
# http://localhost:3000
# Username: admin
# Password: admin123
```

### **3. 워크로드 배포 (ns: shop)**

```bash
kubectl create ns shop --context $CLUSTER1
kubectl create ns shop --context $CLUSTER2
```

```yaml
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: shop
spec:
  replicas: 1
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
      - name: backend
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: shop
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: shop
spec:
  replicas: 1
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      containers:
      - name: frontend
        image: nicolaka/netshoot
        command: ["tail"]
        args: ["-f", "/dev/null"]
```

```bash
# 배포
kubectl apply -f backend.yaml  --context $CLUSTER1
kubectl apply -f backend.yaml  --context $CLUSTER2
kubectl apply -f frontend.yaml --context $CLUSTER1
kubectl apply -f frontend.yaml --context $CLUSTER2

# 롤아웃 확인
kubectl rollout status deploy/backend  -n shop --context $CLUSTER1
kubectl rollout status deploy/frontend -n shop --context $CLUSTER1
kubectl rollout status deploy/backend  -n shop --context $CLUSTER2
kubectl rollout status deploy/frontend -n shop --context $CLUSTER2
```

---

## **🟢 Stage 1: Allow-All (베이스라인 가시화)**

### **1. 정책 적용**

```yaml
# allow-all.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-all
  namespace: shop
spec:
  endpointSelector: {}
  ingress:
    - fromEntities: [all]
  egress:
    - toEntities: [all]
```

```bash
kubectl apply -f allow-all.yaml --context $CLUSTER1
kubectl apply -f allow-all.yaml --context $CLUSTER2
```

### **2. FE→BE HTTP 확인**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS http://backend.shop.svc.cluster.local/get | jq -r .url; echo
  
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  curl -sS http://backend.shop.svc.cluster.local/get | jq -r .url; echo

# 결과
http://backend.shop.svc.cluster.local/get
http://backend.shop.svc.cluster.local/get
```

- FE → BE **기본 통신** 허용 상태 확인

### **3. Hubble로 Allow-All 매칭 확인**

**(1) 정책 판정 흐름 전체**

```bash
hubble observe -n shop --type policy-verdict --follow
```

✅ **출력**

```bash
Oct  2 14:06:19.406: shop/frontend-79f5dbfc6c-d6gb7:60788 (ID:114060) -> kube-system/kube-dns-75df64b86-xrfwz:53 (ID:67025) policy-verdict:all EGRESS ALLOWED (UDP)
Oct  2 14:06:19.464: shop/frontend-79f5dbfc6c-d6gb7:55544 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:all EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:06:19.464: shop/frontend-79f5dbfc6c-d6gb7:55544 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:all INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:06:28.561: shop/frontend-79f5dbfc6c-r56dx:53700 (ID:135917) -> kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) policy-verdict:all EGRESS ALLOWED (UDP)
Oct  2 14:06:28.576: shop/frontend-79f5dbfc6c-r56dx:35010 (ID:135917) -> shop/backend-6c4c67dc75-mqm9j:80 (ID:163287) policy-verdict:all EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:06:28.576: shop/frontend-79f5dbfc6c-r56dx:35010 (ID:135917) -> shop/backend-6c4c67dc75-mqm9j:80 (ID:163287) policy-verdict:all INGRESS ALLOWED (TCP Flags: SYN)
```

- DNS(53/UDP) 및 FE→BE(80/TCP) 트래픽 허용 관찰

**(2) JSON으로 허용 정책명 확인**

```bash
hubble observe -n shop --type policy-verdict --from-label app=frontend --to-label app=backend --last 1 -o jsonpb \
  | jq '.flow | {verdict:.verdict,ing:(.ingress_allowed_by//[]|map(.name)),eg:(.egress_allowed_by//[]|map(.name))}'
```

✅ **출력**

```bash
{
  "verdict": "FORWARDED",
  "ing": [
    "allow-all"
  ],
  "eg": []
}
{
  "verdict": "FORWARDED",
  "ing": [
    "allow-all"
  ],
  "eg": []
}
```

- **allow-all**이 **수신(ingress)** 측에서 실제 허용자로 매칭됨

### **4. 트래픽 캡처**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 60); do curl -sS --http1.1 -o /dev/null http://backend.shop.svc.cluster.local/anything/ok; sleep 0.1; done'
```

**(1) Hubble UI**

- `frontend → backend:80/TCP`가 연속 **ALLOWED/FORWARDED**

![](https://velog.velcdn.com/images/tlsalswls123/post/edac3dc3-7daf-4ae1-aa9a-9cbfcf7c2754/image.png)

**(2) Grafana**

- **Flow Types는 증가**, **Forwarded vs Dropped는 Forwarded 우세**로 나타남
- **TCPv4 패널의 SYN/SYN-ACK 스파이크**와 **Top 10 Port에서 80/TCP 상위**를 통해 전면 허용 베이스라인 트래픽임을 확인

![](https://velog.velcdn.com/images/tlsalswls123/post/dc8d2368-3d29-4c50-ade1-c9a1933d90f1/image.png)

---

## **🔵 Stage 2: DNS 정책 분리 (kube-dns:53)**

### **1. 정책 적용**

```yaml
# allow-dns.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-dns
  namespace: shop
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
          rules:
            dns:
              - matchPattern: "*" 
```

```bash
kubectl apply -f allow-dns.yaml --context $CLUSTER1
kubectl apply -f allow-dns.yaml --context $CLUSTER2
```

### **2. DNS 정상 조회 확인**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  nslookup backend.shop.svc.cluster.local

kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  nslookup backend.shop.svc.cluster.local
```

✅ **출력**

```bash
# Cluster1
Name:	backend.shop.svc.cluster.local
Address: 34.118.239.118

# Cluster2
Name:	backend.shop.svc.cluster.local
Address: 34.118.228.223
```

- 두 클러스터 모두 **DNS 질의 성공**

### **3. Hubble로 DNS 허용 확인 (프로토콜 뷰)**

```bash
hubble observe -n shop --protocol DNS --last 10
```

✅ **출력**

```bash
# Cluster1
Oct  2 14:14:50.446: shop/frontend-79f5dbfc6c-d6gb7:56300 (ID:114060) -> kube-system/kube-dns-75df64b86-wjls9:53 (ID:67025) dns-request proxy FORWARDED (DNS Query backend.shop.svc.cluster.local. A)
Oct  2 14:14:50.453: shop/frontend-79f5dbfc6c-d6gb7:56300 (ID:114060) <- kube-system/kube-dns-75df64b86-wjls9:53 (ID:67025) dns-response proxy FORWARDED (DNS Answer "34.118.239.118" TTL: 30 (Proxy backend.shop.svc.cluster.local. A))
Oct  2 14:14:50.458: shop/frontend-79f5dbfc6c-d6gb7:53175 (ID:114060) -> kube-system/kube-dns-75df64b86-wjls9:53 (ID:67025) dns-request proxy FORWARDED (DNS Query backend.shop.svc.cluster.local. AAAA)
Oct  2 14:14:50.459: shop/frontend-79f5dbfc6c-d6gb7:53175 (ID:114060) <- kube-system/kube-dns-75df64b86-wjls9:53 (ID:67025) dns-response proxy FORWARDED (DNS Answer  TTL: 4294967295 (Proxy backend.shop.svc.cluster.local. AAAA))

# Cluster2
Oct  2 14:15:18.524: shop/frontend-79f5dbfc6c-r56dx:38458 (ID:135917) -> kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) dns-request proxy FORWARDED (DNS Query backend.shop.svc.cluster.local. A)
Oct  2 14:15:18.528: shop/frontend-79f5dbfc6c-r56dx:38458 (ID:135917) <- kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) dns-response proxy FORWARDED (DNS Answer "34.118.228.223" TTL: 30 (Proxy backend.shop.svc.cluster.local. A))
Oct  2 14:15:18.531: shop/frontend-79f5dbfc6c-r56dx:42314 (ID:135917) -> kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) dns-request proxy FORWARDED (DNS Query backend.shop.svc.cluster.local. AAAA)
Oct  2 14:15:18.532: shop/frontend-79f5dbfc6c-r56dx:42314 (ID:135917) <- kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) dns-response proxy FORWARDED (DNS Answer  TTL: 4294967295 (Proxy backend.shop.svc.cluster.local. AAAA))
```

- **프록시 경유(FORWARDED)**로 DNS L7 룰 동작 확인

### **4. Hubble policy-verdict (to port 53)**

```bash
hubble observe -n shop --type policy-verdict --to-port 53 --last 5
```

✅ **출력**

```bash
# Cluster1
Oct  2 14:14:50.458: shop/frontend-79f5dbfc6c-d6gb7:53175 (ID:114060) -> kube-system/kube-dns-75df64b86-wjls9:53 (ID:67025) policy-verdict:L3-L4 EGRESS ALLOWED (UDP)

# Cluster2
Oct  2 14:15:18.531: shop/frontend-79f5dbfc6c-r56dx:42314 (ID:135917) -> kube-system/kube-dns-55899b7fc-2rdqk:53 (ID:156453) policy-verdict:L3-L4 EGRESS ALLOWED (UDP)
```

- **53/UDP EGRESS 허용**이 L3/L4에서 적용됨

### **5. 트래픽 캡처 (Grafana)**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 40); do nslookup backend.shop.svc.cluster.local >/dev/null 2>&1; sleep 0.15; done'
```

- **Flow Types는 증가**, **Forwarded vs Dropped는 Forwarded 우세**로 나타남
- **L7 Flow Distribution에서 DNS가 뚜렷하게 상승하고 HTTP도 동반 증가**

![](https://velog.velcdn.com/images/tlsalswls123/post/88e6da59-873c-4123-bc83-4d2f85c873ef/image.png)

---

## **🟠 Stage 3: L4 최소 허용 (Ingress + Egress 쌍)**

### **1. 정책 적용**

```yaml
# allow-frontend-to-backend.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop
spec:
  endpointSelector:
    matchLabels: { app: backend }
  ingress:
    - fromEndpoints:
        - matchLabels: { app: frontend }
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
```

```yaml
# allow-frontend-egress.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-egress
  namespace: shop
spec:
  endpointSelector:
    matchLabels: { app: frontend }
  egress:
    - toEndpoints:
        - matchLabels: { app: backend }
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
```

```bash
kubectl apply -f allow-frontend-to-backend.yaml --context $CLUSTER1
kubectl apply -f allow-frontend-egress.yaml --context $CLUSTER1
kubectl apply -f allow-frontend-to-backend.yaml --context $CLUSTER2
kubectl apply -f allow-frontend-egress.yaml --context $CLUSTER2
```

### **2. FE→BE HTTP 200 확인**

```bash
# Cluster1
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS http://backend.shop.svc.cluster.local/anything/ok -w "\nHTTP %{http_code}\n"

# Cluster2
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  curl -sS http://backend.shop.svc.cluster.local/anything/ok -w "\nHTTP %{http_code}\n"
```

✅ **출력**

```bash
...
"url": "http://backend.shop.svc.cluster.local/anything/ok"
HTTP 200

...
"url": "http://backend.shop.svc.cluster.local/anything/ok"
HTTP 200
```

- 두 클러스터 모두 **`frontend → backend:80`** 통신 성공

### **3. 외부 egress 동작 상태 점검(베이스라인 유지)**

```bash
# Cluster1
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  sh -c 'curl -sS -m 3 https://example.com || echo BLOCKED'
  
# Cluster1
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  sh -c 'curl -sS -m 3 https://example.com || echo BLOCKED'  
```

✅ **출력(공통)**

```bash
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
...    
```

- 아직 **Allow-All**이 존재하므로 인터넷 egress는 **허용 상태**

### **4. Hubble: L4 허용 흐름 확인**

```bash
hubble observe -n shop --verdict FORWARDED --from-label app=frontend --to-label app=backend --follow
```

✅ **출력**

```bash
# Cluster1
Oct  2 14:23:07.053: shop/frontend-79f5dbfc6c-d6gb7:54130 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:23:07.053: shop/frontend-79f5dbfc6c-d6gb7:54130 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:23:07.053: shop/frontend-79f5dbfc6c-d6gb7:54130 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) to-endpoint FORWARDED (TCP Flags: SYN)
...

# Cluster2
Oct  2 14:23:18.701: shop/frontend-79f5dbfc6c-r56dx:52560 (ID:135917) -> shop/backend-6c4c67dc75-mqm9j:80 (ID:163287) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:23:18.701: shop/frontend-79f5dbfc6c-r56dx:52560 (ID:135917) -> shop/backend-6c4c67dc75-mqm9j:80 (ID:163287) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:23:18.701: shop/frontend-79f5dbfc6c-r56dx:52560 (ID:135917) -> shop/backend-6c4c67dc75-mqm9j:80 (ID:163287) to-endpoint FORWARDED (TCP Flags: SYN)
...
```

- **L3/L4 EGRESS/INGRESS ALLOWED**로 연속 관찰 → **L4 최소 허용 정책 유효**

### **5. 트래픽 캡처**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 80); do curl -sS --http1.1 -o /dev/null http://backend.shop.svc.cluster.local/anything/ok; sleep 0.1; done'
```

**(1) Hubble UI**

- `policy-verdict:L3-L4`로 **FE→BE:80 허용**
- **egress/ingress 쌍** 모두 확인(L7 정보 없음)

![](https://velog.velcdn.com/images/tlsalswls123/post/1e8ad11d-1c84-4bef-bcdf-c974817b2485/image.png)

**(2) Grafana**

- **Forwarded 우세, Flow Types 증가**
- **L7은 HTTP 상승**

![](https://velog.velcdn.com/images/tlsalswls123/post/30959a8a-cbc5-4308-9230-ea2b99b46586/image.png)

---

## **🟣 Stage 4: L7 필터링**

### **1. 정책 적용** (`GET /status/200` 허용)

```yaml
# allow-frontend-l7.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-l7
  namespace: shop
spec:
  endpointSelector:
    matchLabels: { app: backend }
  ingress:
    - fromEndpoints:
        - matchLabels: { app: frontend }
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "^/status/200$"
```

```bash
kubectl apply -f allow-frontend-l7.yaml --context $CLUSTER1
kubectl apply -f allow-frontend-l7.yaml --context $CLUSTER2
```

### **2. L4 Ingress 삭제 (우회 방지)**

```bash
kubectl delete cnp allow-frontend-to-backend -n shop --context $CLUSTER1 || true
kubectl delete cnp allow-frontend-to-backend -n shop --context $CLUSTER2 || true
```

### **3. 허용/비허용 경로 테스트**

```bash
# 허용 경로
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://backend.shop.svc.cluster.local/status/200
  
# 결과
HTTP 200
```

- `GET /status/200` **허용** 확인

```bash
# 비허용 경로
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://backend.shop.svc.cluster.local/status/500
  
# 결과
HTTP 500   # httpbin 응답 (Allow-All이 살아있어 통과)
```

- 현재는 **Allow-All**이 남아 비허용 경로도 통과(→ Stage 5에서 `403` 기대)

### **4. Hubble로 L7 허용자 확인**

**(1) HTTP 레벨 관찰**

```bash
hubble observe -n shop --protocol http \
  --from-label app=frontend --to-label app=backend --last 10
```

✅ **출력**

```bash
Oct  2 14:32:42.874: shop/frontend-79f5dbfc6c-d6gb7:55886 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) http-request FORWARDED (HTTP/1.1 GET http://backend.shop.svc.cluster.local/status/200)
Oct  2 14:32:51.156: shop/frontend-79f5dbfc6c-d6gb7:55894 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) http-request FORWARDED (HTTP/1.1 GET http://backend.shop.svc.cluster.local/status/500)
```

- **HTTP L7 이벤트**가 보이며, `/status/200` 요청이 프록시 경유로 **포워딩**됨을 확인

**(2) 정책 판정자 확인**

```bash
hubble observe -n shop --type policy-verdict \
  --from-label app=frontend --to-label app=backend --last 10 -o jsonpb \
| jq -r 'select(.flow) | "verdict=" + .flow.verdict + " ing=" + ((.flow.ingress_allowed_by//[]|map(.name))|join(","))'
```

✅ **출력**

```bash
verdict=REDIRECTED ing=allow-frontend-l7
verdict=FORWARDED ing=
```

- **L7 프록시 리다이렉트(`ing=allow-frontend-l7`) → 포워드** 흐름 확인

### **5. 트래픽 캡처**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 40); do curl -sS -o /dev/null http://backend.shop.svc.cluster.local/status/200; sleep 0.15; done'
```

**(1) Hubble UI**

- **L7 Info: `GET /status/200`** 반복 기록
- **Verdict: `redirected → forwarded`** 순으로 연속 표기 → **L7 HTTP 정책 적용 확인**

![](https://velog.velcdn.com/images/tlsalswls123/post/5eb66a9c-09e7-4d6f-aa72-e8743fc71c86/image.png)

**(2) Grafana**

- **L7 Flow Distribution에서 HTTP 상승**
- **Forwarded vs Redirected에 Redirected 라인 확인** → **L7 프록시 경유 허용**

![](https://velog.velcdn.com/images/tlsalswls123/post/6e16960f-a896-4d90-9e9c-7a7141e14cac/image.png)

---

## **⚫ Stage 5: Allow-All 제거 + Default-Deny 명시 (최종 제로트러스트)**

### **1. 정책 적용**

```yaml
# default-deny.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: shop
spec:
  endpointSelector: {}
  ingress: []
  egress: []
```

```bash
kubectl delete -f allow-all.yaml --context $CLUSTER1
kubectl delete -f allow-all.yaml --context $CLUSTER2
kubectl apply -f default-deny.yaml --context $CLUSTER1
kubectl apply -f default-deny.yaml --context $CLUSTER2
```

- `shop` 네임스페이스 기본 통신 **전면 차단**
- **명시 허용**(`allow-dns`, `allow-frontend-egress`, `allow-frontend-l7`)만 통과

### **2. 기능 테스트**

**(1) DNS 유지**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  nslookup backend.shop.svc.cluster.local

kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  nslookup backend.shop.svc.cluster.local
```

✅ **출력**

```bash
# Cluster1
Name:	backend.shop.svc.cluster.local
Address: 34.118.239.118

# Cluster2
Name:	backend.shop.svc.cluster.local
Address: 34.118.228.223
```

- 두 클러스터 모두 정상 응답 → `allow-dns` 유효

**(2) L7 허용 경로(`/status/200`) 통과**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" \
  http://backend.shop.svc.cluster.local/status/200
  
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" \
  http://backend.shop.svc.cluster.local/status/200  
```

✅ **출력**

```bash
HTTP 200
```

- `allow-frontend-l7`에 의해 `GET /status/200` 허용

**(3) L7 비허용 경로 차단**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" \
  http://backend.shop.svc.cluster.local/get
  
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" \
  http://backend.shop.svc.cluster.local/get
```

✅ **출력**

```bash
HTTP 403
```

- 비허용 경로 차단 확인

**(4) 외부 egress 차단**

```bash
kubectl exec -it deploy/frontend -n shop --context $CLUSTER1 -- \
  sh -c 'curl -sS --max-time 3 https://example.com || echo BLOCKED'
  
kubectl exec -it deploy/frontend -n shop --context $CLUSTER2 -- \
  sh -c 'curl -sS --max-time 3 https://example.com || echo BLOCKED'  
```

✅ **출력**

```bash
curl: (28) Connection timed out after 3002 milliseconds
BLOCKED
```

- `allow-frontend-egress`는 **BE:80만 허용**

### **3. Hubble로 최종 검증**

**(1) 트래픽 생성**

```bash
kubectl exec deploy/frontend -n shop --context $CLUSTER1 -- \
  sh -lc 'for i in $(seq 1 10); do \
    printf "%02d " "$i"; \
    curl -sS --http1.1 --max-time 3 -o /dev/null -w "HTTP %{http_code}\n" \
      http://backend.shop.svc.cluster.local/status/200 || echo "BLOCKED"; \
    sleep 0.3; done'
```

✅ **출력**

```bash
01 HTTP 200
02 HTTP 200
03 HTTP 200
04 HTTP 200
05 HTTP 200
06 HTTP 200
07 HTTP 200
08 HTTP 200
09 HTTP 200
10 HTTP 200
```

**(2) 허용자 확인 - EGRESS (FE→BE)**

```bash
hubble observe -n shop --since 2m --type policy-verdict \
  --traffic-direction egress \
  --from-label app=frontend --to-label app=backend
```

✅ **출력**

```bash
Oct  2 14:47:43.789: shop/frontend-79f5dbfc6c-d6gb7:41728 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.132: shop/frontend-79f5dbfc6c-d6gb7:41736 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.491: shop/frontend-79f5dbfc6c-d6gb7:41748 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.826: shop/frontend-79f5dbfc6c-d6gb7:41762 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.160: shop/frontend-79f5dbfc6c-d6gb7:41768 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.495: shop/frontend-79f5dbfc6c-d6gb7:41782 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.835: shop/frontend-79f5dbfc6c-d6gb7:41792 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.175: shop/frontend-79f5dbfc6c-d6gb7:41798 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.513: shop/frontend-79f5dbfc6c-d6gb7:41806 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.849: shop/frontend-79f5dbfc6c-d6gb7:41818 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 EGRESS ALLOWED (TCP Flags: SYN)
```

- 연속 `L3/L4 EGRESS ALLOWED`(SYN) → **허용자: `allow-frontend-egress`**

**(3) 허용자 확인 - INGRESS (FE→BE)**

```bash
hubble observe -n shop --since 2m --type policy-verdict \
  --traffic-direction ingress \
  --from-label app=frontend --to-label app=backend
```

✅ **출력**

```bash
Oct  2 14:47:43.789: shop/frontend-79f5dbfc6c-d6gb7:41728 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.132: shop/frontend-79f5dbfc6c-d6gb7:41736 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.491: shop/frontend-79f5dbfc6c-d6gb7:41748 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:44.826: shop/frontend-79f5dbfc6c-d6gb7:41762 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.160: shop/frontend-79f5dbfc6c-d6gb7:41768 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.495: shop/frontend-79f5dbfc6c-d6gb7:41782 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:45.835: shop/frontend-79f5dbfc6c-d6gb7:41792 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.175: shop/frontend-79f5dbfc6c-d6gb7:41798 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.513: shop/frontend-79f5dbfc6c-d6gb7:41806 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Oct  2 14:47:46.849: shop/frontend-79f5dbfc6c-d6gb7:41818 (ID:114060) -> shop/backend-6c4c67dc75-g22jn:80 (ID:69364) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
```

- 연속 `L3/L4 INGRESS ALLOWED`(SYN) → **허용자: `allow-frontend-l7`** (L7 리다이렉트 후 포워딩 로그)

### **4. 최종 요약**

- **Default-Deny 적용** → 기본 통신 차단, **명시 정책만 통과**
- 유지 정책: **allow-dns**, **allow-frontend-egress(FE→BE:80)**, **allow-frontend-l7(BE Ingress: `GET /status/200`)**
- **비허용 L7 경로는 403**, **외부 egress는 차단(DROP)**
- Hubble에서 **`egress=allow-frontend-egress`**, **`ingress=allow-frontend-l7`**로 최종 매칭 확인 (`allow-all` 제거 확인)

### **5. 트래픽 캡처**

**(1) 허용(L7 `/status/200`)**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 8); do curl -sS -o /dev/null http://backend.shop.svc.cluster.local/status/200; sleep 0.2; done'
sleep 2
```

**(1.1) Hubble UI**

- `frontend → backend:80` 요청이 **GET /status/200**로 표시되고 **redirected → forwarded** 흐름 확인(L7 프록시 경유 허용)

![](https://velog.velcdn.com/images/tlsalswls123/post/ed1c0ca2-4f49-4b6a-b28f-9ca79fccd1a5/image.png)

**(1.2) Grafana**

- **L7 Flow Distribution**에서 **HTTP 피크 상승**으로 허용 트래픽 반영

![](https://velog.velcdn.com/images/tlsalswls123/post/c2f56f77-564c-43b5-9d91-d1816a2632b0/image.png)

**(2) 비허용 트래픽 (`GET /get`) 검증**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 6); do curl -sS -m 2 -o /dev/null http://backend.shop.svc.cluster.local/get || true; sleep 0.25; done'
sleep 2
```

**(2.1) Hubble UI**

- `GET /get` 요청이 **redirected → dropped**로 표시
- FE→BE:80 경로에 빨간 점선과 `verdict: dropped` 로그 확인

![](https://velog.velcdn.com/images/tlsalswls123/post/ae809bf1-9e86-48a7-afa2-44e7f5e798ba/image.png)

**(2.2) Grafana**

- **Drop Reason**: `DROP_REASON_UNKNOWN` 스파이크 관찰
- **L7 Flow Distribution**: **HTTP만 소폭 상승**, DNS 변화 없음

![](https://velog.velcdn.com/images/tlsalswls123/post/30d4a48b-3911-426d-b4d4-456cfe4b41a0/image.png)

**(3) 외부 egress(1.1.1.1:80) 차단 확인**

```bash
kubectl exec -n shop deploy/frontend --context "$CLUSTER1" -- sh -lc \
'for i in $(seq 1 6); do curl -sS -m 2 http://1.1.1.1:80 >/dev/null || true; sleep 0.25; done'
```

**(3.1) Hubble UI**

- `frontend → world(1.1.1.1):80/TCP` 경로에 **빨간 점선**, 리스트에 `verdict=dropped`가 연속 기록

![](https://velog.velcdn.com/images/tlsalswls123/post/293fd6c1-8740-4813-90eb-f87599be8cba/image.png)

**(3.2) Grafana**

- **Drop Reason**에 **POLICY_DENIED** 스파이크 → **Default-Deny로 외부 egress 차단**
- **Flows Types**에서는 **PolicyVerdict·Drop 라인 발생**(L7≈0)으로 **L3/L4 정책 차단**임을 뒷받침

![](https://velog.velcdn.com/images/tlsalswls123/post/8edfd545-8ad5-4560-922e-4f9d8aa19adb/image.png)

![](https://velog.velcdn.com/images/tlsalswls123/post/87d30e21-107a-4bb7-9d37-3e5425a90d0e/image.png)

![](https://velog.velcdn.com/images/tlsalswls123/post/2aa06a3f-ac7a-4fa1-897b-df0ea6e447ed/image.png)
