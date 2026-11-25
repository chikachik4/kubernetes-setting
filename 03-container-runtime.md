# 03. Container Runtime Setup

**실행 대상: 전체 노드.** CRI는 kubelet과 Runtime 간의 gRPC 인터페이스다. CRI 자체를 설치하는 것이 아니라 이를 구현하는 containerd를 설치한다. containerd는 이미지와 컨테이너를 관리하고 runc는 OCI 규격에 따라 컨테이너를 실행한다.

## 1. 패키지 및 저장소 확인

```bash
rpm -q containerd containerd.io
rpm -q dnf-plugins-core
```

미설치 시 다음을 실행한다. 기존 런타임이 있다면 충돌 여부를 먼저 확인한다.

```bash
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rocky/docker-ce.repo
dnf repolist
```

Docker 공식 RPM 저장소에서 containerd.io만 설치한다. Docker Engine 전체는 필요 없다.

```bash
dnf install -y containerd.io
containerd --version
runc --version
```

실습 버전은 containerd v2.4.2 / runc v1.5.3. 재현 시 `dnf list --showduplicates containerd.io`로 사용 가능한 버전을 확인한다. 위 명령은 저장소의 현재 버전을 설치하므로 실습과 동일 버전을 보장하지 않는다.

## 2. CRI 활성화 및 기본 설정 생성

```bash
cat /etc/containerd/config.toml
```

실습에서는 패키지의 최소 설정에 `disabled_plugins = ["cri"]`가 존재했다. kubelet이 CRI를 사용할 수 있도록 초기 환경에서 기본 설정을 다시 생성했다. 기존 운영 환경에서는 설정을 무조건 덮어쓰지 않는다.

```bash
cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
containerd config default > /etc/containerd/config.toml
grep -nE 'version =|disabled_plugins|SystemdCgroup' /etc/containerd/config.toml
```

실습 출력: `version = 4`, `disabled_plugins = []`, `SystemdCgroup = false`. 스키마 버전은 containerd가 생성한 값을 그대로 유지한다. 버전에 따라 구조가 달라지므로 인터넷 예제의 섹션을 무조건 복사하지 않는다.

## 3. cgroup Driver 설정

```bash
stat -fc %T /sys/fs/cgroup
vi /etc/containerd/config.toml
```

Rocky Linux 9는 일반적으로 cgroup v2(`cgroup2fs`)를 사용한다. cgroup은 CPU, 메모리 등 프로세스 자원을 제어한다. kubeadm의 기본 kubelet driver인 systemd와 런타임의 관리 방식을 맞춘다.

vi에서 `/SystemdCgroup`으로 검색하여 runc의 options에 있는 기존 값을 다음처럼 수정한다.

```toml
SystemdCgroup = true
```

containerd 2.x에서는 보통 `io.containerd.cri.v1.runtime` 아래 runc options에 해당한다. 실제 생성된 파일의 섹션을 기준으로 수정한다.

```bash
grep -n -B 8 SystemdCgroup /etc/containerd/config.toml
```

## 4. 서비스 시작 및 검증

```bash
systemctl enable --now containerd
systemctl restart containerd
systemctl status containerd --no-pager
ctr plugins ls | grep cri
```

이미 실행 중인 서비스에는 restart로 변경 사항을 적용한다. 목표는 active (running), 필요한 CRI 플러그인 상태 ok다. 소켓 `/run/containerd/containerd.sock`으로 kubelet이 연결한다.

문제 발생 시:

```bash
journalctl -u containerd -n 100 --no-pager
```

- [Docker Rocky Linux 설치 문서](https://docs.docker.com/engine/install/rockylinux/)
- [Kubernetes Container Runtime 문서](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

[다음: Kubernetes Installation](04-kubernetes-installation.md)
