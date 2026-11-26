# 01. Infrastructure & Network

- 학습 목표: Kubernetes 노드의 통신 경로와 네트워크 접근 제어를 이해한다.
- 관련 실습: [Infrastructure Provisioning](../01-infrastructure.md)

## 1. VPC와 Subnet

### VPC (Virtual Private Cloud)

클라우드에서 논리적으로 격리된 가상 네트워크 공간이다. CIDR을 사용하여 VPC에서 사용할 IP 주소 범위를 지정한다.

### Subnet (Subnetwork)

VPC의 IP 주소 범위를 작은 네트워크 영역으로 분할한 것이다. 서비스의 목적이나 보안 요구사항에 따라 서버를 배치할 수 있다.

| 구분 | 실습 설정 |
|---|---|
| VPC | `10.0.0.0/16` |
| Public Subnet | `10.0.1.0/24` |
| Private Subnet | `10.0.2.0/24` |

Subnet을 분리한다고 해서 서브넷 간 통신이 자동으로 차단되는 것은 아니다. 라우팅 및 접근 제어 정책을 별도로 확인해야 한다.

## 2. Public Subnet과 Private Subnet

인터넷으로 직접 연결되는 경로의 유무가 핵심이다.

| 구분 | Public Subnet | Private Subnet |
|---|---|---|
| IGW를 향하는 기본 경로 | 있음 | 없음 |
| 인터넷 직접 통신 | 공인 IP 및 허용 정책 필요 | 불가능 |
| VPC 내부 통신 | 가능 | 가능 |
| NAT Gateway를 통한 외부 통신 | 구성에 따라 가능 | 구성에 따라 가능 |

IGW(Internet Gateway)는 VPC와 인터넷 사이의 통신 경로를 제공한다. 공인 IP를 서버에 할당하는 기능 자체와는 구분해야 한다.

실습에서는 관리 편의성을 위해 세 노드를 Public Subnet에 배치하고 각 노드에 공인 IP를 할당했다. Private Subnet은 생성만 하고 사용하지 않았다.

## 3. Route Table

패킷의 목적지 IP를 기준으로 전달 경로를 결정하는 라우팅 정보다.

| Destination | Public Subnet Target | Private Subnet Target (실습 당시) |
|---|---|---|
| `10.0.0.0/16` | LOCAL | LOCAL |
| `0.0.0.0/0` | IGW | 없음 |

### Longest Prefix Match (최장 접두사 일치)

목적지 IP와 일치하는 여러 경로 중 Prefix 길이가 가장 긴 경로를 선택한다.

예를 들어 Master가 `10.0.1.7`로 통신하면 `/16`과 `/0` 경로에 모두 일치하지만, 더 구체적인 `/16` 경로가 우선한다. 따라서 VPC 내부 LOCAL 경로를 이용한다. `/16`은 앞 16비트를 네트워크 범위로 본다. 실제 노드가 속한 서브넷은 `/24`이며 VPC CIDR과 구분한다.

라우팅 경로가 존재하는 것은 패킷을 전달할 수 있다는 의미이며, 방화벽에서 해당 통신이 허용된다는 의미는 아니다.

## 4. Network ACL과 ACG

### 용어 및 AWS 비교

| 약어 | Full Name | 적용 범위 |
|---|---|---|
| ACL | Access Control List | 접근 제어 규칙 목록 |
| NACL | Network Access Control List | Subnet |
| ACG | Access Control Group | NCP 서버 네트워크 인터페이스 |
| SG | Security Group | AWS 네트워크 인터페이스 |

NCP의 ACG는 AWS Security Group에 대응하는 기능이다. Network ACL은 두 클라우드 모두 서브넷 경계의 트래픽을 제어한다.

| 구분 | ACG / Security Group | Network ACL |
|---|---|---|
| 적용 범위 | 서버 NIC | Subnet 경계 |
| 연결 상태 | Stateful | Stateless |
| 규칙 | Allow | Allow / Deny |
| 평가 방식 | 허용 규칙의 집합 | 우선순위 순서 |

### 적용 위치와 계층

AWS의 EC2 인스턴스 간 통신은 NCP의 Server 간 통신에 해당한다. ACG와 Security Group은 정확히는 컴퓨팅 자원의 네트워크 인터페이스(NIC)에 적용된다. AWS에서는 이를 ENI(Elastic Network Interface)라고 부른다.

| 역할 | AWS | NCP |
|---|---|---|
| 컴퓨팅 자원 | EC2 Instance | Server |
| 가상 네트워크 인터페이스 | ENI | Network Interface |
| 서브넷 경계 접근 제어 | Network ACL | Network ACL |
| 인터페이스 접근 제어 | Security Group | ACG |
| OS 수준 방화벽 | firewalld 등 | firewalld 등 |

```text
VPC (10.0.0.0/16)
├── Subnet A (10.0.1.0/24) [Network ACL A]
│   ├── [ACG / SG] NIC - Master (10.0.1.6)
│   └── [ACG / SG] NIC - Worker 01 (10.0.1.7)
└── Subnet B (10.0.2.0/24) [Network ACL B]
    └── [ACG / SG] NIC - 가상의 Worker (10.0.2.7)
```

아파트 단지에 비유하면 VPC는 단지, Subnet은 건물, Network ACL은 공동현관, ACG는 개별 세대의 현관에 해당한다. firewalld는 서버 OS 내부의 추가 접근 제어에 해당한다.

두 클라우드 방화벽 모두 IP, 프로토콜, 포트 등 주로 OSI L3/L4 정보를 이용한다. 여기서 계층의 차이는 OSI 계층의 차이가 아니라 **방화벽이 적용되는 위치의 차이**이다.

### 패킷 검사 순서와 정책 우선순위

서브넷 경계를 통과하는 일반적인 신규 연결을 기준으로 논리적인 흐름은 다음과 같다.

```text
Inbound : 외부 → Network ACL → ACG/SG → OS 방화벽 → 서비스
Outbound: 서비스 → OS 방화벽 → ACG/SG → Network ACL → 외부
```

위 흐름은 접근 제어 위치를 이해하기 위한 단순화된 모델이다. Network ACL과 ACG 중 하나가 다른 정책보다 우선하여 그 결과를 덮어쓰는 것은 아니다. 적용되는 모든 정책이 통과되어야 한다(AND 조건).

| Network ACL | ACG | 결과 |
|---|---|---|
| 허용 | 허용 | 통과 |
| 허용 | 미허용 | 차단 |
| 차단 | 허용 | 차단 |
| 차단 | 미허용 | 차단 |

이 표는 두 정책이 모두 적용되는 신규 패킷을 기준으로 한다. Network ACL 내부의 규칙 우선순위는 별개의 개념이다. 예를 들어 AWS NACL은 낮은 규칙 번호부터 검사하여 처음 일치하는 규칙을 적용한다. ACG/SG는 허용 규칙이 하나라도 일치하면 해당 방화벽을 통과하며, 일치하는 허용 규칙이 없으면 차단한다.

### 같은 Subnet과 다른 Subnet의 차이

```text
같은 Subnet (사설 IP 직접 통신)
Worker NIC/ACG Outbound → Master NIC/ACG Inbound

다른 Subnet
Worker NIC/ACG Outbound → Worker Subnet ACL Outbound
 → Master Subnet ACL Inbound → Master NIC/ACG Inbound
```

| 통신 유형 | 적용되는 클라우드 접근 제어 |
|---|---|
| 같은 Subnet의 서버 간 직접 통신 | 송신 및 수신 ACG/SG |
| 서로 다른 Subnet의 서버 간 통신 | 송신 및 수신 ACG/SG, 각 Subnet의 Network ACL |
| 인터넷과 서버 간 통신 | 서버 ACG/SG, 해당 Subnet의 Network ACL |

동일 Subnet 내부에서 사설 IP로 직접 통신하는 경우 Network ACL을 거치지 않는다. 하지만 ACG/SG는 여전히 적용된다. 같은 VPC나 같은 Subnet이라는 사실만으로 신규 연결이 자동 허용되지는 않는다. 응답 패킷에도 적용되는 정책은 아래 Stateful/Stateless 방식에 따라 판단한다.

> 예시: Master와 Worker가 같은 Subnet에 있다면 NACL에 TCP 6443 차단 규칙이 있어도 해당 직접 통신에는 적용되지 않는다. Worker Outbound 및 Master Inbound ACG가 허용하고 다른 조건이 정상이라면 접속 가능하다. 서로 다른 Subnet이라면 경계에서 해당 차단 규칙이 적용될 수 있다.

### Stateful과 Stateless

- Stateful: 허용된 연결의 상태를 추적하고 해당 연결에 대한 응답 트래픽을 자동 허용한다.
- Stateless: 요청과 응답의 관계를 기억하지 않고 각 방향의 패킷을 독립적으로 검사한다.

Stateful이라고 모든 Outbound 통신을 허용하는 것은 아니다. 새로운 연결 요청에는 별도의 규칙이 필요하다.

### SSH 연결 예시: TCP 3-Way Handshake

클라이언트의 임시 포트를 50000, 서버의 SSH 포트를 22라고 가정한다.

```text
Client (50000)                         Server (22)
      | -------- SYN (50000 → 22) --------> |
      | <---- SYN+ACK (22 → 50000) -------- |
      | -------- ACK (50000 → 22) --------> |
```

서버 관점에서 요청은 Inbound, 응답은 Outbound이다. TCP는 요청과 응답이 모두 전달되어야 연결이 성립한다.

| 정책 | 요청 | 응답 |
|---|---|---|
| ACG | Inbound 목적지 TCP 22 허용 | 허용된 연결의 응답 자동 허용 |
| Network ACL | Inbound 목적지 TCP 22 허용 | Outbound 목적지 TCP 50000 허용 필요 |

Network ACL의 Outbound 목적지 TCP 22만 허용하면 응답의 목적지 포트인 50000을 차단하므로 SSH 연결이 실패한다. 실제 운영에서는 클라이언트의 임시 포트(Ephemeral Port) 범위와 목적지 IP를 고려해 정책을 설정한다.

## 5. Kubernetes 실습 환경에 적용

### 노드 구성

| 노드 | 역할 | Private IP |
|---|---|---|
| k8s-master | Control Plane | `10.0.1.6` |
| k8s-worker-01 | Worker | `10.0.1.7` |
| k8s-worker-02 | Worker | `10.0.1.8` |

### ACG 정책

세 노드 모두 `k8s-acg`를 적용했다. 기본 Network ACL은 변경하지 않았다.

| 방향 | 대상 | 허용 |
|---|---|---|
| Inbound | 내 공인 IP/32 | TCP 22 |
| Inbound | k8s-acg | TCP 1–65535 |
| Inbound | k8s-acg | UDP 1–65535 |
| Inbound | k8s-acg | ICMP |
| Outbound | 모든 IP | TCP/UDP 전체, ICMP |

노드 간 통신이 가능했던 이유는 같은 VPC라는 사실뿐만 아니라 동일 ACG를 소스로 하는 허용 규칙을 직접 추가했기 때문이다.

| 통신 예시 | 프로토콜 / 목적지 포트 |
|---|---|
| 내 PC → 노드 SSH | TCP 22 |
| Worker → API Server | TCP 6443 |
| Control Plane → kubelet | TCP 10250 |
| Calico VXLAN 노드 간 통신 | UDP 4789 |

실습에서는 편의상 노드 간 포트를 넓게 허용했다. 운영에서는 필요한 통신만 허용하는 최소 권한 정책을 고려해야 한다.

### 동일 ACG 사용과 접근 소스 지정의 차이

같은 `k8s-acg`를 적용했다는 사실만으로 통신이 허용되는 것은 아니다. Inbound의 접근 소스에 `k8s-acg`를 명시하면 해당 그룹에 속한 자원에서 오는 통신을 지정한 프로토콜과 포트에 한해 허용한다.

예를 들어 Outbound 전체 허용 상태에서 다음 Inbound 규칙만 설정한다고 가정한다. 이는 학습용 가상 정책이며 실제 실습 설정을 변경한 것이 아니다.

| 접근 소스 | 프로토콜 | 목적지 포트 |
|---|---|---|
| 내 PC 공인 IP/32 | TCP | 22 |
| k8s-acg | TCP | 6443 |

| 신규 연결 | 결과 | 이유 |
|---|---|---|
| 내 PC → Worker:22 | 허용 | 접근 소스와 포트 일치 |
| Worker → Master:6443 | 허용 | ACG 소스와 포트 일치 |
| Worker → Master:22 | 차단 | SSH 허용 소스가 내 PC 공인 IP로 제한됨 |
| Worker → Master:80 | 차단 | 일치하는 허용 규칙 없음 |

라우팅, OS 방화벽, 서비스 및 적용되는 ACL은 정상이라고 가정한다. `k8s-acg`를 소스로 하는 TCP 6443 규칙까지 제거하면 Worker의 API Server 접속도 차단되므로 `kubeadm join`을 정상적으로 진행할 수 없다.

```bash
kubeadm join 10.0.1.6:6443 --token ...
```

TCP 6443은 kube-apiserver의 기본 HTTPS 포트이며 변경할 수 있다. Worker의 클러스터 참여와 `kubectl`의 API 요청은 API Server와의 통신을 필요로 한다. 6443 접속이 허용된다는 사실만으로 인증이나 전체 클러스터 운영까지 성공하는 것은 아니다.

주요 통신을 구분한다.

- 노드 간 통신: NCP Server(AWS EC2) 사이의 네트워크 통신
- Pod 간 통신: Calico 등의 CNI가 제공하는 Pod 네트워크 통신. 서로 다른 노드에 있는 Pod의 통신도 하위 노드 네트워크에 의존한다.

### API Server에 사설 IP를 지정한 이유

```bash
--apiserver-advertise-address=10.0.1.6
```

노드 간 통신은 VPC 내부에서 수행하므로 API Server의 주소로 공인 IP를 사용할 필요가 없다. 사설 IP를 사용하면 공인 IP 변경과 내부 통신 주소를 분리할 수 있다. 단, advertise-address만으로 외부 접근이 차단되는 것은 아니며 실제 접근 제어는 ACG 등으로 수행한다.

## 6. NAT Gateway

NAT(Network Address Translation) Gateway는 주소 변환을 통해 사설 IP만 가진 서버가 외부 인터넷에 연결할 수 있도록 한다.

```text
Public Subnet:   Server (공인 IP 연결) → IGW → Internet
Private Subnet:  Worker (사설 IP만)   → NAT Gateway → Internet
```

Private Subnet에서 외부 인터넷을 이용하도록 구성하는 경우:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | LOCAL |
| `0.0.0.0/0` | NAT Gateway |

NAT Gateway 및 인터넷 연결 구성이 정상이고 접근 제어가 허용된다고 가정한다.

| 통신 | 가능 여부 |
|---|---|
| Worker → 외부 Registry 이미지 다운로드 | 가능 |
| Registry → 기존 요청에 대한 응답 | 가능 |
| 외부 사용자 → Worker 신규 SSH 연결 | 불가능 |

Private Subnet 서버에 접근하려면 Bastion Host, VPN 등의 별도 접근 경로가 필요하다. 이번 실습에서는 NAT Gateway를 생성하지 않았으며 위 내용은 Private Subnet을 사용하는 가상 시나리오이다.

## 7. 장애 발생 시 확인할 부분

| 상황 | 확인 대상 |
|---|---|
| 노드 간 ping 실패 | IP, Route Table, ACG ICMP 규칙, OS 방화벽 |
| 외부 SSH 접속 실패 | 공인 IP, IGW, ACG TCP 22, Network ACL, sshd |
| Worker에서 API Server 접속 실패 | 사설 IP, ACG TCP 6443, 적용되는 Network ACL, API Server 상태 |
| Private Worker의 이미지 다운로드 실패 | NAT Gateway, 기본 경로, Outbound 규칙, DNS, Registry 접근 |

```bash
ip -br addr                       # NIC 및 IP 확인
ip route                          # OS 라우팅 테이블 확인
ping -c 3 10.0.1.7               # ICMP 통신 확인
curl -I https://www.google.com    # 외부 HTTPS 통신 확인
```

`ip route`는 서버 OS의 라우팅 정보를 보여 주므로 NCP 콘솔의 Route Table 및 보안 정책은 별도로 확인한다. ping 성공은 TCP 포트 연결 성공을 보장하지 않는다.

## 8. 복습 문제

1. VPC와 Subnet의 차이는 무엇인가?
2. Public Subnet과 Private Subnet의 차이는 무엇인가?
3. `10.0.1.7`은 `10.0.0.0/16 → LOCAL`과 `0.0.0.0/0 → IGW` 중 어느 경로를 사용하는가?
4. 같은 VPC에 있는 서버 간 통신은 ACG가 차단하더라도 가능한가?
5. ACG와 Network ACL의 적용 범위 및 Stateful/Stateless 차이는 무엇인가?
6. 서로 다른 Subnet의 Worker와 Master 사이에서 Network ACL이 TCP 6443을 차단하고 ACG는 허용한다면 통신 가능한가?
7. SSH 클라이언트 출발지 포트가 50000일 때 Network ACL Inbound와 Outbound에서 모두 목적지 22번만 허용하면 접속 가능한가?
8. API Server 주소를 공인 IP 대신 사설 IP로 지정한 이유는 무엇인가?
9. Private Subnet의 Worker에 공인 IP가 없고 LOCAL 경로만 있다면 이미지 다운로드를 위해 어떤 구성이 필요한가?
10. NAT Gateway의 공인 IP로 외부에서 Worker에 신규 SSH 접속할 수 있는가?
11. Network ACL과 ACG는 서로 다른 OSI 계층에서 동작하는가? 두 정책의 우선순위는 어떻게 이해해야 하는가?
12. 같은 Subnet 내 Worker와 Master 사이에서 NACL이 TCP 6443을 차단하고 송신/수신 ACG가 허용한다면 사설 IP 직접 통신이 가능한가?
13. 두 노드에 동일 ACG를 적용했지만 Inbound가 내 PC 공인 IP의 TCP 22만 허용하고 Outbound는 전체 허용한다면 Worker에서 Master로 `kubeadm join`이 가능한가?
14. Inbound에 `k8s-acg` 소스의 TCP 6443 허용을 추가하면 Worker에서 Master의 API Server 및 SSH에 각각 접근할 수 있는가?
15. AWS에서 EC2 간 통신에 적용하는 SG는 정확히 어떤 자원에 연결되는가?

<details>
<summary>정답 및 핵심 설명</summary>

1. VPC는 격리된 가상 네트워크 공간, Subnet은 VPC 주소 범위를 분할한 네트워크 영역이다.
2. IGW로 향하는 직접적인 인터넷 경로의 유무이다. Private Subnet도 NAT Gateway를 통해 외부 통신이 가능하다.
3. LOCAL. Longest Prefix Match에 따라 `/16`이 `/0`보다 우선한다.
4. 불가능하다. 라우팅과 접근 제어는 별개의 기능이다.
5. ACG는 서버 NIC에 적용되며 Stateful이다. Network ACL은 서브넷 경계에 적용되며 Stateless이다.
6. 불가능하다. 적용되는 Network ACL의 차단 정책을 ACG 허용으로 우회할 수 없다.
7. 불가능하다. 서버 응답의 목적지는 50000번이므로 Outbound에서도 해당 포트를 허용해야 한다.
8. 노드 간 내부 통신에는 사설 IP로 충분하며 인터넷에 API Server를 공개할 필요가 없다. 별도의 접근 제어는 필요하다.
9. NAT Gateway를 구성하고 Private Subnet의 기본 경로를 NAT Gateway로 지정한다. 관련 접근 제어 및 DNS도 정상이어야 한다.
10. 불가능하다. 일반적인 인터넷 송신용 NAT Gateway는 내부에서 시작한 연결과 응답을 처리한다.
11. 아니다. 둘 다 주로 L3/L4 정보를 사용한다. 적용 위치가 다르며 적용되는 모든 방화벽을 통과해야 한다. Network ACL 내부 규칙의 우선순위와 별개의 개념이다.
12. 가능하다. 같은 Subnet 내부의 직접 통신은 Network ACL의 검사 대상이 아니다. 다른 조건은 정상이라고 가정한다.
13. 불가능하다. Master의 Inbound ACG에 Worker의 TCP 6443 접근을 허용하는 규칙이 없다. 동일 ACG 소속만으로는 자동 허용되지 않는다.
14. API Server 접근은 가능하고 SSH 접근은 불가능하다. SSH 허용 소스는 내 PC 공인 IP이며 Worker는 이에 해당하지 않는다. 다른 조건은 정상이라고 가정한다.
15. ENI(Elastic Network Interface)에 적용된다. NCP의 ACG 역시 네트워크 인터페이스에 적용된다.

</details>

## 핵심 정리

- VPC와 Subnet: 네트워크 공간 및 IP 범위
- Route Table: 패킷 전달 경로
- Network ACL: Subnet 경계 접근 제어(Stateless)
- ACG/SG: NIC 접근 제어(Stateful), 동일 그룹 소속만으로 자동 허용되지 않음
- IGW: 직접적인 인터넷 연결 경로
- NAT Gateway: 공인 IP가 없는 노드의 외부 인터넷 연결

## 참고 문서

- [NCP Network ACL](https://guide.ncloud-docs.com/docs/ko/vpc-nacl-vpc)
- [NCP ACG (VPC)](https://guide.ncloud-docs.com/docs/server-acg-vpc)
- [AWS Network ACL](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [AWS VPC Infrastructure Security](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)
- [AWS Security Group Rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)

[스터디 목차로 돌아가기](README.md)
