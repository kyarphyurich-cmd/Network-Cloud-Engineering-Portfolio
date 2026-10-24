# 🔐 ACL 문제 해결 및 설계 실습

이 실습에서는 Router-on-a-Stick 토폴로지를 기반으로 ACL 설계와 문제 해결을 **3단계**로 진행한다.

* **VLAN 10 – 사용자**
* **VLAN 20 – 관리자**
* **VLAN 30 – 서버**
* **라우터 서브인터페이스의 IP 주소:**
  * VLAN 10 → `192.168.10.1`
  * VLAN 20 → `192.168.20.1` (관리 접속 대상 주소)
  * VLAN 30 → `192.168.30.1`

다음 ACL을 사용하여 **트래픽 접근 제한 정책**을 구현한다.

1. **1단계 — 표준 번호형 ACL**
2. **2단계 — 이름 기반 확장 ACL (`access-restrict`)**
3. **3단계 — 번호형 확장 ACL**

이 문서에서는 실습 과정에서 확인한 설정 오류, 해결 방법 및 주요 학습 내용을 정리한다.

---

# 📘 1단계 — 표준 ACL (VLAN 10 → VLAN 30 차단)

### 목표

**VLAN 10에서 VLAN 30으로 향하는 모든 트래픽을 차단**하고, 그 외 트래픽은 정상적으로 허용한다.

---

## 핵심 개념 — 표준 ACL은 출발지 IP 주소만 검사한다

표준 ACL은 **출발지 IP 주소만** 검사하므로, 특정 목적지로 향하는 트래픽만 제한하려면 일반적으로 **목적지 가까이에 배치**한다.

이번 실습에서는 VLAN 30으로 나가는 인터페이스에 적용하여 차단 범위를 제한한다.

표준 ACL 자체에서는 목적지 VLAN 30을 직접 지정할 수 없다.

출발지 인터페이스의 인바운드 방향에 적용하면 다른 목적지로 향하는 트래픽까지 차단할 수 있으므로 주의해야 한다.

---

## ❌ 발생한 오류 — 잘못된 방향으로 ACL 적용

처음에는 VLAN 30의 서브인터페이스에 ACL을 **인바운드**로 적용했다.

```text
interface g0/0.30
 ip access-group BLOCK-V10 in
```

**결과:**

VLAN 10 장치에서 VLAN 30으로 여전히 통신할 수 있었다.

---

### 실패 원인

VLAN 10에서 출발한 트래픽은 다음 경로로 전달된다.

```text
g0/0.10 (인바운드) → 라우팅 → g0/0.30 (아웃바운드)
```

VLAN 30 인터페이스의 인바운드 ACL은 **VLAN 30에서 들어오는 트래픽**을 검사한다.

따라서 VLAN 30으로 향하는 트래픽은 해당 ACL의 검사 대상이 되지 않는다.

---

## ✔ 해결 방법 — VLAN 30에 ACL을 아웃바운드로 적용

g0/0.30에 ACL을 아웃바운드로 적용하여 라우팅 이후 해당 인터페이스로 나가는 패킷을 검사하도록 수정했다.

이를 통해 VLAN 10 → VLAN 30 트래픽을 차단했다.

**표준 ACL 설정**

![p1-standard-acl-config](./screenshots/p1-standard-acl-config.png)

**잘못된 ACL 적용 위치 및 방향 — VLAN 30 인바운드**

![p1-standard-acl-applied](./screenshots/p1-standard-acl-applied.png)

**수정된 ACL 적용 위치 및 방향 — VLAN 30 아웃바운드**

![p1-corrected-placement-standard-acl](./screenshots/p1-corrected-placement-standard-acl.png)

**ACL 적용 방향 수정 후 VLAN 10에서 ping 테스트**

![p1-vlan10-ping](./screenshots/p1-vlan10-ping.png)

---

### ✔ 1단계 완료

---

# 📘 2단계 — 이름 기반 확장 ACL (`access-restrict`)

이 단계에서는 이름 기반 확장 ACL을 사용하여 **관리용 IP 주소 `192.168.20.1`에 대한 접근**을 제한한다.

---

## 2단계 요구사항

1. VLAN 20에서 `192.168.20.1`로 SSH 접속을 허용한다.
2. VLAN 10에서 해당 주소로 SSH 접속을 차단한다.
3. VLAN 20에서 해당 주소로 ping을 허용한다.
4. VLAN 10에서 해당 주소로 ping을 차단한다.
5. 해당 주소에 대한 HTTP, HTTPS, Telnet, SNMP 및 TFTP 트래픽을 차단한다.
6. 마지막에 `permit ip any any`를 추가하여 나머지 IP 트래픽을 허용한다.

---

# 🔎 발생한 문제 및 해결 방법

## ❌ 1. VLAN 10에서 SSH 및 ICMP 통신이 계속 허용됨

차단 규칙을 설정했지만 VLAN 10에서 관리용 IP 주소로 향하는 트래픽이 차단되지 않았다.

---

## 점검 사항 A — SSH 기본 설정 확인

SSH 접근 제어를 검증하기 전에 SSH 서비스가 정상적으로 동작하는지 확인해야 한다.

SSH 자체가 설정되지 않았다면 접속 실패가 ACL 때문인지 SSH 설정 때문인지 구분하기 어렵다.

다음과 같이 SSH를 설정했다.

```text
enable secret password

ip domain-name lab.local
crypto key generate rsa modulus 2048
username admin secret password
line vty 0 4
 login local
 transport input ssh
```

---

## ❌ 원인 B — VLAN 10 인바운드에 ACL이 적용되지 않음

VLAN 10에서 `192.168.20.1`로 향하는 트래픽은 다음 경로로 처리된다.

```text
VLAN 10의 PC → g0/0.10 (인바운드) → 라우터 내부 처리 → 라우터 자신의 IP 192.168.20.1
```

목적지가 라우터 자신의 IP 주소이므로 해당 패킷은 **VLAN 20 인터페이스의 아웃바운드 방향으로 전달되지 않는다.**

따라서 VLAN 20 아웃바운드에만 ACL을 적용해서는 이 트래픽을 제한할 수 없다.

### ✔ 해결 방법

g0/0.10에 ACL을 **인바운드**로 적용한다.

```text
interface g0/0.10
 ip access-group access-restrict in
```

이렇게 하면 VLAN 10의 트래픽이 라우터로 들어오는 지점에서 ACL 검사가 이루어진다.

---

## ❌ 2. ICMP 규칙에 VLAN 20 허용만 있고 VLAN 10 차단이 없음

기존 규칙은 다음과 같다.

```text
30 permit icmp 192.168.20.0 0.0.0.255 host 192.168.20.1 echo
```

이 규칙은 VLAN 20에서 관리용 IP 주소로 보내는 ICMP Echo Request를 허용한다.

그러나 VLAN 10의 트래픽을 직접 차단하지는 않는다.

특히 마지막에 `permit ip any any`가 있으므로, 별도의 차단 규칙이 없으면 VLAN 10의 ICMP 트래픽도 허용된다.

### ✔ 해결 방법 — 명시적 차단 규칙 추가

```text
35 deny icmp 192.168.10.0 0.0.0.255 host 192.168.20.1 echo
```

해당 규칙을 추가하여 VLAN 10에서 관리용 IP 주소로 보내는 ping을 차단했다.

---

# 최종 이름 기반 확장 ACL (`access-restrict`)

```text
ip access-list extended access-restrict
 10 permit tcp 192.168.20.0 0.0.0.255 host 192.168.20.1 eq 22
 20 deny   tcp 192.168.10.0 0.0.0.255 host 192.168.20.1 eq 22
 30 permit icmp 192.168.20.0 0.0.0.255 host 192.168.20.1 echo
 35 deny   icmp 192.168.10.0 0.0.0.255 host 192.168.20.1 echo
 40 deny   tcp any host 192.168.20.1 eq www
 50 deny   tcp any host 192.168.20.1 eq 443
 60 deny   tcp any host 192.168.20.1 eq telnet
 70 deny   udp any host 192.168.20.1 range snmp 162
 80 deny   udp any host 192.168.20.1 eq tftp
 90 permit ip any any
```

* ✔ VLAN 20의 SSH 및 ping 허용 정책을 정의한다.
* ✔ VLAN 10의 SSH 및 ping을 차단한다.
* ✔ ACL이 검사하는 트래픽 중 지정된 관리 프로토콜을 차단한다.
* ✔ 나머지 IP 트래픽은 허용한다.

**적용 범위 참고:**

ACL 규칙은 해당 ACL이 적용된 인터페이스와 방향을 통과하는 트래픽에만 적용된다.

g0/0.10 인바운드에만 적용하면 VLAN 10의 트래픽만 검사한다. VLAN 20이나 다른 VLAN의 트래픽까지 제한하려면 해당 트래픽의 유입 경로에도 적용 여부를 검토해야 한다.

또한 마지막 `permit ip any any`에 따라, 명시적으로 차단하지 않은 트래픽은 허용된다.

### 2단계 — 이름 기반 확장 ACL 스크린샷

**VLAN 20에서 SSH 접속 성공 — 예상 결과: 허용**

![p2-admin-ssh](./screenshots/p2-admin-ssh.png)

**서브인터페이스에 이름 기반 ACL 적용**

![p2-named-acl-on-sub](./screenshots/p2-named-acl-on-sub.png)

**ACL 확인 (`show access-lists`)**

![p2-show-access](./screenshots/p2-show-access.png)

**VLAN 10에서 SSH 접속 실패 — 예상 결과: 차단**

![p2-ssh-fail-vlan-10](./screenshots/p2-ssh-fail-vlan-10.png)

**35번 규칙 추가 후 VLAN 10에서 ping 실패**

![p2-vlan10-ping](./screenshots/p2-vlan10-ping.png)

---

# 📘 배운 점 — ACL 핵심 개념

### 1. ACL은 실제로 통과하는 트래픽만 검사한다

인터페이스와 적용 방향을 정확히 확인해야 한다.

### 2. 확장 ACL은 일반적으로 출발지 가까이에 배치한다

이번 실습에서는 VLAN 10의 인바운드에 적용하여 관리용 IP 주소로 향하는 트래픽을 검사했다.

### 3. 표준 ACL은 일반적으로 목적지 가까이에 배치한다

1단계에서 목적지 인터페이스의 아웃바운드 방향에 적용하여 차단 범위를 제한했다.

### 4. SSH ACL 검증 전에 SSH 서비스의 정상 동작을 확인한다

SSH 설정 오류와 ACL 차단을 구분해야 정확한 검증이 가능하다.

### 5. 마지막 허용 규칙보다 앞에 필요한 차단 규칙을 작성한다

`permit ip any any`가 있으면 그 전에 차단되지 않은 트래픽은 허용된다.

### 6. 허용 및 차단 대상 트래픽을 모두 테스트한다

ping, SSH 및 VLAN 간 통신을 각각 검증한다.

---

# 📘 3단계 — 번호형 확장 ACL (서버 접근 제어)

### 3단계 목표

**번호형 확장 ACL 110**을 생성하여 다음 정책을 구현한다.

1. **VLAN 20의 관리자는 VLAN 30의 모든 서버에 접근할 수 있다.**
2. **VLAN 10의 사용자는 웹 서버 `192.168.30.30`의 TCP 80/443 포트에만 접근할 수 있다.**
3. **그 외 VLAN 10 → VLAN 30 트래픽은 모두 차단한다.**
4. 마지막에 `permit ip any any`를 추가하여 관련 없는 나머지 트래픽은 허용한다.

---

## 최종 ACL 110 설정

```text
access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 110 permit tcp 192.168.10.0 0.0.0.255 host 192.168.30.30 eq www
access-list 110 permit tcp 192.168.10.0 0.0.0.255 host 192.168.30.30 eq 443
access-list 110 deny   ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 110 permit ip any any
```

---

## ✔ 동작 원리

### 1. VLAN 20의 서버 접근 허용

첫 번째 규칙은 VLAN 20에서 VLAN 30으로 향하는 모든 IP 트래픽을 허용한다.

### 2. VLAN 10의 웹 접근 제한

두 번째와 세 번째 규칙은 **특정 호스트 `192.168.30.30`**에 대한 HTTP 및 HTTPS 트래픽만 허용한다.

### 3. 그 외 VLAN 10 → VLAN 30 트래픽 차단

네 번째 규칙은 앞선 웹 접근 허용 규칙에 일치하지 않은 VLAN 10 → VLAN 30 IP 트래픽을 차단한다.

### 4. 목적지 인터페이스에서 정책 적용

ACL을 **g0/0.30의 아웃바운드**에 적용한다.

* VLAN 10과 VLAN 20의 트래픽은 VLAN 30에 도달하기 위해 모두 이 인터페이스를 통해 나간다.
* VLAN 10 인바운드에만 적용하면 VLAN 20의 트래픽은 검사하지 않는다.
* VLAN 20 인바운드에만 적용하면 VLAN 10의 트래픽은 검사하지 않는다.

따라서 이번 구성에서는 VLAN 30 아웃바운드에 적용하여 **두 출발지의 트래픽을 한 지점에서 검사**한다.

확장 ACL은 일반적으로 출발지 가까이에 배치하지만, 이번 실습처럼 공통 목적지에서 정책을 통합하여 적용할 수도 있다.

**참고:** 일반적인 ACL은 연결 상태를 추적하지 않는다. 따라서 VLAN 30에서 VLAN 10으로 먼저 시작한 통신의 응답 패킷도 차단될 수 있으므로, 해당 통신이 필요하다면 추가 검증과 정책 조정이 필요하다.

---

### 3단계 — 번호형 확장 ACL 스크린샷

**VLAN 20 → 서버 ping 성공**

![p3-admin-ping](./screenshots/p3-admin-ping.png)

**VLAN 10 → 서버 ping 실패**

![p3-pc-ping](./screenshots/p3-pc-ping.png)

**ACL 110 및 규칙별 일치 횟수 확인**

![p3-show-access-list](./screenshots/p3-show-access-list.png)

---

## ✔ 3단계 테스트 결과

* VLAN 20 → 모든 서버: **허용**
* VLAN 10 → `192.168.30.30`의 HTTP/HTTPS: **허용**
* VLAN 10 → VLAN 30의 그 외 목적지 또는 프로토콜: **차단**
* 위 차단 조건에 해당하지 않는 나머지 IP 트래픽: **허용**

---

이 실습에서 다룬 내용은 다음과 같다.

* 표준 ACL의 동작 원리와 적용 방향
* 장비 접근 보호를 위한 이름 기반 확장 ACL
* 서버 접근 제어를 위한 번호형 확장 ACL
* 트래픽 경로에 맞는 인터페이스 선택
* 인바운드 및 아웃바운드 방향 분석
* 단계별 문제 해결 절차
* CCNA/CCNP 학습 및 실무에 활용할 수 있는 ACL 설계 사고방식

---
