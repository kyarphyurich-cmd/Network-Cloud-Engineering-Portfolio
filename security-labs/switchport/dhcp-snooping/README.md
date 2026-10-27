# DHCP Snooping 실습

## 📖 개요

이 저장소는 Cisco 스위치를 사용한 **DHCP Snooping** 실습을 기록합니다.

정상적인 설정에서 DHCP Snooping의 동작을 확인한 후, 의도적으로 잘못된 신뢰 설정을 적용하여 비인가 DHCP 서버의 응답이 허용될 때 발생하는 결과를 관찰합니다.

실습은 두 단계로 진행합니다.

- **1단계: 정상 설정** — 정상 DHCP 서버의 응답이 통과하도록 신뢰 포트를 구성합니다.
- **2단계: 의도적인 잘못된 설정** — 비인가 DHCP 서버가 연결된 코어 스위치의 Gi0/1을 신뢰 포트로 변경하여 결과를 확인합니다.

Cisco의 `show` 명령어와 Wireshark 패킷 캡처를 통해 동작을 검증합니다.

---

## 🖥️ 네트워크 구성

### 코어 스위치

- 정상 DHCP 서버 연결 포트: 신뢰 포트
- 비인가 DHCP 서버 연결 포트: 1단계에서는 비신뢰, 2단계에서는 신뢰 포트
- 액세스 스위치 방향 트렁크: 신뢰 포트

### 액세스 스위치

- 코어 스위치 방향 업링크: 신뢰 트렁크 포트
- 클라이언트 PC 연결 포트 Gi0/1 및 Gi0/2: 비신뢰 액세스 포트

---

## ⚙️ 1단계: 정상 설정

### 두 스위치의 공통 설정

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option
```

### 코어 스위치

```cisco
! 정상 DHCP 서버 연결 포트
interface Gi0/2
 ip dhcp snooping trust

! 액세스 스위치 방향 트렁크
interface Gi0/3
 ip dhcp snooping trust

! 비인가 DHCP 서버 연결 포트
interface Gi0/1
 no ip dhcp snooping trust
```

### 액세스 스위치

```cisco
! 코어 스위치 방향 업링크
interface Gi0/0
 ip dhcp snooping trust

! 클라이언트 연결 포트
interface Gi0/1
 no ip dhcp snooping trust

interface Gi0/2
 no ip dhcp snooping trust
```

> 비신뢰 상태는 기본값입니다. 위의 `no ip dhcp snooping trust` 명령어는 해당 포트의 비신뢰 상태를 명시적으로 설정합니다.

### 관찰 결과

- 액세스 스위치에서 `show ip dhcp snooping binding`을 실행하여 클라이언트 바인딩을 확인했습니다.
- Wireshark에서 다음 과정을 확인했습니다.
  - 클라이언트가 DHCP Discover 메시지를 전송했습니다.
  - 정상 DHCP 서버의 Offer 및 ACK만 클라이언트에 도달했습니다.
  - 클라이언트가 IP 주소를 할당받은 후, 해당 주소에 대한 ARP Probe 또는 Announcement를 전송했습니다.

✅ **결과:** 정상 DHCP 서버만 클라이언트에 IP 주소를 할당했습니다.

---

## ⚠️ 2단계: 의도적인 잘못된 설정

### 설정 변경

**비인가 DHCP 서버가 연결된 코어 스위치의 Gi0/1**을 신뢰 포트로 변경했습니다.

```cisco
interface Gi0/1
 ip dhcp snooping trust
```

### 관찰 결과

- 클라이언트가 DHCP Discover 메시지를 전송했습니다.
- 정상 서버와 비인가 서버가 모두 DHCP Offer를 전송했습니다.
- 이 실습에서는 비인가 서버의 Offer가 먼저 도착했고, 클라이언트가 해당 서버를 선택하는 DHCP Request를 전송했습니다.
- 정상 서버의 Offer는 선택되지 않았습니다.
- Wireshark에서 다음 순서를 확인했습니다.
  - DHCP Discover
  - 두 서버의 DHCP Offer
  - 비인가 서버를 선택하는 DHCP Request
  - 비인가 서버의 DHCP ACK
- 이후 클라이언트가 할당받은 IP 주소를 알리는 ARP 브로드캐스트를 전송했습니다.

❌ **결과:** 비인가 DHCP 서버 연결 포트를 신뢰 포트로 변경하자, 해당 서버가 클라이언트에 IP 주소를 할당할 수 있었습니다.

---

## 🔎 실습을 통해 배운 점

- DHCP Snooping은 신뢰 경계를 기준으로 DHCP 서버 응답을 허용하거나 차단합니다.
- DHCP Snooping이 정상 서버를 우선 선택해 주는 것은 아닙니다.
- **비인가 서버의 연결 포트를 신뢰 포트로 지정하면, 해당 포트로 들어오는 서버 응답이 허용됩니다.**
- 이 실습에서는 먼저 도착한 비인가 서버의 Offer가 선택되었지만, 클라이언트가 항상 첫 번째 Offer를 선택한다고 일반화할 수는 없습니다.
- Wireshark를 통해 DHCP 메시지 교환 과정과 실제로 선택된 서버를 확인할 수 있습니다.

---

## 🛠️ 주요 검증 명령어

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
show vlan brief
show interfaces trunk
```

---

## 📸 실습 스크린샷

### 네트워크 토폴로지

![네트워크 토폴로지](./screenshots/topology.png)

---

### DHCP 서버 설정

#### 정상 DHCP 서버의 주소 풀

![정상 서버의 DHCP 주소 풀](./screenshots/cisco-pool.png)

#### 비인가 DHCP 서버의 주소 풀

![비인가 서버의 DHCP 주소 풀](./screenshots/rogue-pool.png)

---

### 코어 및 액세스 스위치의 DHCP Snooping 설정

#### 코어 스위치의 인터페이스 설정

![코어 스위치 DHCP Snooping 설정](./screenshots/core-dhcp-snooping.png)

#### 코어 스위치의 검증 명령어 실행 결과

![코어 스위치 검증 결과](./screenshots/core-verify.png)

#### 액세스 스위치의 인터페이스 설정

![액세스 스위치 DHCP Snooping 설정](./screenshots/access-dhcp-snooping.png)

#### 액세스 스위치의 검증 명령어 실행 결과

![액세스 스위치 검증 결과](./screenshots/access-verify.png)

---

### 정상 설정에서의 DHCP 패킷 캡처 및 IP 할당

#### Wireshark의 DHCP 패킷 캡처

![DHCP 메시지 교환 과정](./screenshots/dhcp-process-capture.png)

#### VLAN 10의 PC가 정상적으로 IPv4 주소를 할당받은 결과

![PC의 IPv4 주소 할당 확인](./screenshots/pc2-ip-confirmation.png)

#### DHCP Snooping 바인딩 테이블

DHCP 주소 할당을 완료한 클라이언트의 IP 주소와 MAC 주소 등의 정보를 확인합니다.

![DHCP Snooping 바인딩 테이블](./screenshots/snooping-binding.png)

---

### 의도적인 잘못된 설정에서의 결과

#### 비인가 DHCP 서버의 응답 및 주소 할당 과정

![비인가 DHCP 서버 패킷 캡처](./screenshots/rogue-capture.png)

#### VLAN 10의 PC가 비인가 서버에서 IPv4 주소를 할당받은 결과

![비인가 서버에서 할당받은 IP 주소](./screenshots/pc3-ip-from-rogue.png)

---

## 📌 결론

이 실습을 통해 DHCP Snooping의 신뢰 포트 설정이 갖는 중요성을 확인했습니다.

- **1단계:** 비인가 서버의 DHCP Offer가 차단되고, 정상 서버에서만 IP 주소가 할당되었습니다.
- **2단계:** 비인가 서버의 연결 포트를 신뢰 포트로 변경하자, 해당 서버의 응답이 허용되어 클라이언트에 IP 주소가 할당되었습니다.

**핵심 교훈:** DHCP Snooping을 활성화하는 것만으로는 충분하지 않습니다. 서버 응답이 들어오는 경로를 확인하고, 필요한 포트에만 신뢰 설정을 적용해야 합니다.
