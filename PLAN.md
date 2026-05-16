# Astroquad 구현 계획서

최종 업데이트: 2026-05-16 KST

기준 문서: `development-log/RESEARCH.md`

구현 상태: 승인된 계획 기준 구현 완료. 아래 계획 본문은 추적용으로 유지하며,
실제 변경 결과는 이 섹션을 기준으로 본다.

완료 결과:

- `line_follow_node`에 `--line-mode auto|light_on_dark|dark_on_light` 옵션을 추가했다.
- MAVLink `RC_CHANNELS`를 `AutopilotState`에 반영하도록 adapter를 확장했다.
- SITL은 RC assumed-present, Pixhawk1 serial은 RC required profile로 분리했다.
- Pixhawk1 command-producing mission은 `--allow-arm-takeoff`가 있어도 fresh RC가
  확인되지 않으면 `GUIDED`, arm, takeoff, velocity command 전에 종료한다.
- 비행 중 RC loss는 LAND fallback, `GUIDED` 이탈은 operator takeover로 처리한다.
- `SafetyMonitor` unit test와 RC decode test를 추가/보강했다.
- WSL 로컬 검증에서 build와 `ctest` 14/14가 통과했다.

남은 실기체 gate:

- 라즈베리파이/Pixhawk 연결 후 `--strict-local-estimate`, `--strict-rc`,
  `--mavlink-smoke` no-arm gate를 다시 통과해야 한다.
- props-off motor order/direction, arm/disarm/LAND command path 확인이 필요하다.
- battery/failsafe와 `ARMING_CHECK` 위험은 실제 비행 전 별도로 판단해야 한다.

목표: 이번 구현 단계가 끝나면 사용자가 바로 현실에서 짧은 라인 트레이싱
테스트를 준비할 수 있어야 한다. 현실 비행 목표는 다음 MVP로 한정한다.

```text
자동 이륙
  -> 짧은 직선 라인 추종
  -> ArUco marker 발견
  -> 3초 hover
  -> 자동 착륙
```

이 문서는 원래 구현 계획서였고, 현재는 승인 후 실행된 작업의 추적 문서다.
아래 단계별 계획은 구현 의도와 검증 순서를 보존하기 위해 유지한다.

## 1. 이번 단계의 완료 기준

이번 단계는 단순히 코드가 빌드되는 것이 아니라, 현실 비행 직전까지 필요한
MVP 안전장치와 운용 명령이 완성된 상태를 목표로 한다.

완료 조건:

- `line_follow_node`가 `--line-mode auto|light_on_dark|dark_on_light`를 받는다.
- SITL에서는 RC 연결을 가정하여 기존 Gazebo line-follow smoke가 계속 비행한다.
- 실제 Pixhawk serial target에서는 RC 입력이 MAVLink `RC_CHANNELS`로 확인되기
  전까지 `GUIDED`, `arm`, `takeoff`, velocity command를 보내지 않는다.
- 실제 Pixhawk serial target에서 비행 중 RC 입력이 사라지면 companion control을
  중단하고 deterministic fallback으로 들어간다.
- 실제 Pixhawk serial target에서 비행 중 mode가 `GUIDED`가 아니게 되면
  operator takeover로 보고 velocity command를 멈춘다.
- 새 안전 로직은 unit test로 검증한다.
- 구현 완료 후 작업자는 사용자가 그대로 따라 할 수 있는 "명령어 실행 순서
  체크리스트"를 최종 답변에 반드시 제공한다.

이번 단계에서 하지 않는 것:

- full snake/grid exploration
- marker revisit
- GCS command UI
- RC override backend 구현
- battery monitor parameter 자동 변경
- Pixhawk parameter write 자동화 추가

## 2. 현재 전제

현재 라즈베리파이는 연결되어 있지 않다. 그러므로 구현자는 로컬 repo에서
코드와 설정을 수정하고, 로컬 build/test와 가능한 SITL 검증까지만 수행한다.

실기체 관련 현재 사실:

- Pixhawk USB serial stable path:
  `/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00`
- `runtime.pixhawk1.toml`은 serial target과 `rpicam` vision profile을 사용한다.
- `mavlink_probe --target pixhawk1 --strict-local-estimate`는 이전 bench에서 통과했다.
- 현재 기록상 RC는 아직 보이지 않는다: `rc: channels=0 rssi=255`.
- 현재 기록상 battery telemetry는 없다: `BATT_MONITOR=0`.
- 현재 기록상 `ARMING_CHECK=0`이다.

실제 비행 전 hard gate:

- `mavlink_probe --target pixhawk1 --strict-rc`가 통과해야 한다.
- props-off motor order/direction 확인이 끝나야 한다.
- props-off arm/disarm/land command path 확인이 끝나야 한다.
- battery/failsafe 위험을 사용자가 명시적으로 수용하거나 별도 설정해야 한다.

## 3. 작업 원칙

- `vision_debug_node`와 `line_follow_node`의 detector 코드를 분리하지 않는다.
  이미 공유 중인 `VisionProcessor`를 계속 사용한다.
- SITL과 실제 Pixhawk용 mission logic을 분리하지 않는다. 차이는 runtime config와
  safety config로 표현한다.
- 실제 serial target에서 command-producing path는 보수적으로 막는다.
- 기본값은 개발 편의보다 현실 안전을 우선한다. 단, SITL smoke는 기존처럼 바로
  돌 수 있어야 한다.
- 라즈베리파이에 연결되지 않은 상태에서 실제 Pixhawk 명령을 실행하려 하지 않는다.
- 코드 구현 후 README나 RESEARCH 업데이트가 필요하면, 실행 명령과 safety gate
  변경점만 짧게 갱신한다.

## 4. 구현 Phase 1: RC 상태를 AutopilotState에 추가

대상 파일:

- `uav-onboard/src/autopilot/AutopilotState.hpp`
- `uav-onboard/src/autopilot/AutopilotMavlinkAdapter.cpp`
- `uav-onboard/tests/test_autopilot_poll_drain.cpp`

구현 내용:

- `AutopilotState`에 RC 관측 필드를 추가한다.
  - `std::optional<int> rc_channel_count`
  - `std::optional<int> rc_rssi`
  - 필요하면 `std::array<std::uint16_t, 18> rc_channels_pwm`
  - `std::chrono::steady_clock::time_point last_rc_channels_time`
- `AutopilotMavlinkAdapter::processMessage()`에서
  `MAVLINK_MSG_ID_RC_CHANNELS`를 decode한다.
- `mavlink_probe.cpp`의 기존 decode 방식을 참고한다.
- `requestDefaultStreams()`의 `RC_CHANNELS` request는 유지한다.

테스트:

- `test_autopilot_poll_drain.cpp`의 fake transport 메시지 목록에
  `RC_CHANNELS` MAVLink message를 추가한다.
- `adapter.poll()` 이후 다음을 assert한다.
  - `rc_channel_count.has_value()`
  - `rc_channel_count > 0`
  - `rc_rssi.has_value()`
  - `last_rc_channels_time`이 설정됨

완료 기준:

- 기존 heartbeat/local-position test가 깨지지 않는다.
- RC message가 들어오면 adapter state에 반영된다.
- RC message가 없으면 state는 empty 상태를 유지한다.

## 5. 구현 Phase 2: SafetyConfig에 RC gate 설정 추가

대상 파일:

- `uav-onboard/src/safety/SafetyMonitor.hpp`
- `uav-onboard/src/safety/SafetyMonitor.cpp`
- `uav-onboard/config/safety.toml`
- `uav-onboard/config/runtime.sitl.toml`
- `uav-onboard/config/runtime.pixhawk1.toml`
- 필요 시 `uav-onboard/tools/line_follow_node.cpp`의 config loading 부분

구현 내용:

- `SafetyConfig`에 다음 필드를 추가한다.
  - `bool rc_required`
  - `bool assume_rc_present`
  - `int rc_lost_ms`
- base 또는 SITL 기본값은 SITL smoke가 깨지지 않게 둔다.
  - `assume_rc_present = true`
  - `rc_required = false`
- Pixhawk runtime overlay는 현실 비행 safety를 우선한다.
  - `assume_rc_present = false`
  - `rc_required = true`
  - `rc_lost_ms`는 1000-2000ms 범위에서 보수적으로 선택한다.
- `line_follow_node`의 `loadRuntimeConfig()`가 runtime overlay의 RC safety 값을
  읽도록 한다.

테스트:

- `SafetyMonitor` unit test가 아직 별도 파일로 없다면
  `tests/test_safety_monitor.cpp`를 추가한다.
- CMake에 `test_safety_monitor`를 등록한다.
- 테스트 케이스:
  - `assume_rc_present=true`이면 RC 입력 없이도 mission safety가 통과한다.
  - `rc_required=true`이고 RC 입력이 없으면 start gate가 실패한다.
  - mission 중 RC timestamp가 `rc_lost_ms`보다 오래되면 Land 또는 Abort decision을 낸다.

완료 기준:

- safety config가 base/SITL/Pixhawk profile별로 다르게 적용된다.
- unit test로 SITL-friendly path와 Pixhawk-strict path가 모두 검증된다.

## 6. 구현 Phase 3: 실제 serial preflight RC gate 추가

대상 파일:

- `uav-onboard/tools/line_follow_node.cpp`
- 필요 시 `uav-onboard/src/safety/SafetyMonitor.*`

구현 위치:

- `waitHeartbeat()`와 `requestDefaultStreams()` 이후
- `setGuidedMode()`, `arm()`, `takeoff()` 이전

구현 내용:

- `line_follow_node`에 preflight gate 함수를 만든다.
  예: `waitRcReady(AutopilotMavlinkAdapter&, SafetyConfig, timeout)`
- real serial target이고 `rc_required=true`이면 fresh `RC_CHANNELS`를 기다린다.
- fresh 조건:
  - `rc_channel_count.has_value()`
  - `rc_channel_count > 0`
  - `last_rc_channels_time`이 현재 시각 기준 `rc_lost_ms`보다 오래되지 않음
- timeout 안에 RC가 준비되지 않으면:
  - 명확한 에러 로그 출력
  - nonzero exit
  - `setGuidedMode`, `arm`, `takeoff`, `sendBodyVelocity` 호출 금지
- SITL 또는 `assume_rc_present=true`인 profile은 기존처럼 통과한다.

테스트:

- 가능하면 pure helper 함수로 분리해 unit test를 작성한다.
- 최소한 adapter RC decode test와 safety monitor test로 gate 판단을 검증한다.
- 실제 Pixhawk가 없는 상태에서는 real serial 실행 테스트를 하지 않는다.

완료 기준:

- `--target sitl`은 RC 없이 기존 동작을 유지한다.
- `--target pixhawk1 --allow-arm-takeoff`는 RC가 없으면 flight command 전에 중단한다.
- 중단 로그는 사용자가 왜 이륙하지 않았는지 바로 이해할 수 있어야 한다.

## 7. 구현 Phase 4: 비행 중 RC loss와 operator takeover 처리

대상 파일:

- `uav-onboard/tools/line_follow_node.cpp`
- `uav-onboard/src/safety/SafetyMonitor.*`
- 필요 시 `uav-onboard/src/autopilot/AutopilotState.hpp`

구현 내용:

- mission loop의 `SafetyInput`에 RC 상태와 mode 상태를 전달한다.
- `SafetyMonitor`가 다음 상황을 판단하도록 확장한다.
  - RC required인데 RC가 stale이면 fallback
  - `assume_rc_present=false`인데 RC channel count가 0이면 fallback
  - mission 중 mode가 `GUIDED`가 아니면 operator takeover
- fallback 정책:
  - vehicle mode가 아직 `GUIDED`이면 `Land` action
  - mode가 이미 `GUIDED`가 아니면 companion velocity command를 중단하고
    mission을 `ABORT` 또는 operator takeover reason으로 종료
- 어떤 경우에도 RC loss/operator takeover 이후 forward line-follow velocity를
  계속 보내지 않는다.

테스트:

- `test_safety_monitor`에 다음 케이스를 추가한다.
  - fresh RC가 있으면 `SafetyAction::None`
  - stale RC이면 `SafetyAction::Land` 또는 지정한 fallback action
  - mode가 `GUIDED`가 아니면 operator takeover action/reason
- `line_follow_node` loop에서 safety action 처리 후 velocity command가 추가로
  나가지 않는 구조인지 코드 리뷰한다.

완료 기준:

- RC loss와 operator mode takeover가 line lost와 별도 reason으로 로그에 남는다.
- fallback 이후 companion이 manual/operator control과 싸우지 않는다.

## 8. 구현 Phase 5: line_follow_node line-mode CLI 추가

대상 파일:

- `uav-onboard/tools/line_follow_node.cpp`

구현 내용:

- `Options`에 `std::string line_mode_override`를 추가한다.
- usage에 다음 옵션을 추가한다.

```text
--line-mode <auto|light_on_dark|dark_on_light>
```

- parser에서 옵션을 읽는다.
- `loadRuntimeConfig()`에서 base config와 runtime overlay 적용 후 CLI override를
  마지막에 적용한다.
- 허용 값은 `auto`, `light_on_dark`, `dark_on_light`로 제한한다.
- startup log에 선택된 line mode를 출력한다.

참고:

- `vision_debug_node.cpp`의 `--line-mode` 구현을 그대로 참고한다.
- detector path는 `VisionProcessor`를 계속 사용한다.

테스트:

- 최소 빌드 테스트로 parser compile을 확인한다.
- 가능하면 `--vision-smoke-count`와 함께 line mode가 startup/vision smoke 로그에
  반영되는지 수동 실행한다.

완료 기준:

- SITL 실행 명령에 `--line-mode light_on_dark`를 붙일 수 있다.
- 실제 Pi 실행 명령에 `--line-mode light_on_dark` 또는 `dark_on_light`를 붙일 수 있다.
- 공유 config 파일을 매번 수정하지 않고 현장 라인 색을 선택할 수 있다.

## 9. 구현 Phase 6: 문서와 실행 명령 정리

대상 파일:

- `development-log/RESEARCH.md`
- `development-log/PLAN.md`
- `uav-onboard/README.md`

작업 내용:

- 구현 완료 후 `RESEARCH.md`에 실제로 구현된 safety gate와 남은 blocker만 갱신한다.
- `README.md`의 real-hardware command 예시에 `--line-mode`와 RC gate 주의사항을
  추가한다.
- `PLAN.md`에는 완료 상태나 다음 단계만 짧게 반영한다.
- 최종 답변에는 반드시 명령어 실행 순서 체크리스트를 제공하라고 기록한다.

문서에 반드시 남길 내용:

- SITL은 RC assumed-present profile이다.
- Pixhawk serial은 RC required profile이다.
- `--allow-arm-takeoff`를 붙여도 RC gate가 실패하면 이륙하지 않는다.
- `vision_debug_node`와 `line_follow_node`는 동시에 GCS video/telemetry sender로
  실행하지 않는다.

## 10. 로컬 검증 계획

라즈베리파이가 연결되어 있지 않은 현재 상태에서 가능한 검증:

```powershell
cd c:\Users\mseoky\Documents\astro\astroquad\uav-onboard
cmake --build build
ctest --test-dir build --output-on-failure
```

WSL/Gazebo/SITL 환경이 준비되어 있으면 추가 검증:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node --config config --target sitl \
  --vision gazebo --line-mode light_on_dark --video
```

SITL 기대 결과:

- RC receiver 없이도 실행된다.
- startup log에 line mode와 RC assumed-present 설정이 드러난다.
- 기존처럼 marker approach, 3초 hover, land, complete에 도달한다.

## 11. 라즈베리파이 연결 후 체크리스트

구현 작업자가 최종 답변에 사용자용 명령어 실행 순서 체크리스트를 반드시 제공해야
한다. 체크리스트는 아래 순서를 기반으로 작성한다.

### 11.1 배포와 빌드

```bash
ssh astroquad@astroquad.local
cd /home/astroquad/astroquad/uav-onboard
git pull
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

### 11.2 Pixhawk no-arm gate

```bash
./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 12000 --strict-local-estimate

./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 30000 --strict-rc

./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

통과 기준:

- local estimate/range/flow gate 통과
- strict RC gate 통과
- smoke mode에서 mode/arm/takeoff/velocity command가 나가지 않음

### 11.3 카메라와 라인 색 확인

GCS를 켠 뒤 vision-only로 라인 색을 확인한다.

```bash
./build/vision_debug_node --config config \
  --line-only --line-mode light_on_dark --video
```

현장 라인이 어두운 라인이면:

```bash
./build/vision_debug_node --config config \
  --line-only --line-mode dark_on_light --video
```

통과 기준:

- GCS overlay에서 line center가 안정적으로 보인다.
- ArUco marker 테스트도 필요하면 별도 vision-only로 확인한다.
- 이 단계에서 `line_follow_node`는 동시에 실행하지 않는다.

### 11.4 props-off command bench

이 단계는 props removed 상태에서만 한다.

```bash
./build/mavlink_motor_test --config config --target pixhawk1 \
  --motor 1 --percent 5 --seconds 1 --props-removed
```

이후 motor 2, 3, 4를 같은 방식으로 확인한다.

추가로 arm/disarm/land command path 확인이 필요하면 구현 완료 후 제공되는 별도
명령을 사용한다. `--allow-arm-takeoff` full mission은 아직 실행하지 않는다.

### 11.5 첫 실제 line-follow 실행

모든 no-arm/props-off gate가 통과한 뒤에만 진행한다.

밝은 라인:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --vision rpicam --line-mode light_on_dark --video --allow-arm-takeoff
```

어두운 라인:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --vision rpicam --line-mode dark_on_light --video --allow-arm-takeoff
```

실행 전 조건:

- transmitter on/bound
- `mavlink_probe --strict-rc` 통과
- GCS video/telemetry 확인
- 배터리와 failsafe 위험 확인
- operator가 mode switch/kill/land 절차를 즉시 수행할 수 있음
- 주변 안전 확보

## 12. 실패 시 중단 기준

다음 중 하나라도 발생하면 실제 flight command 단계로 넘어가지 않는다.

- strict RC 실패
- local estimate/range/flow gate 실패
- GCS 영상 또는 line overlay가 불안정
- Pixhawk mode/armed 상태가 예상과 다름
- `line_follow_node --mavlink-smoke`가 command를 보낸 흔적이 있음
- motor order/direction이 확실하지 않음
- operator takeover 절차가 확인되지 않음

## 13. 최종 답변 요구사항

이 계획서를 실행하는 구현 작업자는 완료 후 최종 답변에 아래를 반드시 포함한다.

- 변경한 파일 요약
- 추가/수정한 safety gate 요약
- 실행한 테스트와 결과
- 실행하지 못한 테스트와 이유
- 라즈베리파이 연결 후 사용자가 그대로 따라 할 명령어 실행 순서 체크리스트
- 실제 `--allow-arm-takeoff` 실행 전 남은 hard blocker 여부
