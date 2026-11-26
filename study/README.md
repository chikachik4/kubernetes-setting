# Kubernetes 구축 스터디

NCP에서 kubeadm 기반 Kubernetes 클러스터를 구축한 뒤, 각 설정의 목적과 동작 원리를 복습하기 위한 학습 기록이다.

구축 가이드는 **어떻게 설정하는가**, 스터디 문서는 **왜 설정하는가, 문제가 생기면 무엇을 확인하는가**에 집중한다. 명령어 암기보다 개념 이해와 장애 분석 능력을 목표로 한다.

## 학습 목차

| 단원 | 학습 내용 | 상태 |
|---|---|---|
| 01 | [Infrastructure & Network](01-infrastructure.md) | 학습 완료 |
| 02 | OS Preparation — Swap, SELinux, firewalld, Kernel Module, sysctl | 학습 예정 |
| 03 | Container Runtime — CRI, OCI, containerd, runc, cgroup | 학습 예정 |
| 04 | Kubernetes Components — kubeadm, kubelet, kubectl, Control Plane | 학습 예정 |
| 05 | Control Plane Initialization — Static Pod, 인증서, kubeconfig | 학습 예정 |
| 06 | CNI & Calico — Pod CIDR, VXLAN, IPAM | 학습 예정 |
| 07 | Worker Join — Bootstrap Token, 노드 등록 | 학습 예정 |
| 08 | Cluster Verification — Pod, Service, DNS, EndpointSlice | 학습 예정 |

## 학습 방법

1. 기존 구축 가이드에서 실제 수행한 작업을 확인한다.
2. 개념과 설정의 목적을 이해한다.
3. 실제 실습 환경에 적용하여 생각한다.
4. 복습 문제를 문서 없이 풀고, 부족한 개념을 다시 확인한다.

학습을 마친 단원부터 순차적으로 문서를 추가한다.

[구축 가이드로 돌아가기](../README.md)
