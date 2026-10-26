# NAT Overload (PAT) — GNS3 실습 문서

## 개요

이 실습에서는 Cisco IOS 라우터와 외부 호스트, 그리고 세 개의 VLAN(10, 20, 30)을 구성한 L2 스위치를 사용하여 **NAT Overload(PAT)**의 동작을 확인합니다.

각 VLAN에는 호스트가 한 대씩 있으며, 모든 내부 네트워크의 호스트는 라우터의 외부 인터페이스에 설정된 하나의 공인 IP 주소를 공유하여 외부 호스트와 통신합니다.

실습에 포함된 스크린샷은 다음 내용을 보여 줍니다.

- 네트워크 토폴로지
- 라우터 및 스위치 인터페이스 상태
- VLAN 설정
- 실행 중인 설정의 NAT 관련 항목
- PAT 변환 테이블
- 각 내부 VLAN에서 수행한 Ping 테스트
- 외부 호스트에서 수행한 Ping 테스트

이를 통해 **여러 서브넷에서 PAT가 정상적으로 동작하는지** 확인하고, **Router-on-a-Stick과 NAT의 연동 상태**를 검증합니다.

---

## 네트워크 구성 요약

- **외부 호스트:** 203.0.113.1/24
- **라우터 Gi0/0:** 203.0.113.2/24 — NAT 외부 인터페이스
- **라우터 Gi0/1:** 802.1Q 트렁크 — VLAN별 NAT 내부 서브인터페이스
- **L2 스위치:** VLAN 10, 20, 30 구성
- **내부 호스트:** PC1(VLAN 10), PC2(VLAN 20), PC3(VLAN 30)

![네트워크 토폴로지](./screenshots/NAT-ROUTER-TOPOLOGY.png)

---

## 라우터 주요 설정

### 내부 인터페이스 — 서브인터페이스

```cisco
interface GigabitEthernet0/1
no ip address

interface GigabitEthernet0/1.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
ip nat inside

interface GigabitEthernet0/1.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
ip nat inside

interface GigabitEthernet0/1.30
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0
ip nat inside
```

### 외부 인터페이스

```cisco
interface GigabitEthernet0/0
ip address 203.0.113.2 255.255.255.0
ip nat outside
no shutdown
```

### 기본 경로

```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### PAT 설정

```cisco
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 permit 192.168.20.0 0.0.0.255
access-list 10 permit 192.168.30.0 0.0.0.255

ip nat inside source list 10 interface GigabitEthernet0/0 overload
```

---

## 검증 스크린샷

### 1. GNS3 네트워크 토폴로지

![GNS3 토폴로지](./screenshots/NAT-ROUTER-TOPOLOGY.png)

---

### 2. NAT 라우터 인터페이스 상태 — `show ip interface brief`

![라우터 인터페이스 상태](./screenshots/ip-int-brief.png)

---

### 3. 스위치 VLAN 설정 — `show vlan brief`

![VLAN 설정](./screenshots/vlan-brief.png)

---

### 4. 스위치 트렁크 상태 — `show interfaces trunk`

![트렁크 인터페이스 상태](./screenshots/int-trunk.png)

---

### 5. NAT 관련 설정 — `show run | section nat`

![NAT 관련 설정](./screenshots/Nat-config-section.png)

---

### 6. NAT 변환 테이블 — `show ip nat translations`

![NAT 변환 테이블](./screenshots/ip-nat-trans.png)

---

### 7. 내부 호스트에서 외부 호스트로 Ping 성공

![내부 호스트 Ping 테스트](./screenshots/pc-success-ping.png)

---

### 8. 외부 호스트에서 라우터의 공인 IP 주소로 Ping 테스트

![외부 호스트 Ping 테스트](./screenshots/outside-ping.png)

---

### 추가 NAT 통계

![NAT 통계](./screenshots/nat-stats.png)

---

## 검증 결과 요약

모든 검증 단계가 정상적으로 완료되었습니다.

- Router-on-a-Stick 트렁크 정상 동작
- 모든 VLAN의 연결성 확인
- PAT 오버로드 주소 변환 동작 확인
- 모든 내부 호스트의 NAT 변환 항목 생성 확인
- 외부 호스트와의 통신 성공
- 종단 간 연결성 검증 완료

---

## 결론

이 실습을 통해 다음 내용을 확인했습니다.

- 여러 VLAN을 이용한 네트워크 분리
- 서브인터페이스를 이용한 VLAN 간 라우팅
- NAT Overload(PAT)를 이용한 여러 내부 호스트의 주소 변환
- 하나의 공인 IP 주소를 공유하는 내부 호스트들의 외부 통신

이 실습은 다음과 같은 심화 NAT 실습의 기초가 됩니다.

- 정적 NAT
- NAT 주소 풀
- 포트 포워딩
- ACL 기반 NAT 정책
- 다중 라우터 환경의 NAT 설계
