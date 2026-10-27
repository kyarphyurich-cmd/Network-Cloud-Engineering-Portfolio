
# STP 실습 — 루트 브리지 선출, 중복 링크 추가 및 BPDU Guard 검증

이 실습에서는 중복 경로가 있는 Layer 2 토폴로지에서 스패닝 트리 프로토콜(STP)의 동작을 확인합니다.

루트 브리지 선출, 우선순위 변경을 통한 루트 브리지 지정, 중복 링크 추가 후 STP 재계산, 액세스 포트의 BPDU Guard 보호 동작을 검증합니다.

Cisco Packet Tracer에서 분배 계층 스위치 3대와 BPDU Guard 테스트용 액세스 스위치 1대를 사용하여 구성했습니다.

---

## 1. 실습 목표

1. 초기 STP 루트 브리지 선출 과정 확인
2. STP 우선순위를 조정하여 다른 스위치를 루트 브리지로 지정
3. 중복 링크를 추가하고 STP가 포트를 차단하여 루프를 방지하는 과정 확인
4. 액세스 포트에 BPDU Guard를 활성화하고 BPDU 수신 시 err-disabled 전환 확인

이를 통해 STP의 루프 방지 원리, 루트 브리지 위치에 따른 전달 경로 변화, BPDU Guard의 보호 동작을 이해합니다.

---

## 2. 네트워크 구성

멀티레이어 스위치 SW1, SW2, SW3을 GigabitEthernet 트렁크 링크로 연결하여 삼각형 형태의 중복 토폴로지를 구성했습니다.

SW4는 BPDU Guard 테스트용 액세스 스위치입니다.

| 구성 항목 | 연결 및 인터페이스 |
|---|---|
| 기본 토폴로지 | SW1, SW2, SW3 간 삼각형 연결 |
| 트렁크 포트 | SW1, SW2, SW3의 Gi0/1 및 Gi0/2 |
| BPDU Guard 테스트 | SW2 Fa0/1에 SW4 연결 |
| 추가 중복 링크 | SW1 Fa0/1 ↔ SW3 Fa0/1 |

> 물리적인 중복 경로가 있어도 STP가 적절한 포트를 차단하면 실제 프레임 전달 루프는 형성되지 않습니다.

---

## 3. STP 기본 설정

SW1, SW2, SW3에서 PVST+를 사용합니다.

다음은 각 스위치에 적용한 기본 설정 예시입니다.

```cisco
spanning-tree mode pvst

interface g0/1
 switchport mode trunk

interface g0/2
 switchport mode trunk
```

이 실습에서는 **VLAN 1의 STP 인스턴스**를 기준으로 동작을 확인합니다.

---

## 4. 1단계 — 초기 루트 브리지 선출

우선순위를 변경하기 전에 다음 명령어를 실행했습니다.

```cisco
show spanning-tree vlan 1
```

확인 결과, **SW3이 루트 브리지로 선출**되었습니다.

SW3의 Bridge ID가 가장 낮았기 때문입니다. Bridge ID는 우선순위, 확장 시스템 ID(VLAN ID), MAC 주소로 구성되며, 동일한 VLAN에서 우선순위가 같으면 MAC 주소가 낮은 스위치가 선출됩니다.

### 초기 동작

- **SW3:** 스위치 간 연결 포트가 지정 포트(Designated Port) 역할 수행
- **SW1 및 SW2:** 루트 브리지 방향의 루트 포트(Root Port)를 각각 선택
- 중복 경로에서는 STP 계산 결과에 따라 일부 포트가 Blocking 상태로 전환

---

## 5. 2단계 — SW2를 루트 브리지로 지정

SW2의 STP 우선순위를 낮춰 루트 브리지로 선출되도록 설정했습니다.

```cisco
spanning-tree vlan 1 priority 4096
```

### 수렴 후 결과

- SW2가 루트 브리지로 선출되었습니다.
- SW2의 활성 스위치 간 연결 포트는 지정 포트 역할을 수행했습니다.
- SW1과 SW3은 SW2까지의 경로 비용을 기준으로 루트 포트를 다시 선택했습니다.
- 중복 경로의 일부 포트는 Blocking 상태가 되었습니다.

### 검증 명령어

```cisco
show spanning-tree vlan 1
show spanning-tree root
```

> Priority 4096은 이 실습의 다른 스위치보다 낮은 값이므로 SW2가 선출되었습니다. 다른 장비에 더 낮은 우선순위가 설정되어 있다면 결과가 달라질 수 있습니다.

---

## 6. 3단계 — 중복 링크 추가 및 루프 방지 확인

**SW1 Fa0/1과 SW3 Fa0/1** 사이에 중복 링크를 추가했습니다.

추가 링크가 VLAN 1에 참여하는 상태에서 STP의 포트 역할 및 상태 변화를 확인했습니다.

### 관찰 결과

- STP가 새 링크를 반영하여 포트 역할과 상태를 계산했습니다.
- 중복 경로의 일부 포트가 Blocking 상태로 유지되거나 전환되었습니다.
- STP가 프레임 전달 루프를 방지하여 브로드캐스트 폭풍이 발생하지 않았습니다.

이를 통해 STP가 경로 비용과 Bridge ID 등의 정보를 비교하여 중복 경로를 제어하는 동작을 확인했습니다.

### 검증 명령어

```cisco
show spanning-tree vlan 1
```

> 출력에 `Altn`이 표시되는 경우 포트 역할을, `BLK`는 차단 상태를 나타냅니다. 포트 역할과 상태를 구분하여 확인합니다.

---

## 7. 4단계 — BPDU Guard에 의한 Err-Disable 테스트

SW2의 액세스 포트에 BPDU Guard를 활성화했습니다.

```cisco
interface fa0/1
 switchport mode access
 spanning-tree bpduguard enable
```

이후 SW4를 SW2 Fa0/1에 연결했습니다.

### 관찰 결과

- SW2 Fa0/1이 SW4에서 전송한 BPDU를 수신했습니다.
- BPDU Guard가 동작하여 해당 인터페이스를 **err-disabled** 상태로 전환했습니다.

### 검증 명령어

```cisco
show interfaces status err-disabled
show spanning-tree interface fa0/1 detail
```

이 결과를 통해 BPDU 수신이 예상되지 않는 단말 연결 포트에서, 스위치 연결로 발생할 수 있는 토폴로지 변경을 차단하는 동작을 확인했습니다.

> BPDU Guard는 연결 장비의 신원을 인증하는 기능이 아닙니다. 해당 포트에서 BPDU를 수신하면 포트를 비활성화하는 보호 기능입니다.

---

## 8. 핵심 관찰 내용

- STP는 가장 낮은 Bridge ID를 가진 스위치를 루트 브리지로 선출합니다.
- 우선순위를 조정하면 의도한 스위치가 루트 브리지로 선출되도록 제어할 수 있습니다.
- STP는 중복 경로의 일부 포트를 차단하여 Layer 2 전달 루프를 방지합니다.
- 루트 브리지의 위치는 루트 포트 선택과 실제 전달 경로에 영향을 줍니다.
- BPDU Guard는 BPDU를 수신한 포트를 err-disabled 상태로 전환하여 네트워크를 보호합니다.

---

## 9. 실습 스크린샷

다음 화면을 기록합니다.

1. 전체 네트워크 토폴로지
2. SW2를 루트로 지정하기 전의 `show spanning-tree vlan 1`
3. SW2가 루트로 선출된 후의 `show spanning-tree vlan 1`
4. 중복 링크 추가 후 포트 역할 및 Blocking 상태
5. BPDU Guard 동작 후 인터페이스의 err-disabled 상태
6. 트렁크 및 BPDU Guard 설정

---

## 10. 폴더 내 파일

- Packet Tracer `.pkt` 프로젝트 파일 — 별도 업로드
- 스크린샷 폴더
- `README.md` — 실습 설명 문서
