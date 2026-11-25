# 05. Control Plane Initialization

**실행 대상: Master만.** kubeadm init은 인증서, kubeconfig, Static Pod manifest, bootstrap token 등을 생성한다. kubelet이 containerd를 통해 API Server, etcd, Scheduler, Controller Manager를 실행한다.

## 1. 사전 검증

```bash
systemctl status containerd --no-pager
kubeadm version
kubelet --version
```

Node/VPC, Pod, Service CIDR이 서로 겹치지 않는지 확인한다. 이번 VPC는 10.0.0.0/16, Pod는 192.168.0.0/16, Service는 10.96.0.0/12다. 실제 연결되는 다른 네트워크 대역과의 중복도 확인한다.

## 2. 초기화

Master 사설 IP를 실제 값으로 바꾼다.

```bash
kubeadm init \
  --kubernetes-version=v1.36.6 \
  --apiserver-advertise-address=10.0.1.6 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

| 옵션 | 목적 |
|---|---|
| kubernetes-version | Control Plane 버전 지정 |
| apiserver-advertise-address | 다른 노드에서 사용할 API Server 주소 |
| pod-network-cidr | Pod 네트워크 범위 |
| cri-socket | containerd 연결 소켓 |

Service CIDR은 기본값 10.96.0.0/12를 사용한다. 성공 메시지는 `Your Kubernetes control-plane has initialized successfully!`이다. 출력된 join 토큰은 공개하지 않는다.

## 3. kubeconfig 설정

root 계정 기준:

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chmod 600 $HOME/.kube/config
```

kubectl의 기본 설정 파일에 API 주소 및 관리자 인증 정보를 제공한다. 관리자 권한을 갖는 민감한 파일이므로 Git에 올리지 않는다.

```bash
ls -l /etc/kubernetes/manifests/
kubectl get nodes -o wide
kubectl get pods -A
```

Static Pod manifest에는 etcd.yaml, kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml이 존재한다. CNI 설치 전 Node NotReady 및 CoreDNS Pending은 정상적인 초기 상태일 수 있다.

## 4. 실습 중 발생한 경고

- `SystemVerification`: Rocky 9의 5.14 커널이 kubeadm 검사기의 지원 LTS 목록에 없다는 경고. RHEL 계열 커널은 백포트를 사용하지만 경고가 공식 지원을 보장하지는 않는다. 이번 실습은 커널을 교체하지 않고 이후 통신 검증까지 성공했다.
- `Hostname`: `lookup k8s-master ... no such host`. `/etc/hosts` 매핑으로 이름 해석을 보완했다(02 참고). 경고만으로 init을 다시 실행할 필요는 없다.

실제 오류 발생 시 무조건 무시하거나 init을 반복하지 않고 `journalctl -u kubelet -n 100 --no-pager` 및 `crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a`로 원인을 확인한다.

- [kubeadm 클러스터 생성 공식 가이드](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

[다음: Calico Installation](06-calico-installation.md)
