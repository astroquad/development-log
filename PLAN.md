# Astroquad Implementation Plan

최종 업데이트: 2026-05-15 KST, Pixhawk serial bench 이후

이 문서는 `development-log/RESEARCH.md`를 기반으로 다음 에이전트가 구체적인 작업을 진행하기 위한 실행 계획이다. 목표는 **현재 SITL에서 검증된 mission/control 흐름을 유지하면서, Raspberry Pi 4와 실제 Pixhawk1 사이의 MAVLink serial 연결과 no-arm bench 검증 경로를 안전하게 완성하는 것**이다.

현재 요청 조건:

- 코드 수정은 Raspberry Pi에서 직접 하지 않는다.
- 코드 수정은 로컬 PC repo에서 수행하고, commit/push 후 Raspberry Pi에서 `git pull`로 받아 테스트한다.
- Pixhawk parameter 변경은 Raspberry Pi에서 직접 수행해도 된다.
- 단, 적용할 Pixhawk 설정 값은 repo에 설정 파일로 남겨 원격 저장소에 보존한다.
- 시뮬레이터와 실제 Pixhawk가 서로 다른 mission logic 코드를 쓰면 안 된다. 설정값만 바꾸면 transport와 runtime target이 바뀌어야 한다.

## 0. 현재 진행 상태

완료:

- `SerialMavlinkTransport` 구현.
- `mavlink_probe` no-arm bench tool 구현.
- `mavlink_motor_test` 저출력 모터 테스트 도구 구현.
- `line_follow_node --target pixhawk1 --mavlink-smoke` 안전 smoke 경로 구현.
- 실제 serial target에서 명시 safety flag 없이 자동 arm/takeoff mission을 거부하도록 guard 추가.
- `config/runtime.pixhawk1.toml`을 Pixhawk USB stable by-id path로 변경.
- `config/pixhawk1_usb.params`, `config/pixhawk1_usb.expected.toml` 추가.
- local Linux build/CTest 통과.
- commit `030a476` push 완료.
- Pi에서 `git pull`, build, CTest `13/13` 통과.
- Pi에서 `mavlink_probe --strict-local-estimate` exit status `0`.
- Pixhawk parameter `EK3_SRC1_POSXY=0`, `EK3_SRC1_VELZ=0` 적용 완료.

남은 immediate 작업:

- 사용자가 물리적으로 기체 주변을 정리한 상태에서 `mavlink_motor_test`로 모터별 회전 확인.
- RC receiver 입력이 왜 `RC_CHANNELS count=0`인지 확인.
- power module/battery monitor 설정 확인. 현재 `BATT_MONITOR=0`이라 battery telemetry가 없다.
- `ARMING_CHECK=0`을 최종 비행 전에 더 안전한 값으로 재설정할지 결정.
- props-off arm/disarm/mode command bench.

## 1. 목표 상태

최종 목표:

```text
같은 mission/control 코드
  + runtime 설정 target=sitl     -> UDP MAVLink transport + Gazebo vision
  + runtime 설정 target=pixhawk1 -> Serial MAVLink transport + rpicam vision
```

허용되는 차이:

- transport 종류: UDP vs serial
- frame source: Gazebo camera vs rpicam
- runtime config: endpoint, baudrate/device, safety timeout, tuning 값
- test command: no-arm probe, props-off command bench, full mission run

허용되지 않는 차이:

- 실기체 전용 mission state machine 별도 구현
- SITL과 실기체에서 다른 line-follow 판단 코드 사용
- GCS가 mission 판단을 대신 수행
- `vision_debug_node` detector 코드를 mission executable에 복사

## 2. 현재 확인된 사실

요약:

- Pixhawk USB MAVLink는 Pi에서 `/dev/ttyACM0` 및 stable by-id path로 통신 가능하다.
- Stable path는 `/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00`이다.
- `/dev/serial0`은 현재 `ttyS0` Linux console/getty에 묶여 있으므로 USB Pixhawk 테스트 기본값으로 쓰면 안 된다.
- Pixhawk firmware는 ArduCopter `V4.6.3`, frame은 `QUAD/X`, 현재 mode는 `STABILIZE`, disarmed다.
- Parameter list는 읽힌다.
- `FLOW_TYPE=5`, `RNGFND1_TYPE=10`, `EK3_ENABLE=1` 등 MTF-01 관련 설정 후보는 존재한다.
- 2026-05-15 bench 이후 `LOCAL_POSITION_NED`, `DISTANCE_SENSOR`, `RANGEFINDER`, `OPTICAL_FLOW`, `EKF_STATUS_REPORT`가 확인됐다.
- `RC_CHANNELS count=0`, `BATT_MONITOR=0`, `ARMING_CHECK=0` 상태도 확인됐다.
- `line_follow_node`는 serial target을 지원하되, real serial에서는 `--mavlink-smoke`, `--no-arm`, `--dry-run`, `--allow-arm-takeoff` 중 하나 없이는 실행을 거부한다.

첫 구현 작업인 **no-arm MAVLink probe + serial transport**는 완료됐다. 다음 작업은 RC/battery/failsafe 확인, 물리적으로 관찰하는 motor test, 그리고 props-off command bench다.

## 3. 절대 금지사항

실기체 안전:

- 현재 `line_follow_node`를 실제 Pixhawk에 바로 연결해 실행하지 않는다.
- no-arm/dry-run/probe 경로가 생기기 전에는 실제 Pixhawk에 `setGuidedMode`, `arm`, `takeoff`, `land`, `sendBodyVelocity`, `RC_CHANNELS_OVERRIDE`를 보내지 않는다.
- props 장착 상태 테스트는 이 계획 범위 밖이다.
- `ARMING_CHECK=0`인 현재 상태를 안전한 상태로 간주하지 않는다.
- `RC_CHANNELS count=0` 상태에서 RC takeover가 가능하다고 가정하지 않는다.
- `LOCAL_POSITION_NED`와 range/flow가 확인되지 않은 상태에서 GUIDED velocity line-follow를 시도하지 않는다.

코드/운영:

- Raspberry Pi에서 직접 코드 수정하지 않는다.
- Pi에서 임시로 고친 파일을 테스트 후 방치하지 않는다.
- 로컬 변경 없이 Pi에서만 빌드 결과를 바꾸지 않는다.
- `/dev/serial0`을 현재 USB Pixhawk 기본 endpoint로 설정하지 않는다.
- SITL용 mission logic과 Pixhawk용 mission logic을 분리하지 않는다.
- full snake/revisit/GCS command UI를 현재 blocker 해결 작업에 섞지 않는다.

## 4. 작업 흐름 규칙

코드 작업 workflow:

```text
local PC repo에서 수정
  -> local build/test
  -> git status 확인
  -> commit
  -> push
  -> SSH into Pi
  -> cd /home/astroquad/astroquad/uav-onboard
  -> git pull
  -> cmake/build/test
  -> Pixhawk bench probe
```

Pixhawk parameter 작업 workflow:

```text
local repo에 parameter 파일 작성/수정
  -> commit/push
  -> Pi에서 git pull
  -> Pi에서 no-arm parameter diff 확인
  -> 필요할 때만 Pi에서 Pixhawk에 parameter 적용
  -> 적용 후 다시 parameter dump
  -> dump 결과를 local repo에 반영하거나 별도 기록 파일로 저장
```

권장 SSH:

```bash
ssh astroquad@astroquad.local
```

Pi repo path:

```bash
/home/astroquad/astroquad/uav-onboard
```

## 5. Phase 0: 기준 동기화 - 완료

목표: local repo와 Pi repo 기준을 맞추고, 이후 테스트가 어떤 commit에서 수행됐는지 추적 가능하게 한다.

작업:

- local `uav-onboard`와 remote Pi `uav-onboard`의 branch/HEAD 차이를 확인한다.
- Pi의 현재 HEAD는 조사 시점에 `b337d02`, local은 `23214a4`였다. 다음 작업 전 반드시 다시 확인한다.
- local 변경사항이 있으면 내용을 이해하고 보존한다.
- Pi repo에 local-only 변경이 있으면 바로 덮어쓰지 말고 diff를 확인한다.
- 이후 구현 기준 commit을 하나로 정한다.

성공 기준:

- local과 Pi가 같은 branch를 보고 있다.
- 어떤 commit을 Pi에 배포해 테스트할지 명확하다.
- `git status --short`가 local/Pi 모두 의도한 상태다.

## 6. Phase 1: no-arm MAVLink probe 추가 - 완료

목표: 실제 Pixhawk와 통신하되 절대 arm/takeoff/mode 변경을 하지 않는 독립 실행 파일을 만든다.

권장 실행 파일:

```text
uav-onboard/tools/mavlink_probe.cpp
```

권장 CLI:

```bash
./build/mavlink_probe --config config --target pixhawk1
./build/mavlink_probe --autopilot serial:///dev/ttyACM0:115200
./build/mavlink_probe --autopilot serial:///dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00:115200
./build/mavlink_probe --strict-local-estimate
./build/mavlink_probe --dump-params
```

필수 동작:

- serial endpoint open
- companion/GCS heartbeat 송신
- Pixhawk heartbeat 수신
- mode/armed/sysid/component 출력
- message interval request:
  - `SYS_STATUS`
  - `LOCAL_POSITION_NED`
  - `GLOBAL_POSITION_INT`
  - `DISTANCE_SENSOR`
  - `RANGEFINDER`
  - `OPTICAL_FLOW`
  - `OPTICAL_FLOW_RAD`
  - `EKF_STATUS_REPORT`
  - `RC_CHANNELS`
- 주요 parameter read/dump
- missing gate를 명확한 exit code로 반환

절대 금지:

- `setGuidedMode`
- `setLandMode`
- `arm`
- `disarm`
- `takeoff`
- `sendBodyVelocity`
- `RC_CHANNELS_OVERRIDE`
- `COMMAND_LONG` 중 actuator/flight mode에 영향을 주는 명령

성공 기준:

- Pixhawk heartbeat를 30초 안에 수신한다.
- `mode=STABILIZE`, `armed=false` 같은 상태를 정확히 출력한다.
- parameter list 또는 selected parameter가 읽힌다.
- missing `LOCAL_POSITION_NED`, range, flow, RC 상태를 실패/경고로 명확히 표시한다.
- 이 도구를 여러 번 실행해도 Pixhawk mode/armed 상태가 바뀌지 않는다.

## 7. Phase 2: SerialMavlinkTransport 구현 - 완료

목표: 기존 `MavlinkTransport` interface를 유지하면서 serial/USB MAVLink endpoint를 직접 지원한다.

권장 파일:

```text
uav-onboard/src/autopilot/SerialMavlinkTransport.hpp
uav-onboard/src/autopilot/SerialMavlinkTransport.cpp
```

요구사항:

- POSIX `termios` raw mode
- `/dev/ttyACM0`, `/dev/ttyUSB0`, `/dev/serial/by-id/...`, 추후 `/dev/ttyAMA0` 지원
- configurable baudrate
- USB CDC는 baudrate가 nominal이어도 config를 유지
- `recvMessage(mavlink_message_t&, timeout_ms)`에서 `select` timeout 처리
- byte stream MAVLink parser 처리
- partial read/write 처리
- `EINTR`, `EAGAIN`, short write 대응
- close/restore RAII
- permission denied, no such device, busy device, timeout 에러 메시지 명확화
- UDP transport와 같은 `MavlinkTransport` interface 사용

테스트:

- serial parsing unit test는 pseudo terminal 또는 작은 parser test로 가능하면 추가한다.
- 기존 UDP transport test를 깨지 않는다.
- `mavlink_probe`가 `SerialMavlinkTransport`를 통해 `/dev/ttyACM0` heartbeat를 읽는다.

성공 기준:

- SITL 기존 `UdpMavlinkTransport` test 통과.
- Pi에서 serial transport로 Pixhawk heartbeat 수신.
- serial transport 실패 시 actionable error가 나온다.

## 8. Phase 3: Pixhawk parameter config 파일 관리 - 1차 완료

목표: Pixhawk 설정 변경을 사람이 기억에 의존하지 않고 repo에서 추적한다.

권장 파일:

```text
uav-onboard/config/pixhawk1_usb.params
uav-onboard/config/pixhawk1_usb.expected.toml
uav-onboard/config/pixhawk1_usb.observed.example.toml
```

역할:

- `pixhawk1_usb.params`: ArduPilot parameter apply용 canonical 파일.
- `pixhawk1_usb.expected.toml`: probe가 읽어야 할 핵심 기대값과 gate 기준.
- `pixhawk1_usb.observed.example.toml`: 조사 시점 관찰값 예시 또는 template.

초기 expected에 포함할 항목:

```toml
[identity]
sysid = 1
frame_class = 1
frame_type = 1
firmware_family = "ArduCopter"

[mavlink]
transport_device = "/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00"
baudrate = 115200

[ekf]
ahrs_ekf_type = 3
ek3_enable = 1
ek3_flow_use = 1
ek3_src1_posxy = 0
ek3_src1_velxy = 5
ek3_src1_posz = 1
ek3_src1_velz = 0
ek3_src1_yaw = 1

[rangefinder]
rngfnd1_type = 10
rngfnd1_orient = 25
rngfnd1_min_cm = 1
rngfnd1_max_cm = 800

[flow]
flow_type = 5

[safety]
arming_check_expected_nonzero = true
battery_monitor_expected_nonzero = true
rc_channels_required = true
```

주의:

- 현재 관찰값은 `ARMING_CHECK=0`, `BATT_MONITOR=0`, `RC_CHANNELS count=0`이다.
- 이 값들을 그대로 "좋은 설정"으로 고정하지 않는다.
- `expected` 파일에는 목표/gate 기준을 쓰고, `observed` dump에는 현재 사실을 쓴다.
- 실제 parameter apply는 no-arm probe와 diff 출력이 안정된 뒤 진행한다.

성공 기준:

- repo에 Pixhawk 설정 목표와 관찰 dump 형식이 생긴다.
- probe가 expected file과 현재 Pixhawk parameter를 비교할 수 있다.
- 변경한 Pixhawk parameter는 commit history로 추적 가능하다.

## 9. Phase 4: runtime config 통합 - 1차 완료

목표: 설정값만 바꾸면 SITL과 Pixhawk가 같은 mission/control code path를 사용하게 한다.

수정 방향:

- `runtime.sitl.toml`:
  - transport `udp`
  - vision `gazebo`
  - SITL-specific topic/timeout/tuning
- `runtime.pixhawk1.toml`:
  - transport `serial`
  - vision `rpicam`
  - serial device는 current wiring 기준 stable by-id path
  - no-arm smoke mode가 기본이거나 별도 probe tool 사용

권장 Pixhawk runtime 기본 serial:

```toml
[serial]
device = "/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00"
baudrate = 115200
```

주의:

- `/dev/serial0`은 현재 USB Pixhawk 테스트 기본값으로 쓰지 않는다.
- 나중에 TELEM UART로 옮기려면 별도 `runtime.pixhawk1_uart.toml`을 만들고 console/getty disable 절차를 문서화한다.

성공 기준:

- `--target sitl`은 기존 SITL smoke를 유지한다.
- `--target pixhawk1`은 serial endpoint를 선택한다.
- mission/control 코드 중 endpoint 종류를 직접 분기하는 부분이 최소화된다.
- transport 선택은 factory/config 레벨에서 끝난다.

## 10. Phase 5: Pi 배포와 no-arm bench 테스트 - 완료

목표: 로컬에서 구현한 serial/probe를 Pi에 배포해 props-off 상태에서 통신 gate를 확인한다.

절차:

```bash
# local PC
git status --short
cmake --build build
ctest --test-dir build --output-on-failure
git add ...
git commit -m "Add no-arm MAVLink serial probe"
git push

# Raspberry Pi
ssh astroquad@astroquad.local
cd /home/astroquad/astroquad/uav-onboard
git pull
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
./build/mavlink_probe --config config --target pixhawk1
```

성공 기준:

- Pi build succeeds.
- tests pass or known unrelated failures are documented.
- `mavlink_probe` opens Pixhawk USB serial.
- heartbeat/mode/armed/sysid/component are printed.
- no flight-affecting command is sent.
- Pixhawk remains disarmed and in the same operator-selected mode.

## 11. Phase 6: MTF-01/EKF local estimate gate - no-arm bench 통과

목표: GUIDED velocity를 쓰기 전에 MTF-01 range/flow와 EKF local estimate를 실제로 확인한다.

Probe가 확인해야 하는 것:

- `LOCAL_POSITION_NED` appears.
- local `x/y/z`와 `vx/vy/vz`가 finite 값이다.
- range telemetry appears:
  - `DISTANCE_SENSOR` 또는 ArduPilot-specific `RANGEFINDER`
- optical flow telemetry appears:
  - `OPTICAL_FLOW` 또는 `OPTICAL_FLOW_RAD`
- 바닥 높이 변화에 range가 반응한다.
- 기체를 손으로 천천히 움직이면 flow/local velocity가 반응한다.
- EKF flags가 local estimate gate를 만족한다.
- RC input이 `RC_CHANNELS`로 보인다.

실패 시 해야 할 일:

- 코드 mission 확장으로 넘어가지 않는다.
- Pixhawk parameter expected/observed를 비교한다.
- MTF-01가 연결된 Pixhawk port/protocol/baud 설정을 확인한다.
- 필요한 parameter 변경은 repo의 params file에 먼저 반영한다.
- Pi에서 parameter apply 후 다시 dump/probe한다.

성공 기준:

- no-arm probe에서 local estimate/range/flow/RC가 모두 확인된다.
- 결과가 `development-log/RESEARCH.md` 또는 별도 bench log에 갱신된다.
- 다음 props-off command bench로 넘어갈 수 있다.

## 12. Phase 7: line_follow_node 안전 옵션 추가 - 1차 완료

목표: 실제 Pixhawk target에서도 같은 executable을 쓰되, 실기체 bench 단계에서 arm/takeoff를 막을 수 있게 한다.

권장 CLI:

```bash
./build/line_follow_node --config config --target pixhawk1 --vision rpicam --no-arm
./build/line_follow_node --config config --target pixhawk1 --vision rpicam --dry-run
./build/line_follow_node --config config --target pixhawk1 --vision rpicam --mavlink-smoke
```

권장 의미:

- `--mavlink-smoke`: heartbeat/streams/state only, mission start 없음.
- `--dry-run`: vision/control 계산은 하되 Pixhawk에 flight-affecting command 송신 없음.
- `--no-arm`: mode/request stream은 허용할 수 있으나 arm/takeoff 금지. 정확한 허용 범위는 코드에서 명시적으로 분리한다.

주의:

- `--no-arm`이 있어도 mode change가 안전한지 별도로 판단한다.
- 첫 버전은 mode change도 하지 않는 `--mavlink-smoke`를 우선 구현한다.
- 실제 command path는 명시적 플래그 없이는 열리지 않게 한다.

성공 기준:

- Pixhawk target으로 실행해도 기본 또는 smoke 모드에서 arm/takeoff가 발생하지 않는다.
- SITL full mission smoke는 기존처럼 가능하다.
- command-producing branch가 로그에 명확히 표시된다.

## 13. Phase 8: props-off command bench - 다음 단계

전제:

- Phase 1-7 완료.
- no-arm probe에서 local estimate/range/flow/RC 확인.
- operator가 props off 상태를 직접 확인.
- GCS 또는 Mission Planner로 상태 관찰 가능.

작업 순서:

1. Stream request/state read만 반복.
2. Land command를 disarmed/ground 상태에서 어떻게 처리하는지 확인.
3. Mode switch command를 짧게 확인하되, operator mode switch로 즉시 되돌릴 수 있게 준비.
4. Arm command는 마지막에 별도 명시 승인을 받고 수행.
5. Arm 성공 시 즉시 disarm path 확인.
6. takeoff는 이 phase에서 하지 않는다.

성공 기준:

- mode/armed 상태 변화가 정확히 telemetry에 반영된다.
- arm/disarm path가 props-off에서 확인된다.
- RC takeover/mode switch 절차가 확인된다.
- 실패 시 Pixhawk가 안전 상태로 돌아온다.

## 14. Phase 9: 첫 실기체 line-follow 준비

전제:

- props-off command bench 완료.
- MTF-01 local estimate gate 완료.
- RC takeover 확인.
- battery monitor/failsafe 또는 대체 운용 절차 확인.
- IMX519 camera/focus/FOV/line threshold 현장 확인.
- prop 장착 전/후 체크리스트 작성.

첫 실기체 목표:

```text
auto takeoff
  -> 즉시 land
  -> 짧은 2-5m 직선 line follow
  -> line end/lost/marker/timeout에서 land
```

금지:

- snake
- marker revisit
- 교차점 회전
- 장거리 autonomous mission

## 15. 문서 갱신 규칙

작업 중 갱신할 파일:

- `development-log/RESEARCH.md`: 조사 결과, 실제 관찰값, 문제 원인.
- `development-log/PLAN.md`: 현재 실행 계획과 checklist.
- `development-log/TROUBLESHOOTING.md`: 반복 문제와 해결 이력.
- `uav-onboard/config/pixhawk1_usb.*`: Pixhawk 설정 목표/관찰/적용값.

기록해야 하는 값:

- 실행 commit hash
- Pi hostname/IP
- serial device path
- Pixhawk firmware version
- mode/armed state
- parameter diff
- observed telemetry messages
- missing telemetry messages
- pass/fail와 다음 action

## 16. 다음 에이전트 첫 작업

다음 에이전트가 실제 구현을 시작한다면 이 순서로 진행한다.

1. `RESEARCH.md`와 이 `PLAN.md`를 읽는다.
2. local/Pi repo revision 차이를 확인한다.
3. `mavlink_probe` 설계를 코드에 반영한다.
4. `SerialMavlinkTransport`를 구현한다.
5. `pixhawk1_usb.expected.toml`과 `pixhawk1_usb.params` 초안을 추가한다.
6. local build/test를 통과시킨다.
7. commit/push 한다.
8. Pi에서 `git pull` 후 build/test 한다.
9. props-off `mavlink_probe`만 실행한다.
10. 결과를 `RESEARCH.md`와 필요 시 `TROUBLESHOOTING.md`에 갱신한다.

첫 구현 PR/commit의 완료 조건:

- 코드가 serial transport를 지원한다.
- no-arm probe가 실제 Pixhawk heartbeat를 읽는다.
- Pixhawk에 arm/takeoff/mode-change/velocity command를 보내지 않는다.
- SITL UDP path가 깨지지 않는다.
- runtime config만으로 SITL/Pixhawk transport 선택 방향이 정리된다.
