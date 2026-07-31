# 🏆 vSphere to OpenStack 하이브리드 인프라 마이그레이션
vSphere 환경에서 Kubernetes 클러스터를 구축하고, OpenStack 기반 프라이빗 클라우드로 V2V 마이그레이션하는 프로젝트입니다.
<br/>
<br/>

## 🗺️ 프로젝트 로드맵
<details>

<summary><strong> 가상화 인프라 구성 및 네트워크 망 분리</strong></summary>

**[👉 ESXi 네트워크 설계 및 pfSense VPN 구축 상세 기록 보러가기](./docs/01-vmware-pfsense-network.md)**
- **ESXi vSwitch 망 분리:** 내부망 전용 vSwitch-LAN 생성 및 pfSense 방화벽을 통한 라우팅.
- **pfSense & WireGuard VPN:** 홈랩 대역(172.16.x.x) 및 외부망 라우터 대역(192.168.45.x) 트래픽 라우팅 분리 및 방화벽 룰 적용.
</details>

<details>
  
<summary><strong> Kubernetes 클러스터 구축 및 네트워크 최적화</strong></summary>

**[👉 K8s 기본 튜닝 및 구축 상세 기록 보러가기](./docs/02-k8s-basic-setup.md)**
- **kubeadm 클러스터 구성:** Master 1대, Worker 2대 구성. Operator 패턴을 활용한 CNI(Calico) 배포.
- **etcd 데이터 백업/복원:** etcdctl 스냅샷 백업 구성 및 특정 네임스페이스 강제 삭제 후 복구(Restore) 테스트 검증.
- **네트워크 및 DNS 최적화:** Kube-proxy iptables와 IPVS 성능 비교 및 fortio 기반 패킷 지연 시간(Latency) 측정. CoreDNS ndots:5 설정 트래픽 오버헤드 추적 및 dnsConfig 패치 적용.
</details>

<details>

<summary><strong> CNI (Calico) 오버레이 네트워크 설계 및 GitOps 배포</strong></summary>

**[👉 Calico 아키텍처 튜닝 및 GitOps 배포 상세 기록 보러가기](./docs/03-cni-calico-architecture.md)**
- **Tigera Operator 도입:** 명령형(Imperative) 배포를 지양하고, 무중단 업그레이드 및 수명주기 관리가 가능한 선언적(Declarative) 인프라 구성.
- **네트워크 대역 격리 및 IPPool 최적화:** 관리망(172.16.x.x)과의 라우팅 충돌 방지를 위한 대역 분리(172.20.0.0/16) 및 노드 단위 IP 할당 블록(blockSize: 26) 적용.
- **오버레이 캡슐화(VXLAN):** 물리 스위치 장비의 BGP 연동 없이 파드 간 통신을 처리하는 표준 터널링 아키텍처 구현.
- **외부 통신 제어:** `natOutgoing` 설정을 통해 파드의 사설 IP를 노드의 물리 IP로 변환(SNAT)하여 아웃바운드 인터넷 통신 허용.

</details>

<details>

<summary><strong> CNI (Cilium) 기본 설정 차이 및 iptables와 IPVS 차이</strong></summary>

</details>


<details>

<summary><strong> 쿠버네티스 네트워크 흐름 with OS</strong></summary>

</details>

<details>

<summary><strong> 쿠버네티스 자원 with OS</strong></summary>

</details>


## 🛠️ Directory Structure
- **`cni/`** : K8s 데이터플레인 네트워크 매니페스트 (Calico, Cilium)
- **`apps/`** : 클러스터 배포용 애플리케이션 리소스 (ArgoCD, Harbor 등 자동화 타겟)
- **`docs/`** : 원인-조치-재발 방지(CAP) 사이클에 맞춘 인프라 튜닝 및 트러블슈팅 문서
<br/>
