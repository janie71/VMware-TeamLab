# 🤝 Day 01 - Strategic Planning & Conceptual Design

> **2026.03.06 금요일**

## 1. 목표

단일 장애점을 제거한 **고가용성 이중화 데이터센터** 구축을 최종 목표로, Day 01에서는 전체 아키텍처 설계 및 ESXi 초기 환경 구성을 진행했습니다.

ESXi 클러스터 → vCenter 이중화 → VLAN 분리 → vSAN 기반 분산 스토리지로 이어지는 전체 구조를 설계했습니다.

## 2. 왜 클러스터 + 이중 vCenter 구조로 설계하였는가?

두 개의 네트워크 대역 (`172.30.x.x/16`, `10.30.x.x/16`) 으로 분리하고, 각 대역 간 VyOS 라우터로 통신하는 구조를 채택했습니다.

단일 vCenter 장애 시 전체 관리 불가 문제를 방지하기 위해 SSO 도메인을 통일, 한쪽 vCenter가 다운되어도 나머지 vCenter에서 동일한 자격증명으로 VM 관리가 가능하도록 설계했습니다.

## 3. ESXi 설치

물리 서버 1대에 USB 부팅으로 ESXi를 설치하고, Management Network IP를 수동으로 할당했습니다.

### ⚠️ Troubleshooting

* **🔴 문제** : ESXi 설치(USB)를 위한 BIOS 진입 불가  

* **🔍 원인** : 레거시 BIOS 방식 사용  

* **✅ 해결** : `Enter` 키로 BIOS 진입 성공

> 장비마다 BIOS 진입 방식이 다를 수 있습니다.