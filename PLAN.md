# Current Step Plan

최종 업데이트: 2026-05-14

이 문서는 `RESEARCH.md`를 바탕으로 작성한 **다음 구현 계획서**다. 사용자가 이 계획을 검토하고 승인하기 전까지 코드는 작성하지 않는다.

## Current Task

현재 스텝의 목표는 Astroquad onboard runtime을 Gazebo SITL과 실제 Raspberry Pi 4 + Pixhawk1 환경 사이에서 쉽게 전환할 수 있는 구조로 확장하는 것이다.

우선순위:

```text
1. MAVLink heartbeat false-positive abort 해결
2. Gazebo camera 기반 vision 입력 경로 설계/구현
3. 기존 Raspberry Pi 4 + IMX519 vision 코드 재사용 구조 정리
4. 실행 옵션/설정만 바꿔 SITL과 실제 환경을 오갈 수 있는 runtime profile 정리
```

이번 단계의 핵심은 **ROS는 사용하지 않지만 ROS식 역할 분리**를 지키는 것이다.

```text
FrameSource
  -> VisionProcessor
  -> Mission
  -> Control
  -> AutopilotAdapter

Vision/Flight telemetry
  -> GCS publisher
```

Vision module은 image를 perception result로 바꾸는 일만 담당한다. MAVLink 제어 명령은 계속 mission/control/autopilot 경계를 통해서만 나간다.

## Current Status

이미 완료된 것:

- OpenCV/opencv_contrib `4.10.0`이 `/usr/local` 기본 C++ OpenCV로 설치됐다.
- `uav-onboard`는 `BUILD_TOOLS=ON`으로 전체 빌드된다.
- `line_follow_node`, `vision_debug_node`, `aruco_detector_tester`가 같은 OpenCV 4.10에 링크된다.
- 기존 unit test는 `7/7` 통과했다.
- Gazebo + ArduCopter SITL에서 Astroquad `line_follow_node`만으로 다음 smoke sequence를 확인했다.

```text
heartbeat 수신
  -> GUIDED
  -> arm
  -> takeoff
  -> fake line input 기반 BODY_NED velocity command
  -> LAND
  -> COMPLETE
```

현재 남은 문제:

- 기본 `pixhawk_heartbeat_lost_ms = 2000`에서 mission 중 `pixhawk heartbeat lost` false-positive abort가 발생할 수 있다.
- 원인은 Gazebo heartbeat를 사용자가 직접 설정하지 않아서가 아니라, 현재 MAVLink receive path가 SITL telemetry stream을 충분히 drain하지 못하는 구조에 가깝다.
- 현재 `line_follow_node`는 `--vision fake`만 구현되어 있고, Gazebo camera image 기반 line/ArUco 인식은 아직 연결되지 않았다.
- 기존 `VisionDebugPipeline`은 Raspberry Pi camera debug runtime으로는 잘 동작하지만, mission/control 입력으로 바로 쓰기에는 camera loop, processing, telemetry/video publish가 한 클래스에 묶여 있다.
- `--target pixhawk1`과 `serial://...` parsing은 있지만 serial transport 자체는 아직 구현되어 있지 않다.

## Scope

이번 계획에서 구현할 것:

- `AutopilotMavlinkAdapter`와 `UdpMavlinkTransport`의 MAVLink receive path를 개선한다.
- heartbeat가 다른 telemetry message에 밀려 stale로 보이는 문제를 줄인다.
- SITL smoke test가 임시 safety timeout override 없이 완료되는지 확인한다.
- 기존 vision 코드를 재사용 가능한 `VisionProcessor` 형태로 분리하는 계획을 코드에 반영한다.
- `line_follow_node` 또는 후속 onboard runtime이 fake/gazebo/rpicam vision source를 선택할 수 있게 만든다.
- Gazebo camera frame을 OpenCV `cv::Mat`으로 받아 기존 line/ArUco detector에 넣을 수 있는 source abstraction을 추가한다.
- GCS telemetry/video 송신은 기존 `UdpTelemetrySender`, `UdpMjpegStreamer`를 재사용하는 방향으로 연결한다.
- SITL과 실제 Raspberry Pi 4 환경을 config/CLI profile로 전환할 수 있게 정리한다.

이번 계획에서 구현하지 않을 것:

- F450 정밀 Gazebo model 제작.
- 실제 IMX519 광학계와 완전히 동일한 Gazebo lens/distortion calibration.
- GPS-denied EKF 튜닝 전체.
- Optical flow sensor Gazebo plugin 세팅.
- 실제 Pixhawk1 props-on 비행.
- GCS UI 대규모 개편.
- Vision module이 MAVLink command를 직접 보내는 구조.

## Architecture Target

목표 runtime 구조:

```text
FrameSource
  FakeFrameSource        # unit/smoke test
  GazeboCameraSource     # SITL camera topic/frame
  RpicamFrameSource      # Raspberry Pi 4 + IMX519

VisionProcessor
  ArucoDetector
  LineMaskBuilder
  LineDetector
  LineStabilizer
  IntersectionDetector
  MarkerStabilizer

Mission
  LineFollowMission
  future: marker hover / grid mission

Control
  GuidedVelocityController

Autopilot
  UdpMavlinkTransport    # SITL
  SerialMavlinkTransport # Pixhawk1, later step
  AutopilotMavlinkAdapter

Publishers
  UdpTelemetrySender
  UdpMjpegStreamer
```

중요한 경계:

- `FrameSource`는 frame을 가져오기만 한다.
- `VisionProcessor`는 `VisionResult`만 만든다.
- `Mission`은 상태 전이와 mission event만 판단한다.
- `Control`은 `VisionResult`/mission state를 velocity setpoint로 바꾼다.
- `AutopilotAdapter`만 MAVLink packet을 만들고 보낸다.
- `TelemetryPublisher`는 관측/상태를 GCS로 보낼 뿐 제어하지 않는다.

## Step 1: Heartbeat Receive Path Fix

### Problem

현재 구조:

```text
line_follow loop
  -> autopilot.poll(1)
  -> SafetyMonitor checks last_heartbeat_time
```

`AutopilotMavlinkAdapter::poll()`은 호출당 MAVLink message 하나만 처리한다. `UdpMavlinkTransport::recvMessage()`도 UDP datagram 안에서 첫 MAVLink frame을 찾으면 바로 반환한다.

SITL/MAVProxy stream에는 heartbeat 외에도 `LOCAL_POSITION_NED`, `DISTANCE_SENSOR`, `GLOBAL_POSITION_INT` 등이 들어온다. message queue가 밀리면 heartbeat가 실제로는 들어오고 있어도 `last_heartbeat_time`이 제때 갱신되지 않아 safety monitor가 abort할 수 있다.

### Planned Changes

1. `UdpMavlinkTransport`가 한 UDP datagram 안의 MAVLink frame을 모두 보존하도록 pending queue를 둔다.
2. `recvMessage()`는 pending queue가 있으면 먼저 반환하고, 없을 때만 socket에서 새 datagram을 읽는다.
3. `AutopilotMavlinkAdapter::poll(timeout_ms)`는 첫 message를 timeout 안에서 기다린 뒤, 같은 loop에서 non-blocking drain을 수행한다.
4. drain에는 무한 loop 방지를 위한 max message budget 또는 time budget을 둔다.
5. heartbeat 처리 시 `last_heartbeat_time`, mode, armed state가 계속 갱신되는지 log/debug 확인 경로를 둔다.
6. Safety timeout은 끄지 않는다. receive starvation을 고친 뒤에도 SITL jitter가 있으면 profile별 timeout만 분리한다.

### Candidate Interface

기존 public API는 유지한다.

```cpp
bool AutopilotMavlinkAdapter::poll(int timeout_ms);
```

동작만 다음처럼 바꾼다.

```text
poll(timeout_ms)
  -> wait one message up to timeout_ms
  -> process it
  -> recv pending/non-blocking messages with timeout 0
  -> process up to max_drain_messages
  -> return whether any message was processed
```

### Validation

- Fake transport unit test를 추가해 heartbeat가 non-heartbeat message 뒤에 있어도 한 번의 poll/drain에서 갱신되는지 확인한다.
- UDP transport pending queue는 가능하면 datagram 안에 여러 MAVLink frame이 있는 case를 test한다.
- 기존 `ctest` 전체 통과를 확인한다.
- `fly_test.sh` + `line_follow_node --target sitl`을 기본 config로 실행해 `pixhawk heartbeat lost` 없이 `COMPLETE` 되는지 확인한다.

성공 기준:

```text
[mavlink] heartbeat ok ...
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF ...
[mission] LINE_FOLLOW
[mission] LAND reason=duration complete
[mission] COMPLETE
```

## Step 2: Runtime Profile 정리

현재 `line_follow_node`에는 다음 경로가 있다.

```bash
--target sitl
--target pixhawk1
--autopilot udp://0.0.0.0:14550
--autopilot serial:///dev/serial0:115200
--vision fake
```

이를 다음처럼 확장한다.

```bash
./build/line_follow_node --config config --target sitl --vision gazebo
./build/line_follow_node --config config --target pixhawk1 --vision rpicam
./build/line_follow_node --config config --target sitl --vision fake
```

목표 profile:

```text
target=sitl
  autopilot: udp://0.0.0.0:14550
  vision: gazebo or fake
  frame source: Gazebo camera topic/source
  safety: SITL-friendly timeout, still failsafe enabled

target=pixhawk1
  autopilot: serial:///dev/serial0:115200
  vision: rpicam
  frame source: Raspberry Pi 4 + IMX519
  safety: real vehicle conservative timeout
```

설정 파일 방향:

```text
config/
  autopilot.toml
  mission.toml
  safety.toml
  vision.toml
  network.toml
  runtime.sitl.toml      # optional overlay
  runtime.pixhawk1.toml  # optional overlay
```

원칙:

- 코드 수정 없이 CLI option 또는 config overlay만으로 SITL/실기 전환.
- 같은 mission/control/autopilot code path를 SITL과 실제 환경에서 공유.
- source별 차이는 `FrameSource`, transport, camera parameter, safety profile 안에만 둔다.

## Step 3: Existing Vision Code Reuse

### Reusable Today

현재 코드에서 그대로 재사용할 수 있는 부분:

```text
src/vision/ArucoDetector.*
src/vision/LineMaskBuilder.*
src/vision/LineDetector.*
src/vision/LineStabilizer.*
src/vision/IntersectionDetector.*
src/vision/IntersectionStabilizer.*
src/mission/IntersectionDecisionEngine.*
src/mission/GridCoordinateTracker.*
src/network/UdpTelemetrySender.*
src/video/UdpMjpegStreamer.*
src/protocol/TelemetryMessage.*
```

### Needs Refactor

`src/app/VisionDebugPipeline.*`는 다음 역할이 한 클래스 안에 섞여 있다.

```text
Raspberry Pi camera open/read
JPEG decode
ArUco detection
line detection
intersection detection
telemetry build/send
debug video send
runtime loop
```

따라서 `VisionDebugPipeline` 전체를 `line_follow_node`에 그대로 넣기보다는, 내부 processing block을 아래처럼 분리한다.

```text
VisionProcessor
  input: cv::Mat image + frame metadata
  output: VisionResult + timing/debug metrics
```

그 다음:

```text
vision_debug_node
  RpicamFrameSource
  -> VisionProcessor
  -> telemetry/video publish

line_follow_node or onboard_runtime
  Fake/Gazebo/Rpicam FrameSource
  -> VisionProcessor
  -> Mission
  -> Control
  -> Autopilot
  -> telemetry/video publish
```

이렇게 하면 Raspberry Pi 4에서 이미 정상 동작한 detector logic은 유지하고, camera/source/runtime loop만 분리할 수 있다.

## Step 4: Gazebo Camera-Based Vision Plan

### Goal

Gazebo 안에서 아래 흐름을 만든다.

```text
Gazebo world
  line texture / marker plane
  Iris downward camera

Gazebo camera frame
  -> GazeboCameraSource or OpenCvFrameSource
  -> VisionProcessor
  -> LineFollowMission
  -> GuidedVelocityController
  -> MAVLink BODY_NED velocity command
```

첫 버전은 F450 정밀 모델이 아니라 Iris에 downward camera를 붙여 검증한다. 목적은 실제 dynamics 재현이 아니라 **vision -> mission -> control -> autopilot 경계가 end-to-end로 맞는지 확인**하는 것이다.

### Gazebo World

새 test world 또는 model override를 만든다.

```text
world:
  runway/ground plane
  high contrast line
  ArUco marker plane

vehicle:
  Iris base
  downward-facing RGB camera
  camera pose: body center 하단, optical axis down
  resolution/fps: vision.toml과 맞춤
```

초기 권장값:

```text
resolution: 960x720 또는 640x480
fps: 12
line: light_on_dark 또는 dark_on_light 중 하나로 고정
marker: DICT_4X4_50
```

### Camera Source

ROS 없이 진행하므로 후보는 두 가지다.

1. Preferred: Gazebo Transport subscriber
   - `gz::transport`로 camera image topic을 subscribe한다.
   - `gz::msgs::Image`를 `cv::Mat`으로 변환한다.
   - Gazebo topic 이름은 config로 둔다.

2. Fallback: OpenCV-readable stream/device
   - Gazebo camera output을 GStreamer/stream/device로 노출할 수 있을 때 `OpenCvFrameSource`로 읽는다.
   - local 개발환경에서 Gazebo transport dependency가 복잡하면 임시 fallback으로 사용한다.

계획상 우선순위는 Gazebo Transport subscriber다. Gazebo가 이미 `gz sim`으로 실행 중이므로, ROS bridge 없이 Gazebo native transport를 쓰는 편이 구조적으로 맞다.

### FrameSource Interface

공통 interface 예시:

```cpp
struct Frame {
    cv::Mat image_bgr;
    std::uint32_t frame_id;
    std::uint64_t timestamp_ms;
    int width;
    int height;
};

class FrameSource {
public:
    virtual bool open(const RuntimeConfig& config) = 0;
    virtual bool read(Frame& frame) = 0;
    virtual std::string lastError() const = 0;
};
```

구현체:

```text
FakeFrameSource
GazeboCameraSource
RpicamFrameSource
```

주의:

- `RpicamFrameSource`는 기존 `RpicamMjpegSource`를 감싸서 재사용한다.
- `GazeboCameraSource`는 BGR/RGB format 변환을 명확히 처리한다.
- Vision code는 source가 Gazebo인지 IMX519인지 알 필요가 없어야 한다.

## Step 5: Line Follow + ArUco Hover Path

MVP에 맞춘 Gazebo camera mission 흐름:

```text
takeoff
  -> line detected
  -> line center/angle로 forward/yaw/lateral correction
  -> ArUco marker detected
  -> marker center 근처에서 3초 hover
  -> land
```

구현 순서:

1. `VisionResult`에서 line detection을 `LineControlInput`으로 변환한다.
2. 기존 `GuidedVelocityController`가 fake input 대신 실제 line input을 받게 한다.
3. ArUco marker detection event를 mission input으로 전달한다.
4. mission에 `MarkerHover` state를 추가한다.
5. marker center error가 threshold 안에 들어오면 zero velocity를 3초 유지한다.
6. hover 완료 후 LAND로 전환한다.

중요:

- ArUco를 발견했다고 vision module이 직접 hover command를 보내지 않는다.
- Vision은 marker id/center/corners/confidence만 제공한다.
- Hover 판단은 mission이 한다.
- Zero velocity command는 control/autopilot 경계를 통해 나간다.

## Step 6: Raspberry Pi 4 + IMX519 Compatibility

실기 runtime 목표:

```bash
./build/line_follow_node --config config --target pixhawk1 --vision rpicam
```

실제 환경에서 달라지는 것:

```text
autopilot transport: UDP -> serial
camera source: GazeboCameraSource -> RpicamFrameSource
camera parameters: simulated camera -> IMX519 tuning
safety profile: SITL -> real vehicle
```

같게 유지할 것:

```text
VisionProcessor
Line/Aruco detector config schema
Mission state machine
GuidedVelocityController
Telemetry schema
GCS debug video path
```

추가로 필요한 실기 전 단계:

- `SerialMavlinkTransport` 구현.
- props-off Pixhawk1 bench test.
- heartbeat/mode/arm/takeoff command가 serial에서 정상 동작하는지 확인.
- Raspberry Pi 4에서 camera 단독 test, vision 단독 test, GCS telemetry/video test를 먼저 실행.
- 실기체는 `GUIDED` 진입, EKF local estimate, rangefinder/flow availability를 별도 preflight gate로 확인.

## Planned File Changes

예상 변경 대상:

```text
uav-onboard/src/autopilot/
  UdpMavlinkTransport.*
  AutopilotMavlinkAdapter.*

uav-onboard/src/vision/ 또는 src/app/
  VisionProcessor.*
  FrameSource abstraction
  GazeboCameraSource.*      # Gazebo transport dependency가 가능할 경우
  RpicamFrameSource.*       # existing RpicamMjpegSource wrapper

uav-onboard/tools/
  line_follow_node.cpp
  vision_debug_node.cpp     # VisionProcessor 재사용에 맞춘 최소 조정

uav-onboard/config/
  runtime.sitl.toml
  runtime.pixhawk1.toml
  vision.toml               # source별 override가 필요할 경우
  safety.toml               # profile별 timeout 분리 방식 반영

uav-onboard/tests/
  heartbeat/poll drain 관련 test
  VisionProcessor smoke test
```

문서 변경:

```text
development-log/
  RESEARCH.md
  PLAN.md
```

변경 금지/주의:

- 기존 detector algorithm을 불필요하게 재작성하지 않는다.
- Raspberry Pi에서 동작하던 ArUco/line detector API를 OpenCV 4.5 구 API로 되돌리지 않는다.
- Vision module에서 MAVLink command를 보내지 않는다.
- SITL 편의를 위해 safety를 제거하지 않는다.

## Implementation Order

### Phase 1: Heartbeat 안정화

1. `UdpMavlinkTransport` pending message queue 설계.
2. `recvMessage()`가 datagram 내 여러 MAVLink frame을 잃지 않도록 수정.
3. `AutopilotMavlinkAdapter::poll()` drain behavior 추가.
4. heartbeat 관련 unit test 추가.
5. `ctest` 실행.
6. Gazebo SITL smoke test를 기본 config로 재실행.

### Phase 2: VisionProcessor 분리

1. `VisionDebugPipeline` 내부 processing logic을 읽고 독립 가능한 입력/출력 구조를 정한다.
2. `VisionProcessor`를 추가해 `cv::Mat`과 frame metadata를 `VisionResult`로 변환한다.
3. `vision_debug_node`가 새 processor를 사용해 기존 GCS telemetry/video 동작을 유지하는지 확인한다.
4. 기존 detector unit/smoke test가 유지되는지 확인한다.

### Phase 3: FrameSource profile 추가

1. `FrameSource` interface 추가.
2. `FakeFrameSource` 또는 fake vision adapter 유지.
3. `RpicamFrameSource`를 기존 `RpicamMjpegSource` 기반으로 추가.
4. Gazebo image topic 확인 후 `GazeboCameraSource` 추가.
5. `--vision fake|gazebo|rpicam` option을 실제 source 선택으로 연결한다.

### Phase 4: Gazebo camera world

1. Iris downward camera가 붙은 test model/world를 만든다.
2. 바닥 line과 ArUco marker plane을 배치한다.
3. Gazebo camera topic이 frame을 내보내는지 확인한다.
4. `GazeboCameraSource -> VisionProcessor` 단독 smoke를 먼저 돌린다.
5. line/marker detection 결과를 GCS telemetry 또는 console log로 확인한다.

### Phase 5: Vision-driven flight

1. fake line input 대신 `VisionResult.line`을 `GuidedVelocityController` input으로 변환한다.
2. line lost 시 `SafetyMonitor`가 land로 전환하는지 확인한다.
3. marker detection을 mission event로 전달한다.
4. marker hover 3초 state를 추가한다.
5. Gazebo에서 line follow + marker hover + land sequence를 검증한다.

### Phase 6: Real target 준비

1. `--target pixhawk1 --vision rpicam` profile을 완성한다.
2. `SerialMavlinkTransport` 구현 계획을 별도 세부 계획으로 분리한다.
3. Raspberry Pi 4에서 build/runtime dependency 확인 절차를 작성한다.
4. props-off Pixhawk1 bench test checklist를 작성한다.

## Validation Plan

### Build/Test

```bash
cd ~/astroquad/uav-onboard
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DBUILD_TOOLS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

### SITL Heartbeat Smoke

```bash
cd ~
bash ./fly_test.sh
```

다른 터미널:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node --config config --target sitl --vision fake
```

성공 기준:

- 임시 `/tmp` safety override 없이 완료.
- `pixhawk heartbeat lost` abort 없음.
- Gazebo에서 takeoff/move/land 확인.

### Gazebo Camera Vision Smoke

예상 실행:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node --config config --target sitl --vision gazebo
```

성공 기준:

- Gazebo camera frame 수신.
- line detection telemetry/log 확인.
- ArUco detection telemetry/log 확인.
- 아직 비행 제어를 붙이기 전에도 vision 단독 smoke가 가능해야 한다.

### Vision-Driven Flight Smoke

성공 기준:

- takeoff 후 실제 Gazebo camera image 기반 line input 사용.
- line center error에 따라 BODY_NED velocity/yaw-rate command 생성.
- ArUco marker 감지 시 mission이 hover state로 전환.
- 3초 hover 후 land.
- GCS에 vision/flight telemetry가 송신된다.

### Raspberry Pi Compatibility Check

실기 전 확인:

```bash
./build/vision_debug_node --config config --count 100
./build/line_follow_node --config config --target pixhawk1 --vision rpicam
```

성공 기준:

- IMX519 frame capture 정상.
- 기존 line/ArUco detector 결과 유지.
- GCS telemetry/video 송신 정상.
- Pixhawk1 serial 연결은 별도 bench test에서 검증.

## Acceptance Criteria

- 기본 SITL config에서 heartbeat false-positive abort가 재현되지 않는다.
- MAVLink receive path는 heartbeat 외 telemetry stream이 많아도 state cache를 안정적으로 갱신한다.
- 기존 vision detector code는 재사용되고, mission/control/autopilot 경계는 유지된다.
- Gazebo camera frame을 `VisionProcessor`에 넣을 수 있는 구조가 생긴다.
- `--target`과 `--vision` 또는 profile config만으로 SITL/Gazebo와 Raspberry Pi/IMX519 경로를 선택할 수 있다.
- Vision telemetry/video를 GCS로 보내는 기존 경로를 유지하거나 재사용할 수 있다.
- 코드 구현 후 `BUILD_TOOLS=ON` build와 unit test가 통과한다.

## Implementation Result

2026-05-14 구현/검증 결과:

- Phase 1 heartbeat receive path를 수정했다. `UdpMavlinkTransport`는 한 datagram 안의 MAVLink frame을 pending queue에 보존하고, `AutopilotMavlinkAdapter::poll()`은 첫 message 이후 non-blocking drain을 수행한다.
- Phase 2 `VisionProcessor`를 추가해 기존 ArUco/line/intersection detector와 stabilizer를 재사용 가능한 processing block으로 분리했다. `VisionDebugPipeline`은 기존 GCS telemetry/video 동작을 유지하면서 새 processor를 사용한다.
- Phase 3 `FrameSource` abstraction과 `FakeFrameSource`, `GazeboCameraSource`, `RpicamFrameSource`를 추가했다. `line_follow_node`는 `--vision fake|gazebo|rpicam`, `--vision-smoke-count`, `runtime.sitl.toml`, `runtime.pixhawk1.toml` profile을 지원한다.
- Phase 4 Gazebo Iris wrapper world를 추가했다. Downward camera topic은 `config/vision.toml`의 `[source].gazebo_topic`으로 설정되어 있고, Gazebo transport frame을 OpenCV `cv::Mat`으로 변환해 `VisionProcessor`에 넣는다.
- Phase 5 `VisionResult.line`을 `GuidedVelocityController` 입력으로 변환하고, ArUco marker centered event를 mission에 전달하는 `MarkerHover` state를 추가했다. Hover 완료 후 LAND로 전환한다.
- Phase 6 real target 준비로 `--target pixhawk1 --vision rpicam` profile과 serial endpoint 안내를 정리했다. `SerialMavlinkTransport` 구현과 props-off Pixhawk1 bench test는 별도 승인 스텝으로 남긴다.

검증:

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
[vision] frame=1 source=gazebo size=640x480 line=yes offset_px=-0.666656 angle_deg=90 markers=0
```

Gazebo vision-driven flight smoke:

```text
[vision] source=gazebo opened
[mavlink] heartbeat ok system=1 component=1 mode=STABILIZE
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF target=1.2m
[mission] LINE_FOLLOW
[mission] LAND reason=duration complete
[mission] COMPLETE
```

남은 제한:

- Gazebo ground smoke에서는 ArUco marker가 카메라 시야에 들어오는 위치까지 검증하지 못했다. `MarkerHover` 전이는 unit test로 검증했고, marker placement/flight path 조정은 다음 승인 스텝에서 다룬다.
- `SerialMavlinkTransport`는 이번 PLAN의 구현 범위가 아니라 별도 세부 계획/bench test 대상으로 유지한다.
- `line_follow_node`의 GCS telemetry/video wire schema는 새 field 없이 기존 `vision_debug_node` protocol과 호환되도록 유지했다. 실제 mission runtime에서 flight 상태 telemetry field를 추가하는 일은 protocol v1.8 후보로 분리한다.
