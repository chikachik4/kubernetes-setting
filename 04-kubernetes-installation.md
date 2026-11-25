# 04. Kubernetes Installation

**실행 대상: 전체 노드.** kubeadm은 초기화/join, kubelet은 노드의 Pod 관리, kubectl은 API 접근 CLI다. Worker에서 kubectl은 필수가 아니지만 실습에서는 동일하게 설치했다.

## 1. Repository 등록

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<'CONF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
CONF
dnf repolist
```

`baseurl`은 minor 버전별 저장소, `gpgcheck`는 패키지 서명 검증, `exclude`는 일반 OS 업데이트로 Kubernetes가 의도치 않게 변경되는 것을 막는다.

## 2. 패키지 설치

```bash
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

`--disableexcludes=kubernetes`는 이번 설치에만 exclude를 무시한다. 실습에서는 v1.36.6이 설치됐다. 동일 버전을 요구하면 먼저 `dnf list --showduplicates kubeadm kubelet kubectl --disableexcludes=kubernetes`로 제공 버전을 확인하고 명시적으로 동일 버전을 선택한다.

```bash
rpm -q kubeadm kubelet kubectl
kubeadm version
kubelet --version
kubectl version --client
```

세 노드 버전을 맞춘다. 다음 문서의 `--kubernetes-version`도 실제 선택한 버전과 일치시킨다.

## 3. kubelet 활성화

```bash
systemctl enable --now kubelet
systemctl status kubelet --no-pager
```

아직 init/join 전에는 kubelet 설정이 없어서 auto-restart 상태일 수 있다. 클러스터 구성 전의 실패와 실제 초기화 후 장애를 구별한다.

- [kubeadm 설치 공식 가이드](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[다음: Control Plane Initialization](05-control-plane-init.md)
