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

## Follow-up Research: OpenCV, SITL, GCS Vision

사용자 질문 기준 추가 조사 결과:

1. WSL Ubuntu의 OpenCV 버전을 onboard vision 코드에 맞춰야 한다.
2. `fly_test.sh` 실행 후 `line_follow_node`를 빌드/실행하면 Gazebo에서 움직임을 볼 수 있는지 확인해야 한다.
3. 이후 Raspberry Pi 4에서 `line_follow_node`를 실행할 때도 기존 `vision_debug_node`처럼 GCS로 vision 정보를 송신할 수 있는지 확인해야 한다.

### OpenCV ArUco API Compatibility

현재 WSL Ubuntu에 설치된 OpenCV는 apt 기준 `4.5.4+dfsg-9ubuntu4`이다.

확인 결과:

```text
pkg-config --modversion opencv4 -> 4.5.4
opencv_version -> 4.5.4
python cv2.__version__ -> 4.5.4
python cv2.aruco.ArucoDetector -> 없음
python cv2.aruco.generateImageMarker -> 없음
```

현재 onboard vision 코드의 `src/vision/ArucoDetector.cpp`는 다음 modern ArUco API를 사용한다.

```text
cv::aruco::PredefinedDictionaryType
cv::aruco::generateImageMarker
cv::aruco::ArucoDetector
cv::aruco::DetectorParameters
```

WSL의 `/usr/include/opencv4/opencv2/aruco.hpp`에는 구 API인 `PREDEFINED_DICTIONARY_NAME`, `drawMarker`, free function 기반 `detectMarkers`가 존재하지만, 현재 코드가 사용하는 `ArucoDetector`, `generateImageMarker`, `PredefinedDictionaryType`는 없다. 따라서 `BUILD_TOOLS=ON`에서 `onboard_vision` target을 빌드하면 다음 계열의 에러가 발생한다.

```text
cv::aruco::PredefinedDictionaryType does not name a type
cv::aruco::generateImageMarker is not a member of cv::aruco
cv::aruco::ArucoDetector has not been declared
```

`scripts/setup_rpi_dependencies.sh` 확인 결과:

- Raspberry Pi용 setup script는 `libopencv-dev`와 가능한 경우 `libopencv-contrib-dev`를 apt로 설치한다.
- OpenCV version을 명시적으로 pin하지 않는다.
- 따라서 Pi에서 실제로 설치됐던 OpenCV version은 당시 Raspberry Pi OS/Debian suite와 apt repository 상태에 의해 결정된다.
- 같은 script라도 Bullseye/Bookworm/Trixie 기반 여부에 따라 다른 OpenCV가 설치될 수 있다.

참고 가능한 Debian apt package 기준:

```text
Debian Bullseye libopencv-dev -> 4.5.1+dfsg-5
Debian Bookworm libopencv-dev -> 4.6.0+dfsg-12
Debian Trixie libopencv-dev -> 4.10.0+dfsg-5
Ubuntu Jammy libopencv-dev -> 4.5.4+dfsg-9ubuntu4
```

따라서 지금 Pi가 없으면 과거 Pi의 정확한 OpenCV version을 확정할 수는 없다. 다만 현재 목표에서는 Pi version 자체가 반드시 필요한 것은 아니다. 더 중요한 기준은 **현재 onboard vision 코드가 요구하는 ArUco API를 만족하는 개발환경**이다.

결론:

- Raspberry Pi 4에서 현재 vision 코드가 정상 동작하므로, 코드를 OpenCV 4.5.4 구 API에 맞춰 되돌리면 안 된다.
- WSL 개발환경의 OpenCV를 onboard 코드가 요구하는 API에 맞추는 방향이 맞다.
- Ubuntu Jammy apt 기본 저장소는 OpenCV 4.5.4까지만 제공하므로 apt upgrade만으로는 해결되지 않는다.
- Pi가 없더라도 WSL에는 OpenCV와 opencv_contrib를 source build로 설치해 현재 onboard vision 코드가 요구하는 API를 만족시키면 된다.
- 현재 코드 기준으로는 OpenCV `4.10.x`를 개발환경 target으로 삼는 것이 현실적이다. Debian Trixie의 apt package도 `4.10.0` 계열이고, 현재 코드가 쓰는 modern ArUco API를 만족한다.

현재 선택한 환경 구성:

```text
/usr/local/bin/opencv_version
/usr/local/include/opencv4
/usr/local/lib/libopencv_*.so
/usr/local/lib/cmake/opencv4/OpenCVConfig.cmake
/usr/local/lib/pkgconfig/opencv4.pc
```

빌드 시에는 별도 `OpenCV_DIR`를 주지 않아도 CMake가 `/usr/local`의 OpenCV 4.10을 기본으로 찾는 상태를 목표로 한다.

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_TESTS=ON \
  -DBUILD_TOOLS=ON
```

런타임에서도 `/usr/local/lib`의 OpenCV shared library가 잡히도록 `ldconfig` 상태를 확인한다.

```bash
sudo ldconfig
ldconfig -p | rg 'libopencv_(core|objdetect|imgproc|videoio)'
```

주의:

- apt OpenCV header와 custom OpenCV library가 섞이면 더 위험한 빌드/런타임 문제가 생길 수 있다.
- 지금은 Pi가 없으므로 Pi version 확인은 blocker가 아니다.
- 추후 Pi에 다시 접근 가능해지면 동일한 API check만 수행하면 된다.
- Pi의 동작 중인 vision 코드를 기준으로 삼고, 소스 코드는 바꾸지 않는다.

Pi에서 확인할 명령:

```bash
opencv_version
pkg-config --modversion opencv4
python3 - <<'PY'
import cv2
print(cv2.__version__)
print(hasattr(cv2.aruco, "ArucoDetector"))
print(hasattr(cv2.aruco, "generateImageMarker"))
PY
```

참고한 공식 문서:

- OpenCV 4.x ArUco group: <https://docs.opencv.org/4.x/de/d67/group__objdetect__aruco.html>
- OpenCV `cv::aruco::ArucoDetector`: <https://docs.opencv.org/4.x/d2/d1a/classcv_1_1aruco_1_1ArucoDetector.html>
- Debian Bookworm `libopencv-dev`: <https://packages.debian.org/bookworm/libopencv-dev>
- Debian Bullseye `libopencv-dev`: <https://packages.debian.org/bullseye/libopencv-dev>
- Debian Trixie `libopencv-dev`: <https://packages.debian.org/trixie/libopencv-dev>

### Gazebo Visibility After Running line_follow_node

다음 스텝에서 `fly_test.sh`로 Gazebo + ArduCopter SITL을 띄운 뒤, 별도 터미널에서 `line_follow_node`를 빌드/실행하면 Gazebo 상에서 기체 움직임을 확인할 수 있다.

단, 현재 의미는 다음과 같다.

- Gazebo의 기체는 현재 `fly_test.sh`가 띄우는 Iris 모델이다.
- `line_follow_node`는 현재 fake line input을 사용한다.
- 따라서 Gazebo 화면에서는 takeoff, 짧은 guided velocity 이동, land 같은 기체 motion을 볼 수 있다.
- 아직 Gazebo 카메라 이미지에서 라인을 검출해서 따라가는 end-to-end vision simulation은 아니다.
- `takeoff_move_land.cpp`와 `line_follow_node`를 동시에 같은 MAVLink 포트에 붙여 실행하면 안 된다.

OpenCV 문제가 해결되기 전에도 SITL smoke test만 하려면 `BUILD_TOOLS=OFF`로 `line_follow_node`를 빌드할 수 있다.

```bash
cd ~/astroquad/uav-onboard
cmake -S . -B build-sitl \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_TESTS=ON \
  -DBUILD_TOOLS=OFF
cmake --build build-sitl --target line_follow_node
./build-sitl/line_follow_node --config config --target sitl
```

실행 순서:

```text
Terminal 1: ./fly_test.sh
Terminal 2: ./build-sitl/line_follow_node --config config --target sitl
```

`fly_test.sh`가 foreground에서 Gazebo/MAVProxy session을 점유하므로 `line_follow_node`는 다른 터미널에서 실행하는 것이 좋다.

### Future GCS Vision Telemetry From line_follow_node

추후 Raspberry Pi 4에서 `line_follow_node`를 실제 실행할 때, 기존 `vision_debug_node`처럼 GCS로 vision 정보를 송신하는 것은 가능하다.

현재 이미 재사용 가능한 구성요소가 있다.

```text
src/net/UdpTelemetrySender.*
src/net/UdpMjpegStreamer.*
src/protocol/TelemetryMessage.*
src/vision/VisionDebugPipeline.*
```

현실적인 구현 방향:

1. `VisionDebugPipeline`에만 묶여 있는 camera/vision 처리 흐름을 재사용 가능한 vision pipeline으로 분리한다.
2. `line_follow_node` 또는 이후의 onboard main runtime에서 같은 vision 결과를 mission/control 입력과 GCS telemetry 입력으로 동시에 전달한다.
3. GCS telemetry schema에 `mission_state`, `autopilot_mode`, `armed`, `altitude_or_range`, `safety_event` 같은 flight 상태를 추가한다.
4. GCS 쪽 parser/UI도 새 필드를 표시하도록 맞춘다.
5. debug video는 기존 `UdpMjpegStreamer` 경로를 재사용한다.

중요한 제약:

- Pi의 CSI camera는 `vision_debug_node`와 `line_follow_node`가 동시에 열면 충돌할 수 있다.
- 최종 구조는 두 프로세스를 동시에 실행하는 방식이 아니라, 하나의 onboard runtime 안에서 camera/vision, mission/control, telemetry/video publish를 함께 구성하는 방식이 적절하다.
- Vision module은 계속 perception result만 산출해야 하며 MAVLink 제어 명령을 직접 내리면 안 된다.
- Mission/control/autopilot boundary를 유지해야 ROS 철학과 현재 architecture goal에 맞는다.

## Next Recommended Step

추천 순서:

1. `AutopilotMavlinkAdapter::poll()`이 한 주기 안에서 여러 MAVLink message를 drain하도록 조정한다.
2. SITL profile과 Pixhawk1 profile의 heartbeat timeout을 분리해 SITL false-positive abort를 줄인다.
3. altitude source 우선순위와 mode confirmation log를 smoke test 기준으로 정리한다.
4. `VisionDebugPipeline`을 재사용 가능한 vision pipeline으로 분리하고, `line_follow_node` 계열 runtime에서 GCS vision telemetry/video publish를 함께 수행하도록 계획한다.
5. Gazebo world에 line/ArUco marker를 배치하고, Iris top-down camera image를 onboard vision pipeline 입력으로 연결한다.
6. 그 다음에 serial transport를 구현해 `--target pixhawk1`을 props-off Pixhawk bench test까지 연결한다.

바로 다음 한 스텝으로는 **MAVLink receive loop의 heartbeat stale false-positive를 없애는 것**을 추천한다. OpenCV 4.10 개발환경과 `BUILD_TOOLS=ON` 전체 빌드는 완료됐으므로, 이제 SITL motion smoke의 신뢰도를 높이는 쪽이 다음 병목이다.

## Execution Update: Astroquad SITL Smoke

2026-05-14 추가 실행 결과:

### OpenCV Environment

Baseline:

```text
OS: Ubuntu 22.04.5 LTS (jammy)
cmake: 3.22.1
g++: 11.4.0
opencv_version: 4.5.4
pkg-config opencv4: 4.5.4
OpenCVConfig.cmake: /usr/lib/x86_64-linux-gnu/cmake/opencv4/OpenCVConfig.cmake
opencv4.pc: /usr/lib/x86_64-linux-gnu/pkgconfig/opencv4.pc
```

추가 실행 결과:

```text
/usr/local/bin/opencv_version -> 4.10.0
pkg-config --modversion opencv4 -> 4.10.0
pkg-config --cflags opencv4 -> -I/usr/local/include/opencv4
OpenCV_DIR -> /usr/local/lib/cmake/opencv4
opencv2/aruco.hpp -> /usr/local/include/opencv4/opencv2/aruco.hpp
```

`/usr/local`에 OpenCV/opencv_contrib `4.10.0` source build를 설치했고, `libopencv-dev`, `libopencv-contrib-dev`, `python3-opencv` apt package는 제거했다. apt OpenCV runtime package 일부는 남아 있을 수 있지만, CMake/pkg-config/runtime linker 확인 결과 Astroquad C++ build는 `/usr/local` OpenCV를 사용한다.

설치된 OpenCV 4.10 header에서 현재 onboard vision 코드가 요구하는 API를 확인했다.

```text
cv::aruco::ArucoDetector
cv::aruco::generateImageMarker
cv::aruco::PredefinedDictionaryType
```

결론:

- OpenCV 4.10 기본 C++ 개발환경 전환은 완료됐다.
- 일반 build 절차에서 별도 `OpenCV_DIR` 또는 `LD_LIBRARY_PATH` 지정이 필요 없다.
- `BUILD_TOOLS=ON` 전체 vision build가 정상화됐다.

### Build/Test

초기에는 OpenCV와 무관한 SITL 제어 노드 검증을 위해 `BUILD_TOOLS=OFF`로 빌드했다.

```bash
cd ~/astroquad/uav-onboard
cmake -S . -B build-sitl -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DBUILD_TOOLS=OFF
cmake --build build-sitl
ctest --test-dir build-sitl --output-on-failure
```

결과:

```text
line_follow_node build 성공
6/6 tests passed
```

OpenCV 4.10 설치 후 `BUILD_TOOLS=ON` 전체 빌드를 재확인했다.

```bash
cmake -S . -B build-opencv410 -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DBUILD_TOOLS=ON
cmake --build build-opencv410
ctest --test-dir build-opencv410 --output-on-failure
```

결과:

```text
line_follow_node build 성공
onboard_vision build 성공
camera_preview build 성공
aruco_detector_tester build 성공
line_detector_tuner build 성공
grid_image_smoke build 성공
marker_grid_replay build 성공
vision_debug_node build 성공
replay_vision build 성공
mock_gcs_command build 성공
video_streamer build 성공
7/7 tests passed
```

링크 확인:

```text
OpenCV_DIR: /usr/local/lib/cmake/opencv4
vision_debug_node -> /usr/local/lib/libopencv_*.so.410
aruco_detector_tester -> /usr/local/lib/libopencv_*.so.410
```

### SITL Smoke

`fly_test.sh`는 Gazebo + ArduCopter SITL launcher로만 사용하고, 제어는 Astroquad `line_follow_node`만 사용했다. `takeoff_move_land.cpp`는 실행하지 않았다.

첫 실행:

```text
[mavlink] heartbeat ok system=1 component=1 mode=STABILIZE
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF target=1.2m
[mission] LINE_FOLLOW
[mission] ABORT reason=pixhawk heartbeat lost
[mission] COMPLETE
```

의미:

- SITL heartbeat 연결, GUIDED mode, arm, takeoff, line-follow 진입까지 성공했다.
- 기본 `pixhawk_heartbeat_lost_ms = 2000`은 현재 SITL message stream과 adapter polling 방식에 비해 너무 짧다.
- `AutopilotMavlinkAdapter::poll()`이 한 번 호출될 때 한 메시지만 처리하므로, telemetry stream이 많을 때 heartbeat age가 stale처럼 보일 수 있다.
- 이것은 Gazebo에서 사용자가 heartbeat를 수동 설정해야 한다는 의미가 아니다. ArduPilot/MAVProxy는 heartbeat를 내보냈고, onboard adapter가 다른 telemetry message에 밀려 heartbeat cache를 제때 갱신하지 못한 쪽에 가깝다.

두 번째 실행:

- repo config는 수정하지 않았다.
- `/tmp/astroquad-sitl-config/safety.toml`에서만 `pixhawk_heartbeat_lost_ms = 10000`으로 임시 override했다.

결과:

```text
[mavlink] heartbeat ok system=1 component=1 mode=LAND
[mavlink] GUIDED confirmed
[mavlink] armed
[mission] TAKEOFF target=1.2m
[mission] LINE_FOLLOW
[mission] LAND reason=duration complete
[mission] COMPLETE
```

의미:

- Astroquad `line_follow_node`만으로 Gazebo/ArduCopter SITL에서 `GUIDED -> arm -> takeoff -> short line-follow velocity -> LAND -> COMPLETE` smoke sequence를 통과했다.
- 현재 smoke는 fake line input 기반이며, Gazebo camera 기반 line detection은 아직 아니다.

### Immediate Technical Follow-Up

다음 구현 후보:

1. `AutopilotMavlinkAdapter::poll()`이 제한 시간 내 여러 MAVLink message를 drain하도록 조정한다.
2. SITL profile의 heartbeat timeout을 기본 2초보다 넉넉하게 분리한다.
3. `fly_test.sh` 안내 문구를 Astroquad SITL launcher 기준으로 정리한다.
4. Gazebo camera 기반 line/ArUco end-to-end simulation 계획을 별도 구현 스텝으로 분리한다.
