# 02. OS Preparation

**실행 대상: 전체 노드.** 기본 요구사항과 이번 containerd/Calico 구성에 필요한 설정을 준비한다. 이미 충족한 항목은 변경하지 않는다.

## 1. 시스템 업데이트 및 커널 확인 (권장)

```bash
dnf update -y
uname -r
rpm -q kernel-core
grubby --default-kernel
```

`uname -r`은 실행 중인 커널, `rpm -q kernel-core`는 설치된 커널 목록, `grubby --default-kernel`은 기본 부팅 커널을 보여 준다. 새 커널 설치만으로 실행 중인 커널이 바뀌지 않는다. 업데이트 완료 후 새 커널 적용이 필요하면 재부팅한다.

```bash
reboot
```

SSH 재접속 후 `uname -r`로 검증한다. 실습에서 5.14.0-687.15.1.el9_8.x86_64에서 5.14.0-687.66.1.el9_8.x86_64로 변경됐다. 패키지 버전은 시점마다 달라질 수 있다.

## 2. Hostname 및 이름 해석

```bash
hostname
getent hosts "$(hostname)"
```

각 hostname은 고유해야 한다. IP 기반 join에서는 전체 hosts 매핑이 필수는 아니지만, 실습 중 kubeadm의 hostname lookup 경고가 발생하여 전체 노드에 아래 매핑을 추가했다. Route Table/ACG는 이름 해석을 대신하지 않는다.

```bash
vi /etc/hosts
```

기존 localhost 항목을 유지하고 실제 IP로 추가한다.

```text
10.0.1.6 k8s-master
10.0.1.7 k8s-worker-01
10.0.1.8 k8s-worker-02
```

검증: `getent hosts k8s-master k8s-worker-01 k8s-worker-02`. 재부팅은 불필요하다.

## 3. Swap

기본 kubelet 설정은 활성 Swap을 허용하지 않는다. 이번 구성에서는 Swap을 끈다.

```bash
free -h
swapon --show
cat /etc/fstab
```

활성 Swap이 있다면 `swapoff -a` 실행. `/etc/fstab`에 Swap 항목이 있으면 백업 후 해당 줄만 주석 처리한다. 별도 swap unit이나 zram이 있는 환경이면 해당 활성화 설정도 확인한다.

```bash
swapon --show
free -h
```

Swap 0B, `swapon --show` 출력 없음이 목표다.

## 4. SELinux 및 firewalld

```bash
getenforce
systemctl status firewalld --no-pager
```

실습 이미지에서 Master의 SELinux는 Disabled, firewalld는 disabled/inactive였으므로 변경할 필요가 없었다. 다른 노드도 각각 확인한다.

SELinux가 Enforcing이면 이번 RPM 설치 가이드 기준으로 Permissive를 사용한다(보안 완화이므로 운영 환경에서는 정책을 별도 검토).

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Calico가 관리하는 패킷 필터링 규칙과 firewalld의 충돌을 방지하기 위해 설치되어 있고 활성화/자동 시작 상태인 경우 다음을 실행한다.

```bash
systemctl disable --now firewalld
```

`getenforce`, `systemctl status firewalld --no-pager`로 재확인한다. NCP ACG는 호스트 보안 기능을 완전히 대체하지 않는다.

## 5. 커널 모듈

`overlay`는 containerd의 OverlayFS 계층형 파일시스템에 사용된다. `br_netfilter`는 Linux bridge 트래픽에 Netfilter 처리를 적용할 수 있게 한다. Calico의 모든 구성에서 bridge 관련 설정이 필수인 것은 아니다.

```bash
lsmod | grep -E 'overlay|br_netfilter'
modprobe overlay
modprobe br_netfilter
lsmod | grep -E 'overlay|br_netfilter'
```

실습에서는 최초 출력이 없었으며 로드 후 overlay, br_netfilter와 의존 모듈 bridge가 확인됐다. Used by 0은 오류가 아니다.

재부팅 후에도 로드되도록 파일을 생성한다.

```bash
cat > /etc/modules-load.d/k8s.conf <<'CONF'
overlay
br_netfilter
CONF
cat /etc/modules-load.d/k8s.conf
```

파일명은 관리자가 정하며 `.conf` 파일을 부팅 시 systemd가 읽는다.

## 6. sysctl

```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

실습 Master의 값은 각각 0, 1, 1이었다. IPv4 Forwarding은 노드 내부에서 다른 인터페이스/Pod로 패킷을 전달하는 기능이다. VPC 라우팅과 다른 계층이다.

```bash
cat > /etc/sysctl.d/99-k8s.conf <<'CONF'
net.ipv4.ip_forward = 1
CONF
sysctl --system
sysctl net.ipv4.ip_forward
```

목표는 `net.ipv4.ip_forward = 1`. bridge 설정은 실습에서 이미 1이었으므로 수정하지 않았다. IPv6 bridge 필터는 IPv4 전용 구성의 필수 조건이 아니다. 파일명 `99-k8s.conf`는 관례이며 뒤쪽에 적용되게 하려는 목적이다. 우선순위는 `man sysctl.d`에서 확인한다.

## 완료 기준

모든 노드에서 고유 hostname, Swap 비활성화, 이번 실습의 SELinux/firewalld 상태, 모듈 및 IPv4 forwarding을 확인한다.

- [kubeadm 설치 공식 가이드](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Container Runtime 요구사항](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

[다음: Container Runtime Setup](03-container-runtime.md)
