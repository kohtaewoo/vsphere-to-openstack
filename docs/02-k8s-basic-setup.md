---

# 🚀 1단계: Kubernetes 핵심 인프라 (Rocky Linux) 기본 구축

본 문서는 Rocky Linux 9 기반의 베이스 VM 생성부터 Kubernetes Control-plane 및 Worker 노드 구성, 그리고 CNI(Calico) 설치까지의 과정을 기록합니다. 

---

## 1. 베이스 VM 템플릿 구성 및 기본 설정

### 1-1. Rocky Linux 설치 스펙

* **CPU:** 2~4 Core
* **RAM:** 4GB 이상
* **Disk:** 30GB
* **네트워크:** 관리망 대역 (172.16.0.0/24)

### 1-2. 패키지 업데이트 및 시간 동기화

K8s 클러스터는 노드 간 API 통신, 인증서(Certificate) 만료 검증, 분산 로그 기록이 핵심이므로 완벽한 시간 동기화가 필수적입니다.

```bash
dnf update -y
dnf install chrony -y
systemctl enable --now chronyd
timedatectl set-timezone Asia/Seoul

# 시간 동기화 정상 작동 확인
chronyc tracking

```

---

## 2. K8s 구동을 위한 커널 및 OS 튜닝 (핵심)

패킷을 직접 라우팅하고 제어하는 "가상의 네트워크 인프라"로 동작시키기 위한 필수 커널 튜닝 과정입니다.

### 2-1. Swap 비활성화

* **명령어:** `swapoff -a` 및 `/etc/fstab` 수정 (swap 라인 주석 처리)
* **K8s 아키텍처 철학:** K8s의 스케줄러(kube-scheduler)는 노드의 가용 메모리 상태를 정확히 계산하여 파드를 배치해야 합니다. Swap이 켜져 있으면 시스템이 메모리 부족 상태를 디스크 페이징으로 숨기게 되며, 이는 극심한 디스크 I/O 병목(성능 저하)을 유발합니다. K8s는 "느려진 상태로 좀비처럼 살아있는 것보다, 메모리가 부족하면 깔끔하게 OOM(Out of Memory)으로 파드를 죽이고 새로 띄우는 것(Fail-Fast)"을 지향합니다.

### 2-2. SELinux 및 Firewall 비활성화

* **명령어:** `setenforce 0`, `/etc/selinux/config` (SELINUX=disabled), `systemctl disable firewalld --now`
* **이유:** K8s는 수많은 컨테이너 네트워크 인터페이스(CNI)와 가상 볼륨(HostPath, CSI 등)을 쉴 새 없이 동적으로 생성하고 삭제합니다. 이 과정에서 정적인 OS 보안 모듈(SELinux, Firewalld)의 규칙과 충돌하여 권한 거부 오류가 잦게 발생하므로, CNI(Calico/Cilium) 자체의 네트워크 정책으로 보안을 통제하기 위해 기본 OS 방화벽은 비활성화합니다.


### 2-3. 커널 모듈(Kernel Module) 적재

* **명령어:**

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

```

* **동작 원리 및 적용 이유:**

#### 1) `overlay` (OverlayFS 파일시스템)

* **미적용 시 문제점:** K8s는 컨테이너(파드)마다 독립적인 파일시스템을 요구합니다. OverlayFS가 없다면 파드를 하나 생성할 때마다 수 GB에 달하는 OS 베이스 이미지를 디스크에 통째로 복사(Full Copy)해야 하므로, 디스크 용량이 즉시 고갈되고 I/O 병목으로 인해 파드 생성 속도가 심각하게 지연됩니다.
* **동작 원리:** 여러 개의 디렉토리를 수직으로 쌓아 단일 파일시스템처럼 마운트하는 기술입니다.
* **Lowerdir (읽기 전용):** 컨테이너의 베이스 이미지 공간입니다. 노드에 생성되는 수십 개의 파드가 하나의 Lowerdir을 공유하여 디스크 공간을 절약합니다.
* **Upperdir (쓰기 가능):** 개별 파드에 할당되는 고유의 빈 공간입니다.
* **CoW (Copy-on-Write) 매커니즘:** 파드가 데이터를 읽을 때는 공유된 Lowerdir을 바로 읽습니다. 기존 파일의 수정이 발생할 때만, 해당 파일을 Upperdir로 복사(Copy)한 뒤 수정(Write)합니다. 이를 통해 파드 생성 시 복사 작업을 생략하고, 즉각적인 구동(밀리초 단위)을 가능하게 합니다.
<img width="1280" height="874" alt="Image" src="https://github.com/user-attachments/assets/698a5639-1c74-464a-8dd1-d9a62effbe89" />

#### 2) `br_netfilter` (브리지 네트워크 필터링)

* **미적용 시 문제점:** 리눅스 커널의 네트워크 스택은 L2(MAC 기반 브리지 스위칭)와 L3/L4(IP/Port 기반 라우팅 및 iptables 방화벽) 계층이 분리되어 있습니다. 동일 노드 내에서 파드 A가 파드 B와 통신할 때 트래픽은 가상 브리지(L2)를 통해 스위칭됩니다. 성능 최적화를 위해 리눅스는 기본적으로 L2 브리지를 지나는 패킷을 L3/L4 방화벽(iptables)으로 보내지 않고 즉시 포워딩합니다.
* **동작 원리:** L2 계층의 브리지 코드와 상위 계층의 Netfilter(iptables) 코드 사이에 훅(Hook)을 생성하는 커널 모듈입니다. 브리지를 통과하는 패킷을 강제로 낚아채어 상위 iptables 방화벽 체인으로 전달합니다.
* **K8s 적용 이유:** K8s의 핵심인 Service(ClusterIP를 타겟 파드 IP로 변환하는 NAT)와 NetworkPolicy(특정 트래픽 차단)는 오직 iptables 규칙을 기반으로 동작합니다. 이 모듈이 로드되지 않으면 동일 노드 내의 파드 트래픽이 iptables를 우회해버리므로, 보안 정책이 무력화되거나 서비스 라우팅이 불가능해집니다.



### 2-4. 네트워크 파라미터 튜닝 (sysctl)

* **명령어:**
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

```


* **동작 원리 및 적용 이유:**
* **`net.ipv4.ip_forward = 1` (패킷 포워딩)**
* **디폴트(0)인 이유:** 일반적인 리눅스 운영체제는 네트워크의 '종단점(End-Host)'으로 동작하도록 설계되어 있습니다. 자신의 IP가 목적지가 아닌 패킷을 수신하면, 보안 유지 및 무한 라우팅 루프 방지를 위해 패킷을 즉시 폐기(Drop)합니다.
* **설정 이유:** K8s 노드는 내부에 별도의 가상 IP 대역(예: 172.20.x.x)을 가진 수많은 파드를 품고 있는 일종의 '가상 라우터' 역할을 수행해야 합니다. 물리 NIC로 들어온 트래픽이 노드 내부의 파드 가상 인터페이스로 전달(Forwarding)되게 하려면 이 라우팅 기능을 강제로 활성화해야 합니다.


* **`net.bridge.bridge-nf-call-iptables = 1` (브리지 트래픽 방화벽 연동)**
* **디폴트(0)인 이유:** L2 계층의 단순 브리지 스위칭 작업 중인 트래픽을, 굳이 L3/L4 계층의 iptables로 넘겨 방화벽 규칙을 검사하게 만들면 패킷 처리 오버헤드와 지연(Latency)이 발생합니다. 따라서 성능 확보를 위해 리눅스의 기본값은 이 기능이 비활성화되어 있습니다.
* **설정 이유:** K8s의 핵심 네트워크 기능인 서비스(ClusterIP, NodePort) 통신과 NetworkPolicy(L4 보안 통제)는 패킷의 NAT(네트워크 주소 변환) 및 필터링 처리를 커널의 iptables 규칙에 전적으로 의존합니다. 설정값을 1로 변경하여, L2 브리지를 지나는 파드 통신 트래픽이 반드시 커널 방화벽 규칙을 통과하도록 강제해야만 K8s의 논리적 네트워크 라우팅이 정상 동작합니다.

---

## 3. 컨테이너 런타임 & K8s 패키지 설치

### 3-1. Containerd 설치 및 Cgroup 드라이버 동기화

* **명령어:** Docker 레포지토리 추가 및 `containerd.io` 설치
* **튜닝:** `/etc/containerd/config.toml`에서 `SystemdCgroup = false`를 `true`로 변경
* **이유:** 리눅스 환경의 기본 자원(CPU, Memory) 관리자는 `systemd`입니다. 하지만 Containerd의 기본 설정은 `cgroupfs`를 사용합니다. 자원 통제 주체가 2개로 나뉘면 충돌(스케줄링 오작동, OOM 추적 실패)이 발생하므로, K8s 권장 사항에 따라 모든 자원 관리를 `systemd` 하나로 통일시킵니다.

### 3-2. Kubeadm, Kubelet, Kubectl 설치 (버전 통제)

* **명령어:** K8s 공식 레포지토리(`pkgs.k8s.io`) 등록 후, `--disableexcludes=kubernetes` 옵션을 주어 패키지 설치
* **이유:** K8s는 마이너 버전 불일치에도 시스템이 붕괴할 수 있을 만큼 버전 호환성에 극도로 민감합니다. 따라서 레포지토리 설정에 `exclude=kubelet kubeadm kubectl`을 미리 걸어두어 평소 `dnf update` 시 K8s 패키지가 멋대로 업데이트되는 것을 원천 차단하고, 설치 시에만 예외 옵션을 주어 안전하게 배포합니다.

---

## 4. 클러스터 프로비저닝 (VM 복제 및 초기화)

### 4-1. VM 복제 및 네트워크 맵핑

베이스 VM 종료 후 Master 1대, Worker 2대로 복제합니다.

* **명령어:** `hostnamectl set-hostname [이름]`
* **/etc/hosts 맵핑:** K8s 내부 컴포넌트들이 IP가 아닌 도메인 이름으로 서로를 찾을 수 있도록 3대의 IP와 호스트네임을 모두 등록합니다. (예: 172.16.0.10 k8s-master)

### 4-2. Master 노드 초기화 (네트워크 대역 분리)

* **명령어:** `kubeadm init --pod-network-cidr=172.20.0.0/16`
* **이유:** 홈랩의 관리망(VM IP) 대역이 `172.16.0.0/24`입니다. Calico의 기본 설정인 `192.168.0.0/16`을 그대로 쓰거나 관리망과 겹치게(Overlap) 설정하면 패킷이 길을 잃는 라우팅 충돌(Split-Brain)이 발생합니다. 따라서 파드 전용 네트워크를 `172.20.0.0/16`으로 완전히 격리하여 충돌 장애점을 사전 제거합니다.

---

## 5. CNI (Calico) GitOps 기반 설치

단순한 튜토리얼용 `.yaml` 배포 방식(명령형)을 버리고, 향후 무중단 업그레이드 및 선언적 인프라(GitOps) 관리를 위해 **Tigera Operator** 방식을 채택합니다.

* **1단계:** Calico CRD(Custom Resource Definition) 및 Operator 배포
* **2단계:** Custom Resource(`custom-resources.yaml`) 튜닝 적용
* `cidr: 172.20.0.0/16`: 앞서 `kubeadm init`에서 설계한 대역과 정확히 일치시킵니다.
* `encapsulation: VXLAN`: 물리 L3 스위치의 BGP 프로토콜을 제어할 수 없는 홈랩 환경의 한계를 극복하기 위해, 패킷을 한 번 더 감싸서 통신하는 오버레이(VXLAN) 터널링 방식을 채택합니다.
* `natOutgoing: Enabled`: 내부 파드가 외부 인터넷과 통신할 수 있도록 SNAT를 활성화합니다.


* **결과 검증:** `kubectl get nodes`를 통해 모든 노드가 `Ready` 상태로 전환되는 것을 확인합니다.
