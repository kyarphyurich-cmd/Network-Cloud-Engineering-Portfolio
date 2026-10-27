# MyTown Media Solutions — 기업 네트워크 구축 실습

*GRE 터널, OSPF 라우팅, VLAN 분리, 중앙 집중형 서비스 및 접근 제어를 적용한 다중 사이트 네트워크 실습입니다.*

![Status](https://img.shields.io/badge/Status-Completed-brightgreen)
![Tool](https://img.shields.io/badge/Tool-Cisco%20Packet%20Tracer-blue)
![Category](https://img.shields.io/badge/Category-Enterprise%20Networking-orange)

---

## 📌 프로젝트 개요

이 저장소는 **MyTown Media Solutions** 프로젝트의 기업 네트워크 구축 실습을 정리합니다.

본사와 지사가 ISP 네트워크를 통해 연결되는 환경을 모델로 구성하고, **GRE 터널**과 **OSPF Area 0**을 활용하여 사이트 간 통신을 구현했습니다.

Layer 3 라우팅, NAT 경계, DHCP 서비스, VLAN, ACL 및 장비 보안 설정을 구성하고 검증했습니다.

Cisco IOS 기술을 활용하여 다음 기능을 실습합니다.

- 다중 사이트 연결
- 중앙 집중형 네트워크 서비스
- VLAN 기반 네트워크 분리
- ACL 기반 접근 제어
- SSH 기반 장비 관리

---

## 🏗️ 네트워크 아키텍처 요약

### 본사 — Headquarters

- VLAN 구성: Management, Sales, Engineering, Holding
- HQ-RTR에서 Router-on-a-Stick 구현
- 본사 스위치 간 EtherChannel(PAgP) 구성
- **DHCP, DNS, TFTP, HTTP** 서비스 중앙 집중화
- 내부 네트워크의 외부 통신을 위한 NAT Overload 구성
- GRE 터널을 통한 지사와의 OSPF 경로 교환

### 지사 — Branch Office

- VLAN 구성: Admin, Guest, Holding
- 본사 서버를 이용하기 위한 DHCP Relay 구성
- Router-on-a-Stick 구현
- 외부 통신을 본사의 NAT 경계로 전달
- GRE 터널을 통한 OSPF 인접 관계 형성

### ISP 라우터

- 공인 WAN 구간 시뮬레이션
- 두 사이트의 GRE 터널 끝점 간 전송 경로 제공

---

## 🔑 주요 기능

### ✔ VLAN 및 IP 주소 설계

- **192.168.99.0/24** 주소 대역에 VLSM을 적용하여 내부 서브넷 구성
- GRE 터널에 **192.168.98.0/30** 사용
- 각 서브넷의 첫 번째 사용 가능한 IP 주소를 게이트웨이로 지정

### ✔ 라우팅

- OSPF Area 0 기반 경로 교환
- GRE 터널을 통한 논리적인 지점 간 연결 및 OSPF 인접 관계 형성
- 단일 영역 설계를 통한 라우팅 구성 단순화

### ✔ NAT 및 외부 네트워크 연결

- HQ-RTR에 NAT Overload(PAT) 구성
- ISP-RTR을 통한 외부 네트워크 시뮬레이션
- 지사도 본사의 NAT 경계를 통해 외부 네트워크에 접근

### ✔ 중앙 서비스 — HQ-SRV

- 본사 및 지사 VLAN을 위한 DHCP 서비스
- 내부 DNS 서비스
- 장비 설정 백업용 TFTP 서비스
- 내부 HTTP 웹 서버

### ✔ 접근 제어

- Sales와 Engineering VLAN 간 양방향 통신 차단
- Management VLAN의 전체 네트워크 접근 허용
- 서브인터페이스의 **인바운드 방향**에 ACL 적용
- 추가 ACL을 통한 DNS 및 웹 서비스 접근 제한

### ✔ 스위칭 및 Layer 2 기능

- HQ-SW1을 STP 루트 브리지로 지정
- 본사 스위치 간 EtherChannel 구성
- 사용하지 않는 포트를 VLAN 99에 할당하고 관리적으로 비활성화

### ✔ 장비 보안 설정

- 원격 관리 접속을 SSH로 제한
- 각 라우터 및 스위치에서 RSA 키 생성
- 로그인 및 MOTD 배너 설정
- 포트 보안 적용
  - Sticky MAC 학습
  - 최대 허용 MAC 주소 수: 2개
  - 위반 모드: Restrict
- WAN 연결 인터페이스에서 CDP 비활성화
- 비밀번호 보호 설정 적용

> GRE 자체는 트래픽을 암호화하지 않습니다. 이 프로젝트의 GRE 연결과 IPsec을 결합한 암호화 터널은 구분해야 합니다.

---

## 📁 저장소 구성

다음 파일과 검증 자료를 포함할 예정입니다.

### 1. `MyTownMediaSolutions.pkt`

- 전체 Cisco Packet Tracer 프로젝트
- 저장된 장비 설정
- 장비 이름, 인터페이스 및 네트워크 라벨

### 2. `screenshots/` 폴더

- 전체 네트워크 토폴로지
- 라우팅 및 OSPF 상태
  - `show ip route`
  - `show ip ospf neighbor`
- ACL 동작 검증
- NAT 변환 테이블
- STP 루트 브리지 확인
- EtherChannel 요약 정보
- SSH 접속 및 배너
- DHCP 및 DNS 테스트
- TFTP 백업 과정

### 3. `network-addressing-table.txt`

- VLSM 서브넷 설계표
- VLAN과 서브넷 매핑
- 게이트웨이 주소
- 사용 가능한 호스트 주소 범위

---

## 🛠️ 문제 해결 기록

구축 과정에서 다음 문제를 확인하고 수정했습니다.

### DHCP 주소 할당 실패

DHCP 메시지 전달 경로를 점검하면서 다음 설정 문제를 확인했습니다.

- 필요한 인터페이스의 `ip helper-address` 누락
- 잘못된 Native VLAN 설정

DHCP Relay 및 VLAN 구성을 수정한 후 주소 할당 과정을 다시 검증했습니다.

### ACL 적용 방향 오류

초기 ACL을 아웃바운드 방향에 적용했을 때 의도한 트래픽 제어가 이루어지지 않았습니다.

실제 패킷 경로와 인터페이스 방향을 확인한 후, 설계에 맞는 인바운드 위치로 수정했습니다.

### SSH 접속 실패

RSA 키가 생성되지 않은 장비에서 SSH 접속이 실패했습니다.

장비별 RSA 키 생성과 관련 SSH 설정을 확인한 후 접속을 검증했습니다.

### 특정 네트워크의 OSPF 경로 학습 실패

지사에서 VLAN 99 네트워크를 학습하지 못하는 문제가 있었습니다.

OSPF Area 설정과 해당 서브넷의 광고 여부를 점검하여 수정했습니다.

### GRE 터널을 통한 OSPF 인접 관계 문제

터널 출발지 설정과 원격 터널 끝점까지의 도달성을 확인했습니다.

설정을 수정한 후 GRE 터널을 통한 OSPF 인접 관계를 검증했습니다.

각 문제는 원인 분석, 설정 수정 및 재검증 과정과 함께 기록했습니다.

---

## 🎯 프로젝트 목적

중소·중견 규모 기업의 본사와 지사 환경을 모델로 다음 구성을 실습했습니다.

- 다중 사이트 라우팅
- 사이트별 VLAN 설계
- 중앙 집중형 네트워크 서비스
- 접근 정책에 따른 네트워크 분리
- WAN 연결 구조
- 장비 관리 접속 보안

이를 통해 다음 분야의 기초 역량을 강화합니다.

- 네트워크 지원 및 네트워크 엔지니어링
- 인프라 운영
- 보안 운영에 필요한 네트워크 이해
- CCNA 수준의 종합 실습 설계
- 실제 운영 환경을 고려한 네트워크 설계 및 검증
