# Current Step Plan

최종 업데이트: 2026-05-15

이 문서는 `RESEARCH.md`를 바탕으로 작성한 **Gazebo + GCS 테스트 환경 구현 계획서**다. 사용자가 승인하기 전까지 코드는 수정하지 않는다. 승인 후에는 이 문서의 순서대로 진행한다.

## Goal

현재 우선 목표는 라인 트레이싱 MAVLink 제어 성능 튜닝이 아니라, 아래 테스트 환경을 안정적으로 만드는 것이다.

```text
Windows 11
  -> uav_gcs_vision_debug.exe --config config
  -> telemetry UDP 14550 대기
  -> MJPEG video UDP 5600 대기

WSL Ubuntu
  -> fly_test.sh
  -> Gazebo Sim + ArduCopter SITL
  -> Iris + downward camera
  -> 폭 10cm 흰색 직선 라인
  -> 출발 지점 기준 3m 전방에 50cm x 50cm ArUco ID 1 marker

uav-onboard
  -> Gazebo downward camera frame 수신
  -> 기존 onboard vision 처리
  -> GCS로 raw/top-down camera video 송출
  -> GCS로 line/marker/intersection telemetry 송출
  -> 이후 같은 vision 결과로 line_follow_node SITL 제어 검증
```

## Non-Goals

이번 승인 스텝에서 하지 않는다.

- F450 정밀 Gazebo model 제작.
- 실제 IMX519 lens distortion 정밀 재현.
- optical flow/range Gazebo plugin 정밀 튜닝.
- Pixhawk1 serial transport 구현.
- 실기체 props-on 비행.
- GCS UI 대규모 개편.
- Vision module에서 MAVLink command를 직접 보내는 구조.
- Protocol v1.8 확정. 기존 v1.7 telemetry/video wire format을 우선 유지한다.

## Design Rules

- 실기체 기본값을 Gazebo 값으로 덮지 않는다.
- Raspberry Pi 4 + IMX519 경로는 계속 `rpicam` source로 유지한다.
- Gazebo/SITL 값은 `runtime.sitl.toml`, Gazebo asset, CLI option에서만 선택한다.
- `VisionProcessor`는 image를 perception result로만 바꾼다.
- GCS video/telemetry 송출은 관제/debug publisher이며 mission-critical 경로가 아니다.
- `vision_debug_node`에는 MAVLink 제어 코드를 넣지 않는다.
- `line_follow_node`는 같은 vision publisher를 재사용하되, mission/control/autopilot 경계는 유지한다.

## Target Commands

승인 후 구현이 끝나면 다음 명령이 동작해야 한다.

Windows GCS:

```powershell
cd astroquad\uav-gcs
.\build\uav_gcs_vision_debug.exe --config config
```

WSL Gazebo/SITL:

```bash
bash ~/fly_test.sh
```

WSL vision-only GCS smoke:

```bash
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"

cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-mode light_on_dark \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

WSL marker-only fixture smoke:

```bash
cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --aruco-only \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

WSL vision-driven flight smoke:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

기존 실기체 vision debug는 계속 유지한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
```

## Work Breakdown

### Phase 0: Preflight Snapshot

목적: 사용자가 만든 변경과 현재 작업 상태를 건드리지 않고 기준점을 확인한다.

작업:

- `development-log`, `uav-onboard`, `uav-gcs` 각각 `git status --short` 확인.
- `/home/mseoky/test_aruco_marker` asset 확인.
- 현재 `vision_debug_node --help`, `line_follow_node --help` option 확인.
- Gazebo version과 world load 가능 여부 확인.

검증:

```bash
find /home/mseoky/test_aruco_marker -maxdepth 1 -type f -printf '%f\n' | sort
gz sim --versions
```

성공 기준:

```text
aruco1.png
aruco2.png
aruco3.png
aruco4.png
Vmarker.png
```

### Phase 1: Gazebo Course Asset 정리

목적: Gazebo world가 요구한 물리 조건을 명확히 갖게 한다.

변경 파일:

```text
uav-onboard/sim/gazebo/models/astroquad_vision_course/model.config
uav-onboard/sim/gazebo/models/astroquad_vision_course/model.sdf
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/aruco1.png
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/aruco2.png
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/aruco3.png
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/aruco4.png
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/aruco_id1.png
uav-onboard/sim/gazebo/models/astroquad_vision_course/materials/textures/Vmarker.png
uav-onboard/sim/gazebo/models/astroquad_static_downward_camera/model.config
uav-onboard/sim/gazebo/models/astroquad_static_downward_camera/model.sdf
uav-onboard/sim/gazebo/worlds/astroquad_iris_vision.sdf
uav-onboard/sim/gazebo/worlds/astroquad_line_camera_fixture.sdf
uav-onboard/sim/gazebo/worlds/astroquad_marker_center_fixture.sdf
uav-onboard/sim/gazebo/README.md
```

구현 내용:

- `/home/mseoky/test_aruco_marker`의 PNG asset을 repo 안 Gazebo model texture directory로 복사한다.
- `astroquad_vision_course` model을 새로 만든다.
- dark ground plane을 course model에 둔다.
- 흰색 line은 폭 `0.10m`, 길이 `7.0m`로 둔다.
- 기본 course marker는 검증된 `aruco_id1.png`를 사용한다. 구현 중 확인 결과 `/home/mseoky/test_aruco_marker/aruco2.png`가 OpenCV `DICT_4X4_50` 실제 ID 1이었다.
- marker physical size는 `0.50m x 0.50m`.
- marker pose는 시작점 기준 전방 `3.0m`에 둔다.
- `Vmarker.png`는 이번 world에 기본 배치하지 않는다. vertiport 후보 asset으로만 보관한다.
- `astroquad_iris_vision.sdf`는 inline course visual을 줄이고 `model://astroquad_vision_course`를 include한다.
- `astroquad_line_camera_fixture.sdf`와 `astroquad_marker_center_fixture.sdf`는 ArduPilot 없이 고정 하향 camera로 detector smoke를 빠르게 돌리는 fixture다.

주의:

- marker texture가 Gazebo material mapping에서 뒤집히면 SDF pose yaw 또는 texture orientation을 조정한다.
- plane texture mapping이 Gazebo에서 반복/타일링되면 fallback으로 marker를 box-cell geometry로 생성하되, source image는 `aruco_id1.png` 기준으로 유지한다.
- line detector smoke와 marker detector smoke를 분리할 수 있게 fixture world를 둔다.

검증:

```bash
cd ~/astroquad/uav-onboard

GZ_SIM_SYSTEM_PLUGIN_PATH="$HOME/ardupilot_gazebo/build:${GZ_SIM_SYSTEM_PLUGIN_PATH:-}" \
GZ_SIM_RESOURCE_PATH="$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$PWD/sim/gazebo/models:$PWD/sim/gazebo/worlds:${GZ_SIM_RESOURCE_PATH:-}" \
timeout 20s gz sim -s -v4 -r sim/gazebo/worlds/astroquad_iris_vision.sdf
```

성공 기준:

- world load 중 fatal error 없음.
- `iris_with_downward_camera`와 `astroquad_vision_course`가 포함됨.
- known warning인 upstream `gz_frame_id` warning 외 새로운 asset missing error 없음.

### Phase 2: `fly_test.sh`를 Astroquad World 기준으로 정리

목적: 사용자가 `bash ~/fly_test.sh`만 실행해도 Astroquad vision world와 ArduCopter SITL이 같이 뜨게 한다.

변경 파일:

```text
uav-onboard/scripts/fly_test.sh
/home/mseoky/fly_test.sh
uav-onboard/README.md
uav-onboard/sim/gazebo/README.md
```

구현 내용:

- repo 안에 versioned `scripts/fly_test.sh`를 만든다.
- home-level `/home/mseoky/fly_test.sh`는 repo script를 호출하는 thin wrapper로 바꾸거나, 같은 내용으로 갱신한다.
- `GAZEBO_WORLD_FILE`을 `iris_runway.sdf`가 아니라 `astroquad_iris_vision.sdf`로 바꾼다.
- `GZ_SIM_SYSTEM_PLUGIN_PATH`에 `$HOME/ardupilot_gazebo/build`를 포함한다.
- `GZ_SIM_RESOURCE_PATH`에 다음을 포함한다.

```text
$HOME/ardupilot_gazebo/models
$HOME/ardupilot_gazebo/worlds
$HOME/astroquad/uav-onboard/sim/gazebo/models
$HOME/astroquad/uav-onboard/sim/gazebo/worlds
```

- script 출력에서 MAVLink UDP 14550과 Astroquad JSON telemetry UDP 14550을 구분해서 설명한다.
- Windows GCS IP 확인 방법을 script 출력에 안내한다.

검증:

```bash
bash ~/fly_test.sh
```

성공 기준:

- Gazebo가 Astroquad course world를 연다.
- ArduCopter SITL/MAVProxy console이 열린다.
- script가 onboard 실행 예시를 `--target sitl --vision gazebo` 기준으로 출력한다.

### Phase 3: Runtime Profile Loader 정리

목적: `vision_debug_node`와 `line_follow_node`가 같은 target/source profile을 해석하게 한다.

변경 파일 후보:

```text
uav-onboard/src/common/RuntimeProfileConfig.hpp
uav-onboard/src/common/RuntimeProfileConfig.cpp
uav-onboard/CMakeLists.txt
uav-onboard/config/runtime.sitl.toml
uav-onboard/config/runtime.pixhawk1.toml
uav-onboard/tools/vision_debug_node.cpp
uav-onboard/tools/line_follow_node.cpp
```

구현 내용:

- `--target sitl|pixhawk1` parsing을 `vision_debug_node`에도 추가한다.
- `--vision fake|gazebo|rpicam` parsing을 `vision_debug_node`에도 추가한다.
- 기존 `vision_debug_node --config config --line-only --line-mode light_on_dark --video`는 default `rpicam`으로 계속 동작하게 한다.
- `runtime.sitl.toml` default vision은 `gazebo`로 바꾼다.
- fake smoke가 필요하면 CLI에서 `--vision fake`로 명시한다.
- runtime overlay가 다음 값을 덮어쓸 수 있게 한다.
  - `[runtime].vision`
  - `[vision.source].gazebo_topic`
  - `[vision.source].read_timeout_ms`
  - `[debug_video].send_fps`
  - `[debug_video].jpeg_quality`
  - `[line_follow].duration_s`
  - `[line_follow].forward_mps`
  - `[marker_hover].hold_s`
  - `[marker_hover].center_tolerance_px`

권장 `runtime.sitl.toml`:

```toml
[runtime]
target = "sitl"
vision = "gazebo"

[transport]
kind = "udp"
listen_host = "0.0.0.0"
listen_port = 14550

[vision.source]
gazebo_topic = "/world/astroquad_iris_vision/model/iris_with_downward_camera/link/downward_camera_link/sensor/downward_camera/image"
read_timeout_ms = 500

[debug_video]
send_fps = 12
jpeg_quality = 70

[line_follow]
forward_mps = 0.3
duration_s = 15.0

[marker_hover]
hold_s = 3.0
center_tolerance_px = 80.0
```

검증:

```bash
cd ~/astroquad/uav-onboard
./build/vision_debug_node --help
./build/line_follow_node --help
```

성공 기준:

- `vision_debug_node` help에 `--target`과 `--vision`이 보인다.
- 기존 rpicam 명령은 option conflict 없이 유지된다.
- `line_follow_node --target sitl --vision fake`도 여전히 가능하다.

### Phase 4: `vision_debug_node`를 Gazebo FrameSource에 연결

목적: MAVLink 제어 없이도 Gazebo camera frame을 GCS로 보내고 overlay를 확인한다.

변경 파일 후보:

```text
uav-onboard/src/app/VisionDebugPipeline.hpp
uav-onboard/src/app/VisionDebugPipeline.cpp
uav-onboard/tools/vision_debug_node.cpp
uav-onboard/src/vision/FrameSource.hpp
uav-onboard/src/vision/GazeboCameraSource.*
uav-onboard/src/vision/RpicamFrameSource.*
uav-onboard/CMakeLists.txt
```

구현 내용:

- `VisionDebugPipelineOptions`에 `vision_source`를 추가한다.
- `vision_source=rpicam`이면 `RpicamFrameSource`를 사용한다.
- `vision_source=gazebo`이면 `GazeboCameraSource`를 사용한다.
- `vision_source=fake`는 `vision_debug_node`에서는 smoke/debug 용도로만 허용하거나 unsupported로 명확히 에러 처리한다.
- `RpicamFrameSource`의 `jpeg_data`가 있으면 기존처럼 raw MJPEG를 GCS로 보낸다.
- `GazeboCameraSource`는 BGR frame만 있으므로 `cv::imencode(".jpg", ...)`로 JPEG를 만들어 `UdpMjpegStreamer`에 넣는다.
- telemetry JSON shape은 v1.7 그대로 유지한다.
- `camera.sensor_model` 또는 `debug.note`에 source가 `gazebo`임을 남긴다.
- `--line-only`를 쓰면 ArUco marker detection은 꺼진다. marker 검증은 `--aruco-only` 또는 detector 둘 다 켠 default run에서 한다.

검증:

Gazebo 실행 후:

```bash
cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-only \
  --line-mode light_on_dark \
  --count 30
```

성공 기준:

```text
source=gazebo
frame 증가
line=yes
decode/frame read fatal error 없음
```

Windows GCS를 켠 상태:

```bash
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-mode light_on_dark \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

성공 기준:

- Windows GCS camera window에 Gazebo top-down frame이 뜬다.
- line overlay가 표시된다.
- vision log에서 telemetry seq가 증가한다.
- `[video-rx] completed`가 증가한다.

### Phase 5: ArUco ID 1 Fixture Smoke

목적: Gazebo marker texture가 실제 onboard ArUco detector에서 ID 1로 검출되는지 먼저 분리 검증한다.

변경 파일:

```text
uav-onboard/sim/gazebo/worlds/astroquad_marker_center_fixture.sdf
uav-onboard/config/runtime.sitl.toml
```

구현 내용:

- marker-center fixture world에서 `aruco_id1.png`가 camera FOV 중앙 근처에 보이도록 둔다.
- 필요하면 `runtime.sitl.toml`의 gazebo topic을 fixture world name에 맞춰 override할 방법을 문서화한다.
- detector가 ID 1을 못 읽으면 texture orientation, alpha channel, border margin, material lighting을 순서대로 확인한다.
- `Vmarker.png`는 이 smoke에 쓰지 않는다.

검증:

```bash
cd ~/astroquad/uav-onboard
GZ_SIM_SYSTEM_PLUGIN_PATH="$HOME/ardupilot_gazebo/build:${GZ_SIM_SYSTEM_PLUGIN_PATH:-}" \
GZ_SIM_RESOURCE_PATH="$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$PWD/sim/gazebo/models:$PWD/sim/gazebo/worlds:${GZ_SIM_RESOURCE_PATH:-}" \
gz sim -v4 -r sim/gazebo/worlds/astroquad_marker_center_fixture.sdf
```

다른 터미널:

```bash
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --aruco-only \
  --count 30
```

성공 기준:

```text
markers=1
marker_id=1
```

GCS 성공 기준:

- marker box/corners/center overlay가 표시된다.
- log에 marker ID 1이 표시된다.

### Phase 6: `line_follow_node`에 GCS Publisher 연결

목적: SITL 제어 중에도 GCS에서 같은 Gazebo camera/vision overlay를 볼 수 있게 한다.

변경 파일 후보:

```text
uav-onboard/tools/line_follow_node.cpp
uav-onboard/src/app/VisionDebugPipeline.*
uav-onboard/src/app/VisionDebugPublisher.hpp
uav-onboard/src/app/VisionDebugPublisher.cpp
uav-onboard/CMakeLists.txt
uav-onboard/src/protocol/TelemetryMessage.*
```

구현 내용:

- `line_follow_node`에 다음 option을 추가한다.
  - `--gcs-ip <ip>`
  - `--video`
  - `--no-video`
  - `--no-telemetry`
  - `--telemetry-port <n>`
  - `--video-port <n>`
- `VisionDebugPipeline` 안의 telemetry/video 송출 코드를 그대로 복사하지 않는다.
- 공통 publisher component를 만든다.
- `vision_debug_node`와 `line_follow_node`가 같은 publisher를 사용하게 한다.
- 초기 구현은 protocol v1.7 shape를 유지한다.
- mission/autopilot state는 당장 필수 field로 추가하지 않는다. 필요하면 `debug.note`에 `line_follow_node source=gazebo` 정도만 남긴다.
- `VisionResult.line`은 기존처럼 `GuidedVelocityController` input으로 사용한다.
- marker ID 1 centered event가 들어오면 기존 `MarkerHover` state로 연결한다.

검증:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

성공 기준:

- MAVLink sequence:

```text
[mavlink] heartbeat ok
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF
[mission] LINE_FOLLOW
```

- GCS:

```text
camera frame 표시
line overlay 표시
telemetry seq 증가
video completed frame 증가
```

### Phase 7: End-To-End Marker Hover Flight

목적: 3m 전방 ID 1 marker가 실제 flight path에서 검출되고 hover/land까지 이어지는지 확인한다.

사전 조건:

- Phase 1-6 성공.
- Windows GCS UDP firewall 허용.
- `runtime.sitl.toml` line-follow duration이 12-15초 이상.
- marker fixture에서 `marker_id=1` 검출 성공.

실행:

```bash
bash ~/fly_test.sh
```

다른 터미널:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

성공 기준:

```text
[mavlink] heartbeat ok
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF
[mission] LINE_FOLLOW
[vision] marker id=1 centered=yes
[mission] MARKER_HOVER
[mission] LAND reason=marker hover complete
[mission] COMPLETE
```

GCS 성공 기준:

- top-down camera가 계속 표시된다.
- line overlay가 line-follow 중 유지된다.
- marker ID 1 overlay가 보인다.
- video/telemetry 수신이 mission loop를 막지 않는다.

## Build And Test Plan

기본 build/test:

```bash
cd ~/astroquad/uav-onboard
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DBUILD_TOOLS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

추가 unit/integration test 후보:

```text
tests/test_runtime_profile_config.cpp
tests/test_vision_debug_publisher.cpp
```

추가 test가 과도하면 최소한 기존 12개 test가 계속 통과해야 한다.

## Failure Triage

Gazebo world가 열리지 않음:

- `GZ_SIM_RESOURCE_PATH`에 `uav-onboard/sim/gazebo/models`가 있는지 확인.
- `GZ_SIM_SYSTEM_PLUGIN_PATH`에 `ardupilot_gazebo/build`가 있는지 확인.
- missing texture/model error를 먼저 해결.

Gazebo camera frame timeout:

- `gz topic -l | grep downward_camera` 확인.
- world name이 바뀌면 `runtime.sitl.toml`의 topic도 같이 바꾼다.
- camera sensor `always_on=true`, `update_rate=12` 확인.

GCS에 video가 안 뜸:

- `--video`가 켜졌는지 확인.
- `--gcs-ip`가 Windows GCS IP인지 확인.
- Windows Defender Firewall inbound UDP 5600/14550 허용 확인.
- GCS log의 `video_sent`, `chunks_last`, `[video-rx] completed` 확인.

Line은 보이는데 marker가 안 잡힘:

- `--line-only`를 쓰고 있지 않은지 확인.
- `--aruco-only`로 fixture부터 분리 검증.
- marker texture orientation과 border margin 확인.
- `aruco1.png` alpha channel/material lighting이 Gazebo render에서 흐려지지 않는지 확인.

MAVLink는 되는데 GCS 송출 때문에 loop가 느려짐:

- video send FPS를 `runtime.sitl.toml [debug_video].send_fps`로 낮춘다.
- publisher는 latest-frame/drop-old 구조를 유지한다.
- mission/control loop가 video send thread를 기다리지 않게 한다.

## Acceptance Criteria

승인 스텝 완료 조건:

- `~/fly_test.sh`가 Astroquad Gazebo vision world를 실행한다.
- world에는 폭 10cm 흰색 직선 라인이 있다.
- world에는 출발 지점 기준 3m 전방에 50cm x 50cm `aruco_id1.png`/ID 1 marker가 있다.
- `vision_debug_node --target sitl --vision gazebo --video`가 Windows GCS에 Gazebo top-down camera를 보낸다.
- GCS가 기존 overlay로 line telemetry를 표시한다.
- marker-center fixture에서 `marker_id=1` 검출이 확인된다.
- `line_follow_node --target sitl --vision gazebo --video` 실행 중에도 GCS video/telemetry가 유지된다.
- 기존 `vision_debug_node --config config --line-only --line-mode light_on_dark --video` rpicam 경로는 깨지지 않는다.
- `cmake --build build`와 `ctest --test-dir build --output-on-failure`가 통과한다.

## After This Step

다음 승인 후보:

1. Protocol v1.8 설계: mission state, autopilot mode/armed/altitude/range, runtime profile field 추가.
2. `SerialMavlinkTransport` 구현과 Pixhawk1 props-off bench test.
3. Raspberry Pi 4에서 `--target pixhawk1 --vision rpicam` camera/vision/GCS 송출 smoke.
4. Gazebo marker 여러 개와 `Vmarker.png` vertiport visual fixture 설계.

## Implementation Result 2026-05-15

완료:

- `astroquad_iris_vision.sdf` main world가 Gazebo Sim 8.10에서 load되고 downward camera topic을 publish함.
- `astroquad_vision_course`에 폭 10cm 흰색 line과 3m 전방 50cm x 50cm ID 1 marker를 배치함.
- static camera fixture 2개를 추가해 ArduPilot 없이 detector를 검증할 수 있게 함.
- `vision_debug_node --target sitl --vision gazebo`가 Gazebo frame을 읽고 GCS telemetry/video publisher를 통해 MJPEG를 송출함.
- `line_follow_node --target sitl --vision gazebo --video`가 SITL heartbeat, GUIDED, arm, takeoff, line-follow, land, complete까지 통과함.
- `fly_test.sh`는 repo script로 이동하고 home wrapper가 이를 호출한다. MAVProxy map module은 기본 비활성화하고 `MAVPROXY_MAP=1`로 opt-in하게 했다.

검증 결과:

```text
cmake --build build: passed
ctest --test-dir build --output-on-failure: 12/12 passed
marker fixture: marker_id=1, line=yes, video_sent 증가 확인
main world: default Gazebo topic frame read 확인
line_follow_node SITL: heartbeat ok -> GUIDED -> armed -> TAKEOFF -> LINE_FOLLOW -> LAND -> COMPLETE
```

주의:

- `/home/mseoky/test_aruco_marker/aruco1.png`는 실제 detector ID 4로 확인됨. active ID 1 texture는 `aruco2.png`를 복사한 `aruco_id1.png`다.
- 메인 world에서 지상 초기 camera frame은 카메라가 낮아 `line=no`가 나올 수 있다. 실제 line-follow 검증은 takeoff 후 `line=yes`로 확인했다.
