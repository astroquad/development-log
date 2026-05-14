# Astroquad Implementation Plan

최종 업데이트: 2026-05-15

이 문서는 `RESEARCH.md` 기반으로 다음 에이전트가 바로 구현에 착수할 수 있도록 작성한 구체 계획서다.

중요: 이 문서를 작성하는 현재 단계에서는 **코드를 수정하지 않는다.** 다음 작업 턴에서만 아래 계획에 따라 구현한다.

## 0. 이번 작업 목표

현재 Gazebo SITL + Windows GCS 경로는 정상이다. 남은 핵심은 `line_follow_node`의 제어/상태머신을 실질적인 line-follow mission으로 만드는 것이다.

구현 목표:

```text
GUIDED 진입
  -> arm
  -> 2m takeoff
  -> 착륙 전까지 2m altitude hold 의도 유지
  -> line center가 camera center에 오도록 lateral/yaw 제어
  -> line angle이 desired angle에 수렴하도록 yaw 제어
  -> ArUco marker 인식
  -> marker 중심 좌표 위에서 3초 hover
  -> marker 위치에서 land
  -> COMPLETE
```

실기체 target은 A안이다.

```text
A: EKF local estimate + ARM/GUIDED + LOCAL_POSITION_NED + BODY_NED velocity/setpoint
B: ALT_HOLD + RC_CHANNELS_OVERRIDE fallback 설계만 준비
```

## 1. 반드시 지킬 경계와 금지사항

- `vision_debug_node` detector 코드를 mission executable에 복사하지 않는다.
- `VisionDebugPipeline` 내부에 MAVLink/control 코드를 넣지 않는다.
- GCS가 mission 판단을 하게 만들지 않는다.
- GCS가 line/marker/intersection detection을 다시 수행하게 만들지 않는다.
- debug video를 mission-critical 경로로 넣지 않는다.
- full snake/revisit/command UI를 이번 line-follow MVP의 선행 조건으로 만들지 않는다.
- MTF-01/EKF/GUIDED gate 없이 실기체 자동비행 준비 완료로 간주하지 않는다.
- unrelated refactor, 대규모 UI 전환, protocol 대개편을 섞지 않는다.
- 이번 계획을 구현할 때도 기존 `vision_debug_node --target sitl --vision gazebo --video` 경로가 깨지면 안 된다.

## 2. 현재 코드에서 시작할 위치

주요 구현 파일 후보:

- `uav-onboard/tools/line_follow_node.cpp`
- `uav-onboard/src/mission/LineFollowMission.*`
- `uav-onboard/src/control/GuidedVelocityController.*`
- `uav-onboard/src/control/ControlSetpoint.hpp`
- `uav-onboard/src/autopilot/AutopilotMavlinkAdapter.*`
- `uav-onboard/src/autopilot/AutopilotState.hpp`
- `uav-onboard/src/safety/SafetyMonitor.*`
- `uav-onboard/config/mission.toml`
- `uav-onboard/config/runtime.sitl.toml`
- `uav-onboard/config/autopilot.toml`
- `uav-onboard/config/safety.toml`
- `uav-onboard/tests/`

필요하면 추가할 수 있는 파일:

- `uav-onboard/src/mission/MissionStateMachine.*`
- `uav-onboard/src/control/LineFollowController.*`
- `uav-onboard/src/control/ControlBackend.hpp`
- `uav-onboard/src/control/GuidedVelocityBackend.*`
- `uav-onboard/src/control/RcOverrideBackend.*`
- `uav-onboard/src/autopilot/SerialMavlinkTransport.*`

단, 첫 구현에서는 지나치게 큰 프레임워크를 만들지 말고, 현재 `line_follow_node`에서 안전하게 사용할 수 있는 작은 class부터 분리한다.

## 3. Config 변경 계획

### 3.1 고도 기본값

`uav-onboard/config/mission.toml`:

```toml
[takeoff]
target_altitude_m = 2.0
altitude_reached_ratio = 0.9

[altitude_hold]
target_altitude_m = 2.0
kp = 0.4
max_vz_mps = 0.35
deadband_m = 0.08
```

`target_altitude_m`은 takeoff와 line-follow 동안 공통으로 2m를 뜻하게 한다. ArduPilot NED 기준 `vz_down_mps`는 아래 방향이 positive이므로, 실제 제어식에서 sign을 반드시 확인한다.

### 3.2 Line-follow controller 설정

`uav-onboard/config/mission.toml` 또는 별도 `[line_controller]`:

```toml
[line_follow]
forward_mps = 0.25
duration_s = 60.0
desired_angle_deg = 0.0
marker_hover_s = 3.0

[line_controller]
offset_kp = 0.35
angle_yaw_kp = 1.0
offset_yaw_kp = 0.25
max_lateral_mps = 0.35
max_yaw_rate_rad_s = 0.6
max_forward_mps = 0.35
min_confidence = 0.35
offset_deadband_norm = 0.03
angle_deadband_deg = 3.0
```

처음 SITL에서는 conservative하게 시작한다. 실제 line 추종이 확인되기 전에는 forward 속도를 낮게 둔다.

### 3.3 Marker hover/land 설정

```toml
[marker_hover]
hold_s = 3.0
center_tolerance_px = 80.0
center_kp = 0.25
max_lateral_mps = 0.25
```

ArUco marker가 인식되면 즉시 land하지 말고, marker 중심이 camera center에 오도록 3초 동안 hover/centering을 시도한 뒤 land한다.

## 4. 상태머신 설계 계획

현재 `LineFollowMission`은 작고 빠르게 만든 staging 상태머신이다. 다음 구현에서는 유지보수성을 위해 state와 action을 분리한다.

목표 상태:

```text
IDLE
WAIT_AUTOPILOT
SET_GUIDED
ARM
TAKEOFF
LINE_FOLLOW
MARKER_APPROACH
MARKER_HOVER
LAND
COMPLETE
ABORT
```

첫 구현에서 `WAIT_AUTOPILOT`, `SET_GUIDED`, `ARM`은 `line_follow_node` orchestration에 남겨도 된다. 다만 mission class의 public API는 새 상태 추가가 쉬운 형태로 둔다.

권장 구조:

```text
MissionInput
  now
  autopilot_state snapshot
  vision_result snapshot
  safety_decision
  operator_command

MissionOutput
  state
  desired_control_mode
  control_intent
  should_takeoff
  should_land
  landing_reason
```

상태 전환 규칙:

- `IDLE -> TAKEOFF`: onboard local start 또는 CLI start.
- `TAKEOFF -> LINE_FOLLOW`: altitude >= 2.0m * reached_ratio.
- `LINE_FOLLOW -> MARKER_APPROACH`: ArUco marker detected.
- `MARKER_APPROACH -> MARKER_HOVER`: marker center가 tolerance 안에 들어오거나, marker approach timeout이 지나면 hover로 넘어간다.
- `MARKER_HOVER -> LAND`: centered hover 누적 시간이 3초가 되면 land.
- `LINE_FOLLOW -> LAND`: line lost timeout, mission timeout, operator land.
- `LAND -> COMPLETE`: disarmed 확인.
- `any -> ABORT`: heartbeat loss, severe failsafe, RC takeover, EKF/GUIDED 불가.

나중에 full snake mission을 붙일 때 `LINE_FOLLOW` 뒤에 `GRID_EXPLORE`, `NODE_DECISION`, `TURN`, `MARKER_RECORD`, `REVISIT`을 추가할 수 있어야 한다. 이를 위해 `switch(state)` 안에 큰 제어 코드를 직접 넣지 말고, state transition과 control intent 생성을 분리한다.

## 5. ROS-like 설계 철학 적용 계획

ROS를 직접 쓰지 않지만 node graph처럼 typed message를 흘린다.

권장 runtime graph:

```text
FrameSource
  -> VisionProcessor
  -> MissionStateMachine
  -> GuidanceController
  -> ControlBackend
  -> AutopilotMavlinkAdapter

Observers:
  SafetyMonitor
  VisionDebugPublisher
  TelemetryPublisher
  LogPublisher
```

구현 원칙:

- 각 단계는 input snapshot을 받고 output struct를 반환한다.
- `VisionProcessor`는 MAVLink를 모른다.
- `MissionStateMachine`은 pixel image/JPEG를 모른다. marker center, line offset, line angle 같은 typed vision result만 본다.
- `GuidanceController`는 mission state와 vision error를 velocity/yaw/altitude setpoint로 바꾼다.
- `ControlBackend`는 setpoint를 A안 GUIDED velocity 또는 B안 RC override로 변환한다.
- `AutopilotMavlinkAdapter`는 MAVLink packet 송수신만 한다.
- GCS publisher는 observer이며 mission/control 루프를 막으면 안 된다.

## 6. Line-follow 제어 구현 계획

현재 vision은 다음 값을 제공한다.

```text
line_center_x
image_center_x
line_angle
confidence
```

제어 error:

```text
offset_error = line_center_x - image_center_x
angle_error  = line_angle - desired_angle
```

권장 내부 표현:

```text
offset_error_norm = offset_error / (image_width * 0.5)
angle_error_rad = deg_to_rad(line_angle_deg - desired_angle_deg)
```

주의할 점:

- 현재 `toLineControlInput()`은 `center_offset_px / half_width`를 `center_error_m` 이름의 필드에 넣는다. 구현 시 필드명을 `center_error_norm` 등으로 바로잡거나, 최소한 comment/config 이름을 normalized 기준으로 맞춘다.
- camera image X positive 방향과 `MAV_FRAME_BODY_NED`의 `vy_right_mps` positive 방향이 일치하는지 SITL에서 검증한다.
- `line_angle_deg` sign과 ArduPilot yaw-rate sign이 일치하는지 SITL에서 검증한다.

초기 controller 식:

```text
vx_forward_mps = forward_mps
vy_right_mps   = clamp(offset_kp * offset_error_norm, -max_lateral_mps, max_lateral_mps)
yaw_rate_rad_s = clamp(angle_yaw_kp * angle_error_rad + offset_yaw_kp * offset_error_norm,
                       -max_yaw_rate_rad_s,
                       max_yaw_rate_rad_s)
vz_down_mps    = altitude_hold_controller(current_altitude_m, target_altitude_m)
```

Sign 검증 후 필요하면 `vy_right_mps` 또는 `yaw_rate_rad_s`의 sign을 config로 뒤집을 수 있게 한다.

추천 config:

```toml
[line_controller]
invert_lateral = false
invert_yaw = false
```

Line lost 처리:

- 짧은 line lost: `vx = 0`, `vy = 0`, `yaw_rate = 0`, altitude hold만 유지.
- `line_lost_ms` 초과: mission output이 `LAND`.
- confidence가 낮으면 제어 gain을 줄이거나 hold 처리한다.

## 7. 2m altitude hold 구현 계획

요구사항은 “이륙 시 2m까지 올라가고 착륙 전까지 항상 2m 고도를 유지하려고 한다”이다.

구현 방향:

- takeoff command는 `MAV_CMD_NAV_TAKEOFF` altitude 2.0m.
- takeoff 완료 판단은 `AutopilotMavlinkAdapter::bestAltitudeM()` 기준 `>= 2.0 * altitude_reached_ratio`.
- line-follow 동안 `SET_POSITION_TARGET_LOCAL_NED` body velocity에 `vz_down_mps`를 포함한다.
- altitude source 우선순위는 현재 코드처럼 distance sensor, local position, relative altitude 순서를 유지하되, 실기체에서는 MTF-01 range 안정성을 우선 확인한다.

Altitude hold P 제어:

```text
altitude_error_m = target_altitude_m - current_altitude_m
vz_down_mps = -clamp(kp * altitude_error_m, -max_vz_mps, max_vz_mps)
if abs(altitude_error_m) < deadband_m:
    vz_down_mps = 0
```

NED sign 설명:

- current altitude가 target보다 낮으면 `altitude_error_m > 0`, 더 올라가야 하므로 `vz_down_mps`는 negative.
- current altitude가 target보다 높으면 `altitude_error_m < 0`, 내려가야 하므로 `vz_down_mps`는 positive.

검증:

- SITL에서 2m 도달 전 `LINE_FOLLOW`로 넘어가지 않는지 확인한다.
- line-follow 중 `vz_down_mps`가 altitude error에 맞게 작아지고, 2m 근처에서 0으로 수렴하는지 log로 확인한다.

## 8. ArUco marker hover 후 landing 계획

요구사항:

- Gazebo line 끝에 ArUco marker가 있다.
- ArUco marker가 인식되면 marker 중심 좌표 기준으로 3초간 hover.
- 3초 후 그 자리에서 바로 착륙.

구현 방향:

1. `VisionResult.markers`가 비어 있지 않으면 marker detected.
2. 첫 marker 또는 ID 1 marker를 target marker로 선택한다. 지금 Gazebo fixture는 ID 1 기준이므로 가능하면 ID 1을 우선한다.
3. marker center error를 계산한다.

```text
marker_offset_x = marker.center_px.x - image_center_x
marker_offset_y = marker.center_px.y - image_center_y
```

4. `MARKER_APPROACH` 또는 `MARKER_HOVER`에서는 line-follow forward command를 멈추고 marker center가 camera center로 오도록 body velocity를 낸다.
5. marker center가 tolerance 안에 들어온 시간만 누적해 3초 hover를 판단한다.
6. marker가 일시적으로 사라지면 짧게 hold하고, marker lost timeout이 길어지면 safety land 또는 line-follow 복귀 중 하나를 선택한다. 이번 MVP는 safety land가 더 안전하다.
7. 3초가 지나면 `LAND` 상태로 전환하고 `setLandMode()` 또는 `MAV_CMD_NAV_LAND`를 사용한다.

초기 marker centering 식:

```text
vx_forward_mps = clamp(marker_y_kp * marker_offset_y_norm, -max_marker_mps, max_marker_mps)
vy_right_mps   = clamp(marker_x_kp * marker_offset_x_norm, -max_marker_mps, max_marker_mps)
yaw_rate_rad_s = 0
vz_down_mps    = altitude hold until LAND starts
```

marker image Y sign은 반드시 SITL에서 확인한다. 하향 카메라에서 image down이 body forward인지 backward인지는 camera mounting/orientation에 따라 다를 수 있으므로 `invert_marker_x`, `invert_marker_y` config를 둘 수 있다.

## 9. MAVLink/ControlBackend 계획

### 9.1 A안 구현 목표

A안은 primary다.

```text
EKF local estimate
ARM
GUIDED
LOCAL_POSITION_NED or range altitude
MAV_FRAME_BODY_NED velocity setpoint
SET_POSITION_TARGET_LOCAL_NED
```

필요 조건:

- heartbeat 수신
- GUIDED mode set/confirm
- arm confirm
- takeoff command
- altitude estimate valid
- line-follow 중 주기적으로 body velocity setpoint 송신
- land mode set/confirm

`AutopilotMavlinkAdapter` 점검:

- 현재 `sendBodyVelocity()`는 `MAV_FRAME_BODY_NED`와 velocity+yaw_rate setpoint를 보낸다.
- type mask는 position/accel/yaw ignore, velocity/yaw_rate 사용 형태로 유지한다.
- `vz_down_mps`를 altitude hold에서 적극 사용하도록 controller output을 연결한다.
- `LOCAL_POSITION_NED`, `DISTANCE_SENSOR`, `GLOBAL_POSITION_INT` 수신 상태를 log/telemetry에 드러낸다.

### 9.2 B안 fallback 설계

B안은 이번에 실비행 primary로 만들지 않는다. 다만 구조상 추가 가능하게 설계한다.

```text
ALT_HOLD + RC_CHANNELS_OVERRIDE
```

Fallback 발동 후보:

- GUIDED mode 진입 실패
- EKF local estimate unavailable
- local position/range invalid
- MTF-01 optical flow/range failure
- repeated setpoint rejection 또는 failsafe

B안 구현 시 경계:

- mission state machine은 A/B backend를 몰라야 한다.
- 공통 `ControlSetpoint`를 만든 뒤 backend만 `GuidedVelocityBackend` 또는 `RcOverrideBackend`로 바꾼다.
- RC override는 실기체 위험도가 높으므로 props-off bench, low-throttle inhibit, RC takeover 우선순위 확인 후에만 활성화한다.

이번 계획에서 구현할 최소 항목:

- `ControlBackend` interface 설계.
- A안 backend는 기존 MAVLink body velocity path로 구현.
- B안은 config/documentation과 placeholder class 또는 명확한 TODO까지만 허용. 실제 RC override 전송은 별도 안전 검토 후.

## 10. Telemetry/Logging 계획

GCS overlay는 이미 잘 된다. 이번 작업에서 디버깅을 위해 control/mission 상태를 log에 남겨야 한다.

권장 telemetry/log fields:

```json
"mission": {
  "state": "LINE_FOLLOW",
  "target_altitude_m": 2.0,
  "landing_reason": ""
},
"control": {
  "offset_error_norm": 0.12,
  "angle_error_deg": -4.5,
  "vx_forward_mps": 0.25,
  "vy_right_mps": 0.04,
  "vz_down_mps": -0.02,
  "yaw_rate_rad_s": -0.08
},
"autopilot": {
  "mode": "GUIDED",
  "armed": true,
  "altitude_m": 1.98,
  "range_m": 1.98,
  "local_position_valid": true
}
```

Protocol 변경이 부담되면 우선 stdout log로 남긴다. GCS parser가 unknown field를 무시하는 구조이면 additive JSON fields를 추가할 수 있지만, 양쪽 `docs/PROTOCOL.md` 갱신을 같이 한다.

## 11. Test 계획

### 11.1 Unit/focused tests

추가 권장 테스트:

- `GuidedVelocityController` 또는 새 `LineFollowController`
  - positive offset일 때 expected `vy_right_mps` sign 확인.
  - positive angle error일 때 expected yaw sign 확인.
  - deadband 안에서는 lateral/yaw가 0 또는 작게 나오는지 확인.
  - output clamp 확인.
  - line lost 입력에서 stop/hold setpoint 확인.
- altitude hold
  - current 1.5m, target 2.0m이면 `vz_down_mps < 0`.
  - current 2.5m, target 2.0m이면 `vz_down_mps > 0`.
  - current 2.02m, target 2.0m이면 deadband로 0.
- mission state machine
  - takeoff altitude 도달 전에는 `LINE_FOLLOW` 진입 금지.
  - ArUco detected -> marker hover -> 3초 후 land.
  - line lost timeout -> land.
  - heartbeat lost -> abort.

### 11.2 SITL smoke

기존 순서:

```powershell
cd astroquad\uav-gcs
.\build\uav_gcs_vision_debug.exe --config config
```

```bash
bash ~/fly_test.sh
```

```bash
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"

cd ~/astroquad/uav-onboard
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

SITL 성공 기준:

- `[mission] TAKEOFF target=2.0m`
- altitude 2m 근처 도달 후 `LINE_FOLLOW`.
- line offset이 줄어드는 방향으로 `vy/yaw_rate`가 출력된다.
- 드론이 라인을 벗어나지 않고 라인 끝까지 접근한다.
- ArUco ID 1 인식.
- marker center 기준 hover 3초.
- 그 자리에서 `LAND`.
- disarmed 후 `COMPLETE`.

### 11.3 현실 세계 점검

구현 완료 후 실기체 적용 가능성 점검 항목:

- Pixhawk1에서 MTF-01 optical flow/range 수신 확인.
- ArduPilot EKF가 GPS 없이 GUIDED/local position velocity 제어를 받아들일 상태인지 확인.
- `LOCAL_POSITION_NED` 또는 equivalent local estimate가 안정적으로 갱신되는지 확인.
- range altitude가 2m 근처에서 튀지 않는지 확인.
- props off 상태에서 heartbeat, mode change, arm inhibit, takeoff command reject/accept path 확인.
- RC takeover가 언제나 onboard command보다 우선하는지 확인.
- 실제 line width, camera FOV, altitude 2m에서 line detection confidence가 충분한지 확인.
- SITL gain을 그대로 실기체에 쓰지 말고, forward/lateral/yaw gain을 50% 이하로 낮춰 첫 hover/line test를 시작한다.

## 12. 구현 순서 체크리스트

1. [ ] 현재 `RESEARCH.md`, `MVP_PLAN.md`, `PROJECT_SPEC.md` 재확인.
2. [ ] `mission.toml`/`runtime.sitl.toml`에서 2m altitude와 line/marker controller 설정 확장.
3. [ ] `ControlSetpoint`가 `vx`, `vy`, `vz`, `yaw_rate`를 명확히 표현하는지 확인하고 이름/주석 정리.
4. [ ] `GuidedVelocityController`를 line center/angle 기반 controller로 정리하거나 새 `LineFollowController`로 분리.
5. [ ] altitude hold P controller 추가.
6. [ ] offset/angle sign 검증을 위한 unit tests 추가.
7. [ ] mission state machine을 `TAKEOFF`, `LINE_FOLLOW`, `MARKER_APPROACH`, `MARKER_HOVER`, `LAND`, `COMPLETE`, `ABORT` 중심으로 정리.
8. [ ] ArUco marker centering + 3초 hover + land transition 구현.
9. [ ] `line_follow_node` orchestration이 mission output과 controller output을 연결하도록 정리.
10. [ ] control setpoint와 mission/autopilot state를 stdout 또는 telemetry에 노출.
11. [ ] A안 GUIDED body velocity path 회귀 확인.
12. [ ] B안 RC override fallback은 interface/설계 수준으로만 남기고 실제 활성화는 별도 safety gate로 미룬다.
13. [ ] `ctest --test-dir build --output-on-failure` 실행.
14. [ ] Windows GCS + WSL Gazebo SITL smoke 실행.
15. [ ] 구현 후 현실 적용 가능성 점검 결과를 `RESEARCH.md` 또는 `TROUBLESHOOTING.md`에 기록.

## 13. 작업 완료 기준

코드 구현 작업이 완료되었다고 판단하려면 다음을 만족해야 한다.

- takeoff target이 2m이고, `LINE_FOLLOW` 중 altitude hold setpoint가 계속 2m를 유지하려 한다.
- line controller가 `offset_error = line_center_x - image_center_x`, `angle_error = line_angle - desired_angle`를 기준으로 명확히 동작한다.
- 제어 output sign이 unit test와 SITL log로 검증된다.
- ArUco marker 인식 시 marker center 기준 3초 hover 후 land한다.
- 상태머신은 새 상태를 추가하기 쉬운 입력/출력 구조를 가진다.
- `vision_debug_node`와 GCS overlay 기존 동작이 유지된다.
- 실기체 A안 적용 가능성을 점검했고, B안 fallback은 구조상 추가 가능하게 남아 있다.
