# 08. Cluster Verification

**실행 대상: Master.** Node Ready만 확인하지 않고 실제 워크로드 배포, 노드 간 Pod 통신, Service 및 DNS를 검증한다.

## 1. 시스템 구성 요소

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ds -A
kubectl get tigerastatus
```

목표: Node 3/3 Ready, Control Plane 및 CoreDNS Running, calico-node/kube-proxy 3/3 Ready, Calico Available=True/Degraded=False.

실습에서 모든 항목을 충족했으며 시스템 Pod 재시작 횟수는 0이었다. NCP 공인 IP가 있어도 Kubernetes EXTERNAL-IP는 클라우드 연동이 없으면 `<none>`일 수 있다.

## 2. Nginx 배포

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl rollout status deployment/nginx-test --timeout=180s
kubectl get pods -o wide
```

API Server, etcd, Controller, Scheduler, kubelet, containerd, Calico가 협력해 Pod를 실행하고 IP를 할당하는지 확인한다. 실습 결과는 Worker 01의 192.168.36.194와 Worker 02의 192.168.118.66이었다. 재구축 시 이름과 IP는 달라진다. 두 Pod가 다른 노드에 배치됐는지 확인한다(자동 배치는 보장되지 않음).

## 3. Cross-Node Pod 통신

실제 이름/IP를 확인한 후 아래 변수를 지정한다. 예시 값은 실습 당시의 값이다.

```bash
POD1=nginx-test-c8697b5cc-sj8f4
POD2=nginx-test-c8697b5cc-rscqv
IP1=$(kubectl get pod "$POD1" -o jsonpath='{.status.podIP}')
IP2=$(kubectl get pod "$POD2" -o jsonpath='{.status.podIP}')
kubectl exec "$POD1" -- curl -I --max-time 10 "http://$IP2"
kubectl exec "$POD2" -- curl -I --max-time 10 "http://$IP1"
```

두 Pod가 서로 다른 Worker에 있어야 이 테스트가 Cross-Node 통신을 검증한다. 동일 노드라면 별도 테스트 Pod를 각 Worker에 지정하여 검증한다. 실습에 사용한 nginx 이미지에는 curl이 있었으나 향후 이미지에서 빠져 있다면 curl 지원 테스트 이미지로 대체한다.

실제 결과: 양방향 `HTTP/1.1 200 OK`.

## 4. ClusterIP 및 EndpointSlice

```bash
kubectl expose deployment nginx-test --port=80 --type=ClusterIP
kubectl get svc nginx-test
kubectl get endpointslices -l kubernetes.io/service-name=nginx-test
SVC_IP=$(kubectl get svc nginx-test -o jsonpath='{.spec.clusterIP}')
kubectl exec "$POD1" -- curl -I --max-time 10 "http://$SVC_IP"
```

Service는 변경될 수 있는 Pod IP 대신 안정적인 접근 지점을 제공한다. EndpointSlice에서 두 Pod IP를 확인한다. kube-proxy가 Service에 대한 전달 규칙을 구성한다.

실제 결과: ClusterIP 10.104.30.118, Endpoint 192.168.36.194 및 192.168.118.66, HTTP 200.

## 5. CoreDNS

```bash
kubectl exec "$POD1" -- curl -I --max-time 10 http://nginx-test
```

Service 이름을 DNS로 해석한 후 ClusterIP를 통해 HTTP 통신하는지 확인한다. 실제 결과: HTTP 200. DNS/Service 접근 성공을 확인했지만 Service가 두 백엔드에 트래픽을 분산하는 비율까지 검증한 것은 아니다.

## 6. 테스트 리소스 정리

```bash
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
kubectl get pods,svc
```

기본 `kubernetes` Service는 삭제하지 않는다. 실습을 종료할 때는 NCP 콘솔에서 VM을 정지하여 가동 비용을 줄인다. 디스크 및 공인 IP 등의 유지 비용은 별도이며, 재구축이 필요 없으면 필요한 데이터 백업 후 관련 리소스를 반납한다.

## 최종 결과 (2026-11-25 실습)

| 검증 | 결과 |
|---|---|
| Node | 3/3 Ready |
| 시스템 Pod | Running, 재시작 0 |
| Calico / kube-proxy | 3/3 |
| Nginx Deployment | 2/2 Running, Worker별 배치 |
| Cross-Node HTTP | 양방향 200 |
| ClusterIP HTTP | 200 |
| Service DNS HTTP | 200 |

[README로 돌아가기](README.md)
