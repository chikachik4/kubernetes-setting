# 01. Infrastructure Provisioning

## 목적

Kubernetes 노드가 동작할 VM과 네트워크를 구성한다. Route Table은 패킷 전달 경로, Network ACL은 서브넷 접근 제어, ACG는 서버 인터페이스 접근 제어를 담당한다.

## 1. VPC 및 Subnet 생성

NCP 콘솔에서 Korea 리전, VPC 플랫폼을 선택한다. Networking → VPC에서 다음을 생성한다.

| 항목 | 값 |
|---|---|
| VPC | k8s-vpc |
| 유형 | Normal |
| CIDR | 10.0.0.0/16 |
| Public Subnet | k8s-public-subnet / 10.0.1.0/24 / KR-1 / Internet Gateway Y / 일반 |
| Private Subnet | k8s-private-subnet / 10.0.2.0/24 / KR-1 / Internet Gateway N / 일반 |
| Network ACL | 기본값 유지 |

Normal VPC는 일반 워크로드를 배치하는 용도다. Transit VPC는 네트워크 허브가 필요한 경우에 고려한다. Private Subnet은 이번 실습에서 사용하지 않는다.

Route Table과 연결된 서브넷을 확인한다.

| Route Table | Destination | Target |
|---|---|---|
| Public | 10.0.0.0/16 | LOCAL |
| Public | 0.0.0.0/0 | IGW |
| Private | 10.0.0.0/16 | LOCAL |

Public Subnet이라도 개별 공인 IP 등 인터넷 연결 수단이 필요하다. 이번에는 NAT Gateway를 만들지 않고 각 노드에 공인 IP를 할당한다.

## 2. ACG 생성

Compute → Server → ACG에서 `k8s-acg`를 생성한다. VPC 기본 ACG는 변경하지 않는다.

로컬 PowerShell에서 공인 IP 확인:

```powershell
curl.exe https://api.ipify.org
```

Inbound:

| 프로토콜 | 소스 | 포트 |
|---|---|---|
| TCP | 내 공인 IP/32 | 22 |
| TCP | k8s-acg | 1-65535 |
| UDP | k8s-acg | 1-65535 |
| ICMP | k8s-acg | - |

Outbound:

| 프로토콜 | 목적지 | 포트 |
|---|---|---|
| TCP | 0.0.0.0/0 | 1-65535 |
| UDP | 0.0.0.0/0 | 1-65535 |
| ICMP | 0.0.0.0/0 | - |

ACG는 Stateful이며 동일 ACG를 소스로 지정하여 노드 간 통신을 허용한다. API Server TCP 6443, kubelet TCP 10250, Calico VXLAN UDP 4789 등도 이 내부 규칙에 포함된다. 외부 전체에 SSH나 API Server를 공개하지 않는다. 이 규칙은 실습 편의를 위한 것으로 운영 환경에서는 필요한 포트와 목적지만 허용한다.

## 3. 서버 생성

Compute → Server → Server 생성. 공인 IP 동시 신청을 위해 한 대씩 생성한다.

| 항목 | 값 |
|---|---|
| 이미지 | rocky-9.8-base |
| Hypervisor | KVM / G3 |
| 타입 | c2-g3a (2 vCPU, 4 GB) |
| 요금제 | 시간제 |
| 서버 이름 | k8s-master, k8s-worker-01, k8s-worker-02 |
| hostname 동일 설정 | 체크 |
| VPC / Subnet | k8s-vpc / k8s-public-subnet |
| NIC | eth0 하나, 사설 IP 자동 할당 |
| Public IP | 각 노드에 신규 할당 |
| Storage | CB2, 50 GB, 추가 디스크 없음 |
| 인증키 | k8s-key 신규 생성 후 나머지 노드에서 재사용 |
| ACG | k8s-acg만 적용 |
| Init Script | 없음 |

인증키 파일은 안전하게 보관하고 Git에 올리지 않는다. 실제 할당된 사설 IP를 기록한다. eth1은 추가하지 않는다.

## 4. SSH 및 네트워크 검증

콘솔의 서버 관리 및 설정 변경 → 관리자 비밀번호 확인에서 인증키로 각 서버 비밀번호를 확인한다. 로컬 PC에서 접속한다.

```bash
ssh root@<PUBLIC_IP>
```

각 서버에서:

```bash
hostnamectl
ip -br addr
ip route
nproc
free -h
lsblk
curl -I https://www.google.com
```

각 노드에서 나머지 두 노드의 사설 IP로 ping을 확인한다. 예: Master에서

```bash
ping -c 3 10.0.1.7
ping -c 3 10.0.1.8
```

실패하면 ACG, Network ACL, 서브넷 연결, OS 방화벽 및 공인 IP를 확인한다. IP 통신 검증은 DNS 이름 해석과 별개다.

## 참고

- [NCP 콘솔](https://console.ncloud.com/)
- [NCP VPC 가이드](https://guide.ncloud-docs.com/docs/vpc-start-vpc)
- [NCP 요금표](https://www.ncloud.com/charge/price/ko)

[다음: OS Preparation](02-os-preparation.md)
