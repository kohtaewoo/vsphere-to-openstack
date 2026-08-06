# 🚀 1.6단계: CNI (Cilium) eBPF 기반 네트워크 설계 및 배포

본 문서는 Kube-proxy를 대체하는 eBPF 기반의 고성능 CNI인 **Cilium**의 아키텍처 설계와 배포 과정을 기록합니다.

## 1. Cilium 도입 배경 및 Calico와의 아키텍처 차이점

기존 Calico 환경과 신규 Cilium 환경의 가장 큰 차이는 **Kube-proxy의 유무**와 패킷 처리 계층(eBPF)에 있습니다.

- **Calico (iptables 기반):** K8s의 기본 컴포넌트인 `kube-proxy`에 의존합니다. 서비스와 파드가 증가할수록 OS의 iptables 룰(Rule)이 늘어나며, 순차 탐색 방식($O(N)$)으로 인해 트래픽 라우팅 지연(Latency)과 CPU 오버헤드가 발생합니다.
- **Cilium (eBPF 기반):** `kube-proxy`를 클러스터에서 완전히 제거합니다. 대신 Linux 커널의 **eBPF(Extended Berkeley Packet Filter)** 기술을 사용하여 커널 레벨에서 네트워크 패킷을 직접 통제합니다. 해시 테이블 방식($O(1)$)으로 패킷을 처리하여 노드 및 서비스 확장 시에도 일관된 고성능과 안정성을 보장합니다.

## 2. 배포 파이프라인 설계 (Helm + Values.yaml)

Cilium은 **Helm 패키지 매니저와 사용자 정의 `values.yaml` 파일**을 결합한 방식으로 배포합니다.

- **단일 설정 파일 관리:** 모든 네트워크 튜닝 옵션을 `cni/cilium/cilium-values.yaml` 단일 파일에 선언하여 Git 저장소에 기록합니다.
- **GitOps 친화성 보장:** 향후 ArgoCD 등의 지속적 배포(CD) 도구 연동 시, 해당 `values.yaml` 파일의 형상만 추적하여 클러스터 상태를 자동으로 동기화할 수 있도록 설계했습니다.

## 3. 네트워크 아키텍처 튜닝 세부 분석 (`cilium-values.yaml`)

아래는 본 프로젝트에 적용된 핵심 설정과, 해당 파라미터들이 시스템 아키텍처에 미치는 영향에 대한 상세 분석입니다.

YAML

```
kubeProxyReplacement: true
k8sServiceHost: 172.16.0.21
k8sServicePort: 6443
ipam:
  mode: kubernetes
```

### ① Kube-proxy 완전 대체 (`kubeProxyReplacement: true`)

- **동작 원리:** K8s 클러스터 내부의 Service 라우팅(ClusterIP, NodePort, LoadBalancer 등)을 처리하던 `kube-proxy` 데몬을 비활성화하고, 그 역할을 Cilium 에이전트가 eBPF 프로그램을 통해 직접 수행하도록 권한을 이관합니다.
- **적용 사유:** `kubeadm init` 단계에서 `-skip-phases=addon/kube-proxy` 옵션을 사용하여 원천적으로 Kube-proxy를 생성하지 않았습니다. 이 설정이 활성화되어야만 클러스터 내부의 정상적인 서비스 디스커버리 및 통신이 가능해집니다. 이를 통해 시스템 리소스 사용량을 최적화합니다.

### ② API 서버 엔드포인트 직접 지정 (`k8sServiceHost` / `k8sServicePort`)

- **동작 원리:** 각 노드에 배포되는 Cilium 데몬셋(DaemonSet)이 K8s Control Plane(API 서버)과 통신하기 위한 목적지 IP(`172.16.0.21`)와 포트(`6443`)를 하드코딩 방식으로 주입합니다.
- **적용 사유:** Kube-proxy가 존재하지 않는 환경에서는 워커 노드의 에이전트들이 API 서버의 위치를 동적으로 알아낼 방법이 없습니다. 따라서 물리적인 마스터 노드의 IP를 명시하여 초기 네트워크 구성 시 발생할 수 있는 통신 데드락(Deadlock) 상태를 사전에 방지합니다.

### ③ 파드 IP 할당 위임 (`ipam.mode: kubernetes`)

- **동작 원리:** IPAM(IP Address Management) 역할을 Cilium 자체 컨트롤러가 아닌, K8s의 기본 `kube-controller-manager`에게 위임합니다.
- **적용 사유:** 두 클러스터 간의 네트워크 대역 겹침을 방지하기 위해 `kubeadm init` 시 신규 클러스터의 Pod CIDR을 `172.21.0.0/16`으로 격리 설계했습니다. `kubernetes` 모드를 사용하면 K8s 컨트롤러가 이 대역을 각 노드 단위로(`/24`) 분할하여 할당하는 기존의 논리적 라우팅 체계를 Cilium이 그대로 상속받아 사용하게 됩니다. 이를 통해 클러스터 관리의 일관성을 유지합니다.
