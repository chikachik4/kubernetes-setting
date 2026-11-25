# 06. Calico Installation

Calico는 Pod IP 할당, 노드 간 패킷 전달, NetworkPolicy 적용을 담당한다. 이번에는 VXLAN 캡슐화(UDP 4789)를 사용하고 BGP는 끈다. Pod 패킷은 노드 사설 IP 간 통신으로 캡슐화되므로 VPC에 Pod CIDR별 경로를 추가하지 않는 구성이다.

## 1. NetworkManager 설정 (전체 노드)

```bash
systemctl status NetworkManager --no-pager
```

실행 중이면 Calico가 관리하는 인터페이스를 NetworkManager의 관리 대상에서 제외한다.

```bash
cat > /etc/NetworkManager/conf.d/calico.conf <<'CONF'
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
CONF
nmcli general reload conf
```

## 2. CRD 및 Tigera Operator 설치 (Master)

실습에서 사용한 Calico v3.33.0 기준이다. Kubernetes 버전을 바꾸면 Calico 공식 호환 범위를 확인한다.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

CRD는 Calico 전용 리소스 유형을 등록하고 Operator는 선언된 설치 설정에 따라 Calico 구성 요소를 관리한다.

## 3. Installation 리소스 생성 (Master)

```bash
cat > calico-installation.yaml <<'YAML'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - blockSize: 26
        cidr: 192.168.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
YAML
kubectl create -f calico-installation.yaml
```

| 설정 | 의미 |
|---|---|
| cidr | kubeadm init의 Pod CIDR과 일치 |
| blockSize | IPAM 할당 블록 크기 |
| encapsulation | VXLAN 사용 |
| bgp | BGP 비활성화 |
| natOutgoing | IP pool 외부 목적지로 나가는 트래픽에 NAT 적용 |
| nodeSelector | 전체 노드 대상 |

## 4. 검증 (Master)

```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system -o wide
kubectl get tigerastatus
kubectl get nodes -o wide
```

실습에서는 Calico AVAILABLE=True, PROGRESSING=False, DEGRADED=False 및 Master Ready를 확인했다. calico-node, calico-kube-controllers, calico-typha, csi-node-driver가 Running이었다. csi-node-driver는 Calico용 소켓 제공 구성 요소이며 일반 워크로드용 영구 스토리지 CSI 설치를 의미하지 않는다.

문제 발생 시 `kubectl describe pod -n calico-system <POD>`와 `kubectl logs -n calico-system <POD> --all-containers=true`를 확인한다. Worker 가입 전 일부 Pod는 Pending일 수 있다.

- [Calico 설치 공식 가이드](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Calico 시스템 요구사항](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)

[다음: Worker Join](07-worker-join.md)
