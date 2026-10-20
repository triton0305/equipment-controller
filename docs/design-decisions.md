# Equipment Controller — Design Decisions & Collaboration

기준일: 2026-10-13 · 현재 구현 기준선: `c5cbb81`.

장비 운용 요구, 실제 보드 관찰과 후속 구현을 바탕으로 주요 설계 결정과 사용자·ChatGPT·Codex의 역할을 기록합니다. 사용자 발언은 원문 말투를 최대한 살리고, 맞춤법과 어색한 표현만 조금 다듬어 발언 요지로 표기했습니다. Host와 Hardware 확인 범위는 [Verification Report](verification.md)에 연결합니다.

## 역할과 근거

| 주체 | 확인된 책임 |
|---|---|
| 사용자 | 운용 요구·공정 시간·READY 범위 결정, 실제 보드 조작과 관찰, 구현 결과 재평가, 기준선 관리 |
| ChatGPT | 요구·로그 해석, 상태·안전·통신 책임의 구조화, 대안 검토와 검증 기준 정리 |
| Codex | 확정된 명세의 Controller / STM32 / CLI 구현, Build·Host regression과 관련 문서 작업 |

설계 결정의 근거, 구현된 코드, 자동 테스트와 사용자 실물 관찰을 각각 구분합니다. Git commit은 구현의 연결 근거로 사용하고, 그것만으로 제안자나 실물 검증자를 귀속하지 않습니다.

## 1. Pump / Heater 공정 재설계

초기 실물 실행에서 Pump와 Heater 단계가 기대한 공정 형태만큼 명확하게 구분되지 않았습니다.

> **발언 요지:** “아직도 히터랑 펌프가 동시에 켜지네.”

사용자는 전체 30초를 유지하며 다음 시간 구성을 제안했습니다.

> **발언 요지:** “START 입력받고 이상 없으면 6초 뒤에 Pump ON, 다시 6초 뒤에 Heater ON, 6초 뒤에 Heater OFF, 6초 뒤에 Pump OFF, 마지막 6초 뒤에 공정 종료. 이러면 총 30초지?”

| START 수락 후 | 동작 |
|---|---|
| 0초 | PROCESS START |
| 6초 | Pump ON |
| 12초 | Heater ON |
| 18초 | Heater OFF |
| 24초 | Pump OFF |
| 30초 | PROCESS COMPLETE |

Blocking delay 대신 Runtime timer를 사용하고 Safety event를 정상 공정보다 우선 처리합니다. 실제 보드에서 이벤트 순서를 확인했습니다.

사용자는 단계별 운용 요구와 시간표를 결정하고 실제 출력을 확인했습니다. ChatGPT는 시간·출력·Safety 우선순위를 명세로 정리했습니다. Codex는 경과시간 기반 sequence와 회귀 테스트를 구현했습니다.

**상태: HOST / HW VERIFIED — 순서·출력·정상 완료**

## 2. READY를 시작 자격으로 확장

> **발언 요지:** “READY 조건이 Door CLOSED 5초만 있는 게 아니라, 압력이랑 온도도 START 조건에 넣어야 한다.”

사용자는 READY가 Door 상태뿐 아니라 실제 시작 가능 조건을 나타내야 한다고 정리했습니다.

> **발언 요지:** “온도는 20–40°C로 잡고, 압력은 1000–2000으로 잡을까?”

최종 정책은 Idle / Armed, 정상 통신, fresh Door CLOSED / E-Stop OFF, Temperature 20–40°C와 Pressure ADC 1000–2000을 5초 연속 유지하는 것입니다. 조건 이탈·invalid·stale은 시간을 초기화하며 정상 복귀 후 새 연속 5초를 요구합니다. Humidity는 Critical Safety에 유지합니다.

사용자는 가변저항을 실제로 조작해 범위 이탈의 Ready=0, 복귀 후 재qualification과 Ready=1을 확인했습니다. ChatGPT는 qualification·취소·START 순간 재검사 조건을 정리했고, Codex는 Runtime gate와 경계 regression을 반영했습니다.

**Pressure / 5초 requalification: HOST / HW VERIFIED**  
Temperature 범위 밖의 HW 시험은 후속 항목이며 정상 범위의 실제 READY 진행을 관찰했습니다.

## 3. 시작 조건과 Critical Safety 분리

시작에 적합한 정상 범위와 운전 중 Error로 정지해야 하는 위험 범위는 각각 다른 정책입니다.

| 정책 | 판단 |
|---|---|
| READY / START | 새 공정을 시작할 수 있는가 |
| Critical Safety | 운전 중 위험 측정이 확인됐는가 |
| RESET guard | Fault 해소 후 Error를 해제할 수 있는가 |

READY 범위를 조금 벗어난 RUN 측정만으로 Critical Error를 만들지 않습니다. Critical은 Temperature ≤5 / ≥50°C, Humidity ≤15 / ≥90%, Pressure ADC ≤200 / ≥3900의 같은 방향 새 유효 측정 2회로 판단합니다.

사용자는 이전 논의에서 정지 기준·복구 조건·공정 조건을 분리하도록 확인했습니다. ChatGPT는 그 의미를 구분해 정책으로 정리했고 Codex는 measurement count와 시작 gate를 각각 구현했습니다. 현재 정책은 [Process Control](process-control.md)에 기록합니다.

## 4. CommunicationFault와 E-Stop, Linux와 MCU 책임

실제 통신 단절 시험에서 관찰 흐름을 사용자가 정정했습니다.

> **발언 요지:** “아니, 실제로는 CommunicationFault 발생 → Shutdown → 통신 재연결 → Error(빨간 LED/부저) → RESET으로 해제되는 순서였어.”

통신이 끊긴 경로로 새로운 표시 명령을 보낼 수 있는지 사용자가 질문했습니다.

> **발언 요지:** “통신이 끊어지면 Linux에서 보낸 명령이 STM32에 전달되지 않으니까, LED도 켤 수 없는 거 아니야?”

최종 책임은 다음과 같습니다.

| 구성 | 책임 |
|---|---|
| STM32 | 물리 I/O · 독립 command-loss timeout · 출력 차단 · local RED |
| Linux | 상위 State Machine · CommunicationFault Alarm · Error / RESET |
| 물리 E-Stop | EmergencyStop 원인의 Safety input |

사용자는 실제 local shutdown과 후속 Linux ERROR 표시 순서를 확인하고 통신 단절 표시 요구를 정리했습니다. ChatGPT는 Alarm 원인과 물리 출력 책임을 분리했습니다. Codex는 local fail-safe / latch / RED ownership과 회귀를 구현했습니다.

Command-loss 출력 차단은 HW VERIFIED입니다. 최신 Local RED 인계와 timing의 Host / HW 범위는 [검증 보고서](verification.md)에서 구분합니다.

## 5. UART 이상을 원인 미확정 상태로 관리

실물 startup / reconnect에서 `DODOOR:OPEN`, `DDOOR:OPEN`, `PDOOR:OPEN` 등을 관찰했습니다. Malformed line이 있어도 Alarm=0 / Idle을 유지한 실행과 recovery 후 Alarm=6으로 전환한 실행이 각각 있었습니다.

따라서 malformed 입력과 유효 telemetry 공백을 같은 원인으로 묶지 않았습니다. Host에서 delay / partial 조건을 재현한 결과와 실제 MCU / VCP / USB-IP 경로의 원인 판정을 구분했습니다.

사용자는 실제 반복 실행과 로그·계측 결과를 제공했습니다. ChatGPT는 관측 현상, 원인 후보, Host 재현과 HW 결론을 분리했습니다. Codex는 synchronization 방어·timing recorder·regression과 출력 갱신 최적화를 구현했습니다.

후속 계측의 일부 Fault 구간에서는 Linux polling이 이어지면서 약 1초 유효 telemetry 공백이 나타났습니다. 발생 계층은 확정하지 못해 조사 보류로 관리합니다. 재개 시 STM32 TX와 Linux RX 동시 계측을 우선합니다.

**상태: HW 이상 재현 / 원인 미확정**

## 6. Runtime과 Operator CLI 분리

기존 `./run` 터미널에 함께 있던 주기 상태 로그와 명령 입력을 별도의 Operator Console로 분리했습니다.

| 실행 | 책임 |
|---|---|
| Terminal 1 · `./run` | UART · Controller · Safety · Sequence · 상태 로그 |
| Terminal 2 · `./ctl` | 입력·응답 · Unix socket으로 Runtime에 요청 |

기존 stdin과 신규 IPC는 공통 Runtime dispatcher를 사용합니다. CLI는 UART를 열거나 별도의 시작·안전 정책을 소유하지 않습니다.

사용자는 실제 CLI 연결, HELP / STATUS, START 거부·수락, 기존 30초 공정과 RESET을 확인했습니다. ChatGPT는 인터페이스와 제어 정책의 책임을 검토하고 인수인계·검증 범위를 정리했습니다. Codex는 CLI / socket / dispatcher 연결과 process regression을 구현했습니다.

**상태: HOST / HW VERIFIED — 기본 CLI 운전 경로**

## 7. 동작한 STOP을 공개 CLI에서 제거

초기 CLI에서는 실제 `start → OK`, `stop → OK`를 확인했습니다. 이후 사용자가 운용 의미를 다시 검토했습니다.

> **발언 요지:** “STOP은 빼야 하는 거 아님?”

> **발언 요지:** “공정 중에 `ctl`에서 STOP을 보내면 그냥 파란 LED 상태로 넘어가는 게 좀 이상한데?”

STOP이 정상 완료와 유사한 상태 표시로 이어지는 점이 문제였습니다. 정상 완료, Operator Abort와 물리 EmergencyStop을 구분할 필요가 있었고 현재 범위에서는 새 Aborted 상태보다 공개 CLI에서 STOP을 제거하는 방향을 선택했습니다.

최종 명령은 `start reset status help exit`입니다. CLI는 STOP을 로컬에서 거부하고 IPC로 전달하지 않습니다. 기존 Core / stdin STOP은 유지합니다.

사용자는 실제 동작 뒤 운용 의미의 문제를 제기했습니다. ChatGPT는 Completion / Abort / E-Stop의 차이와 변경 범위를 정리했고 Codex는 CLI parsing과 테스트·문서를 최종 정책에 맞췄습니다.

**상태: CLI STOP 로컬 차단 HOST / HW VERIFIED**

## 8. 최종 설계·검증 원칙

요구와 판단은 실제 장비 운용에서 시작하고, 명세·구현·자동 테스트·실물 확인 후 다시 평가합니다.

- START 자격, Critical Fault와 RESET 조건을 각각 관리합니다.
- CommunicationFault와 물리 E-Stop의 Alarm 원인을 구분합니다.
- Linux의 제어 판단과 STM32의 독립 출력 안전정지를 구분합니다.
- Operator interface는 기존 Runtime 정책을 사용합니다.
- Host 재현과 Hardware root cause를 각각 판단합니다.
- Build / Test PASS와 실물 확인 범위를 각각 기록합니다.
- 동작한 기능도 실제 운용 의미를 확인해 유지·변경·제거합니다.

관련: [CLI](cli.md), [Process Control](process-control.md), [Troubleshooting](troubleshooting.md), [Verification Report](verification.md).

초기 설계 논의와 사용자·ChatGPT·Codex의 상세 역할 이력은 [초기 설계·협업 기록](design-history.md)에 별도로 보존합니다.
