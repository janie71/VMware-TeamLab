# 🏢 VMware 기반 듀얼사이트 데이터센터 구축

> 우리FISA 클라우드 엔지니어링 6기 VMware 팀 프로젝트

## 👩🏻‍💻 About Team Members

| <img src="https://github.com/wooxxo.png" width="120"/> | <img src="https://github.com/Kumin-91.png" width="120"/> | <img src="https://github.com/seajihey.png" width="120"/> | <img src="https://github.com/janie71.png" width="120"/> | <img src="https://github.com/Soooonnn.png" width="120"/> | <img src="https://github.com/dbehdrbs0806.png" width="120"/> | 
|:--:|:--:|:--:|:--:|:--:|:--:|
| [**우승연**](https://github.com/wooxxo) | [**최승민**](https://github.com/Kumin-91) | [**서지혜**](https://github.com/seajihey) | [**이유진**](https://github.com/janie71) | [**권순재**](https://github.com/Soooonnn) | [**유동균**](https://github.com/dbehdrbs0806) | 

## 📖 Overview

물리 서버 1대 위에 **Nested Virtualization**으로 Seoul / Jeju **듀얼 사이트 데이터센터**를 구현한 프로젝트입니다.

단일 장애점(SPOF)을 제거하기 위해 **vCenter ELM(Enhanced Linked Mode)**, **VLAN 기반 트래픽 격리**, **vSAN 분산 스토리지**, **HA/DRS 클러스터링**을 적용하였으며, 최종적으로 **vCenter API 기반 VM 프로비저닝 웹 포털**까지 구현하였습니다.

## 💻 Architecture 

### Infra Architecture

![Infra Architecture](./Images/Infra_architecture.png)

### vSAN Architecture

![vSAN Architecture](./Images/vSAN_architecture.png)

## 🌐 Network Design

### ESXi 호스트 배치 및 네트워크 설정

두 개의 독립된 네트워크 대역을 VyOS 라우터로 연결하는 **멀티 사이트 구조**를 설계했습니다.

```
    ┌──────────────────────────────────────────────────────────────────┐
    │                       WAN Transit (KT-WAN)                       │
    │                192.168.30.129 ←──→ 192.168.30.130                │
    ├────────────────────────────────┬─────────────────────────────────┤
    │           Seoul Area           │            Jeju Area            │
    │          10.30.x.x/16          │          172.30.x.x/16          │
    ├────────────────────────────────┼─────────────────────────────────┤
    │  VLAN  0  │ Management         │  VLAN  0  │ Management          │
    │  VLAN 20  │ Storage (iSCSI)    │  VLAN 20  │ Storage (iSCSI)     │
    │  VLAN 30  │ vMotion            │  VLAN 30  │ vMotion             │
    │  VLAN 40  │ Fault Tolerance    │  VLAN 40  │ Fault Tolerance     │
    ├────────────────────────────────┼─────────────────────────────────┤
    │  ESXi-01    10.30.10.10        │  ESXi-01    172.30.10.10        │
    │  ESXi-02    10.30.10.20        │  ESXi-02    172.30.10.20        │
    │  ESXi-03    10.30.10.30        │  ESXi-03    172.30.10.30        │
    │  vCenter    10.30.10.102       │  vCenter    172.30.10.102       │
    │  TrueNAS    10.30.20.40        │  TrueNAS    172.30.20.40        │
    │  DNS        10.30.10.77        │  DNS        172.30.10.77        │
    └────────────────────────────────┴─────────────────────────────────┘
```

### Design Rationale

#### 1. SSO 도메인 통합 및 ELM 기반의 멀티 리전 관리 가용성 확보

* 서울과 제주의 vCenter를 단일 SSO 도메인 (`seoul.seung.fisa`)으로 통합하여 ELM 환경을 구축했습니다.

* 단일 계정 로그인을 통해 두 사이트의 자원을 한 화면에서 제어하는 통합 가시성을 확보했습니다.

* 한쪽 리전의 vCenter 장애 시에도 타 리전의 관리 거점을 통해 인프라 상태를 모니터링하고 즉각 대응할 수 있는 운영 연속성을 마련했습니다.

#### 2. VLAN 기반 트래픽 격리를 통한 성능 최적화 및 보안 강화

* **서비스별 대역 분리:** Management, Storage, vMotion, FT 트래픽을 VLAN 0, 20, 30, 40으로 완전 격리하여 트래픽 간섭을 차단하고 보안성을 높였습니다

* **하이브리드 라우팅 설계:** VyOS 라우터에서 Untagged (Management)와 Tagged (VLAN 20/30/40) 트래픽을 동시에 처리하는 효율적인 서브인터페이스 구조를 채택했습니다.

* **트렁크 최적화:** VyOS 라우터 연결 Port Group을 VLAN 4095로 설정하여, vSwitch 레벨에서 모든 태그된 패킷을 유연하게 수용하도록 구성했습니다.

* **공유 스토리지 가용성:** TrueNAS iSCSI 타겟을 전용 Storage VLAN (20)에 배치하여, 서비스 트래픽과 분리된 안정적인 고속 공유 데이터스토어 환경을 구현했습니다.

#### 3. Nested ESXi 환경 최적화를 위한 계층별 네트워크 설계

* **Base Layer (WS-ESXi)**

    * 물리 호스트는 단일 vSwitch 구성을 통해 물리 NIC 자원을 통합 관리합니다.
    
    * VyOS 라우터가 모든 트래픽을 제어할 수 있도록 VGT (Virtual Guest Tagging) 모드를 적용했습니다.
    
    * 각 서비스별 Port Group에는 VLAN Tagging을 적용하여 논리적 네트워크 분리를 구현했습니다.

* **Nested Layer (ESXi)**

    * 각 Nested ESXi 호스트에는 용도별로 독립된 vNIC을 할당하여 트래픽 간섭을 최소화했습니다.

    * VLAN 대역마다 전용 vSwitch를 1:1로 매핑하는 방식을 채택하여, 중첩 환경 내에서도 실제 물리 서버와 유사한 대역폭 최적화 환경을 구현했습니다.

* **VLAN 태깅 단일화 원칙**

    * Nested ESXi 내부 Port Group의 VLAN을 0으로 설정하여, WS-ESXi에서 이미 태깅된 패킷이 추가로 태깅되는 이중 태깅 문제를 방지했습니다.

    * 이를 통해 중첩 가상화의 고질적인 문제인 이중 태깅에 따른 CPU 오버헤드를 차단하고 통신 효율을 극대화했습니다.

> vSwitch/Port Group 상세 설계, VyOS 라우터 설정, Nested ESXi 매핑 등은 [📄 Day 02](docs/Day_02.md) 에서 확인할 수 있습니다.

## 🛠️ Infrastructure Features

| Feature | 설명 | 적용 목적 |
|:---|:---|:---|
| **Nested Virtualization** | 물리 서버 1대 위에 가상 ESXi 호스트를 중첩 구성 | 제한된 하드웨어에서 멀티 호스트 클러스터 환경 재현 |
| **Enhanced Linked Mode** | 두 vCenter를 동일 SSO 도메인으로 통합 관리 | 한쪽 vCenter 장애 시에도 동일 자격증명으로 VM 관리 가능 |
| **VLAN Segmentation** | Management / Storage / vMotion / FT 트래픽을 VLAN 0, 20, 30, 40으로 격리 | 트래픽 간섭 방지 및 보안성 확보 |
| **vSAN** | 여러 ESXi 호스트의 로컬 디스크를 하나의 분산 스토리지로 통합 (SDS) | Scale-Out 기반 확장, 정책 기반 스토리지 관리 |
| **DRS** | 클러스터 내 호스트 간 리소스 사용량을 모니터링하고 VM을 자동 재배치 | CPU/Memory 로드 밸런싱, Affinity/Anti-Affinity 규칙 적용 |
| **HA** | 호스트 장애 감지 시 해당 호스트의 VM을 다른 호스트에서 자동 재시작 | 단일 장애점 제거, 서비스 연속성 보장 |
| **FT** | Primary VM과 동일한 Secondary VM을 실시간 동기화하여 무중단 보호 | 미션 크리티컬 VM의 다운타임 제로 |
| **vMotion** | 실행 중인 VM을 다른 호스트로 무중단 라이브 마이그레이션 | DRS/HA의 기반 기술, 호스트 유지보수 시 서비스 중단 없이 이동 |
| **Content Library** | VM 템플릿, ISO 이미지를 중앙 저장소에서 관리 및 사이트 간 배포 | 멀티 사이트 간 템플릿 동기화, 표준화된 VM 배포 |
| **iSCSI Shared Storage** | TrueNAS 기반 ZFS Pool을 iSCSI Target으로 공유 | vMotion/HA를 위한 공유 Datastore 환경 구성 |

<br/>

## 📅 Daily Progress

> 각 Day별 상세 구축 과정과 설정 가이드는 아래 링크를 통해 확인할 수 있습니다.

| Day | Title | Key-Points | Docs |
|:--:|:--|:--|:--:|
| 1 | Strategic Planning & Conceptual Design | TBD | [📄 바로가기](./docs/Day_01.md) |
| 2 | Building Core Infrastructure & Management Plane | TBD | [📄 바로가기](./docs/Day_02.md) |
| 3 | Detailing Our Infrastructure - I | TBD | [📄 바로가기](./docs/Day_03.md) |
| 4 | Detailing Our Infrastructure - II | TBD | [📄 바로가기](./docs/Day_04.md) |
| 5 | vSAN and VM Provisioning Portal | TBD | [📄 바로가기](./docs/Day_05.md) |

## ⚠️ Troubleshooting

프로젝트 진행 중 겪은 주요 이슈와 해결 과정을 정리했습니다.

**Nested ESXi VLAN 이중 태깅 이슈**

- **문제**: Nested ESXi 내부 Port Group에 VLAN을 설정하자 통신 불가  

- **원인**: WS-ESXi Port Group에서 이미 태깅된 패킷에 Nested ESXi가 추가 태깅 → 이중 태깅 발생
- **해결**: Nested ESXi 내부 모든 Port Group의 VLAN을 0으로 설정 (L2 태깅 단일화 원칙 적용)

**Windows VM 템플릿에서 Sysprep 미실행 이슈**

- **문제**: 템플릿에서 복제한 Windows VM들이 Sysprep 일반화 과정을 수행하지 않아, 클론된 VM에서 SID 충돌 및 네트워크 설정 오류 발생

- **원인**: 전체 사용자용으로 등록되지 않은 특정 언어팩 패키지가 Sysprep의 일반화 과정을 방해함

- **해결**: 템플릿 생성 전 PowerShell 명령어로 해당 언어팩을 모든 사용자에서 제거 → Sysprep 재실행하여 일반화 성공

> 각 이슈의 상세 분석은 Day별 문서에서 확인할 수 있습니다. → [Day 02](./docs/Day_02.md) | [Day 03 - Windows 10 Sysprep](./Troubles/Day_03_Windows_10_Sysprep.md)




---

## 👩🏻‍💻 LAB - VMware VM Provisioning Portal

![vmweb1.png](./Images/vmweb1.png)
![vmweb2.png](./Images/vmweb2.png)

Spring Boot 기반으로 구현한 **VMware 가상머신 프로비저닝 자동화 웹 포털**입니다.

**vCenter, ESXi, 네트워크, 스토리지로 구성된 인프라 구조를 학습한 뒤 이를 실제 서비스 형태로 적용**해보기 위해 진행했습니다.<br> 

기존에는 vCenter UI에서 템플릿 선택, CPU/Memory 설정, 네트워크 구성 등의 작업을 수동으로 수행해야 했지만,
이를 **웹 요청 → API 기반 자동화 흐름으로 전환**하여 반복 작업을 단순화했습니다.<br>

VM 생성 이후에는 **Apache Guacamole과 자동으로 연결되어 화면을 미러링 할 있는 구조**를 구현했습니다. 

---

### 🔍 Overview

* vCenter에서 수행하던 작업을 **API 단위로 분해하여 자동화**
* VM 생성부터 접속까지의 흐름을 **하나의 서비스로 통합**

---

### ⚙️ Key Features

1. **vCenter API 기반 VM 자동 생성**

  * Datacenter / Cluster / Network / Datastore / Template 검증
  * Template 기반 VM Clone

2. **리소스 및 네트워크 설정 자동화**

  * CPU / Memory 설정
  * Guest Customization 기반 IP 설정

3. **원격 접속 자동 연결**

  * Guacamole API 기반 접속 URL 생성

4. **배포 이력 관리**

  * VM 생성 요청 및 결과를 DB에 저장하여 추적 가능

---

### 🔄 Flow 및 상세

```text
Web UI → Spring Boot → vCenter API → VM 생성 → Guacamole → 접속 URL 반환
```
 

* 현재 배포 환경이 아니기에 Rocky Linux VM 배포를 기준으로 구성했습니다.

* ✨ 해당내용은 [VMwareWeb](https://github.com/seajihey/VMwareWeb)을 참고하세요

## 📂 Project Structure
```
VMware-TeamLab/
├── README.md              # 프로젝트 메인 (현재 문서)
├── docs/
│   ├── Day01.md           # Day 01 - 설계 및 초기 구성
│   ├── Day02.md           # Day 02 - 핵심 인프라 구축
│   ├── Day03.md           # Day 03 - 인프라 상세 설정 I
│   ├── Day04.md           # Day 04 - 인프라 상세 설정 II
│   └── Day05.md           # Day 05 - vSAN 및 웹 포털
├── Images/                # 아키텍처 다이어그램 및 스크린샷
├── Troubles/              # 트러블슈팅 기록
└── .gitignore
```

## 📚 Tech Stacks

| Category | Technologies |
|:---|:---|
| **Hypervisor** | VMware ESXi 7.0 |
| **Management** | vCenter Server 7.0.3 |
| **Network** | VyOS Router, VLAN (802.1Q), Static Routing, NAT |
| **Storage** | TrueNAS (iSCSI, ZFS), vSAN |
| **DNS** | Windows Server 2022 |
| **Application** | Spring Boot, Apache Guacamole |
| **OS** | Ubuntu, Rocky Linux |
