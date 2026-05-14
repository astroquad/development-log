# Current Step Research

최종 업데이트: 2026-05-14

이 문서는 현재 스텝에서 조사/구현/검증한 사실과 다음 추천 스텝을 기록한다. 장기 시스템 기준은 `SYSTEM_SPEC.md`, MVP 범위는 `MVP_PLAN.md`, 실행 계획은 `PLAN.md`를 따른다.

## Current Task

Astroquad `uav-onboard`를 기존 Gazebo Iris + ArduCopter SITL MAVLink 제어 경로에 연결하기 위한 첫 구현을 진행했다.

목표 sequence:

```text
SITL heartbeat 수신
  -> GUIDED mode
  -> arm
  -> takeoff
  -> 짧은 BODY_NED velocity stream
  -> LAND mode
```

이번 스텝은 실제 카메라/vision loop 통합이 아니라, `autopilot`, `control`, `mission`, `safety` 경계를 먼저 만드는 작업이다.

## Implemented

`uav-onboard`에 다음 모듈을 추가했다.

```text
src/autopilot/
  AutopilotState.hpp
  MavlinkTransport.hpp
  UdpMavlinkTransport.*
  AutopilotMavlinkAdapter.*

src/control/
  ControlSetpoint.hpp
  GuidedVelocityController.*

src/mission/
  LineFollowMission.*

src/safety/
  SafetyMonitor.*

tools/
  line_follow_node.cpp
```

구현 범위:

- UDP MAVLink transport.
- ArduPilot heartbeat discovery.
- GUIDED/LAND mode command.
- arm/disarm command.
- GUIDED takeoff command.
- `SET_POSITION_TARGET_LOCAL_NED` + `MAV_FRAME_BODY_NED` velocity/yaw-rate stream.
- `DISTANCE_SENSOR`, `LOCAL_POSITION_NED`, `GLOBAL_POSITION_INT`, heartbeat 기반 altitude/mode/armed snapshot.
- Fake line input 기반 `line_follow_node`.
- `--target sitl`, `--target pixhawk1`, `--autopilot udp://...`, `--autopilot serial://...` CLI profile/override 경로.
- `GuidedVelocityController` unit test.

## Architecture Notes

현재 코드 경계:

```text
FakeVisionSource
  -> LineFollowMission
  -> GuidedVelocityController
  -> AutopilotMavlinkAdapter

SafetyMonitor observes Autopilot/Vision timing snapshots
```

중요한 점:

- Vision module 내부에는 MAVLink/control 코드가 들어가지 않았다.
- Mission은 MAVLink packet을 만들지 않는다.
- Control은 velocity/yaw-rate setpoint 계산만 한다.
- Autopilot adapter는 송수신과 state cache만 담당한다.
- 실제 `VisionDebugPipeline`은 이번 스텝에서 수정하지 않았다.

## Config/Runtime Selection

`config/autopilot.toml`에 transport와 MAVLink ID 값을 추가했다.

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

실행 target:

```bash
./build/line_follow_node --config config --target sitl
./build/line_follow_node --config config --target pixhawk1
./build/line_follow_node --config config --autopilot udp://0.0.0.0:14550
./build/line_follow_node --config config --autopilot serial:///dev/serial0:115200
```

현재 `--target pixhawk1`과 `serial://...`은 profile/URI parsing 경로만 있다. serial transport 자체는 이번 스텝 범위 밖이라 명확한 안내와 함께 종료한다.

## Validation

성공:

```bash
cmake -S . -B build-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DBUILD_TOOLS=OFF
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

결과:

```text
6/6 tests passed
```

추가 확인:

```bash
./build-tests/line_follow_node --help
./build-tests/line_follow_node --config config --target pixhawk1
git -C astroquad/uav-onboard diff --check
git -C astroquad/development-log diff --check
```

결과:

- CLI help 정상 출력.
- `--target pixhawk1` 선택 시 serial endpoint를 표시하고, serial transport 미구현 안내와 함께 종료.
- diff whitespace check 통과.

## Known Issues

- `BUILD_TOOLS=ON` 전체 빌드는 기존 `ArucoDetector`와 현재 설치된 OpenCV aruco API 버전 차이 때문에 실패한다.
- 위 문제는 이번 MAVLink/control 구현 때문이 아니라 기존 vision tool build compatibility 문제다.
- 이번 스텝에서는 승인 범위 밖이라 `ArucoDetector`를 수정하지 않았다.
- SITL 실제 비행 smoke test는 아직 수행하지 않았다. Gazebo/MAVProxy session을 띄운 뒤 별도 검증이 필요하다.

## Next Recommended Step

추천 순서:

1. `BUILD_TOOLS=ON`에서 기존 vision tools가 빌드되도록 OpenCV aruco API 호환 문제를 해결한다.
2. Gazebo/MAVProxy를 띄우고 `line_follow_node --target sitl` 실제 SITL smoke test를 수행한다.
3. SITL smoke 결과를 바탕으로 `AutopilotMavlinkAdapter` timeout, altitude source 우선순위, mode confirmation 로그를 조정한다.
4. `line_follow_node` 결과를 GCS에서 볼 수 있도록 telemetry에 `mode`, `armed`, `altitude/range`, `mission_state`, `safety_event` 필드를 추가할 계획을 세운다.
5. 그 다음에 serial transport를 구현해 `--target pixhawk1`을 props-off Pixhawk bench test까지 연결한다.

바로 다음 한 스텝으로는 **OpenCV aruco API 호환 문제 해결 후 전체 onboard build를 정상화**하는 것을 추천한다. 그래야 `vision_debug_node`와 `line_follow_node`가 같은 build에서 함께 검증되고, 이후 camera/vision 통합으로 넘어갈 때 빌드 상태가 흔들리지 않는다.
