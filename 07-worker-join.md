# 07. Worker Join

## 1. Join 명령 생성 (Master)

```bash
kubeadm token create --print-join-command
```

Worker의 초기 인증을 위한 토큰과 CA 검증 해시가 포함된 명령어를 생성한다.

## 2. Join 실행 (Worker 2대)

Master에서 출력된 실제 명령어를 각각 실행한다. 아래는 형식 예시이며 그대로 실행하지 않는다.

```bash
kubeadm join 10.0.1.6:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

`--token`은 초기 노드 인증, `--discovery-token-ca-cert-hash`는 신뢰할 클러스터 CA 검증에 사용한다. 실제 인증 토큰은 저장소에 기록하지 않는다.

Join 이후 kubelet이 Node를 등록하고 DaemonSet에 의해 calico-node와 kube-proxy가 배포된다. Worker에 Calico를 따로 설치하지 않는다.

## 3. 검증 (Master)

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ds -n calico-system
```

실습에서 join 직후 Worker는 NotReady였으나 Calico 초기화 후 모두 Ready가 되었다. `calico-node`의 DESIRED/CURRENT/READY가 3/3/3인지 확인한다. Worker ROLES `<none>`은 역할 Label이 없다는 뜻이며 오류가 아니다.

지속적으로 NotReady라면:

```bash
kubectl describe node k8s-worker-01
kubectl get pods -A -o wide
```

해당 Worker에서 `journalctl -u kubelet -n 100 --no-pager`를 확인한다. 토큰 만료 시 Master에서 join 명령을 새로 생성한다. API 연결 문제라면 사설 IP와 TCP 6443 ACG 규칙을 확인한다.

[다음: Cluster Verification](08-cluster-verification.md)
