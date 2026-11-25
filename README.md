# NCP Kubernetes 구축 가이드

2026-11-25 실습 기록을 기반으로 작성한 kubeadm 기반 3노드 클러스터 재구축 가이드. 관리형 NKS는 사용하지 않는다. 각 문서는 순서대로 실행하며, 명령어는 별도 표시가 없으면 root 기준이다.

## 구축 환경

| 항목 | 실습 값 |
|---|---|
| Cloud | Naver Cloud Platform, Korea / KR-1, VPC |
| OS | Rocky Linux 9.8 (rocky-9.8-base) |
| 서버 | c2-g3a, 2 vCPU / 4 GB, 3대, 시간제 |
| 디스크 | CB2 50 GB / 노드 |
| Kubernetes | v1.36.6 |
| Runtime | containerd 2.4.2, runc 1.5.3 |
| CNI | Calico v3.33.0, Tigera Operator, VXLAN, BGP Disabled |
| VPC / Public / Private | 10.0.0.0/16 / 10.0.1.0/24 / 10.0.2.0/24 |
| Pod / Service CIDR | 192.168.0.0/16 / 10.96.0.0/12 |

| 노드 | 역할 | 실습 사설 IP |
|---|---|---|
| k8s-master | Control Plane | 10.0.1.6 |
| k8s-worker-01 | Worker | 10.0.1.7 |
| k8s-worker-02 | Worker | 10.0.1.8 |

사설 IP는 자동 할당된 실습 값이다. 재구축 시 실제 할당값으로 모든 명령어와 hosts 파일을 수정한다. 공인 IP와 인증 정보는 기록하지 않는다.

## 구축 순서

| 단계 | 문서 | 실행 대상 |
|---|---|---|
| 01 | [Infrastructure Provisioning](01-infrastructure.md) | NCP 콘솔 / 로컬 PC |
| 02 | [OS Preparation](02-os-preparation.md) | 전체 노드 |
| 03 | [Container Runtime Setup](03-container-runtime.md) | 전체 노드 |
| 04 | [Kubernetes Installation](04-kubernetes-installation.md) | 전체 노드 |
| 05 | [Control Plane Initialization](05-control-plane-init.md) | Master |
| 06 | [Calico Installation](06-calico-installation.md) | 전체 노드 사전 설정 / Master 설치 |
| 07 | [Worker Join](07-worker-join.md) | Master / Worker |
| 08 | [Cluster Verification](08-cluster-verification.md) | Master |

## 완료 기준 및 범위

실제 실습에서 Node 3/3 Ready, 시스템 Pod 정상 실행, Calico/kube-proxy 3/3, 양방향 Cross-Node HTTP 200, ClusterIP 및 Service 이름 접근 HTTP 200을 확인했다. 이 문서 자체를 새 환경에서 재실행한 것은 아니다. HA, 영구 스토리지, 외부 서비스 공개, NetworkPolicy, GPU, 운영 보안 강화는 후속 실습 범위다.

단일 Control Plane, 공인 IP 3개, 내부 ACG의 넓은 허용 범위, SELinux/firewalld 완화는 학습용 설계다. 운영 환경에 그대로 적용하지 않는다. 버전과 패키지 제공 여부는 재구축 시 공식 문서와 저장소에서 확인한다. 서버 정지 중에도 디스크와 공인 IP 등의 비용이 발생할 수 있다.
