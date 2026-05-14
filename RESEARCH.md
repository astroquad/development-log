# Current Step Research

최종 업데이트: 2026-05-14

이 문서는 방금 완료한 스텝의 구현/검증 결과와 다음 추천 스텝만 기록한다. 장기 기준은 `SYSTEM_SPEC.md`, MVP 범위는 `MVP_PLAN.md`, 실행 계획은 `PLAN.md`를 따른다.

## Current Step Summary

이번 스텝의 목표는 `uav-onboard` runtime을 Gazebo SITL과 Raspberry Pi 4 + Pixhawk1 환경 사이에서 전환할 수 있는 구조로 정리하는 것이었다.

핵심 흐름:

```text
FrameSource
  -> VisionProcessor
  -> Mission
  -> Control
  -> AutopilotAdapter
```

중요한 경계는 유지했다.

- Vision code는 image를 perception result로만 바꾼다.
- Mission은 state/event 판단만 한다.
- Control은 velocity/yaw-rate setpoint만 만든다.
- Autopilot adapter만 MAVLink packet을 만들고 보낸다.
- GCS protocol schema는 이번 스텝에서 변경하지 않았다.

## Implemented In This Step

### MAVLink Receive Stability

- `UdpMavlinkTransport`에 pending message queue를 추가했다.
- 한 UDP datagram 안에 여러 MAVLink frame이 들어와도 첫 frame 이후 frame을 잃지 않도록 했다.
- `AutopilotMavlinkAdapter::poll()`이 첫 message를 timeout 안에서 기다린 뒤 non-blocking drain을 수행하도록 바꿨다.
- heartbeat가 telemetry stream 뒤에 밀려 `pixhawk heartbeat lost`로 오판되는 문제를 줄였다.

추가 test:

```text
test_autopilot_poll_drain
test_udp_mavlink_transport
```

### VisionProcessor Separation

- 기존 `VisionDebugPipeline` 안에 있던 detector/stabilizer processing block을 `VisionProcessor`로 분리했다.
- `VisionProcessor`는 `cv::Mat`과 frame metadata를 받아 `VisionResult`를 만든다.
- 기존 ArUco, line, intersection detector/stabilizer logic은 재작성하지 않고 재사용했다.
- `VisionDebugPipeline`은 기존 GCS telemetry/video 동작을 유지하면서 새 processor를 사용한다.

추가 test:

```text
test_vision_processor
```

### FrameSource Runtime Profiles

추가한 source abstraction:

```text
FrameSource
  FakeFrameSource
  GazeboCameraSource
  RpicamFrameSource
```

`line_follow_node`는 다음 option/profile을 지원한다.

```bash
./build/line_follow_node --config config --target sitl --vision fake
./build/line_follow_node --config config --target sitl --vision gazebo
./build/line_follow_node --config config --target pixhawk1 --vision rpicam
./build/line_follow_node --config config --target sitl --vision gazebo --vision-smoke-count 3
```

추가 config:

```text
config/runtime.sitl.toml
config/runtime.pixhawk1.toml
config/vision.toml [source]
```

추가 test:

```text
test_fake_frame_source
```

### Gazebo Camera Path

- Gazebo native transport 기반 `GazeboCameraSource`를 추가했다.
- ROS bridge 없이 `gz::msgs::Image`를 OpenCV BGR `cv::Mat`으로 변환한다.
- Gazebo camera topic은 `config/vision.toml`의 `[source].gazebo_topic`에서 설정한다.
- Iris에 downward camera를 붙인 local wrapper model과 test world를 추가했다.

추가 asset:

```text
uav-onboard/sim/gazebo/worlds/astroquad_iris_vision.sdf
uav-onboard/sim/gazebo/models/iris_with_downward_camera/
```

Gazebo world 실행:

```bash
cd ~/astroquad/uav-onboard

GZ_SIM_RESOURCE_PATH="$HOME/ardupilot_gazebo/models:$PWD/sim/gazebo/models:$PWD/sim/gazebo" \
  gz sim -v4 -r sim/gazebo/worlds/astroquad_iris_vision.sdf
```

### Vision-Driven Line Follow And Marker Event Hook

- `VisionResult.line`을 `GuidedVelocityController`의 `LineControlInput`으로 변환한다.
- `line_follow_node --vision gazebo` 실행 시 fake input 대신 Gazebo camera image 기반 line input을 사용할 수 있다.
- ArUco marker detection 결과를 mission input으로 전달한다.
- `LineFollowMission`에 `MarkerHover` state를 추가했다.
- marker가 center tolerance 안에 들어오면 zero velocity hover를 유지하고, hold 시간이 끝나면 LAND로 전환한다.

추가 test:

```text
test_line_follow_mission
```

## Validation Results

Build/test:

```text
cmake --build build
ctest --test-dir build --output-on-failure
-> 12/12 tests passed
```

SITL fake smoke:

```text
[mavlink] heartbeat ok system=1 component=1 mode=STABILIZE
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF target=1.2m
[mission] LINE_FOLLOW
[mission] LAND reason=duration complete
[mission] COMPLETE
```

Gazebo camera vision smoke:

```text
./build/line_follow_node --config config --target sitl --vision gazebo --vision-smoke-count 3

[vision] frame=1 source=gazebo size=640x480 line=yes offset_px=-0.666656 angle_deg=90 markers=0
```

Gazebo vision-driven flight smoke:

```text
./build/line_follow_node --config config --target sitl --vision gazebo

[vision] source=gazebo opened
[mavlink] heartbeat ok system=1 component=1 mode=STABILIZE
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF target=1.2m
[mission] LINE_FOLLOW
[mission] LAND reason=duration complete
[mission] COMPLETE
```

Pixhawk1 profile check:

```text
./build/line_follow_node --config config --target pixhawk1
-> serial endpoint /dev/serial0:115200 selected, serial transport intentionally not implemented in this step
```

Protocol/docs:

- `uav-onboard/docs/PROTOCOL.md`와 `uav-gcs/docs/PROTOCOL.md`를 동기화했다.
- 이번 스텝은 protocol v1.7 schema를 변경하지 않는다.
- fake/gazebo/rpicam source 선택은 telemetry JSON 또는 MJPEG chunk wire format을 바꾸지 않는다.

## Current Limitations

- Gazebo line 기반 vision flight는 동작 확인됐다.
- Gazebo ArUco marker hover end-to-end는 아직 신뢰할 수 없다.
- 현재 Gazebo vision smoke에서 marker 결과는 `markers=0`이었다.
- `MarkerHover` mission transition은 unit test로 확인했지만, 실제 Gazebo camera 시야에서 marker를 검출해 hover/land까지 이어지는 sequence는 미검증이다.
- `SerialMavlinkTransport`는 아직 구현하지 않았다. `--target pixhawk1 --vision rpicam`은 profile/source 준비와 serial endpoint 안내까지만 제공한다.
- Raspberry Pi 4 + IMX519 실기 runtime은 hardware에서 다시 확인해야 한다.
- `line_follow_node`의 mission/flight 상태를 GCS telemetry field로 확장하는 작업은 이번 step에서 하지 않았다. Protocol v1.8 후보로 분리하는 편이 좋다.

## Useful Assumptions For Next Step

- 개발환경의 C++ OpenCV는 `/usr/local`의 OpenCV/opencv_contrib `4.10.0` 기준이다.
- 기존 detector code는 OpenCV modern ArUco API를 사용하므로 OpenCV 4.5 계열 API로 되돌리지 않는다.
- Gazebo camera path는 ROS 없이 Gazebo transport를 우선 사용한다.
- 다음 작업에서도 Vision module이 MAVLink command를 직접 보내면 안 된다.
- SITL 편의를 위해 safety를 제거하지 않는다. 필요한 경우 runtime profile별 timeout을 조정한다.

## Recommended Next Step

다음 한 스텝으로는 **Gazebo ArUco marker hover end-to-end 검증 스텝**을 추천한다.

목표 sequence:

```text
Gazebo world line/marker
  -> downward camera frame
  -> line detected
  -> line follow
  -> marker detected
  -> marker centered
  -> MarkerHover 3s
  -> LAND
  -> COMPLETE
```

추천 구현 범위:

1. Gazebo world의 line path, marker 위치, marker 크기, camera pose/FOV를 조정해 marker가 실제 비행 camera frame에 들어오게 만든다.
2. `--vision-smoke-count`에서 `markers>0`을 먼저 확인한다.
3. 필요하면 marker 위치별 정지 frame smoke를 위한 world fixture를 추가한다.
4. `line_follow_node` console log에 mission state transition과 marker center error를 더 명확히 남긴다.
5. detector algorithm은 먼저 건드리지 말고, world geometry와 camera/FOV 문제인지부터 분리한다.
6. Gazebo vision-driven flight smoke의 성공 기준을 `MarkerHover` 진입과 hover 완료 land까지로 올린다.

이번 다음 스텝에서 하지 않는 편이 좋은 것:

- F450 정밀 Gazebo model 제작.
- 실제 lens distortion/calibration 정밀화.
- `SerialMavlinkTransport` 구현.
- GCS UI 대규모 변경.
- Protocol v1.8 flight telemetry field 추가.
- Vision module에서 직접 MAVLink command를 보내는 구조.

추천 validation:

```bash
cd ~/astroquad/uav-onboard
cmake --build build
ctest --test-dir build --output-on-failure
```

Gazebo camera smoke:

```bash
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --vision-smoke-count 10
```

성공 기준:

```text
line=yes
markers>=1
marker_id=0
```

Gazebo flight smoke 성공 기준:

```text
[mission] TAKEOFF ...
[mission] LINE_FOLLOW
[vision] marker id=0 centered=yes ...
[mission] MARKER_HOVER
[mission] LAND reason=marker hover complete
[mission] COMPLETE
```

그 다음 후보 스텝:

1. `SerialMavlinkTransport` 구현과 Pixhawk1 props-off bench test.
2. Raspberry Pi 4에서 `--target pixhawk1 --vision rpicam` camera/vision 단독 smoke.
3. GCS mission telemetry schema v1.8 설계: mode, armed, heartbeat age, altitude/range, safety state, active runtime profile.
