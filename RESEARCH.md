# Current Step Research

최종 업데이트: 2026-05-14

이번 조사의 목표는 라인 트레이싱 MAVLink 제어 자체보다 먼저, Windows 11 GCS와 WSL Ubuntu의 Gazebo + ArduPilot SITL을 사용해 반복 가능한 vision/GCS 테스트 환경을 만드는 것이다.

원하는 최종 개발 흐름:

```text
Windows 11
  -> uav_gcs_vision_debug.exe 실행
  -> UDP telemetry 14550, UDP MJPEG video 5600 대기

WSL Ubuntu
  -> fly_test.sh 실행
  -> Gazebo Sim + ArduCopter SITL 실행
  -> Iris + downward camera
  -> 어두운 바닥 위 폭 10cm 흰색 직선 라인
  -> 출발 지점 기준 3m 전방에 50cm x 50cm ArUco marker

uav-onboard runtime
  -> Gazebo downward camera frame 수신
  -> 기존 onboard line/ArUco/intersection vision 처리
  -> GCS로 raw/top-down camera MJPEG 송출
  -> GCS로 vision telemetry/log 송출
  -> 필요 시 line_follow_node가 같은 vision 결과로 SITL Iris를 제어
```

실기체 기본값은 Raspberry Pi 4 + IMX519 + Pixhawk1 기준으로 유지해야 한다. Gazebo/SITL 값은 runtime profile 또는 CLI option으로만 선택한다.

## 확인한 현재 상태

### 문서 기준

- 전체 시스템 경계는 `development-log/SYSTEM_SPEC.md` 기준이다.
- MVP 범위는 `development-log/MVP_PLAN.md` 기준이며, 현재 1차 자동비행 목표는 짧은 직선 line follow 후 안전 착륙이다.
- GCS는 미션 판단을 하지 않고, onboard가 보낸 video/telemetry를 수신해 overlay와 log를 표시한다.
- `uav_gcs_vision_debug`는 현재 vision bring-up의 주 도구다.
- `vision_debug_node`는 vision + telemetry + optional debug video 전용으로 유지해야 하며 MAVLink 제어를 넣으면 안 된다.
- `line_follow_node` 또는 `mission_node`는 staging 실행 파일로 허용된다. 안정화 후 최종 `uav_onboard` composition root로 흡수한다.

### 구현 기준선

이미 준비된 것:

- GCS:
  - `uav_gcs_vision_debug`: UDP MJPEG video 수신, UDP JSON telemetry 수신, marker/line/intersection overlay 표시.
  - GCS discovery beacon: UDP 5601로 `AQGCS1 video_port=5600` broadcast.
  - Protocol v1.7 parser는 unknown field를 무시하므로 source/runtime field를 debug note로 추가해도 호환된다.
- Onboard:
  - `vision_debug_node`: Pi `rpicam` frame을 읽어 GCS telemetry/video 송출 가능.
  - `VisionProcessor`: 기존 detector/stabilizer block이 `VisionDebugPipeline`에서 분리되어 있음.
  - `FrameSource`: `FakeFrameSource`, `GazeboCameraSource`, `RpicamFrameSource`가 추가되어 있음.
  - `line_follow_node`: `--target sitl|pixhawk1`, `--vision fake|gazebo|rpicam`, `--vision-smoke-count` 지원.
  - `GazeboCameraSource`: ROS bridge 없이 Gazebo Transport image topic을 구독해 OpenCV BGR frame으로 변환.
  - `sim/gazebo/worlds/astroquad_iris_vision.sdf`: Iris wrapper + downward camera + dark ground + line + hand-built marker world가 있음.
  - `config/runtime.sitl.toml`, `config/runtime.pixhawk1.toml`: SITL/Pixhawk profile 초안이 있음.
- Test asset:
  - `/home/mseoky/test_aruco_marker`에 테스트용 marker image가 추가되어 있음.
  - 파일 목록은 `aruco1.png`, `aruco2.png`, `aruco3.png`, `aruco4.png`, `Vmarker.png`.
  - 구현 검증 결과, OpenCV `DICT_4X4_50` 기준 실제 ID는 `aruco2.png -> ID 1`, `aruco3.png -> ID 2`, `aruco4.png -> ID 3`, `aruco1.png -> ID 4`다. 파일명과 실제 ID가 1씩 어긋나므로 Gazebo active ID 1 texture는 `aruco2.png`를 복사한 `aruco_id1.png`로 둔다.
  - `Vmarker.png`는 추후 vertiport/landing target 시각 자산 후보로 둔다. 현재 ArUco marker detection 검증 대상은 `aruco1.png`-`aruco4.png`다.

로컬 환경 확인:

```text
Gazebo Sim: gz sim 8.10.0
MAVLink headers: ~/ardupilot/build/sitl/libraries/GCS_MAVLink/include
Gazebo transport CMake packages: gz-msgs10, gz-transport13
line_follow_node: build target exists and --vision gazebo option exposed
```

`gz sim -s -v4 -r sim/gazebo/worlds/astroquad_iris_vision.sdf`는 server load까지 확인했다. upstream `iris_with_standoffs`의 `gz_frame_id` SDF warning은 표시되지만 world 초기화와 ArduPilot plugin load는 진행된다.

## 현재 원하는 동작과의 차이

1. `~/fly_test.sh`는 아직 Astroquad world가 아니라 `$HOME/ardupilot_gazebo/worlds/iris_runway.sdf`를 실행한다.
2. 현재 Astroquad world의 line은 폭 `0.04m`이고, 요구사항은 `0.10m`다.
3. 현재 marker는 `y=1.2m` 부근에 hand-built SDF box cell로 들어가 있고, 요구사항은 출발 지점 기준 `3.0m` 전방의 `0.50m x 0.50m` ArUco marker다.
4. hand-built SDF box marker는 OpenCV ArUco canonical bitmap과 정확히 맞는지 보장하기 어렵다. 이전 smoke 결과도 `markers=0`이었다.
5. `line_follow_node --vision gazebo`는 Gazebo vision을 읽고 MAVLink control을 할 수 있지만, GCS로 MJPEG video와 vision telemetry를 아직 보내지 않는다.
6. `vision_debug_node`는 GCS 송출 경로가 완성돼 있지만 camera source가 `RpicamMjpegSource`로 고정되어 있어 Gazebo topic을 직접 볼 수 없다.
7. `line_follow_node` 기본 mission 설정은 `forward_mps=0.3`, `duration_s=3.0`이라 3m 전방 marker까지 가지 못한다. 0.3m/s면 최소 10초 이상이 필요하고, 안정 여유를 포함해 SITL profile에서는 12-15초가 적당하다.
8. Windows GCS와 WSL onboard 사이에서는 UDP broadcast/discovery가 네트워크 모드에 따라 막힐 수 있다. 시뮬레이션 경로는 `--gcs-ip <Windows-host-ip>` 명시를 1차 실행법으로 두는 편이 안전하다.
9. `runtime.pixhawk1.toml`은 serial endpoint를 고르지만 `SerialMavlinkTransport`는 아직 구현되지 않았다. 이번 Gazebo test 환경 구축의 blocker는 아니다.

## 권장 설계 방향

### 핵심 원칙

- 실기체 기본 config를 Gazebo 값으로 덮지 않는다.
- `vision.toml`의 Pi/IMX519 기본값은 유지한다.
- SITL 전용 값은 `runtime.sitl.toml`, Gazebo world asset, 또는 CLI option으로만 고른다.
- `VisionProcessor`는 계속 image -> perception result 역할만 한다.
- GCS video/telemetry 송출은 mission-critical 경로가 아니라 관제용 publisher로 분리한다.
- `vision_debug_node`와 `line_follow_node`가 같은 `FrameSource -> VisionProcessor -> telemetry/video publisher` 조립 부품을 재사용하게 만든다.

### 실행 파일별 목표

Vision-only Gazebo bring-up:

```bash
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-only \
  --line-mode light_on_dark \
  --video \
  --gcs-ip <windows-gcs-ip>
```

이 단계는 MAVLink control 없이도 GCS에 Gazebo downward camera와 overlay가 뜨는지 확인하기 위한 1차 목표다.

Vision-driven SITL flight:

```bash
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip <windows-gcs-ip>
```

이 단계는 같은 Gazebo frame으로 GCS 송출과 MAVLink velocity 제어를 동시에 수행하는 2차 목표다. 위 `--video`, `--gcs-ip` option은 아직 구현 필요다.

실기체 기존 흐름은 계속 유지한다:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
./build/line_follow_node --config config --target pixhawk1 --vision rpicam
```

## 필요한 파일/디렉토리 수정안

### 1. Gazebo world/course asset 정리

현재 inline world geometry를 별도 course model로 빼는 편이 관리하기 쉽다.

추천 구조:

```text
uav-onboard/sim/gazebo/
├─ README.md
├─ models/
│  ├─ iris_with_downward_camera/
│  │  ├─ model.config
│  │  └─ model.sdf
│  └─ astroquad_vision_course/
│     ├─ model.config
│     ├─ model.sdf
│     └─ materials/
│        └─ textures/
│           ├─ aruco1.png
│           ├─ aruco2.png
│           ├─ aruco3.png
│           ├─ aruco4.png
│           ├─ aruco_id1.png
│           └─ Vmarker.png
└─ worlds/
   ├─ astroquad_iris_vision.sdf
   └─ astroquad_marker_center_fixture.sdf
```

`astroquad_vision_course/model.sdf` 권장 geometry:

- ground: dark plane, at least `20m x 20m`.
- line: white box/plane, `0.10m x 7.0m`, centered on the vehicle forward path.
- marker: ArUco `DICT_4X4_50`, ID `1`, physical size `0.50m x 0.50m`, pose `x=0`, `y=3.0`, `z` slightly above line/ground.
- marker texture: active ID 1 texture는 `/home/mseoky/test_aruco_marker/aruco2.png`를 복사한 `aruco_id1.png`를 사용한다. 여러 marker scenario가 필요해지면 실제 detector ID를 먼저 확인한 뒤 canonical texture name으로 추가한다.
- `Vmarker.png`는 ArUco ID marker가 아니라 추후 vertiport 후보 visual로 분리해 둔다.
- marker가 line 위에 놓이면 line detector에는 marker-sized interruption이 생긴다. 기존 detector에 marker mask가 있으므로 실제 mission 조건과도 맞지만, line-only smoke fixture가 필요하면 marker 없는 world도 하나 두면 좋다.

`astroquad_marker_center_fixture.sdf`를 별도로 두는 이유:

- marker가 3m 전방이면 이륙 직후 하향 camera 시야에 들어오지 않는다.
- detector smoke에서 `markers>=1`을 빠르게 확인하려면 marker를 시작 지점 바로 아래 또는 hover 지점 아래에 둔 fixture가 필요하다.
- 3m course world는 end-to-end flight용, marker-center fixture는 detector regression용으로 분리한다.

현재 `iris_with_downward_camera/model.sdf` 검토:

- camera image는 `640x480`, update `12Hz`, horizontal FOV `2.2rad`.
- 고도 `1.2m` 기준 가로 지면 시야는 대략 `2 * 1.2 * tan(1.1) = 4.7m`다.
- 10cm line은 640px 기준 약 13px, 480px process width 기준 약 10px로 현재 `line.min_line_width_px=8` 근처라 검출 가능 범위다.
- 50cm marker는 640px 기준 약 68px라 ArUco 검출에 충분한 크기다.

### 2. `fly_test.sh`를 Astroquad world 기준으로 변경

현재 home-level `~/fly_test.sh`는 `iris_runway.sdf`를 실행한다. repo 안에 versioned script를 두고, 필요하면 home script가 이를 호출하게 하는 편이 좋다.

추천 파일:

```text
uav-onboard/scripts/fly_test.sh
```

핵심 변경:

```bash
ONBOARD_DIR="$HOME/astroquad/uav-onboard"
ARDUCOPTER_DIR="$HOME/ardupilot/ArduCopter"
ASTROQUAD_WORLD="$ONBOARD_DIR/sim/gazebo/worlds/astroquad_iris_vision.sdf"

export GZ_SIM_SYSTEM_PLUGIN_PATH="$HOME/ardupilot_gazebo/build:${GZ_SIM_SYSTEM_PLUGIN_PATH:-}"
export GZ_SIM_RESOURCE_PATH="$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$ONBOARD_DIR/sim/gazebo/models:$ONBOARD_DIR/sim/gazebo/worlds:${GZ_SIM_RESOURCE_PATH:-}"

gz sim -v4 -r "$ASTROQUAD_WORLD"

sim_vehicle.py \
  -v ArduCopter \
  -f gazebo-iris \
  --model JSON \
  --map \
  --console \
  --out 127.0.0.1:14550 \
  --add-param-file "$INDOOR_PARAM_FILE" \
  --wipe-eeprom
```

주의:

- `--out 127.0.0.1:14550`은 WSL 내부 `line_follow_node`의 MAVLink UDP listen port로 보내는 경로다.
- Windows GCS의 telemetry UDP 14550과 개념이 다르다. 하나는 ArduPilot MAVLink, 다른 하나는 Astroquad JSON telemetry다.
- 두 endpoint 모두 14550을 쓰므로 문서와 script 출력에서 이름을 명확히 구분해야 한다.

### 3. runtime profile/config overlay 확장

현재 `line_follow_node`는 `runtime.<target>.toml` overlay를 읽지만 `vision_debug_node`는 읽지 않는다.

추천:

- `src/common/RuntimeProfileConfig.hpp/.cpp` 또는 동등한 helper를 추가한다.
- `vision_debug_node`와 `line_follow_node`가 모두 `--target sitl|pixhawk1`를 받게 한다.
- `--vision fake|gazebo|rpicam`은 target default를 override한다.
- 기존 `vision_debug_node --config config --line-only --line-mode light_on_dark --video`는 그대로 동작해야 한다.

`config/runtime.sitl.toml` 확장 후보:

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
enabled = false
send_fps = 12
jpeg_quality = 70

[line_follow]
forward_mps = 0.3
duration_s = 15.0

[marker_hover]
hold_s = 3.0
center_tolerance_px = 80.0
```

실제 loader가 `[debug_video]`, `[line_follow]`, `[marker_hover]` overlay까지 읽도록 구현해야 한다.

`config/runtime.pixhawk1.toml`은 계속 다음 의미로 유지한다:

```toml
[runtime]
target = "pixhawk1"
vision = "rpicam"

[transport]
kind = "serial"

[serial]
device = "/dev/serial0"
baudrate = 115200
```

### 4. `vision_debug_node`를 `FrameSource` 기반으로 바꾸기

현재 문제:

- `VisionDebugPipeline`이 `camera::RpicamMjpegSource`를 직접 생성한다.
- Gazebo frame은 BGR image로 들어오며 원본 JPEG가 없다.

권장 변경:

```text
FrameSource
  -> VisionProcessor
  -> TelemetryBuilder
  -> OptionalDebugVideoPublisher
```

구현 방향:

- `VisionDebugPipelineOptions`에 `vision_source` 또는 `FrameSourceFactory`를 추가한다.
- `rpicam` source는 기존 MJPEG bytes를 그대로 video sender에 넣어 재인코딩을 피한다.
- `gazebo` source는 BGR frame을 `cv::imencode(".jpg", ...)`로 JPEG 인코딩해 기존 `AQV1` UDP chunk format으로 보낸다.
- telemetry JSON shape은 v1.7 그대로 둔다.
- `camera.sensor_model`은 `gazebo_downward_camera` 같은 값으로 표시하고, `debug.note`는 `vision_debug_node source=gazebo`처럼 남긴다.
- GCS overlay는 protocol 변화 없이 기존 marker/line metadata로 그대로 동작한다.

### 5. `line_follow_node`에 GCS 관제 publisher 붙이기

현재 문제:

- `line_follow_node --vision gazebo`는 frame을 읽고 `VisionProcessor`를 돌리지만, GCS telemetry/video 송출이 없다.

권장 변경:

- `VisionDebugPipeline` 내부 코드를 그대로 복사하지 않는다.
- telemetry/video 송출부를 `src/app/VisionDebugPublisher.hpp/.cpp` 또는 `src/network/VisionTelemetryPublisher.*` 같은 작은 컴포넌트로 분리한다.
- `vision_debug_node`와 `line_follow_node`가 둘 다 이 publisher를 사용한다.
- `line_follow_node` option 후보:
  - `--gcs-ip <ip>`
  - `--video`
  - `--no-video`
  - `--no-telemetry`
  - `--telemetry-port <n>`
  - `--video-port <n>`
- flight 중 telemetry에는 최소 기존 vision fields를 보내고, 가능하면 optional field로 mission/autopilot 상태를 추가한다.

Protocol v1.8 후보 optional fields:

```json
"mission": {
  "state": "LINE_FOLLOW",
  "elapsed_ms": 12345,
  "landing_reason": ""
},
"autopilot": {
  "mode": "GUIDED",
  "armed": true,
  "heartbeat_seen": true,
  "altitude_m": 1.18,
  "range_m": 1.18
},
"runtime": {
  "target": "sitl",
  "vision_source": "gazebo"
}
```

GCS parser는 unknown fields를 무시하므로 additive field는 안전하지만, 문서 버전은 `docs/PROTOCOL.md` 양쪽을 함께 올려야 한다. 단순 overlay 확인만 목표라면 v1.7 유지로 충분하다.

### 6. Windows GCS / WSL network 운용

권장 1차 실행법은 explicit unicast다.

Windows PowerShell:

```powershell
.\build\uav_gcs_vision_debug.exe --config config
```

WSL에서 Windows host IP 후보 확인:

```bash
ip route | awk '/default/ {print $3; exit}'
```

또는 Windows LAN IPv4를 직접 확인해 사용한다.

WSL onboard 실행 예:

```bash
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"

cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-only \
  --line-mode light_on_dark \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

주의:

- WSL NAT 모드에서는 `127.0.0.1`이 Windows GCS를 의미하지 않을 수 있다.
- GCS discovery broadcast가 WSL까지 항상 들어온다고 가정하지 않는다.
- Windows Defender Firewall에서 `uav_gcs_vision_debug.exe` inbound UDP 14550/5600 허용이 필요할 수 있다.

## 추천 구현 순서

1. Gazebo world/course 정리
   - line 폭 10cm.
   - marker 3m 전방, 50cm x 50cm.
   - 단일 marker fixture는 검증된 `aruco_id1.png`를 ID 1 marker texture로 사용.
   - `/home/mseoky/test_aruco_marker`의 원본 PNG는 실제 detector ID를 확인한 뒤 다중 marker scenario용 후보로 보관.
   - `Vmarker.png`는 추후 vertiport visual asset 후보로 보관.
   - marker-center smoke fixture 추가.

2. `fly_test.sh`를 Astroquad world 기준으로 정리
   - repo script 추가.
   - home-level script는 repo script 호출 또는 같은 내용으로 갱신.
   - `GZ_SIM_RESOURCE_PATH`, `GZ_SIM_SYSTEM_PLUGIN_PATH`를 script에서 설정.

3. `vision_debug_node --target sitl --vision gazebo` 구현
   - GCS에 Gazebo downward camera MJPEG 송출.
   - 기존 line/marker telemetry 송출.
   - 이 단계에서 MAVLink 제어는 건드리지 않는다.

4. Gazebo vision-only 검증
   - GCS camera window에 top-down Gazebo view 표시.
   - line overlay 확인.
   - marker-center fixture에서 `markers>=1` 확인.
   - 3m course에서 manual/GUIDED takeoff 후 marker가 시야에 들어오는지 확인.

5. `line_follow_node`에 GCS publisher 연결
   - 같은 frame/result를 GCS로 송출하면서 MAVLink 제어.
   - SITL profile에서 line-follow duration을 12-15초로 확장.
   - marker hover end-to-end 검증.

6. Protocol v1.8 여부 결정
   - 단순 vision overlay는 v1.7 유지.
   - GCS에서 mode/armed/altitude/mission state까지 보려면 양쪽 `docs/PROTOCOL.md`를 v1.8로 갱신.

## 검증 체크리스트

Build/test:

```bash
cd ~/astroquad/uav-onboard
cmake --build build
ctest --test-dir build --output-on-failure
```

Gazebo world load:

```bash
cd ~/astroquad/uav-onboard
GZ_SIM_SYSTEM_PLUGIN_PATH="$HOME/ardupilot_gazebo/build:${GZ_SIM_SYSTEM_PLUGIN_PATH:-}" \
GZ_SIM_RESOURCE_PATH="$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$PWD/sim/gazebo/models:$PWD/sim/gazebo/worlds:${GZ_SIM_RESOURCE_PATH:-}" \
gz sim -s -v4 -r sim/gazebo/worlds/astroquad_iris_vision.sdf
```

Gazebo topic 확인:

```bash
gz topic -l | grep downward_camera
```

Vision smoke:

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
markers>=1   # marker-center fixture 또는 marker가 camera FOV 안에 있을 때
```

Windows GCS end-to-end:

```text
GCS window: raw/top-down Gazebo camera frame 표시
GCS overlay: magenta line contour, red/green line center offset 표시
GCS log: TELEMETRY seq 증가, line fields 갱신
```

Flight smoke:

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

## 참고 자료

로컬 문서:

- `development-log/SYSTEM_SPEC.md`
- `development-log/MVP_PLAN.md`
- `uav-gcs/PROJECT_SPEC.md`
- `uav-onboard/PROJECT_SPEC.md`
- `uav-gcs/docs/PROTOCOL.md`
- `uav-onboard/sim/gazebo/README.md`
- `uav-onboard/README.md`
- `uav-gcs/README.md`

공식/상위 프로젝트 문서:

- ArduPilot Gazebo SITL: https://ardupilot.org/dev/docs/sitl-with-gazebo.html
- ArduPilot Gazebo plugin repo/README: https://github.com/ArduPilot/ardupilot_gazebo
- WSL networking: https://learn.microsoft.com/windows/wsl/networking
- OpenCV ArUco detection/generation: https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html
- SDFormat camera sensor spec: http://sdformat.org/spec?ver=1.9&elem=sensor

## 결론

다음 구현 스텝은 **Gazebo vision-only GCS end-to-end**로 잡는 것이 가장 좋다.

즉, 먼저 드론 제어를 붙이기 전에 다음 상태를 만든다:

```text
Windows uav_gcs_vision_debug
  <- WSL vision_debug_node --target sitl --vision gazebo --video
  <- Gazebo downward camera image
  <- 기존 onboard vision telemetry
```

이게 안정되면 `line_follow_node`에 같은 publisher를 붙여, GCS에서 top-down camera/overlay를 보면서 SITL Iris line-follow와 3m marker hover/land를 검증한다.
