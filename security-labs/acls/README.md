# 🔒 ACL 실습 보고서 — 올바른 설정을 찾기 위한 5번의 시도

## 🎯 실습 목표

- **표준 ACL과 확장 ACL** 설정을 실습한다.
- ACL의 **적용 위치(출발지 또는 목적지 근처)**와 **적용 방향(in/out)**이 트래픽에 미치는 영향을 이해한다.
- 자주 발생하는 설정 오류와 문제 해결 과정을 정리한다.
- 출발지에 가까운 **R1의 G0/1 인터페이스**에 확장 ACL을 적용하여 **PC1에서 PC2로 향하는 ICMP 트래픽을 차단**하고, 나머지 트래픽은 허용한다.

  -----------------------

  # 🖼️ Topology

  PC1 (192.168.10.10) ---- R1 ---- R2 ---- R3 ---- PC2 (192.168.20.10)
  *PC1: 192.168.10.10/24 (gateway R1 G0/0)
  *PC2: 192.168.20.10/24 (gateway R3 G0/1)
  * R1–R2–R3 간 연결 네트워크: 10.0.x.x/30
---

# 🛠 시도 및 결과

---

## 🔧 시도 1 (A1) — R1에 표준 ACL 적용 (인바운드)
  ```cisco
R1(config)# access-list 1 deny host 192.168.10.10
R1(config)# access-list 1 permit any
R1(config)# interface g0/0
R1(config-if)# ip access-group 1 in
```
### 결과:

* PC1 → PC2 ping ❌ (차단됨)
* PC1 → 다른 모든 목적지 ❌ (함께 차단됨 — 과도한 차단)

📸 스크린샷:

(ping은 차단되었지만, 차단 범위가 너무 넓음)

**배운 점:** 표준 ACL은 **출발지 IP 주소**만으로 필터링하므로, 이 설정처럼 출발지 근처에 적용하면 PC1에서 다른 목적지로 향하는 트래픽까지 차단된다.

---

## 🔧 시도 2 (A2) — R3에 표준 ACL 적용 (인바운드, 잘못된 방향)

```cisco
R3(config)# access-list 1 deny host 192.168.10.10
R3(config)# access-list 1 permit any
R3(config)# interface g0/1
R3(config-if)# ip access-group 1 in
```

### 결과:

* PC1 → PC2 ping ✅ (성공 — ACL이 해당 트래픽에 영향을 주지 않음)

📸 스크린샷:

**배운 점:** PC1의 트래픽은 R3의 G0/1 인터페이스로 *들어오는* 것이 아니라, 해당 인터페이스를 통해 *나간다*. 따라서 인바운드로 적용한 ACL은 해당 트래픽을 검사하지 않는다.

---

## 🔧 시도 3 (A3) — R3에 표준 ACL 적용 (아웃바운드)

```cisco
R3(config)# interface g0/1
R3(config-if)# no ip access-group 1 in
R3(config-if)# ip access-group 1 out
```

### 결과:

* PC1 → PC2 ping ❌ (의도대로 차단됨)
* PC2 → PC1 ping ❌ (실패 — 응답 패킷도 차단됨)

📸 스크린샷:

**배운 점:** 표준 ACL은 **연결 상태를 추적하지 않으며**, 출발지 IP 주소만으로 필터링한다. PC1이 보내는 ICMP Echo Reply 패킷(출발지 IP = 192.168.10.10)도 차단 규칙에 일치하므로, PC2 → PC1 ping 역시 실패한다.

---

## 🔧 시도 4 (A4) — R3의 G0/1에 확장 ACL 적용 (아웃바운드)

```cisco
R3(config)# ip access-list extended BLOCK-PC1-PC2
R3(config-ext-nacl)# deny ip host 192.168.10.10 host 192.168.20.10
R3(config-ext-nacl)# permit ip any any
R3(config)# interface g0/1
R3(config-if)# ip access-group BLOCK-PC1-PC2 out
```

### 결과:

* PC1 → PC2 ping ❌ (의도대로 차단됨)
* PC2 → PC1 ping ❌ (다시 실패 — 응답 패킷이 여전히 차단됨)

📸 스크린샷:

**배운 점:** 확장 ACL에서도 포괄적인 `deny ip` 규칙을 사용하면 응답 패킷까지 차단된다. PC1 → PC2 요청 패킷과 PC2의 ping에 대한 PC1의 응답 패킷은 모두 출발지가 PC1, 목적지가 PC2이므로 동일한 차단 규칙에 일치한다.

---

## 🔧 시도 5 (A5) — R1의 G0/1에 확장 ACL 적용 (아웃바운드, 최종 해결)

```cisco
R1(config)# ip access-list extended BLOCK-PC1-PC2-ICMP
R1(config-ext-nacl)# deny icmp host 192.168.10.10 host 192.168.20.10 echo
R1(config-ext-nacl)# permit ip any any
R1(config)# interface g0/1
R1(config-if)# ip access-group BLOCK-PC1-PC2-ICMP out
```

### 결과:

* PC1 → PC2 ping ❌ (차단됨)
* PC2 → PC1 ping ✅ (성공 — 응답 패킷은 차단되지 않음)
* PC1 → 다른 목적지 ✅ (허용됨)

📸 스크린샷:

**배운 점:**

* 확장 ACL을 사용하면 세밀한 필터링이 가능하다. 이번 실습에서는 ICMP Echo Request만 차단한다.
* 확장 ACL을 **출발지 가까이에 배치**하면 불필요한 트래픽을 경로 초기에 차단할 수 있다. 응답 패킷과 다른 트래픽의 허용 여부는 ACL 규칙에 따라 결정된다.
* 포괄적인 `deny ip` 대신 `deny icmp ... echo`를 사용하면 Echo Request만 차단하고 Echo Reply는 이후 허용 규칙으로 통과시킬 수 있다.

---

# 🔍 검증 명령어

* `show access-lists` → PC1에서 PC2로 ping을 보낼 때 해당 차단 규칙의 일치 횟수(match counter)가 증가하는지 확인한다.
* `show ip interface g0/1` → R1의 G0/1 인터페이스에 ACL이 아웃바운드로 적용되었는지 확인한다.
* PC1과 PC2에서 각각 ping 테스트를 수행하여 양방향 통신의 허용 및 차단 여부를 검증한다.

---

# ✅ 최종 정리

1. **A1:** 출발지 근처에 표준 ACL 적용 → 다른 목적지로 향하는 트래픽까지 과도하게 차단됨.
2. **A2:** 표준 ACL을 잘못된 방향으로 적용 → 대상 트래픽이 ACL을 통과하지 않아 차단되지 않음.
3. **A3:** R3에 표준 ACL을 아웃바운드로 적용 → PC1의 요청과 응답을 구분하지 못해 양쪽에서 시작한 ping이 모두 실패함.
4. **A4:** R3에 확장 ACL을 아웃바운드로 적용 → 포괄적인 차단 규칙으로 인해 응답 패킷까지 차단됨.
5. **A5:** R1의 G0/1에 확장 ACL을 아웃바운드로 적용하고 ICMP Echo Request만 차단 → 요구사항 충족.

**시험 및 실무 핵심:**

* 표준 ACL → 출발지 IP 주소만으로 필터링하므로, 일반적으로 목적지 가까이에 배치한다.
* 확장 ACL → 출발지, 목적지, 프로토콜 등을 기준으로 세밀하게 필터링하므로, 일반적으로 출발지 가까이에 배치한다.
* 일반적인 표준·확장 ACL은 연결 상태를 추적하지 않으므로, 항상 양방향 통신을 테스트한다.
* 의도하지 않은 차단을 방지하려면 `deny ip`보다 `deny icmp ... echo`처럼 차단 조건을 구체적으로 설정한다.
* ACL 끝에는 암묵적 차단이 있으므로, 나머지 IP 트래픽을 허용하려면 `permit ip any any`와 같은 명시적 허용 규칙이 필요하다.


