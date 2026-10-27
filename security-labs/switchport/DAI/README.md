# DAI + DHCP Snooping 실습

---

## 📖 개요

이 실습은 기존 DHCP Snooping 실습을 기반으로, 두 스위치에서 **동적 ARP 검사(Dynamic ARP Inspection, DAI)**를 활성화하고 추가 검증 옵션을 적용합니다.

정상적인 DHCP Snooping 바인딩이 있을 때 ARP 및 ICMP 통신이 정상적으로 이루어지는지 확인한 후, 클라이언트의 MAC 주소를 의도적으로 변경하여 DAI의 차단 동작을 관찰합니다.

---

## 🖥️ 네트워크 구성

### 코어 스위치

- 정상 DHCP 서버 연결 포트: 신뢰 포트
- 액세스 스위치와 연결되는 트렁크 포트: 신뢰 포트

### 액세스 스위치

- 코어 스위치 방향 업링크: 신뢰 포트
- **PC1**: Gi0/1에 연결
- **PC2**: Gi0/2에 연결하며, 이후 MAC 주소 변경 테스트 수행

DHCP Snooping 설정은 이전 실습 구성을 유지합니다.

> DHCP Snooping의 신뢰 설정과 DAI의 신뢰 설정은 서로 별개입니다. 각 기능에 맞게 신뢰 포트를 지정해야 합니다.

---

## ⚙️ 주요 설정

### 두 스위치의 공통 설정

```cisco
! DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

! DAI
ip arp inspection vlan 10
ip arp inspection validate src-mac dst-mac ip
```

### 코어 스위치의 신뢰 포트

아래 예시는 Gi0/3이 액세스 스위치 방향 트렁크이고, Gi0/2가 DHCP 서버 연결 포트인 경우입니다.

```cisco
! 액세스 스위치 방향 트렁크
interface Gi0/3
 ip dhcp snooping trust
 ip arp inspection trust

! 정상 DHCP 서버 연결 포트
interface Gi0/2
 ip dhcp snooping trust
 ip arp inspection trust
```

### 액세스 스위치의 업링크

실제 코어 스위치 연결 포트에 다음 설정을 적용합니다.

```cisco
ip dhcp snooping trust
ip arp inspection trust
```

### 액세스 스위치의 클라이언트 포트

PC 연결 포트는 비신뢰 상태로 유지합니다.

```cisco
interface Gi0/1
 switchport mode access
 switchport access vlan 10
 no ip dhcp snooping trust
 no ip arp inspection trust

interface Gi0/2
 switchport mode access
 switchport access vlan 10
 no ip dhcp snooping trust
 no ip arp inspection trust
```

> 코어 스위치의 Gi0/2와 액세스 스위치의 Gi0/2는 서로 다른 장비의 포트입니다.

---

## 🔎 추가 ARP 검증 명령어의 목적

```cisco
ip arp inspection validate src-mac dst-mac ip
```

이 명령어는 다음 세 가지 추가 검사를 활성화합니다.

| 옵션 | 검사 내용 |
|---|---|
| `src-mac` | Ethernet 헤더의 출발지 MAC과 ARP 메시지 내부의 송신자 MAC이 일치하는지 검사합니다. ARP 요청과 응답에 적용됩니다. |
| `dst-mac` | Ethernet 헤더의 목적지 MAC과 ARP 메시지 내부의 대상 MAC이 일치하는지 검사합니다. ARP 응답에 적용됩니다. |
| `ip` | ARP 메시지에 유효하지 않은 IP 주소가 포함되어 있는지 검사합니다. |

이 추가 검사는 DAI의 **DHCP Snooping 바인딩 기반 IP–MAC 검증**과 구분됩니다.

### 두 스위치에서 DAI를 활성화하는 이유

- **액세스 스위치**: 클라이언트가 보낸 위조 ARP 패킷을 네트워크 진입 지점에서 차단합니다.
- **코어 스위치**: 코어의 비신뢰 포트로 유입되는 ARP 패킷을 검사합니다.
- **정책 관리**: 각 장비의 역할에 맞는 ARP 검사 정책을 적용합니다.

> `ip arp inspection trust`가 설정된 포트로 들어오는 ARP 패킷은 DAI 검사를 우회합니다. 따라서 스위치 간 링크를 신뢰 포트로 설정하면, 해당 링크로 들어오는 패킷을 코어에서 다시 검사하지 않습니다.

---

## 🔬 1단계: 정상 통신 확인

### 수행 내용

DAI를 활성화하고 DHCP Snooping 바인딩이 정상적으로 생성된 상태에서 PC2 → PC1으로 ping을 실행합니다.

### 확인 항목

- ping 성공 여부
- ARP 및 ICMP 통신 정상 여부
- DAI 드롭 카운터 증가 여부
- 인터페이스의 정상 동작 상태

✅ **정상 동작 시 결과:** 유효한 바인딩을 가진 호스트의 ARP 패킷이 허용되고, ping이 성공합니다.

---

## ⚠️ 2단계: MAC 주소 위조 테스트

### 수행 내용

PC2의 MAC 주소를 기존 DHCP Snooping 바인딩과 일치하지 않는 주소로 변경한 후, PC1으로 ping을 실행합니다.

기존 바인딩과의 불일치를 확인하는 테스트이므로, 테스트 중 DHCP를 재요청하여 새 MAC 주소로 바인딩이 갱신되지 않았는지 확인합니다.

필요한 경우 기존 ARP 캐시를 비운 후 테스트하여 새로운 ARP 요청이 발생하도록 합니다.

### 확인 항목

- ping 실패 여부
- 인터페이스가 `up/up` 상태를 유지하는지 확인
- DAI 위반 관련 Syslog 메시지 확인
- `show ip arp inspection statistics`의 드롭 카운터 증가 확인
- Wireshark에서 ARP 요청과 스위치 통과 여부 확인

❌ **차단 동작 시 결과:** DAI가 바인딩과 일치하지 않는 ARP 패킷을 폐기하여 주소 확인이 실패하고, 해당 통신의 ICMP 패킷이 전송되지 못합니다.

> DAI는 ARP 패킷을 검사합니다. ICMP를 직접 차단하는 기능은 아니므로, ARP 캐시 상태에 따라 테스트 결과가 달라질 수 있습니다.

---

## 🛠️ 검증 명령어

```cisco
show ip arp inspection vlan 10
show ip arp inspection statistics
show ip dhcp snooping
show ip dhcp snooping binding
show logging
show interfaces status
show mac address-table
```

---

## 🖥️ 네트워크 토폴로지

![네트워크 토폴로지](./screenshots/topology.png)

---

## 📸 1단계 — DAI 설정 및 정상 통신

### 액세스 스위치의 DAI 설정

![DAI 설정](./screenshots/DAI-Config.png)

### 정상 상태의 ARP 검사 통계

![정상 상태 검증 결과](./screenshots/show-commands.png)

### PC2 → PC1 ping 성공

![ping 성공](./screenshots/pc2-ping.png)

---

## 📸 2단계 — 의도적인 MAC 주소 변경

### CLI에서 MAC 주소 변경

![MAC 주소 변경](./screenshots/chg-mac-add.png)

### PC2 → PC1 ping 실패

![ping 실패](./screenshots/pc2-ping-fail.png)

### Wireshark ARP 패킷 캡처

![ARP 패킷 캡처](./screenshots/arp-capture.png)

### 위반 발생 후 DAI 통계

![DAI 드롭 통계](./screenshots/sw-ip-arp-inspec-stats.png)

---

## ✅ 실습을 통해 배운 점

- 이 구성에서 DAI는 DHCP Snooping 바인딩을 활용하여 ARP 패킷의 IP–MAC 매핑을 검증합니다.
- 바인딩과 일치하지 않는 ARP 패킷은 폐기됩니다.
- 인터페이스가 정상적으로 연결되어 있어도 ARP 패킷 차단으로 통신이 실패할 수 있습니다.
- Syslog와 DAI 통계를 함께 확인하면 차단 원인을 파악할 수 있습니다.
- 추가 검증 옵션은 Ethernet 헤더와 ARP 메시지의 MAC 주소 일치 여부 및 IP 주소의 유효성을 검사합니다.
- 신뢰 포트로 들어오는 ARP 패킷은 DAI 검사를 우회하므로, 신뢰 경계를 신중하게 설정해야 합니다.
