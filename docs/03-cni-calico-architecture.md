---

# 🚀 1.5단계: CNI (Calico) 오버레이 네트워크 설계 및 GitOps 배포

본 문서는 Kubernetes 클러스터의 데이터 플레인 네트워크를 담당하는 CNI(Container Network Interface) 플러그인인 **Calico**의 아키텍처 설계와 배포 과정을 기록합니다. 단순 명령형(Imperative) 배포를 지양하고, 선언적(Declarative) 인프라 관리를 위한 **Tigera Operator** 패턴을 적용했습니다.

---

## 1. Tigera Operator 배포 패턴 채택 이유

과거에는 `calico.yaml` 단일 파일을 통해 DaemonSet과 Deployment를 직접 배포하는 방식이 주를 이루었습니다. 하지만 본 프로젝트에서는 **Tigera Operator** 방식을 채택했습니다.

* **Day-2 운영 및 수명주기 관리:** Operator는 K8s 내부에서 독립적인 컨트롤러로 동작하며, Calico 컴포넌트(Node, Typha, Kube-controllers 등)의 상태를 지속적으로 모니터링하고 자가 치유(Self-healing) 및 무중단 업그레이드를 자동화합니다.
* **GitOps 친화적 구조:** 인프라 관리자는 수천 줄의 복잡한 매니페스트를 직접 수정할 필요 없이, 소수의 핵심 설정만 담긴 `Installation` Custom Resource(CR) 하나만 Git으로 관리하면 됩니다. 나머지는 Operator가 알아서 K8s API와 통신하여 리소스를 전개합니다.

<br/>

---

## 2. 매니페스트 구성 및 배포 파이프라인

Git 저장소의 `cni/calico/` 디렉토리는 아래 3개의 파일로 구성되며, 반드시 정해진 순서대로 배포되어야 합니다.

### 2-1. `v1_crd_projectcalico_org.yaml` (1순위)

* **역할:** K8s가 기본적으로 알지 못하는 Calico만의 고유한 리소스(BGPConfiguration, IPPool, GlobalNetworkPolicy 등) 문법을 K8s API 서버에 학습시키는 **CRD(Custom Resource Definition)** 파일입니다.

<br/>

### 2-2. `tigera-operator.yaml` (2순위)

* **역할:** `tigera-operator` 네임스페이스를 생성하고, Operator 파드와 이에 필요한 권한(RBAC)을 부여합니다.
* **핵심 기술 포인트 (`hostNetwork: true`):**
Operator 파드의 스펙을 보면 `hostNetwork: true`로 설정되어 있습니다. 아직 CNI가 설치되지 않아 파드 네트워크(172.20.x.x)가 존재하지 않는 클러스터 초기 상태에서도, Operator가 노드의 물리 네트워크(172.16.x.x)를 직접 빌려 써서 K8s API 서버와 통신하고 Calico 컴포넌트를 배포할 수 있게 만드는 핵심 설정(Chicken-and-Egg 문제 해결)입니다.

<br/>

### 2-3. `custom-resources.yaml` (3순위)

* **역할:** Operator에게 "우리 환경에 맞게 Calico를 이렇게 세팅해 줘"라고 내리는 최종 작업 지시서(CR)입니다.

<br/>

---

## 3. 네트워크 아키텍처 튜닝 세부 분석 (`custom-resources.yaml`)

아래는 본 프로젝트에 적용된 핵심 설정과, 실무(Enterprise) 환경에서 아키텍처 요구사항에 맞춰 조정할 수 있는 Calico IPPool의 주요 파라미터 분석입니다.

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 172.20.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()

```

### ① 망 분리 및 대역 격리 (`cidr: 172.20.0.0/16`)

* **설정 이유:** 호스트(VM)가 위치한 홈랩 관리망의 대역은 `172.16.0.0/24`입니다. K8s 파드 네트워크 대역이 이와 겹치면 패킷이 목적지를 찾지 못하는 라우팅 충돌(Routing Overlap)이 발생합니다. 이를 원천 차단하기 위해 파드 전용 대역을 `172.20.0.0/16`으로 격리했습니다. `kubeadm init` 시의 `--pod-network-cidr` 설정과 반드시 일치해야 합니다.

### ② 노드 단위 IP 할당 블록 (`blockSize: 26`)

* **동작 원리:** Calico는 개별 파드가 생성될 때마다 etcd에 접근하여 IP를 할당하지 않습니다. K8s 노드당 `/26` (64개의 IP) 블록 단위로 IP 대역을 미리 잘라서 할당합니다. 한 노드에서 64개의 파드가 꽉 차면 새로운 `/26` 블록을 추가로 할당받습니다.
* **실무 튜닝 포인트:** 노드 하나에 수백 개의 초경량 파드가 띄워지는 고밀도 환경이라면 `/24` (256개)로 블록 크기를 늘려 블록 고갈 및 할당 오버헤드를 줄입니다. 반대로 노드 수는 매우 많지만 노드당 파드 수가 적다면, IP 낭비를 막기 위해 `/28` (16개) 등으로 크기를 줄여 IP 풀을 효율적으로 관리합니다.

### ③ 네트워크 캡슐화 프로토콜 (`encapsulation: VXLAN`)

* **VXLAN (적용됨):** 파드의 트래픽을 호스트가 내보내기 직전에 표준 UDP 패킷(Port 4789)으로 한 번 더 감싸서 터널링하는 방식입니다. 물리 장비(L3 스위치)의 BGP 라우팅 개입 없이도 파드 간 통신이 가능하므로 홈랩이나 퍼블릭 클라우드(AWS, GCP 등) 환경에서 표준으로 사용됩니다.
* **IPIP (대안):** IP 패킷 안에 IP 패킷을 넣는 고전적인 터널링 방식입니다. Azure 등 일부 퍼블릭 클라우드 환경에서는 IPIP 패킷을 방화벽에서 강제로 드랍(Drop)하는 경우가 있어 최근에는 VXLAN으로 대체되는 추세입니다.
* **None (대안 - BGP Native):** 캡슐화를 아예 하지 않고 파드의 원본 IP를 그대로 물리망에 태워 보냅니다. CPU 캡슐화 오버헤드가 0이므로 **최고의 네트워크 성능**을 냅니다. 단, 이를 위해선 물리 L3 스위치가 K8s 노드들과 BGP 프로토콜을 맺고 라우팅 테이블을 직접 교환하도록 인프라 장비 레벨의 설정이 동반되어야 합니다. (엔터프라이즈 온프레미스의 지향점)

### ④ 캡슐화 범위 제어 (옵션: `vxlanMode` / `ipipMode`)

* 기본적으로 위 설정 파일에는 명시되지 않아 `Always`(모든 파드 간 통신 시 캡슐화)로 동작합니다.
* **CrossSubnet (실무 튜닝 포인트):** 물리적 네트워크 대역(Subnet)이 다른 노드끼리 통신할 때만 캡슐화를 수행하고, 같은 L2 스위치 하단에 있는 노드끼리는 캡슐화 없이 직접 통신(Native)하도록 지시하는 하이브리드 옵션입니다. 불필요한 패킷 캡슐화 연산을 줄여 노드의 CPU 자원을 절약하고 트래픽 처리 속도를 향상시킬 수 있습니다.

### ⑤ 외부 통신 허용 (`natOutgoing: Enabled`)

* **동작 원리:** 파드가 외부 인터넷이나 사내 레거시망(예: 외부 DB API)으로 통신을 시도할 때, K8s 내부용 사설 IP(`172.20.x.x`)를 노드의 물리 IP(`172.16.x.x`)로 변환(Source NAT)하여 내보냅니다.
* **실무 튜닝 포인트:** 외부망 연결이 완벽하게 차단된 폐쇄망(Air-gapped) 환경이나, K8s 파드의 IP가 사내망 방화벽 규칙에 직접 등록되어 원본 IP를 그대로 유지해야 하는 특수 보안 환경에서는 이 옵션을 `Disabled`로 설정합니다.

### ⑥ 노드 그룹별 IP 풀 분리 (`nodeSelector: all()`)

* **동작 원리:** 이 IPPool 설정이 K8s 클러스터 내의 어떤 노드들에 적용될지 레이블(Label) 기반으로 결정합니다. `all()`은 클러스터 내 전체 노드를 대상으로 하나의 대역을 공유함을 의미합니다.
* **실무 튜닝 포인트 (멀티 테넌시/망 분리):** 클러스터 내에 '외부 노출용 DMZ 노드 그룹'과 '내부 DB용 노드 그룹'이 섞여 있을 때 사용합니다. DMZ 노드에는 `172.20.10.0/24` 풀을, 내부 노드에는 `172.20.20.0/24` 풀을 할당하도록 각각 다른 `Installation` CRD를 배포하여, K8s 내부에서도 물리적인 IP 대역 분리를 구현할 수 있습니다.

---
