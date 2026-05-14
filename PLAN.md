# Current Step Plan

최종 업데이트: 2026-05-14

이 문서는 `RESEARCH.md`를 바탕으로 작성한 **현재 스텝 구현 계획서**다. 사용자가 이 계획을 검토하고 승인하기 전까지 코드는 작성하지 않는다.

## Current Task

Astroquad `uav-onboard`를 현재 성공한 Gazebo Iris + ArduCopter SITL MAVLink 경로에 연결한다.

이번 스텝의 목표는 실제 camera/vision loop까지 한 번에 붙이는 것이 아니라, Astroquad 내부에 비행 제어 경계를 먼저 만들고 SITL에서 다음 sequence를 통과시키는 것이다.

```text
SITL heartbeat 수신
  -> GUIDED mode
  -> arm
  -> takeoff
  -> 짧은 BODY_NED velocity stream
  -> LAND mode
```

## Scope

이번에 구현할 것:

- `uav-onboard` 안에 MAVLink adapter의 최소 뼈대를 만든다.
- UDP SITL transport를 먼저 지원한다.
- GUIDED mode, arm/disarm, takeoff, land, body-frame velocity setpoint 송신을 지원한다.
- heartbeat, mode, armed, altitude/range/local-position snapshot을 저장한다.
- 임시 `line_follow_node`를 추가해 fake line input 기반으로 SITL smoke mission을 실행한다.
- mission/control/safety/autopilot 모듈 경계를 코드 구조로 분리한다.
- `config/autopilot.toml`, `config/mission.toml`, `config/safety.toml` 값을 실제 runtime에서 읽을 수 있게 확장한다.
- 기존 `vision_debug_node`, telemetry/video/GCS debug 흐름은 건드리지 않는다.

이번에 구현하지 않을 것:

- F450 Gazebo model 제작.
- Gazebo camera bridge 또는 IMX519 대체 camera source.
- 실제 `VisionDebugPipeline`에서 mission 제어 명령을 직접 내리는 구조.
- full snake mission, marker revisit, grid planner.
- GCS command channel.
- RC override fallback backend.
- 실기체 serial transport의 완전 구현. 단, interface와 config는 serial 전환을 고려해서 설계한다.

## Architecture Boundary

ROS는 쓰지 않지만 ROS node graph처럼 typed data만 넘긴다.

```text
FakeVisionSource
  -> MissionStateMachine
  -> GuidanceController
  -> ControlBackendGuidedVelocity
  -> AutopilotMavlinkAdapter

Support:
  SafetyMonitor observes Mission/Autopilot/Vision snapshots
```

규칙:

- Vision 계층은 `line.detected`, `center_offset_px`, `angle_deg`, `confidence` 같은 관측값만 제공한다.
- Mission은 MAVLink packet을 만들지 않는다.
- Guidance는 비행 모드 전환을 하지 않고 `ControlSetpoint`만 만든다.
- Control backend는 setpoint를 ArduPilot GUIDED velocity 명령으로 변환한다.
- MAVLink adapter는 판단하지 않고 송수신과 autopilot state cache만 담당한다.
- Safety는 inhibit/land/abort intent를 만들지만, 실제 land 명령 실행은 mission/control path를 통해 일관되게 보낸다.

## Planned File Changes

예상 파일 추가:

```text
uav-onboard/
  src/autopilot/
    AutopilotState.hpp
    MavlinkTransport.hpp
    UdpMavlinkTransport.hpp
    UdpMavlinkTransport.cpp
    AutopilotMavlinkAdapter.hpp
    AutopilotMavlinkAdapter.cpp
  src/control/
    ControlSetpoint.hpp
    GuidedVelocityController.hpp
    GuidedVelocityController.cpp
  src/mission/
    LineFollowMission.hpp
    LineFollowMission.cpp
  src/safety/
    SafetyMonitor.hpp
    SafetyMonitor.cpp
  tools/
    line_follow_node.cpp
```

예상 파일 수정:

```text
uav-onboard/
  CMakeLists.txt
  config/autopilot.toml
  config/mission.toml
  config/safety.toml
  tests/CMakeLists.txt
```

MAVLink header 처리 후보:

- 1순위: CMake 옵션 `MAVLINK_INCLUDE_DIR`로 기존 로컬 MAVLink C header 경로를 받는다.
- 2순위: 빌드 환경에 `<mavlink/v2.0/ardupilotmega/mavlink.h>`가 있으면 사용한다.
- 3순위: 없으면 별도 승인 후 generated MAVLink header를 `third_party`에 vendoring한다.

이번 계획 승인 후 구현을 시작할 때, 먼저 현재 머신의 MAVLink header 위치를 확인하고 가장 작은 변경으로 연결한다.

## Config Plan

`config/autopilot.toml` 목표 형태:

```toml
[transport]
kind = "udp"
listen_host = "0.0.0.0"
listen_port = 14550

[serial]
device = "/dev/serial0"
baudrate = 115200

[mavlink]
system_id = 191
component_id = 191
target_system = 1
target_component = 1
setpoint_rate_hz = 20
```

`config/mission.toml`에 MVP line-follow 값을 추가한다.

```toml
[takeoff]
target_altitude_m = 1.2

[line_follow]
forward_mps = 0.3
center_error_m = 0.0
line_angle_deg = 0.0
duration_s = 3.0
```

`config/safety.toml`의 기존 timeout 값을 사용하되, SITL smoke test에서는 mission timeout을 짧게 override할 수 있게 CLI option을 둔다.

## Runtime Target Selection

프로그램 실행 때 옵션만 바꿔 SITL과 실제 Pixhawk1을 전환할 수 있게 한다. 코드에서 endpoint, baudrate, transport 종류를 매번 수정하지 않는다.

목표 CLI:

```bash
# Gazebo/ArduCopter SITL
./build/line_follow_node --config config --target sitl

# 실제 Pixhawk1, Raspberry Pi serial 연결
./build/line_follow_node --config config --target pixhawk1

# 필요할 때만 endpoint 직접 override
./build/line_follow_node --config config --autopilot udp://0.0.0.0:14550
./build/line_follow_node --config config --autopilot serial:///dev/serial0:115200
```

우선순위:

```text
CLI --autopilot explicit URI
  > CLI --target profile
  > config/autopilot.toml default profile
```

`--target sitl`은 UDP transport와 SITL용 timeout/log label을 선택한다. `--target pixhawk1`은 serial/USB transport와 실기체용 conservative timeout/log label을 선택한다.

`mission`, `control`, `safety` 코드는 `--target` 값을 직접 보지 않는다. 실행 entrypoint가 target profile을 `AutopilotEndpointConfig` 같은 typed config로 변환하고, 이후 계층에는 transport 종류와 endpoint가 추상화된 adapter만 전달한다.

## Implementation Steps

- [ ] 현재 MAVLink header 사용 가능 경로를 확인한다.
- [ ] `AutopilotState`와 `MavlinkTransport` interface를 추가한다.
- [ ] UDP transport를 구현한다.
- [ ] `AutopilotMavlinkAdapter`에 heartbeat discovery, message receive loop, command send API를 추가한다.
- [ ] `ControlSetpoint`와 `GuidedVelocityController`를 추가한다.
- [ ] `LineFollowMission` 축소 상태머신을 추가한다.
- [ ] `SafetyMonitor` 최소 timeout check를 추가한다.
- [ ] `line_follow_node`를 추가해 config 기반 SITL smoke mission을 실행한다.
- [ ] `line_follow_node` CLI에 `--target sitl|pixhawk1`과 `--autopilot <uri>` override를 추가한다.
- [ ] CMake target을 연결한다.
- [ ] fake line offset/angle에 대한 controller 단위 테스트를 추가한다.
- [ ] SITL 없이 가능한 unit test를 먼저 통과시킨다.
- [ ] 사용자가 직접 또는 Codex가 승인 후 `fly_test.sh`로 SITL을 띄운 뒤 `line_follow_node` smoke test를 실행한다.

## Reduced State Machine

이번 스텝에서 사용할 상태:

```text
IDLE
TAKEOFF
LINE_FOLLOW
LAND
COMPLETE
ABORT
```

전환:

- `IDLE -> TAKEOFF`: `line_follow_node` 시작 후 heartbeat 확인, GUIDED 설정, arm 성공.
- `TAKEOFF -> LINE_FOLLOW`: altitude/range/local-position 중 사용 가능한 값이 목표 고도의 85% 이상.
- `LINE_FOLLOW -> LAND`: duration 완료, line lost timeout, mission timeout, operator abort placeholder.
- `LAND -> COMPLETE`: disarmed heartbeat 확인 또는 land timeout 후 안전 종료.
- `any -> ABORT`: heartbeat loss, command failure, severe timeout.

## Control Policy

초기 controller는 `takeoff_move_land.cpp`의 보수적인 P 제어를 Astroquad 모듈로 옮긴다.

```text
vx_forward_mps = fixed forward speed
vy_right_mps = clamp(lateral_kp * center_error_m)
vz_down_mps = 0 during line follow
yaw_rate_rad_s = clamp(yaw_kp * line_angle_rad)
```

초기값:

```text
forward_mps: 0.2-0.4
lateral_kp: 0.8
yaw_kp: 1.2
max_lateral_mps: 0.5
max_yaw_rate_rad_s: 0.8
setpoint_rate_hz: 20
```

MAVLink frame:

- `SET_POSITION_TARGET_LOCAL_NED`
- `MAV_FRAME_BODY_NED` 또는 body velocity에 맞는 equivalent frame
- position/accel/yaw는 ignore
- velocity와 yaw_rate만 사용

## Validation Plan

코드 구현 후, 승인된 범위 안에서 다음을 검증한다.

Unit/build:

- [ ] `cmake -S . -B build-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON`
- [ ] `cmake --build build-tests`
- [ ] `ctest --test-dir build-tests --output-on-failure`

SITL smoke:

- [ ] `bash ~/fly_test.sh`로 기존 Iris Gazebo SITL 실행.
- [ ] `./build/line_follow_node --config config --autopilot udp://0.0.0.0:14550 --vision fake`
- [ ] heartbeat 수신 확인.
- [ ] GUIDED mode confirm.
- [ ] arm confirm.
- [ ] takeoff altitude reached confirm.
- [ ] BODY_NED velocity stream 20Hz 확인.
- [ ] LAND mode 전환 확인.
- [ ] disarm 또는 landing complete 확인.

Safety smoke:

- [ ] heartbeat timeout에서 command stream 중단 및 abort/land path 확인.
- [ ] line lost timeout에서 land path 확인.
- [ ] mission timeout에서 land path 확인.

Regression:

- [ ] 기존 `vision_debug_node` build가 깨지지 않는다.
- [ ] 기존 telemetry JSON test가 깨지지 않는다.
- [ ] `uav_gcs_vision_debug`와의 protocol 호환성은 이번 스텝에서 schema 변경을 하지 않아 유지한다.

## Acceptance Criteria

- `uav-onboard`에 MAVLink/control/mission/safety 경계가 생긴다.
- `line_follow_node`가 Gazebo Iris SITL에서 `takeoff -> short velocity -> land`를 수행한다.
- UDP SITL endpoint는 동작하고, serial/USB 전환을 위한 interface가 막히지 않는다.
- vision module 내부에 MAVLink/control 코드가 들어가지 않는다.
- 기존 vision debug/GCS 관제 경로가 깨지지 않는다.
- F450 모델 제작 전에도 Astroquad 본 프로젝트 코드로 SITL 통합 검증을 시작할 수 있다.

## Open Questions Before Implementation

- MAVLink headers는 기존 시스템 include를 쓸지, `uav-onboard`에 vendoring할지.
- `line_follow_node`를 장기적으로 유지할지, 안정화 후 `uav_onboard --mode line-follow`로 흡수할지.
- `MAV_FRAME_BODY_NED`와 `MAV_FRAME_BODY_OFFSET_NED` 중 실제 ArduPilot Copter에서 이번 velocity command에 가장 일관적인 frame을 smoke test로 확정할지.
- altitude reached 판단에서 SITL은 `DISTANCE_SENSOR` 우선으로 볼지, `LOCAL_POSITION_NED.z` fallback을 우선으로 볼지.
