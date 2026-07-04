# Astroquad 트러블슈팅 및 개발 판단 로그

최종 업데이트: 2026-05-21

범위: `uav-gcs`, `uav-onboard` bring-up 과정에서 실제로 발생한 문제, 원인 분석, 해결 방법, 설계 판단을 보고서용 개발로그로 정리한다. 현재 기본 장치는 Raspberry Pi 4 + IMX519-78이며, Raspberry Pi Zero 2 W 관련 내용은 이전 bring-up 단계의 이력으로 남긴다.

## 0. 현재 기준선: `grid_mission_node`와 grid arena snake mission

### 현재 실행 기준

최근 grid mission 개발의 기준 명령은 다음이다.

```bash
bash ~/astroquad/uav-onboard/scripts/grid_arena_test.sh

./build/grid_mission_node --config config --target sitl --vision gazebo \
  --world grid --line-mode dark_on_light --marker-count 4 \
  --video --gcs-ip <windows-gcs-ip>
```

`--world grid`는 `config/runtime.sitl.grid.toml`을 사용해
`grid_arena_test_world` Gazebo camera topic을 선택한다. 수동 topic override는
custom fixture 또는 임시 world 테스트 때만 필요하다.

### 현재 state machine

`grid_mission_node`는 `VisionProcessor`, `IntersectionDecisionEngine`,
`GridCoordinateTracker`, `GridMission`, `SnakePlanner`, `GridControlMapper`,
`AutopilotMavlinkAdapter`, `VisionDebugPublisher`를 묶는 SITL staging
composition root다.

현재 grid mission 흐름:

```text
ARM_TAKEOFF
  -> MARKER_LOCK_YAW
  -> ENTRY_FORWARD
  -> ENTRY_CENTER_ORIGIN
  -> SNAKE_LAUNCH_ALIGN
  -> SNAKE_FORWARD
  -> SNAKE_RECORD_NODE
  -> SNAKE_STOP_AT_CENTER
  -> SNAKE_TURN_90
  -> SNAKE_ADVANCE_ONE_CELL
  -> SNAKE_TURN_90_AGAIN
  -> SNAKE_COMPLETE
  -> LAND
```

새 grid arena에는 vertiport에서 grid까지 이어지는 line이 없으므로
`ENTRY_FORWARD`는 yaw-frozen blind forward다. 첫 L/T/+ 교차점은 보는 즉시
origin으로 확정하지 않고, `ENTRY_CENTER_ORIGIN`에서 X/Y 중심과 저속 gate를
통과한 뒤 local `(0,0)`으로 publish한다.

### 현재 알고리즘 판단

- Gazebo ground-truth pose를 mission 입력으로 쓰지 않는다.
- LOCAL_NED는 짧은 hop 거리와 hover source로만 사용한다.
- `GridCoordinateTracker::update()`는 peek-only이며, mission gate 통과 후
  `commitAdvance()`만 실제 좌표를 전진시킨다.
- `grid_mission_node`는 GCS UDP loss에 대비해 최신 committed `grid_node`를
  매 frame 다시 보낸다. GCS는 node id/coordinate로 dedup한다.
- Cell 사이 이동은 yaw locked `ForwardBlind`가 기본이고, line following은
  `hop_align_start_m..hop_align_end_m`의 짧은 mid-cell alignment window에서만
  열린다.
- Boundary에서는 `SnakePlanner`가 첫 turn direction을 latch하고 이후
  left/right를 strict alternation한다. 기대 branch가 없으면 backtrack하지
  않고 snake complete로 본다.
- Marker commit은 sliding `MarkerWindow`가 같은 non-vertiport ID를 충분히
  관찰했을 때만 수행한다. window 안에 서로 다른 ID가 섞이면 flush한다.

### 현재 남은 범위

- `grid_mission_node --target pixhawk1` real arm/takeoff는 열려 있지 않다.
  현재는 `--no-arm` smoke만 허용한다.
- Marker ID 역순 재방문, 출발점 복귀, official coordinate conversion은 아직
  미구현이다.
- GCS protocol에는 richer mission object가 준비되어 있지만,
  `VisionDebugPublisher` path는 아직 grid mission state를 구조화해서 채우지
  않는다. 현재 상세 state 확인은 `grid_mission_node` console log가 기준이다.

## 1. GCS가 온보드 telemetry를 수신하지 못함

### 증상

Raspberry Pi에서 `uav_onboard`를 실행하면 다음처럼 telemetry가 정상 송신되는 것처럼 보였다.

```text
sent TELEMETRY seq=1 timestamp_ms=...
sent TELEMETRY seq=2 timestamp_ms=...
...
```

하지만 Windows GCS에서는 다음 메시지만 반복되었다.

```text
telemetry timeout after 2000 ms
```

### 원인

초기 `uav-onboard/config/network.toml`의 GCS 목적지 IP가 고정 IP로 설정되어 있었다.

```toml
[gcs]
ip = "192.168.1.100"
```

실제 노트북 IP는 네트워크마다 바뀌기 때문에, Pi가 존재하지 않거나 잘못된 IP로 UDP telemetry를 보내고 있었다.

### 해결

기본 목적지를 local broadcast로 변경했다.

```toml
[gcs]
ip = "255.255.255.255"
telemetry_port = 14550
video_port = 5600
```

또한 UDP sender에서 broadcast 송신을 허용하도록 `SO_BROADCAST`를 설정했다.

### 결과

GCS를 먼저 실행하고 Pi에서 `uav_onboard --config config --count 10`을 실행하면 GCS에서 증가하는 `seq`를 가진 telemetry를 수신할 수 있게 되었다.

## 2. Windows에서 OpenCV가 없어 `uav_gcs_video`가 빌드되지 않음

### 증상

Windows 로컬 빌드에서 다음 CMake warning이 발생했다.

```text
OpenCV was not found; uav_gcs_video will not be built.
```

그 결과 영상 수신 실행 파일이 생성되지 않았다.

### 원인

개발 환경은 MinGW/Ninja/g++ 조합이었다. OpenCV가 설치되어 있지 않았고, Windows에서 OpenCV를 설치하더라도 Visual Studio용 prebuilt OpenCV와 MinGW ABI가 맞지 않을 수 있었다.

### 해결

GCS 영상 창 backend를 두 경로로 분리했다.

- OpenCV가 있으면 `src/ui/VideoWindow.cpp` 사용
- Windows에서 OpenCV가 없으면 `src/ui/VideoWindowWin32.cpp` 사용

Win32 fallback backend는 Windows Imaging Component(WIC)로 JPEG를 decode하고, Win32/GDI로 화면을 그린다.

### 결과

OpenCV가 없는 Windows 환경에서도 `uav_gcs_video.exe`와 `uav_gcs_vision_debug.exe`를 빌드할 수 있게 되었다.

## 3. Pi에서 `video_streamer --source rpicam`이 시작되지 않음

### 증상

Pi에서 카메라는 인식되었다.

```text
rpicam-hello --list-cameras
0 : ov5647 ...
```

하지만 `video_streamer` 실행 시 다음 오류가 발생했다.

```text
failed to open rpicam source: failed to start rpicam-vid
```

### 원인

`RpicamMjpegSource`가 Linux에서도 `popen(command, "rb")`를 사용하고 있었다. Windows `_popen`에서는 `"rb"`가 자연스럽지만, POSIX `popen()`은 `"r"` 또는 `"w"`만 보장한다.

Pi에서 간단한 확인 결과 `"rb"` 모드가 invalid argument로 실패하는 것을 확인했다.

### 해결

플랫폼별 pipe read mode를 분리했다.

- Windows: `"rb"`
- Linux/POSIX: `"r"`

또한 `rpicam-vid`의 stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리하고, `--verbose 0`을 추가했다.

### 결과

Pi에서 `rpicam-vid` 기반 MJPEG frame을 읽고 UDP로 송신할 수 있게 되었다.

```text
sent video frame id=1 bytes=14952
sent video frame id=2 bytes=15038
sent video frame id=3 bytes=14999
```

## 4. `--count` 테스트 종료 후 `rpicam-vid` abort 로그가 보임

### 증상

`video_streamer --source rpicam --count 5`는 지정된 frame 수를 정상 송신했지만, `rpicam-vid`가 다음과 비슷한 로그를 남겼다.

```text
Received signal 13
terminate called after throwing an instance of 'std::runtime_error'
what(): failed to write output bytes
Aborted
```

### 원인

`video_streamer`가 지정된 frame 수를 읽은 뒤 pipe를 닫으면, stdout으로 MJPEG를 계속 쓰던 `rpicam-vid`가 SIGPIPE를 받고 종료한다. 이는 송신 실패라기보다는 테스트 종료 방식의 부작용이다.

### 해결

사용자 콘솔이 혼란스럽지 않도록 `rpicam-vid` stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리했다.

### 결과

터미널에는 송신 결과 중심의 로그만 보이게 되었다. `/tmp/astroquad_rpicam_vid.log`에는 SIGPIPE 종료 기록이 남을 수 있지만, `video_streamer`가 정상 종료하고 `sent video frame ...`이 출력되면 정상 테스트로 본다.

## 5. Broadcast 영상 스트리밍이 불안정함

### 증상

노트북 IP를 직접 지정하면 영상 수신이 안정적이었다.

```bash
./build/video_streamer --source rpicam --config config --gcs-ip 172.20.10.4
```

하지만 `255.255.255.255` broadcast로 영상 frame 자체를 보내면 frame 누락이 많거나 영상이 불안정했다.

### 원인

640x480 MJPEG frame은 대략 14-31 KB 수준이고, UDP payload를 1200 byte로 제한했기 때문에 frame 하나가 여러 UDP packet으로 분할된다. 이 중 packet 하나만 유실되어도 해당 JPEG frame은 완성되지 않는다.

Wi-Fi 환경에서는 broadcast packet이 낮은 전송률로 처리되거나 손실될 수 있다.

실제 측정 결과:

| 목적지 | 송신 frame | 완성 수신 frame | 완성률 |
|---|---:|---:|---:|
| `255.255.255.255` broadcast | 10 | 4 | 40% |
| 노트북 IP unicast | 10 | 10 | 100% |

### 해결

영상 본문은 broadcast로 계속 보내지 않고, GCS discovery 후 unicast로 전송하도록 변경했다.

동작 방식:

1. GCS가 UDP `5601`로 `AQGCS1 video_port=5600` beacon을 broadcast한다.
2. 온보드 `video_streamer` 또는 `vision_debug_node`가 시작 시 3초간 beacon을 기다린다.
3. beacon을 받으면 송신자 IP를 GCS IP로 사용한다.
4. 실제 영상은 해당 IP로 unicast 송신한다.
5. discovery 실패 시 기존 broadcast 주소로 fallback한다.

### 결과

Pi에서 매번 노트북 IP를 직접 입력하지 않아도 자동으로 GCS IP를 찾고, 실제 영상은 unicast로 보내게 되었다.

```text
discovering GCS video receiver for 3000 ms...
discovered GCS video receiver at <laptop-ip>:5600
```

## 6. GCS 영상 창이 중간중간 검은색으로 깜빡임

### 증상

영상이 수신되기는 하지만 GCS 영상 창이 중간중간 검은 화면으로 깜빡였다.

### 원인

두 가지가 겹쳤다.

1. 일정 시간 complete frame을 받지 못하면 `showStatus("waiting for video stream...")`가 호출되어 기존 frame 대신 검은 status 화면이 표시되었다.
2. Windows fallback 창에서 GDI paint 과정 중 배경 erase와 frame redraw가 분리되어 flicker가 보일 수 있었다.

UDP 특성상 incomplete frame drop은 정상적으로 발생할 수 있으므로, frame 하나가 빠질 때마다 검은 화면으로 돌아가면 사용자 경험이 나빠진다.

### 해결

GCS 표시 정책을 변경했다.

- 첫 frame을 받기 전에는 waiting 화면을 표시한다.
- 한 번이라도 frame을 받은 뒤에는 timeout이 발생해도 마지막 complete frame을 유지한다.
- JPEG decode 실패 시 화면을 지우지 않고 stderr warning만 출력한다.
- Win32 fallback backend에서 memory DC에 먼저 그리고 `BitBlt`로 한 번에 복사하는 double buffering을 적용했다.
- `WM_ERASEBKGND`를 처리해 배경 erase로 인한 flicker를 줄였다.

### 결과

일시적인 UDP frame drop이 있어도 마지막 정상 frame이 유지되므로 검은색 깜빡임이 줄었다.

## 7. OpenCV `camera_preview`와 Pi CSI camera 경로 차이

### 증상

초기 OpenCV `VideoCapture` 기반 `camera_preview`에서 frame read가 실패했다.

```text
frame read failed at index 0: failed to read a non-empty camera frame
```

### 원인

Pi CSI camera는 Raspberry Pi OS/libcamera 환경에서 OpenCV V4L2 경로로 항상 안정적으로 열리는 것이 아니다. 반면 `rpicam-hello`, `rpicam-vid`, `rpicam-still`은 libcamera/rpicam 경로를 사용하므로 정상 동작했다.

### 해결

실시간 Pi camera 경로는 OpenCV `VideoCapture`가 아니라 `rpicam-vid --codec mjpeg -o -` stdout을 읽는 방식으로 확정했다.

### 결과

`RpicamMjpegSource` 기반 `video_streamer`와 `vision_debug_node`가 Pi camera frame을 안정적으로 읽는다.

## 8. ArUco detector 빌드 시 OpenCV enum 이름 차이 발생

### 증상

로컬에서는 OpenCV가 없어 해당 target이 skip되었지만, Pi에서 빌드할 때 다음 오류가 발생했다.

```text
error: 'PREDEFINED_DICTIONARY_NAME' in namespace 'cv::aruco' does not name a type
```

### 원인

Pi에 설치된 OpenCV 4.10.0 header에서는 ArUco dictionary enum type이 `cv::aruco::PredefinedDictionaryType`이었다. 코드에서는 다른 OpenCV 버전에서 쓰이는 이름인 `PREDEFINED_DICTIONARY_NAME`을 사용하고 있었다.

### 해결

`ArucoDetector.cpp`의 dictionary mapping type을 Pi OpenCV 4.10.0에 맞게 수정했다.

```cpp
cv::aruco::PredefinedDictionaryType dictionaryFromName(const std::string& name)
```

### 결과

Pi에서 `onboard_vision`, `aruco_detector_tester`, `vision_debug_node`가 모두 빌드되었다.

## 9. `uav_gcs_vision_debug.exe`가 packet을 받지 못함

### 증상

Pi는 GCS beacon을 정상 발견했다.

```text
discovered GCS video receiver at 192.168.0.69:5600
```

하지만 `uav_gcs_vision_debug` 콘솔에는 telemetry packet이 계속 0개로 표시되었다.

```text
[marker] no telemetry packets yet packets=0 dropped=0
```

### 원인

Windows Defender Firewall에 `uav_gcs_vision_debug.exe` inbound rule이 생성되어 있었지만, Action이 `Block`이었다. 기존 `uav_gcs.exe`, `uav_gcs_video.exe`는 Allow였기 때문에 새 실행 파일만 막힌 상태였다.

### 해결

관리자 권한에서 `uav_gcs_vision_debug.exe` inbound UDP rule을 Allow로 변경해야 한다.

예시:

```powershell
Set-NetFirewallRule -DisplayName 'uav_gcs_vision_debug.exe' -Direction Inbound -Action Allow -Profile Private,Public
```

또는 Windows 보안 앱에서 해당 실행 파일의 Public/Private network 접근을 허용한다.

### 결과

방화벽이 허용된 실행 파일 경로로 테스트했을 때 GCS가 Pi에서 보낸 telemetry 20개를 모두 수신했다.

## 10. ArUco 오버레이를 온보드에서 그릴지 GCS에서 그릴지에 대한 설계 판단

### 고민 배경

ArUco marker 인식 후 결과를 영상 위에 표시하는 방법은 두 가지가 있었다.

1. 온보드에서 marker box, id, 방향 등을 영상에 직접 그린 뒤 그 결과 영상을 GCS로 송신한다.
2. 온보드는 marker id/corners/center/orientation만 계산해 telemetry로 보내고, GCS가 원본 영상 위에 overlay를 그린다.

온보드에서 바로 그려서 보내는 방식은 구현이 간단해 보였다. 특히 디버깅 단계에서는 `cv::aruco::drawDetectedMarkers`나 `cv::line`, `cv::putText`를 사용하면 빠르게 화면을 확인할 수 있다.

하지만 프로젝트 최종 목표와 Raspberry Pi Zero 2 W급 저사양 환경을 고려하면 단순 구현 편의성보다 역할 분리가 중요했다.

### 온보드에서 오버레이를 그리는 방식의 장점

- 구현이 직관적이다.
- GCS는 그냥 영상만 띄우면 된다.
- frame과 overlay가 영상 픽셀에 이미 합쳐져 있으므로 동기화 문제가 적어 보인다.
- 초기 데모에서는 빠르게 시각적 결과를 만들 수 있다.

### 온보드에서 오버레이를 그리는 방식의 단점

- 온보드 CPU/GPU 자원을 관제용 drawing에 사용한다.
- 영상에 그려진 overlay는 원본 영상 정보를 훼손한다.
- 실전에서는 overlay를 끄고 싶을 수 있는데, debug/competition 모드를 분리해야 한다.
- line tracing, intersection, marker map 등 mission-critical 계산과 관제용 drawing 코드가 섞일 위험이 있다.
- 나중에 GCS에서 overlay 스타일을 바꾸거나 log/GUI와 연동할 때 유연성이 떨어진다.
- Raspberry Pi Zero 2 W급 환경에서는 불필요한 drawing과 JPEG 재처리가 mission loop에 부담이 될 수 있다.

### GCS에서 오버레이를 그리는 방식의 장점

- 온보드는 인식과 판단에만 집중한다.
- 원본 카메라 영상을 그대로 보존해 GCS로 보낼 수 있다.
- 오버레이 스타일, 색상, 표시 항목, GUI 구성을 GCS에서 자유롭게 바꿀 수 있다.
- 최종 목표인 영상 창, 로그 창, 명령 창 구조와 잘 맞는다.
- ArUco뿐 아니라 line contour, intersection, grid state overlay를 같은 방식으로 확장할 수 있다.
- 온보드에서 관제용 GUI/drawing 연산을 하지 않는 원칙을 지킬 수 있다.

### GCS에서 오버레이를 그리는 방식의 단점

- video frame과 telemetry를 `frame_seq` 기준으로 맞춰야 한다.
- GCS 쪽에 overlay drawing backend가 필요하다.
- Windows에서 OpenCV가 없으므로 Win32/GDI overlay 구현이 추가로 필요하다.

### 최종 판단

GCS에서만 오버레이를 그리는 방식으로 결정했다.

결정 이유:

- 온보드의 mission-critical 비전 처리와 관제용 시각화를 분리해야 한다.
- 원본 영상은 보존하고, overlay는 GCS에서 동적으로 그리는 편이 확장성이 높다.
- Raspberry Pi Zero 2 W급 사양을 고려하면, 관제용 drawing은 노트북 GCS가 담당하는 것이 낫다.
- 최종 GCS 목표가 영상 창, 로그 창, 명령 창으로 분리되는 구조이므로 GCS overlay가 자연스럽다.
- line tracing, intersection detection, marker map, grid state도 모두 metadata telemetry + GCS overlay 구조로 확장할 수 있다.

### 구현 결과

온보드:

- `ArucoDetector`는 marker id, corners, center, orientation만 계산한다.
- `vision_debug_node`는 원본 JPEG frame과 marker telemetry를 송신한다.
- 온보드 코드에는 `drawDetectedMarkers`, `cv::line`, `cv::circle`, `cv::putText` 기반 marker overlay drawing이 없다.

GCS:

- `MarkerOverlay`가 marker telemetry를 overlay primitive로 변환한다.
- `VideoWindow`가 overlay primitive를 실제 영상 위에 그린다.
- OpenCV backend와 Win32/WIC backend 모두 overlay drawing을 지원한다.

### 보고서용 요약

초기에는 구현 편의성을 위해 온보드에서 ArUco marker overlay를 직접 그려 송신하는 방안도 검토했다. 그러나 최종 시스템에서는 온보드가 mission-critical 비전 판단에 집중해야 하고, GCS는 관제용 시각화와 로그를 담당해야 하므로 역할 분리를 우선했다. 이에 따라 온보드는 marker detection 결과만 telemetry로 전송하고, GCS가 원본 영상 위에 overlay를 그리는 구조로 설계했다.

## 11. 현재 권장 실행 순서

### ArUco vision debug

GCS:

```powershell
cd uav-gcs
git pull
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi, metadata-only 기본 실행:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config
```

이 기본 실행은 onboard 부담을 줄이기 위해 debug video를 켜지 않는다. GCS vision log에는 telemetry가 들어오지만 camera window는 `waiting for video stream...` 상태일 수 있다.

GCS camera window와 overlay까지 확인하려면 Pi 명령에 `--video`를 붙인다.

```bash
./build/vision_debug_node --config config --video
```

정상 동작 기준, metadata-only:

- Pi에서 `frame=N markers=M jpeg_bytes=...` 출력
- Pi startup line에 `video: off`, `telemetry: on` 출력
- GCS vision log에 증가하는 `frame`, `seq`, line/marker/system/camera log 출력
- GCS log의 video counters가 `video_sent=0`, `chunks_last=0`, `last_bytes=0`이면 video off 상태로 정상

정상 동작 기준, `--video`:

- Pi에서 `discovering GCS video receiver...` 이후 `discovered GCS video receiver at <laptop-ip>:5600` 또는 fallback 출력
- GCS에 raw camera 영상 창 표시
- ArUco marker가 보이면 GCS 영상 창에 GCS-side marker overlay 표시
- line이 검출되면 GCS 영상 창에 magenta contour와 green tracking point 표시

## 12. 다음에 비슷한 문제가 생겼을 때 확인할 것

1. GCS 실행 파일이 올바른 조합인지 확인한다.
   - telemetry만: `uav_gcs`
   - 영상만: `uav_gcs_video`
   - ArUco overlay: `uav_gcs_vision_debug`
2. Ninja generator 사용 시 실행 파일은 `build/Release/`가 아니라 `build/` 아래에 있다.
3. Pi camera가 `rpicam-hello --list-cameras`에서 보이는지 확인한다.
4. Metadata-only 실행이면 Pi에서 discovery가 출력되지 않는 것이 정상이다. `--video`를 켰을 때만 `discovered GCS video receiver...`를 확인한다.
5. discovery는 되는데 packet이 안 들어오면 Windows Firewall을 확인한다.
6. 영상이 불안정하면 `config/vision.toml`의 `debug_video.send_fps`, `camera.jpeg_quality`, 해상도를 낮춰본다.
7. rpicam 내부 오류는 Pi의 `/tmp/astroquad_rpicam_vid.log`를 확인한다.
8. ArUco overlay가 어긋나면 `camera.frame_seq`와 video `frame_id` 동기화를 먼저 의심한다.

## 13. 라인 contour 상단 절단, 낮은 tracking point, 밝은 바닥 반사 오검출

### 증상

실제 카메라 테스트 캡처 `line1.png`, `line2.png`, `line3.png`, `line4.png`에서 다음 현상을 확인했다.

- 어두운 조명 또는 검정 천 위에서는 흰색 라인이 비교적 잘 잡혔다.
- 밝은 조명이 바닥에 반사되는 환경에서는 흰색 라인뿐 아니라 반사광이 같은 밝은 contour로 붙으면서 magenta contour가 넓게 퍼졌다.
- 가상 격자선 이미지를 태블릿에 띄워 촬영한 경우 십자 형태는 잘 검출됐다.
- 모든 캡처에서 green tracking point가 화면 상하 기준 중점보다 아래쪽에 표시됐다.
- magenta contour가 하단까지는 내려오지만 상단은 잘린 것처럼 보였다.
- 카메라가 완전 탑다운에 가까울 때보다 약간 전방을 보도록 세웠을 때 라인이 더 잘 이어지는 느낌이 있었다.

### 원인

가장 큰 직접 원인은 line config 기본값이었다.

```toml
roi_top_ratio = 0.35
lookahead_y_ratio = 0.70
```

`roi_top_ratio = 0.35`는 영상 상단 35%를 라인 검출에서 제외한다. 따라서 실제 라인이 영상 상단까지 보여도 detector는 그 영역을 보지 못하고, GCS overlay의 contour도 ROI 시작 지점에서 잘린 것처럼 보인다.

`lookahead_y_ratio = 0.70`은 green tracking point를 의도적으로 화면 아래쪽 70% 위치에 둔다. 그래서 정상 검출이어도 tracking point가 중앙보다 낮게 보인다.

밝은 바닥 반사 문제는 별도 원인이 있다. 현재 라인 검출은 onboard 부담을 줄이기 위해 grayscale/local-contrast threshold 기반으로 동작한다. 흰색 라인과 밝은 반사광이 비슷한 밝기로 붙으면 OpenCV contour 단계에서 하나의 큰 component처럼 합쳐질 수 있다. 이 경우 line 자체보다 반사광이 magenta contour에 포함된다.

카메라 각도 문제는 설정값과 시야 특성이 겹친 결과다. 기존 ROI가 하단 위주였기 때문에, 카메라를 조금 전방으로 세워 라인이 하단 ROI를 길게 통과할 때 더 잘 잡히는 것처럼 보였다. 최종 주행에서는 카메라 장착 각도를 고정한 뒤 그 각도에서 `roi_top_ratio`와 `lookahead_y_ratio`를 맞추는 것이 중요하다.

### 적용한 수정

기본 ROI와 tracking point 위치를 다음처럼 변경했다.

```toml
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
```

효과:

- 영상 상단 대부분을 검출에 포함하므로 contour 상단 절단이 크게 줄어든다.
- green tracking point가 화면 중앙에 가까워져 현재 캡처에서 보였던 하단 치우침이 완화된다.
- 너무 먼 상단 0%부터 모두 쓰지는 않아, 카메라가 살짝 전방을 볼 때 생길 수 있는 비바닥 영역의 영향을 조금 줄인다.

밝은 반사광 대응으로 `LineDetector`에 line-branch filtering을 추가했다.

동작 방식:

1. 기존처럼 threshold mask와 contour 후보를 만든다.
2. `lookahead_y_ratio` 위치의 행에서 정상적인 라인 폭 후보만 통과시킨다.
3. 그 행의 tracking x를 기준으로 위/아래 행을 따라가며 정상 라인 폭 run만 추적한다.
4. 너무 넓은 반사광, 화면 가장자리 큰 물체, 십자 교차부처럼 폭이 넓은 span은 selected line branch contour에서 제외한다.
5. GCS에는 이 selected branch contour를 `vision.line.contour_px`로 보내고 magenta overlay로 그린다.

이 방식은 교차점 판단 로직이 아니다. 이번 단계에서는 라인을 따라갈 branch만 안정적으로 시각화하는 것이 목적이다. 따라서 십자 교차부의 가로선 전체를 저장하거나 교차점 좌표를 계산하지 않는다.

실행 전/현장 튜닝을 위해 다음 옵션도 추가했다.

```bash
./build/vision_debug_node --config config --line-only --line-roi-top 0.08 --line-lookahead 0.55
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-threshold 180
./build/line_detector_tuner --config config --image test_data/images/line_sample.jpg --mode light_on_dark --threshold 180 --roi-top 0.08 --lookahead 0.55
```

### 자연광/운동장 바닥에서의 예상

실내 나무 바닥이나 태블릿 화면처럼 반사가 강한 표면은 밝은 라인 검출에 불리하다. 운동장 흙바닥은 일반적으로 정반사가 적기 때문에 `line3.png` 같은 넓은 하이라이트 문제는 줄어들 가능성이 크다.

다만 자연광 환경도 완전히 안전하지는 않다.

- 구름/햇빛 변화로 auto exposure가 흔들릴 수 있다.
- UAV 그림자나 사람 그림자가 라인 근처에 생길 수 있다.
- 라인 재질이 흰색 테이프처럼 반짝이면 특정 각도에서 glare가 생길 수 있다.
- 흙바닥 색과 라인 색의 명암 차이가 작으면 grayscale threshold만으로는 불안정할 수 있다.

현장에서는 먼저 `--line-mode auto`로 확인하고, 라인이 흰색/밝은 색이면 `--line-mode light_on_dark`, 검정/어두운 색이면 `--line-mode dark_on_light`로 고정해서 비교한다. 반사광이 계속 붙으면 `--line-threshold`를 올려 라인보다 어두운 하이라이트를 제외한다.

### 현장 테스트 순서

GCS:

```powershell
cd uav-gcs
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode auto
```

흰색 라인으로 확정되면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

상단이 아직 잘리면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-roi-top 0.03
```

green tracking point를 더 위에서 보고 싶으면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-lookahead 0.45
```

주행 제어용으로 너무 먼 곳을 보고 흔들리면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-lookahead 0.60
```

ArUco와 라인을 동시에 확인하려면 `--line-only`를 빼고 실행한다.

```bash
./build/vision_debug_node --config config --line-mode light_on_dark
```

## 14. 라인 branch filtering 되돌림과 교차점 표시 우선 결정

### 증상

`roi_top_ratio = 0.08`, `lookahead_y_ratio = 0.55`로 바꾼 뒤 green tracking point 위치와 contour 상단 절단 문제는 개선됐다. 하지만 반사광 억제를 위해 추가했던 line-branch filtering이 라인 인식을 너무 보수적으로 만들었다.

특히 `line4.png`처럼 십자 형태가 한 덩어리로 보여야 하는 장면에서, detector가 교차부 전체 contour를 보내지 않고 하단/상단의 일직선 branch 위주로 잘라서 보냈다. 이후 단계에서 교차점 판단을 붙이려면 십자 모양이 하나의 연결 contour로 보이는 것이 더 유리하다.

### 판단

이번 단계에서는 반사광을 과하게 줄이는 것보다 라인과 교차점을 적극적으로 검출하는 쪽이 우선이다.

유지:

- `roi_top_ratio = 0.08`
- `lookahead_y_ratio = 0.55`
- `--line-roi-top`, `--line-lookahead` 실행 옵션
- `auto`, `light_on_dark`, `dark_on_light`, `--line-threshold` 튜닝 옵션

되돌림:

- tracking row 주변의 좁은 branch만 남기던 line-branch filtering
- 교차부나 가로선 span을 magenta contour에서 제외하던 동작

### 현재 동작

`vision.line.contour_px`는 다시 detector가 선택한 연결 contour 전체를 단순화해서 보낸다. 따라서 십자 교차부가 threshold mask에서 하나의 connected component로 잡히면 GCS magenta overlay도 십자 형태로 표시된다.

반사광이 다시 문제가 되면 우선 다음 순서로 조정한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-threshold 180
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-roi-top 0.05 --line-lookahead 0.55
```

반사광 억제와 교차점 contour 보존을 동시에 만족해야 하는 상황이 반복되면, 다음 단계에서 단순 branch trimming이 아니라 `intersection candidate`를 별도 telemetry로 보내는 방식이 맞다. 즉, 라인 주행용 branch와 교차점 인식용 connected contour를 분리해서 둘 다 보존해야 한다.

## 15. 현재까지 문제 발생과 해결 흐름 요약

### 통신/영상 bring-up 단계

| 단계 | 문제 | 원인 | 해결 | 현재 상태 |
|---|---|---|---|---|
| Telemetry 수신 실패 | GCS가 Pi telemetry를 받지 못함 | 온보드 GCS IP가 고정되어 실제 노트북 IP와 불일치 | 기본 목적지를 broadcast로 변경하고 `SO_BROADCAST` 적용 | `uav_onboard` -> `uav_gcs` telemetry 수신 가능 |
| Windows 영상 빌드 실패 | OpenCV가 없어 `uav_gcs_video`가 빌드되지 않음 | Windows MinGW 환경에 OpenCV가 없고 ABI 부담이 있음 | OpenCV backend와 Win32/WIC fallback backend 분리 | OpenCV 없이도 GCS video/vision debug 실행 파일 빌드 가능 |
| Pi camera 실행 실패 | `video_streamer --source rpicam` 시작 실패 | Linux `popen()`에 `"rb"` mode 사용 | Windows는 `"rb"`, POSIX는 `"r"`로 분기 | Pi에서 `rpicam-vid` stdout MJPEG 읽기 가능 |
| `rpicam-vid` abort 로그 | `--count` 종료 후 SIGPIPE성 abort 로그가 보임 | 테스트 종료 시 parent가 pipe를 닫아 `rpicam-vid` stdout write 실패 | stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리 | 사용자 콘솔에는 정상 송신 결과 중심으로 표시 |
| Broadcast 영상 불안정 | 영상 frame 완성률이 낮음 | MJPEG frame이 여러 UDP packet으로 나뉘고 broadcast 손실이 큼 | GCS beacon discovery 후 영상은 unicast로 전송 | Pi가 GCS IP 자동 발견 후 안정적인 unicast 송신 |
| GCS 영상 깜빡임 | packet drop 때 검은 화면이 보임 | timeout 때 status 화면으로 되돌아가고 Win32 paint flicker가 있음 | 마지막 complete frame 유지, double buffering, `WM_ERASEBKGND` 처리 | 영상 drop이 있어도 화면 유지 |

### 비전/오버레이 단계

| 단계 | 문제 | 원인 | 해결 | 현재 상태 |
|---|---|---|---|---|
| Pi ArUco 빌드 실패 | OpenCV enum type compile error | Pi OpenCV 4.10 header의 enum 이름이 코드와 다름 | `cv::aruco::PredefinedDictionaryType` 사용 | Pi에서 ArUco target 빌드 가능 |
| Vision debug packet 미수신 | Pi는 GCS를 발견하지만 GCS packet count가 0 | Windows Defender Firewall이 새 exe inbound UDP를 Block | `uav_gcs_vision_debug.exe` inbound allow rule 필요 | 방화벽 허용 시 telemetry/video 수신 가능 |
| Overlay 위치/역할 결정 | 온보드에서 그릴지 GCS에서 그릴지 결정 필요 | Pi Zero 2 W에서 drawing/JPEG 재처리는 mission-critical 자원 낭비 | 온보드는 metadata만 보내고 GCS가 overlay drawing | ArUco/line 모두 GCS-side overlay 구조 |
| 라인 오검출 | 발/검은 물체가 라인으로 잡힘 | 기존 polarity/후보 선택이 큰 어두운 contour에 취약 | `mode=auto`, `light_on_dark`, `dark_on_light`, threshold override 추가 | 라인 색이 확정되지 않은 환경에 대응 가능 |
| 라인 상단 절단/초록점 낮음 | contour 상단이 잘리고 tracking point가 아래에 치우침 | `roi_top_ratio=0.35`, `lookahead_y_ratio=0.70` | `roi_top_ratio=0.08`, `lookahead_y_ratio=0.55`로 조정 | 상단 절단 완화, tracking point 중앙화 |
| 반사광 억제 시도 | 밝은 바닥 반사가 contour에 섞임 | 흰색 라인과 반사광이 grayscale threshold에서 붙음 | line-branch filtering을 실험적으로 추가 | 교차점 표시에 불리해 되돌림 |
| 십자 교차점 contour 분리 | 십자가 하단/상단 branch로 쪼개짐 | branch filtering이 넓은 교차부를 제외 | branch filtering 제거, connected contour 전체 전송 | `line4.png` 같은 십자 contour 표시 우선 |

### 현재 설계 판단

현재 최종 판단은 다음과 같다.

- 온보드는 camera capture, ArUco/line detection, telemetry generation, raw JPEG debug streaming만 수행한다.
- GCS는 video display, marker/line overlay, vision log window, packet stats 표시를 담당한다.
- GCS video는 best-effort debug channel이며, 향후 제어/이동 판단 경로를 막지 않아야 한다.
- 라인트레이싱 MVP에서는 교차점 좌표 저장이나 grid update를 아직 하지 않는다.
- 교차점 단계에서는 라인 주행용 branch와 교차점 판단용 connected contour를 분리하는 것이 맞다.

### 현재 권장 실행 조합

GCS:

```powershell
cd uav-gcs
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi, 라인만:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Raspberry Pi, ArUco와 라인 동시:

```bash
./build/vision_debug_node --config config --line-mode light_on_dark
```

문제 발생 시 우선순위:

1. GCS가 packet을 못 받으면 Windows Firewall을 확인한다.
2. 영상이 끊기면 GCS discovery/unicast 여부와 network 품질을 확인한다.
3. 라인 상단이 잘리면 `--line-roi-top`을 더 낮춘다.
4. tracking point를 조정하려면 `--line-lookahead`를 조정한다.
5. 흰색 라인은 `--line-mode light_on_dark`, 어두운 라인은 `--line-mode dark_on_light`로 고정해서 비교한다.
6. 반사광이 심하면 `--line-threshold`와 camera angle을 먼저 튜닝한다.

## 16. 1.3 테스트 이후 라인 contour 갈라짐, GCS 프레임 드랍, Pi 발열

### 문제 상황

1.3 테스트에서는 약 2m 고도에서도 검은 천 위 흰색 라인, 아이패드 격자, 흰색 진열장 테두리처럼 대비가 충분한 대상은 이전보다 잘 잡혔다. 특히 십자 교차와 L자 형태도 connected contour로 유지되는 방향이 확인됐다.

반대로 방 바닥 위 휴지 라인은 여전히 잘 잡히지 않았다. 원인은 휴지 표면과 밝은 목재 바닥의 명도 차이가 작고, 바닥 반사와 카메라 자동 노출 때문에 라인과 배경의 local contrast가 약해지는 것으로 판단한다. 이 실내 테스트는 실제 운동장 흙바닥 대비 조건을 완전히 대표하지 못하므로, 경기장과 비슷한 흙색/무광 배경에서 별도 검증해야 한다.

검은 천 위 흰색 라인은 검출되지만 contour가 내부 edge 중심으로 갈라지거나, 라인 폭 전체가 아니라 한쪽 edge만 얇게 잡히는 경우가 있었다. 또한 latency는 줄었지만 GCS 영상 displayed FPS가 낮아져 관제용 영상이 끊겨 보였고, Pi Zero 2 W 발열이 커졌다.

### 원인 판단

- 라인 갈라짐: local contrast 기반 마스크가 라인 전체 면보다 양쪽 edge를 먼저 잡고, morphology 연결이 부족하면 하나의 라인이 두 줄처럼 분리된다.
- 원거리/고고도 검출 저하: 10cm 라인이 2m 고도에서 차지하는 픽셀 폭이 작아지고 JPEG 압축, 노출, 렌즈 왜곡 영향이 커진다.
- GCS 프레임 드랍: onboard latency를 낮추기 위해 낮은 품질/낮은 send FPS로 조정했지만, GCS 쪽에서 UDP 수신과 화면 표시가 같은 흐름에 묶이면 packet drain이 늦어져 incomplete frame이 늘 수 있다.
- CPU/발열: Pi Zero 2 W에서 camera capture, ArUco, line detection, JPEG packet 송신을 동시에 수행하면 미션 제어 여유가 줄어든다. 영상은 반드시 debug/best-effort 채널로 유지해야 한다.

### 적용한 해결

- 라인 마스크 후처리를 `morph_open_kernel`, `morph_close_kernel`, `morph_dilate_kernel`로 분리했다. 기본값은 작은 open, 큰 close로 잡아 얇은 노이즈는 줄이고 라인 내부 edge가 갈라지는 현상을 줄인다.
- projection 기반 run merge에 `line_run_merge_gap_px`를 추가했다. 가까운 밝은 run은 하나의 라인 후보로 합치되, 지나치게 먼 외곽 노이즈는 합치지 않는다.
- onboard debug video는 `fps`, `jpeg_quality`, `send_fps`, `chunk_pacing_us`를 분리했다. 현재 기본값은 12FPS capture, debug video 5FPS send, camera JPEG quality 45, chunk pacing 150us다. 다만 `debug_video.enabled = false`라서 `--video`를 켜지 않으면 video worker는 동작하지 않는다.
- onboard telemetry에 `video_send_ms`, `video_chunk_count`, `video_chunks_sent`, `video_skipped_frames`, `cpu_temp_c`를 추가했다.
- GCS는 UDP 수신을 별도 background thread에서 계속 drain하고, UI는 최신 complete JPEG만 표시하도록 변경했다.
- GCS reassembler에 `completed`, `incomplete`, `old_packets`, `chunk_mismatch_resets`, `last_chunk_count`, `last_frame_bytes` 통계를 추가했다.

### 검증 방법

로컬에서는 다음 검증을 통과했다.

```powershell
cmake --build uav-onboard/build
cmake --build uav-onboard/build
ctest --test-dir uav-onboard/build --output-on-failure
cmake --build uav-gcs/build
cmake --build uav-gcs/build
ctest --test-dir uav-gcs/build --output-on-failure
```

OpenCV가 있는 로컬 빌드에서는 `line_detector_tuner`와 `vision_debug_node`도 별도 Release 빌드로 확인했다. 단, 1.3 이미지 기반 튜너 검증은 GCS overlay가 이미 들어간 screenshot을 사용한 smoke test이므로 실제 raw camera frame 성능을 완전히 대체하지는 않는다.

### Pi 실기 테스트 체크리스트

GCS log에서 다음 항목을 같이 기록한다.

- `processing_latency_ms`, `read_frame_ms`, `jpeg_decode_ms`, `aruco_latency_ms`, `line_latency_ms`
- `display_fps`, `completed`, `incomplete`, `malformed`, `old_packets`, `mismatch_resets`
- `video_send_ms`, `video_chunk_count`, `video_skipped_frames`, `video_dropped_frames`
- `cpu_temp_c`

Pi에서 발열 또는 throttling이 의심되면 다음을 같이 확인한다.

```bash
vcgencmd measure_temp
vcgencmd get_throttled
```

테스트 우선순위는 1) 실제 경기장과 비슷한 흙색 무광 배경, 2) 1.8-2.0m 고도, 3) 흰색 라인과 검정 라인 각각, 4) line-only와 ArUco+line 동시 실행 비교 순서다.

### 추가 판단

Pi Zero 2 W와 Pi Camera만으로 MVP 검증은 가능하지만, ArUco, line tracing, mission 판단, Pixhawk 제어까지 모두 안정적으로 돌리려면 여유가 작다. 현재 기본 장치는 Pi 4 + IMX519로 바뀌었지만 설계 판단은 그대로다. 실제 비행 단계에서는 GCS 영상 FPS보다 제어 루프와 telemetry 안정성을 우선하고, 필요하면 debug video를 끄거나 line-only 모드로 비행 테스트를 시작한다.

## 17. Raspberry Pi 4 + IMX519-78 전환 체크

### 증상

Pi 4로 교체한 뒤 카메라가 보이지 않거나, `vision_debug_node`가 `failed to start rpicam-vid` 또는 frame read 실패로 종료될 수 있다.

### 우선 확인

```bash
rpicam-hello --version
rpicam-hello --list-cameras
rpicam-still -t 1000 --nopreview -o test_data/images/imx519_smoke.jpg
rpicam-vid -t 5000 --nopreview --codec mjpeg --width 640 --height 480 --framerate 12 -o /tmp/imx519_test.mjpeg
```

### 판단

- `rpicam-hello --list-cameras`에 IMX519가 없으면 Astroquad 코드 문제가 아니라 OS/kernel/rpicam/IMX519 driver 또는 CSI cable 문제부터 확인한다.
- 현재 Pi image에서는 IMX519 센서와 `ak7375` lens driver가 잡히지만 libcamera AF algorithm이 없어 `autofocus_mode`/`lens_position`은 적용되지 않을 수 있다.
- 이 경우 `/dev/v4l-subdev1`의 `focus_absolute` V4L2 control을 사용한다.
- Pi 4에서도 debug video는 best-effort다. GCS 영상이 끊겨도 onboard line/marker telemetry와 추후 MAVLink control loop가 우선이다.

### 적용된 1차 대응

- `config/vision.toml`에 Pi 4 + IMX519용 `[camera]` 설정을 추가했다.
- `RpicamMjpegSource`가 rpicam autofocus/focus/exposure/AWB/denoise/orientation 옵션을 받을 수 있게 확장됐다.
- telemetry v1.5에 `system.*`, `camera.*`, `debug.capture_fps`, `debug.processing_fps`, `debug.video_send_failures`를 추가했다.
- GCS는 새 system/camera/debug field를 log window에 표시한다.
- 현재 성능 우선 기본값은 `camera.width=960`, `camera.height=720`, `camera.fps=12`, `camera.jpeg_quality=45`, `debug_video.enabled=false`, `debug_video.send_fps=5`, `debug_video.chunk_pacing_us=150`이다.
- GCS video latency/age 표시는 제거했다. GCS camera overlay는 `frame N`만 표시한다.

### 2026-05-13 focus 확인

Pi에서 다음 로그가 확인됐다.

```text
Could not set AF_MODE - no AF algorithm
Could not set LENS_POSITION - no AF algorithm
lp -1.00
```

즉 `rpicam --lens-position` 경로는 현재 image에서 동작하지 않는다. 하지만 media topology에는 lens subdevice가 있다.

```text
entity 4: ak7375 10-000c
device node name /dev/v4l-subdev1
focus_absolute min=0 max=4095
```

`v4l2-ctl -d /dev/v4l-subdev1 --set-ctrl focus_absolute=<value>`로 초점이 움직이는 것을 확인했고, 현재 테스트 장면에서는 대략 다음 결과가 나왔다.

```text
focus_absolute=1984 -> focus metric 약 1127
focus_absolute=2048 -> focus metric 약 1184
focus_absolute=3072 -> focus metric 약 352
focus_absolute=4095 -> focus metric 약 337
```

현장값은 카메라 높이/대상 거리마다 달라질 수 있으므로 `1984~2112` 근처에서 다시 좁게 sweep한다. 현재 config에는 conservative하게 `focus_absolute = 1984`를 넣었다.

코드 변경:

- `[camera] focus_absolute`와 `focus_device`를 추가했다.
- `RpicamMjpegSource`가 rpicam 실행 전에 `v4l2-ctl`로 focus를 적용한다.
- current config에서는 먹지 않는 rpicam AF 옵션을 비워두고 `lens_position = -1.0`으로 둔다.

## 18. GCS 영상 창이 `waiting for video stream...`에 머무르지만 vision log는 계속 갱신됨

### 증상

Windows에서 다음처럼 GCS vision debug를 실행했다.

```powershell
.\build\uav_gcs_vision_debug.exe --config config
```

Pi에서는 line tracing telemetry가 정상으로 송신됐다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Pi console에는 `frame=N line=yes ... video_sent=0 video_chunks=0`처럼 처리 결과가 계속 출력되고, GCS log window도 frame/line/system/camera telemetry를 받았다. 하지만 GCS camera window는 검은 화면의 `waiting for video stream...` 상태였다.

### 원인

현재 `vision_debug_node` 기본값은 metadata-only다.

```toml
[debug_video]
enabled = false
```

따라서 `--video`를 붙이지 않으면 onboard는 ArUco/line 정보를 telemetry로만 보내고, raw MJPEG debug video는 보내지 않는다. 이때 GCS video receiver는 열려 있어도 받을 video packet이 없으므로 waiting 화면이 정상이다.

확인 신호:

- Pi startup line: `video: off`, `telemetry: on`
- Pi per-frame log: `video_sent=0`, `video_chunks=0`
- GCS packet/video log: `video_sent=0`, `chunks_last=0`, `last_bytes=0`

### 해결

영상과 overlay를 실제로 보고 싶을 때만 Pi 명령에 `--video`를 추가한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
```

네트워크 discovery가 막히면 GCS IP를 직접 지정한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --gcs-ip <laptop-ip>
```

### 설계 판단

이 동작은 버그가 아니라 의도된 기본값이다. 최종 시연에서는 GCS 없이 onboard만 돌릴 수도 있고, mission 판단과 MAVLink 제어가 최우선이다. GCS video는 디버그 관제용이므로 기본값은 꺼 두고 필요할 때만 켠다.

## 19. GCS 영상 latency/age가 음수거나 체감 지연과 맞지 않음

### 증상

GCS camera overlay 또는 log에 표시되던 latency/age가 음수로 나오거나, 양수 20-30ms처럼 표시되는데 실제 체감 지연은 약 500ms 수준이었다.

### 원인

초기 latency 표시는 onboard frame timestamp와 GCS 수신/표시 시각을 단순 비교했다. 하지만 다음 조건 때문에 이 값은 end-to-end video latency로 보기 어렵다.

- Raspberry Pi와 Windows laptop의 wall clock이 정확히 동기화되어 있다고 가정할 수 없다.
- UDP MJPEG video는 best-effort이며 incomplete frame drop, reassembly, UI paint delay가 섞인다.
- GCS는 최신 complete JPEG만 표시하므로 실제 사람이 보는 지연은 network packet timing만으로 설명되지 않는다.
- Debug video는 mission-critical 경로가 아니므로 정확한 latency 계측보다 onboard 처리 시간과 packet health가 더 중요하다.

### 해결

GCS camera overlay와 vision log에서 video latency/age 표시를 제거했다.

현재 표시 정책:

- Camera window overlay: `frame N`만 표시
- Vision log: onboard 처리 시간인 `processing`, `read`, `decode`, `aruco`, `line`, `json`, `tsend`, `vsubmit`, `vsend`를 표시
- Video health: `completed`, `incomplete`, `display_fps`, `chunks_last`, `last_bytes`, `video_sent`, `video_skipped`, `video_dropped`, `video_send_failures`로 판단

`config/ui.toml`에는 의도를 남기기 위해 다음 값을 둔다.

```toml
[video_window]
show_latency = false
```

### 설계 판단

현재 프로젝트에서 필요한 것은 정확한 관제 영상 latency 숫자가 아니라, onboard vision/control loop가 제때 돌고 있는지와 debug video가 관찰 가능한 수준인지다. 정확한 video latency가 필요해지는 경우에는 NTP/PTP 수준 clock sync 또는 GCS에서 송수신 round-trip 측정용 별도 protocol을 설계해야 한다.

## 20. IMX519 화질을 높여도 ArUco 인식 개선 대비 onboard 부담이 큼

### 증상

IMX519 영상이 GCS 화면에서 뿌옇게 보이고 ArUco marker 인식이 불안정해 보여, capture 해상도와 JPEG quality를 올리는 실험을 했다.

실험 방향:

- `camera.width/height`를 `1280x960`까지 올림
- `camera.jpeg_quality`를 `85-90` 수준까지 올림
- ArUco bench tuning용 CLI override 추가

```bash
./build/vision_debug_node --config config --aruco-only --video --camera-quality 90 --lens-position 1.0
```

### 관찰

GCS 영상의 JPEG artifact는 줄어들 수 있지만, 대회 조건에서는 ArUco marker가 50cm x 50cm이고 고도는 약 2m로 예상된다. 이 조건에서는 marker pixel 크기가 충분할 가능성이 높고, 고화질 video가 mission 성능을 결정하는 병목이라고 보기 어렵다.

반대로 고화질 설정은 다음 부담을 키운다.

- rpicam MJPEG frame 크기 증가
- onboard JPEG decode 시간 증가
- ArUco/line detector 입력 픽셀 증가
- optional debug video UDP chunk 수 증가
- Wi-Fi packet loss 가능성 증가
- 추후 mission logic/MAVLink control loop 여유 감소

### 해결

기본값을 성능 우선 설정으로 되돌렸다.

```toml
[camera]
width = 960
height = 720
fps = 12
jpeg_quality = 45

[debug_video]
enabled = false
send_fps = 5
jpeg_quality = 40
chunk_pacing_us = 150
```

Bench tuning이 필요할 때만 CLI override를 쓴다.

```bash
./build/vision_debug_node --config config --aruco-only --camera-quality 90 --lens-position 1.0
./build/vision_debug_node --config config --aruco-only --video --camera-width 1280 --camera-height 960 --camera-quality 85
```

### 설계 판단

현재 단계에서는 화질보다 onboard 처리 여유가 더 중요하다. ArUco가 실제 50cm marker/2m 조건에서 충분히 잡히는지 확인한 뒤, 정말 부족할 때만 해상도나 JPEG quality를 올린다. 기본값은 line tracing, ArUco, 추후 mission 판단, MAVLink 제어를 같이 얹을 수 있는 보수적 설정으로 유지한다.

## 21. Metadata-only 실행에서도 GCS discovery 때문에 startup이 3초 늦어짐

### 증상

이전 구현에서는 `vision_debug_node`를 telemetry-only로 돌려도 GCS video discovery beacon을 최대 3초 기다리는 흐름이 있었다. 최종 운용에서 video를 끄고 onboard만 빠르게 띄우는 경우에는 불필요한 대기였다.

### 원인

GCS discovery는 video destination IP/port를 찾기 위한 기능이다. 하지만 debug video가 꺼진 상태에서도 broadcast address이면 discovery를 시도하면, 실제로 보낼 video가 없는데 startup만 늦어진다.

### 해결

`vision_debug_node`에서 `send_video`가 true일 때만 GCS discovery를 수행하도록 변경했다.

현재 동작:

- 기본 `./build/vision_debug_node --config config`: `debug_video.enabled=false`이므로 discovery 대기 없음
- `--no-video`: discovery 대기 없음
- `--video`: broadcast/default GCS IP일 때 discovery를 최대 3초 수행
- `--video --gcs-ip <laptop-ip>`: 명시 IP가 있으므로 discovery 없이 바로 전송

### 설계 판단

Metadata-only 실행은 향후 mission run에 가장 가까운 형태다. 영상 관제 기능이 꺼져 있을 때는 startup과 처리 경로를 최대한 단순하게 유지한다.

## 22. 넓은 흰색 라인이 얇은 edge만 잡히고 교차점 판단이 흔들림

### 문제 상황

사용자가 제시한 GCS 캡처에서는 굵은 흰색 라인 전체가 magenta overlay로 감싸지지 않고, 한쪽 edge 또는 얇은 strip만 라인으로 잡히는 경향이 있었다. 특히 밝은 목재/흙색 배경에서는 `straight`, `T`, `+`가 `unknown`이나 `L`로 흔들렸다.

### 원인

당시 기본 마스크는 `local_contrast`였다. 이 방식은 배경 대비가 큰 어두운 천 위에서는 잘 동작하지만, 넓은 흰색 테이프/종이처럼 선 내부가 균일하고 배경도 밝은 경우에는 내부 면 전체보다 양쪽 경계 contrast를 더 강하게 잡는다.

그 결과:

- line contour가 실제 선 폭 전체가 아니라 edge 위주로 형성된다.
- intersection center 후보가 실제 교차점 중심이 아니라 한 branch 쪽으로 밀린다.
- 4방향 ray score 중 일부 branch가 threshold 주변에서 깜빡인다.
- `+ -> T`, `T -> L`, `straight -> unknown`처럼 branch 누락형 오분류가 생긴다.

### 해결

`LineMaskBuilder`에 `white_fill` 전략을 추가하고 기본값으로 전환했다.

```toml
[line]
mask_strategy = "white_fill"
white_v_min = 145
white_s_max = 90
fill_close_kernel = 11
fill_dilate_kernel = 3
```

이 전략은 HSV 기준으로 낮은 saturation, 높은 value 영역을 흰색 라인 후보로 잡고 close/dilate morphology로 내부를 채운다. `local_contrast`는 조명 비교 실험용 fallback으로 남겨 두었다.

또한 `LineDetector`는 전체 contour centroid가 아니라 하단 anchor/lookahead band의 X projection으로 tracking X를 계산하도록 바꿨다. L/T 교차점의 가로 branch가 라인 중심을 끌어당기는 문제를 줄이기 위한 조치다.

### 결과

사용자 제공 4개 라인 캡처에서 굵은 흰색 라인이 더 안정적으로 하나의 blob/contour로 잡혔다. 교차점 판단도 원시 detector 단계에서 branch score가 안정화되어 이전보다 `L`, `T`, `+`, `straight` 구분이 잘 유지된다.

## 23. 교차점 판단 정확도가 올라간 이유와 현재 로직

### 현재 교차점 판단 흐름

교차점 판단은 한 frame만 보고 바로 결정하지 않는다. 현재 흐름은 다음과 같다.

1. `IntersectionDetector`가 shared line mask에서 largest blob과 중심 후보를 찾는다.
2. 중심 후보 주변에서 camera-relative 4방향 ray score를 계산한다.
   - front
   - right
   - back
   - left
3. branch score가 `intersection_threshold` 이상이면 raw branch present로 본다.
4. `IntersectionStabilizer`가 raw type을 짧게 smoothing한다.
5. `IntersectionDecisionEngine`이 최근 frame window를 모아 branch evidence를 다시 계산한다.
6. window 안에서 branch별 `present_frames`, `max_score`, `average_score`를 보고 최종 accepted type을 정한다.

### type 결정 기준

`IntersectionDecisionEngine`은 우선순위를 `+ > T > L > straight > unknown`으로 두지만, 단일 frame에서 상위 type이 잠깐 튀는 것을 바로 채택하지 않는다.

- `+`: front/right/back/left 4방향 모두 충분한 frame 수 동안 보이고, 네 방향 모두 `high_confidence_score` 이상이어야 채택한다.
- `T`: 가능한 3방향 조합 중 branch evidence가 가장 좋은 mask를 고른다.
- `L`: 가능한 2방향 직각 조합 중 evidence가 가장 좋은 mask를 고른다.
- `straight`: front/back 또는 left/right 조합이 충분히 보이면 인정한다.
- `unknown`: branch evidence가 부족하거나 일관되지 않을 때 남긴다.

현재 주요 설정:

```toml
[intersection_decision]
cruise_window_frames = 6
turn_confirm_frames = 8
min_branch_score = 0.72
high_confidence_score = 0.85
min_cross_branch_frames = 2
min_t_branch_frames = 2
min_l_branch_frames = 3
record_node_once_frames = 18
```

### 정확도가 좋아진 이유

정확도 개선은 한 가지 수정 때문이 아니라 여러 변경이 겹친 결과다.

- `white_fill`로 넓은 흰색 라인의 내부가 채워져 branch ray가 끊기지 않는다.
- `+`는 네 번째 branch가 약할 때 바로 채택하지 않도록 `high_confidence_score` guard를 추가했다.
- `T`와 `L`은 단일 frame branch count가 아니라 최근 window의 branch evidence로 판단한다.
- line tracking point와 intersection classification을 분리했다. 라인 주행 중심은 anchor band에서 잡고, 교차점 판단은 connected mask topology를 본다.
- GCS overlay를 간소화해 실제 판단에 중요한 line center, offset, 접근 중인 branch 방향만 보이게 했다.

### 보고서용 요약

초기에는 한 frame의 ray score가 threshold 주변에서 흔들리면 교차점 type도 즉시 흔들렸다. 현재는 넓은 흰색 라인을 채워진 mask로 안정화하고, 최근 frame window에서 branch evidence를 누적해 topology를 결정한다. 특히 `+`는 네 방향 모두 높은 확신이 있을 때만 채택하도록 하여 약한 false branch 때문에 `T`가 `+`로 승격되는 문제를 줄였다.

## 24. GCS overlay 정보가 너무 많아 라인트레이싱 제어용 오차를 보기 어려움

### 문제 상황

기존 GCS 영상 overlay에는 line label, tracking point, intersection branch score, decision label 등 많은 정보가 동시에 표시됐다. 라인트레이싱 제어를 준비하려면 “현재 라인 중심이 카메라 중심에서 얼마나 벗어났는지”가 가장 잘 보여야 하는데, 화면이 복잡했다.

### 해결

Line overlay를 다음 세 요소 중심으로 단순화했다.

- magenta: 현재 선택된 line contour/border
- red circle: 현재 라인 중심 X
- green horizontal line: 카메라 중심에서 라인 중심까지의 lateral offset

중요한 표시 규칙은 red circle의 Y가 항상 camera center Y에 고정된다는 점이다. 실제 line tracking point의 세로 위치가 frame 아래쪽/위쪽에 있어도 GCS 표시에서는 같은 수평선 위에 두어, 좌우 오차만 직관적으로 보이게 했다.

교차점 overlay도 간소화했다.

- 화면 상단/현재 접근 영역의 교차점만 표시
- `IX <type>` 형태의 compact label만 표시
- branch score 숫자와 긴 decision label은 영상에서 제거하고 log에 남김

### 결과

영상 창에서 라인트레이싱 제어에 필요한 lateral error를 즉시 확인할 수 있게 되었다. 복잡한 판단 값은 vision log에서 확인하는 구조로 역할을 분리했다.

## 25. 빨간 라인 중심점이 좌우 이동에 조금 늦게 따라오는 느낌

### 문제 상황

라인 자체는 놓치지 않지만, 카메라를 좌우로 빠르게 움직였을 때 GCS 영상의 빨간 점과 초록 offset line이 약간 늦게 따라오는 느낌이 있었다.

### 원인

두 가지 지연 요인이 있었다.

1. onboard `LineStabilizer`가 EMA smoothing, velocity limit, jump rejection을 적용한다.
2. GCS debug video는 `--video`만 켜면 기본 5FPS이고, `--fps 12`를 지정해야 카메라 capture FPS에 가깝게 보인다.

기존 GCS overlay는 filtered `tracking_point_px`를 사용했기 때문에 제어 안정화에는 좋지만, 사람이 화면에서 보는 빨간 점은 raw detector보다 늦어 보일 수 있었다.

### 해결

GCS 영상 overlay의 red circle과 green offset line은 `raw_tracking_point_px`가 있으면 raw 값을 우선 사용하도록 변경했다. telemetry log에는 filtered 값과 raw 값을 모두 남겨, 제어 입력으로 어떤 값을 쓸지는 이후 control loop에서 결정할 수 있게 했다.

실시간 관찰용 권장 실행:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --fps 12
```

### 판단

관제 화면에서는 raw point가 더 직관적이고 빠르다. 반면 실제 제어에는 filtered point가 더 안전할 수 있다. 최종 제어 단계에서는 raw/filtered offset을 모두 기록하면서 lateral controller가 어느 값을 사용할지 별도 실험으로 결정한다.

## 26. GCS grid가 로그 아래에 묻혀 보이지 않음

### 문제 상황

`GridMapTracker`가 ASCII grid를 만들도록 구현했지만, GCS vision log에는 telemetry 값이 너무 많아 grid가 화면 아래로 밀렸다. 사용자가 보기에는 grid가 그려지는 전용 공간이 없는 것처럼 보였다.

### 원인

처음 구현은 `VisionLogFormatter`가 상세 telemetry 문자열 마지막에 `[grid-map]` 텍스트를 붙이는 방식이었다. log window는 하나의 multiline edit control만 사용했기 때문에 grid와 상세 telemetry가 같은 스크롤 영역에서 경쟁했다.

### 해결

GCS `VisionLogWindow`를 두 영역으로 분리했다.

- 상단 고정 pane: `[grid-map]` ASCII map
- 하단 pane: `[vision]`, `[line]`, `[intersection]`, `[intersection-decision]`, `[grid-node]`, `[video-rx]` 상세 telemetry

### 결과

grid는 항상 vision log window 상단에 남고, 상세 telemetry가 길어져도 grid 표시 영역이 밀리지 않는다.

## 27. GCS grid pane이 한 줄로 붙어서 보임

### 문제 상황

상단 grid pane이 생겼지만 Windows에서 다음처럼 줄바꿈 없이 한 줄로 이어져 보였다.

```text
[grid-map] nodes=1 current=(0,0) heading=unknowns|@
```

하단 telemetry도 `[vision] ... [camera] ... [system] ...`이 한 줄로 이어져 표시됐다.

### 원인

Win32 `EDIT` control은 `\n`만 있는 문자열을 안정적인 줄바꿈으로 처리하지 않는다. Windows multiline edit에는 `\r\n` line ending을 넣어야 한다.

### 해결

`VisionLogWindow`에서 UI에 문자열을 넣기 전에 line ending을 정규화했다.

- 입력 문자열의 단독 `\n` 앞에 `\r`을 추가
- UTF-8 -> UTF-16 변환 전에 정규화

### 결과

상단 grid와 하단 telemetry가 의도한 여러 줄 형태로 표시된다.

## 28. GCS grid가 `nodes=1 current=(0,0) heading=unknown`에서 더 이상 확장되지 않음

### 문제 상황

상단 grid pane 자체는 표시되지만 처음부터 끝까지 다음과 비슷한 상태에 머물렀다.

```text
[grid-map] nodes=1 current=(0,0) heading=unknown
```

교차점 판단과 하단 vision telemetry는 계속 갱신되는데, grid는 새 칸을 추가하지 않았다.

### 원인

현재 단계에는 실제 드론 상태머신, IMU heading, MAVLink turn completion event가 없다. 따라서 onboard `GridCoordinateTracker`가 처음 grid node를 local `(0,0)`으로 저장한 뒤에도 `current_heading_ = Unknown` 상태를 유지했다.

기존 좌표 갱신 로직은 heading이 알려져 있을 때만 다음 좌표로 advance한다.

```cpp
if (current_heading_ != GridHeading::Unknown) {
    current_coord_ = advance(current_coord_, current_heading_);
}
```

heading이 unknown이면 이후 node event가 들어와도 좌표가 계속 `(0,0)`이 된다. GCS `GridMapTracker`는 좌표를 key로 map에 저장하므로 같은 `(0,0)`이 반복되어도 node count가 늘지 않는다.

### 해결

임시 vision-only grid smoke용 heading 추정 로직을 추가했다.

- 첫 node에 heading이 없으면 launch line에서 grid로 진입했다고 보고 local heading을 `north`로 둔다.
- 첫 node에서 side branch가 보이면 첫 row/column으로 들어가기 위해 side branch를 우선 선택한다.
- 이후에는 front branch가 있으면 직진한다.
- front branch가 없고 right/left branch가 있으면 해당 방향으로 90도 회전한다고 가정한다.
- row 끝에서 한 칸 올라간 뒤 같은 방향으로 한 번 더 회전해야 하는 snake 특성을 위해 `pending_second_turn`을 둔다.

이 로직은 실제 비행 제어가 아니라 수동 카메라 테스트와 GCS grid 렌더링을 위한 임시 추정이다. 향후 MAVLink/IMU/state machine이 들어오면 `notifyTurnCompleted()`나 mission state가 실제 heading을 공급해야 한다.

### 결과

드론 없이 손으로 카메라를 들고 snake 방식으로 이동하는 테스트에서도 node event가 들어올 때마다 local coordinate가 확장될 수 있게 되었다.

### 남은 리스크

- 초기 진입 방향이 bottom-entry가 아니라 다른 방향이면 local map이 회전된 형태로 그려질 수 있다.
- 실제 드론이 회전 완료를 알려주지 않는 한 heading은 vision-only 추정이다.
- 공식 좌표계 변환은 아직 구현 전이다.
- 같은 교차점을 다시 방문하지 않는 실제 snake policy와 visited guard는 mission layer에서 별도로 완성해야 한다.
- 확실히 회전 완료를 판단하기 애매해서 그리드가 완벽하게 그려지지 않음

## 29. `dark_on_light`에서 굵은 검정 라인이 하나의 라인으로 잡히지 않음

### 문제 상황

2m 고도 탑다운뷰를 가정한 굵은 검정 라인 테스트에서 `--line-mode dark_on_light`로 실행했는데도 검정 라인 전체가 하나의 넓은 contour로 잡히지 않았다. Magenta overlay가 검정 라인의 양쪽 edge 위주로 생기고, red line-center point도 실제 라인 중앙보다 한쪽 edge에 가까웠다.

사용자 캡처 기준 문제 양상:

- 검정 라인 폭 전체가 mask로 채워지지 않음
- contour 후보가 배경 노이즈와 edge 조각으로 많이 쪼개짐
- line offset이 실제 중앙보다 크게 치우침
- `light_on_dark`의 흰색 라인 정확도는 이미 만족스러우므로, 이 경로에는 영향을 주면 안 됨

### 원인

기존 기본 `mask_strategy = "white_fill"`은 이름 그대로 밝은 라인을 채우는 경로가 중심이었다. `dark_on_light`에서는 어두운 선 내부를 직접 채우는 전용 fill 경로가 부족해, local contrast/edge 성분이 더 강하게 남았다.

또한 line 후보 폭 제한이 밝은 라인 기준과 사실상 공유되어 있었다. 2m 고도에서 10cm 정도의 굵은 검정 라인은 화면에서 폭이 넓게 보이므로, 어두운 라인에는 별도 폭 허용치가 필요했다.

정리하면 원인은 두 가지다.

- 검정 라인 내부를 value 기준으로 직접 채우는 `dark_fill` 경로가 없었다.
- `light_on_dark`와 `dark_on_light`의 morphology/width 튜닝이 완전히 분리되어 있지 않았다.

### 해결

`LineMaskBuilder`와 `LineDetector`를 polarity별로 분리했다.

`light_on_dark` 경로:

- HSV low-saturation/high-value 기반 `white_fill`
- `white_v_min`, `white_s_max`
- `fill_close_kernel`, `fill_dilate_kernel`
- `max_line_width_ratio`

`dark_on_light` 경로:

- low-value 기반 `dark_fill`
- `dark_v_max`
- `dark_fill_close_kernel`, `dark_fill_dilate_kernel`
- `dark_max_line_width_ratio`

현재 핵심 설정:

```toml
[line]
mask_strategy = "white_fill"

white_v_min = 145
white_s_max = 90
fill_close_kernel = 11
fill_dilate_kernel = 3
max_line_width_ratio = 0.22

dark_v_max = 85
dark_fill_close_kernel = 13
dark_fill_dilate_kernel = 3
dark_max_line_width_ratio = 0.34
```

`mask_strategy = "white_fill"`이더라도 `dark_on_light`에서는 내부적으로 dark fill 경로로 분기한다. 따라서 기본 전략 이름 때문에 검정 라인이 흰색 fill 로직을 타는 구조가 아니다.

`line_detector_tuner`에도 dark 전용 override를 추가했다.

```text
--dark-v-max
--dark-fill-close
--dark-fill-dilate
--dark-max-width
```

그리고 회귀 테스트를 추가했다.

- 넓은 검정 라인이 `mask_strategy = "white_fill"` 상태에서도 `dark_on_light`로 정상 검출되는지 확인
- 기존 넓은 흰색 라인 `light_on_dark` 검출이 그대로 유지되는지 확인

이 테스트는 한쪽 mode 튜닝이 다른 mode 정확도를 깨는 상황을 막기 위한 guard다.

### 결과

사용자 제공 검정 라인 캡처 기준으로 다음처럼 개선됐다.

```text
before: offset=-62.32px, contours_found=427
after:  offset=-14.07px, contours_found=8
```

수정 후 summary:

```text
detected=true
tracking_point=(468.43,404.54)
center_overlay_y=367
offset=-14.07px
angle=83.78deg
confidence=0.83
contour_points=48
contour_area=83362.50
contour_bounds=(350,59,222,626)
mask_count=1
contours_found=8
candidates_evaluated=3
```

검정 라인이 edge 조각이 아니라 하나의 넓은 line blob으로 잡히며, 배경 노이즈 contour도 크게 줄었다. `uav-onboard` OpenCV tests도 통과했다.

### line mode 해석

`light_on_dark`:

- 어두운 배경 위 밝은 라인을 검출하는 mode
- 흰색, 연노랑, 연한 회색처럼 saturation이 낮고 value가 높은 라인에 유리

`dark_on_light`:

- 밝은 배경 위 어두운 라인을 검출하는 mode
- 검정, 진한 회색, 남색처럼 value가 낮은 라인에 유리

`auto`:

- 밝은 라인 후보와 어두운 라인 후보를 모두 만들고, score가 더 좋은 contour를 선택한다.
- 라인 색이 확정되지 않은 초기 현장 점검에는 유용하다.
- 실제 대회 주행처럼 라인 색과 배경이 정해져 있으면 `light_on_dark` 또는 `dark_on_light`로 고정하는 편이 더 예측 가능하다.

line mode 옵션을 주지 않으면 현재 config 기본값을 따른다. 현재 기본 config는 `mode = "light_on_dark"`이므로, 옵션을 생략한다고 항상 `auto`가 되는 것은 아니다. `auto`를 쓰려면 CLI나 config에서 명시해야 한다.

### 유색 라인 대응

현재 구현은 특정 색 이름을 직접 인식하는 방식이 아니라, 배경 대비와 brightness polarity를 이용한다.

- 연노랑/연두/하늘색처럼 배경보다 충분히 밝고 value가 높으면 `light_on_dark`로 잡힐 가능성이 있다. 단, 채도가 높으면 `white_s_max` 조건에서 빠질 수 있다.
- 진한 초록/남색처럼 배경보다 충분히 어두우면 `dark_on_light`로 잡힐 가능성이 있다.
- 빨강, 진한 초록, 채도 높은 파랑처럼 밝기만으로 분리하기 애매한 색은 현재 light/dark mode만으로는 불안정할 수 있다.

향후 라인 색이 흰색/검정 계열이 아니라 특정 유색으로 확정되면 `red_on_dark`, `green_on_light`처럼 색마다 mode를 늘리기보다, HSV hue range를 설정으로 받는 `color_fill` 전략을 추가하는 편이 낫다. 그래야 색 추가마다 코드를 늘리지 않고 config만 바꿔 현장 색상에 대응할 수 있다.

## 30. ArUco marker가 교차점 위에 있을 때 line/intersection 판단이 오염됨

### 문제 상황

3x3 격자 경기장 이미지에서 ArUco marker가 교차점 중앙에 놓이면 marker 내부의 흰색/검정 패턴이 line mask에 섞였다.

관찰된 증상은 line polarity별로 달랐다.

- `light_on_dark` white line에서는 marker 내부의 밝은 패턴이 라인처럼 잡혀 `T`/`+` 교차점 판별이 흔들렸다.
- marker 위쪽을 지날 때 line tracing 중심이 진행 방향 라인이 아니라 좌우 branch 쪽으로 튀는 경우가 있었다.
- `dark_on_light` black line에서는 교차점 자체는 비교적 유지됐지만, marker 외곽 검정 영역과 라인이 붙으면서 marker 일부만 ArUco 후보로 잡히고 ID 4를 17처럼 오인식하는 경우가 있었다.
- GCS overlay는 온보드 telemetry를 기준으로 그리므로, 온보드가 오염된 line/marker 결과를 보내면 overlay도 그대로 잘못 표시됐다.

### 원인

기존 line/intersection 판단은 line mask에서 가장 큰 blob과 branch ray score를 안정적으로 계산하는 구조였다. 일반 교차점에서는 성능이 충분했지만, marker가 교차점 중앙에 놓이면 marker 내부 패턴이 line 후보와 같은 polarity를 가지는 문제가 생겼다.

핵심 원인은 세 가지다.

- marker 내부 패턴을 line/intersection 후보에서 제외하지 않아 branch score에 가짜 evidence가 섞였다.
- live ArUco fallback이 작은 내부 패턴 조각을 먼저 ID 후보로 받아들일 수 있었다.
- 교차점 근처 line tracing에서 lookahead 영역이 좌우 branch를 더 크게 보면 진행 방향 라인보다 branch를 선택할 수 있었다.

### 해결

기존 line tracing과 교차점 판단 성능을 깨지 않는 것을 최우선 조건으로 두고, marker가 실제로 보이는 경우에만 marker-aware 보정을 적용했다.

온보드 쪽 변경:

- `LineMaskBuilder`/`IntersectionDetector` 경로에서 fresh ArUco marker 영역을 line/intersection mask의 occlusion 영역으로 사용한다.
- marker 내부의 흰색/검정 패턴은 라인 후보가 아니라 장애물처럼 취급해 branch ray score에 섞이지 않도록 한다.
- stale marker box가 드론 이동 후 line mask를 계속 지우지 않도록, line mask occlusion에는 fresh detection만 넘긴다. telemetry 표시는 cached marker를 사용할 수 있다.
- `ArucoDetector` live fallback은 큰 ROI를 먼저 검사하고, marker 크기 plausibility를 통과하지 못하는 부분 후보는 유효 marker로 내보내지 않는다.
- partial marker로 보이는 후보는 잘못된 ID를 보내기보다 미검출로 처리한다.
- `aruco.detect_interval_frames = 3`, `fallback_max_components = 12`, `fallback_max_rois = 120`으로 live ArUco 비용과 fallback 폭주를 제한한다.
- `LineDetector`는 marker/교차점 근처에서 진행 방향 anchor band의 line run을 더 우선해 좌우 branch로 tracking point가 튀는 현상을 줄였다.

GCS 쪽 판단:

- overlay는 계속 GCS에서만 그린다.
- 온보드는 marker id/corners/center/orientation과 line/intersection telemetry만 보낸다.
- GCS는 raw camera frame 위에 marker box, line contour, intersection label, grid log를 그린다.

### 검증

검증 기준은 실제 3x3 격자 경기장, 한 칸 4m, 라인 폭 0.1m, marker 0.5m x 0.5m, 고도 약 2m 탑다운 시야를 가정했다.

검증 산출물:

```text
uav-gcs/logs/20260510-035424-aruco-marker-grid-smoke/
```

결과:

- `white_light_on_dark`: node valid / coord match / event ready `16/16`.
- `black_dark_on_light`: node valid / coord match / event ready `16/16`.
- line segment 검출은 white/black 모두 `15/15`.
- marker 4개 좌표가 `marker_coords.csv`에 저장됐다.
- white marker 위쪽 대표 crop에서 line tracking offset이 약 `1px`로 유지됐다.
- black partial marker 캡처에서는 잘못된 ID를 내지 않고 미검출 처리했다.
- full marker가 충분히 보이는 black replay crop에서는 정상 ID가 검출됐다.

### 판단

ArUco marker가 교차점 위에 있을 때 line/intersection 판단이 흔들리는 문제는 marker-aware mask, partial marker filter, 진행 방향 anchor-band scoring으로 완화됐다. 중요한 설계 기준은 기존 line-only 교차점 판단 성능을 건드리지 않는 것이다. 따라서 marker가 없거나 fresh marker가 없으면 기존 line/intersection 경로가 그대로 유지된다.

## 31. ArUco+line 동시 실행 시 packet/frame 전송이 매우 느려짐

### 문제 상황

사용자 테스트에서 white/black 모두 line-only 모드는 정상 속도로 GCS에 packet/video를 보냈다.

하지만 다음처럼 line과 ArUco를 동시에 켠 모드에서는 `--fps 12`를 지정해도 GCS에 도착하는 frame이 1FPS 미만, 체감상 약 5초에 1장 수준으로 느려졌다.

```bash
./build/vision_debug_node --config config --line-mode light_on_dark --video --fps 12
./build/vision_debug_node --config config --line-mode dark_on_light --video --fps 12
```

### 현재 상태

2026-05-10 기준으로 onboard 처리 루프의 누적 지연 원인은 해결했다.

원인은 line 자체가 아니라 ArUco generic ROI fallback이었다. ArUco marker가 화면에 없어도 direct detection 결과가 비어 있으면, live frame에서 generic fallback이 주기적으로 여러 ROI를 검사했다. 특히 `dark_on_light` 장면에서는 이 fallback이 약 530ms까지 튀며 camera pipe에 backlog를 만들고, video send가 고르게 나가지 못했다.

적용한 수정:

- `c7cfe1d Throttle live ArUco fallback workload`
  - generic full fallback을 매 ArUco detection마다 실행하지 않고 낮은 duty cycle로 제한했다.
  - ROI fallback 내부에서 ROI마다 `cv::aruco::ArucoDetector`를 새로 만들던 비용을 줄이고 detector를 재사용한다.
  - ROI detect 입력은 color crop 변환 대신 gray crop을 직접 사용한다.
- `505d409 Cap live ArUco full fallback spikes`
  - live generic full fallback ROI budget을 `full_fallback_max_rois = 4`로 줄였다.
  - live generic full fallback에서는 2배 upscale detect를 생략한다.
  - partial marker 기반 targeted fallback은 유지해, 실제 marker 조각이 보이는 경우의 복구 가능성은 남겼다.
- `14cf0dd Send every frame when debug FPS matches camera`
  - `--fps 12`가 camera 12FPS와 같을 때 strict time comparison 때문에 한 프레임씩 건너뛰는 문제를 막고, every processed frame을 video sender에 제출한다.

### 검증

로컬 GCS를 실행한 상태에서 Pi 4의 `/home/astroquad/astroquad/uav-onboard`에 `505d409`를 pull/build한 뒤, 다음 조건으로 185초 이상 관찰했다.

```bash
./build/vision_debug_node --config config --line-mode dark_on_light --video --fps 12 --gcs-ip 172.20.10.2
./build/vision_debug_node --config config --line-mode light_on_dark --video --fps 12 --gcs-ip 172.20.10.2
```

검증 로그:

```text
uav-gcs/logs/20260510-044407-pi-aruco-longrun-final/
uav-gcs/logs/20260510-044750-pi-aruco-longrun-final/
uav-gcs/logs/20260510-045552-pi-aruco-longrun-final/
uav-gcs/logs/20260510-045927-pi-aruco-longrun-final/
```

결과:

- `dark_on_light`: `2201` frames, wall `183.3s`, stdout `12.01fps`, `video_sent=2200`, `video_dropped=0`, `video_skipped=0`, `aruco_max=64.43ms`.
- `light_on_dark`: `2201` frames, wall `183.3s`, stdout `12.01fps`, `video_sent=2200`, `video_dropped=0`, `video_skipped=0`, `aruco_max=63.97ms`.
- 수정 전 `dark_on_light`에서는 full fallback spike가 약 `573ms`까지 발생했지만, 수정 후에는 12fps budget 안으로 들어왔다.
- frame 출력이 시간이 갈수록 `12fps -> 5fps -> 1fps -> 5-10초당 1장`으로 무너지는 현상은 재현되지 않았다.

### 남은 분리 이슈

GCS discovery beacon은 위 테스트 환경에서 여전히 Pi가 수신하지 못했다.

```text
no GCS discovery beacon received; falling back to 255.255.255.255:5600
```

`--gcs-ip 172.20.10.2`로 unicast를 고정하면 onboard 처리와 video send는 안정적이다. `--gcs-ip` 없이 broadcast fallback을 쓰는 경우 onboard 처리 루프는 안정적이지만, Wi-Fi broadcast packet loss 때문에 GCS 표시 안정성은 떨어질 수 있다.

따라서 이 항목의 onboard ArUco 병목은 해결됐고, 남은 작업은 GCS discovery가 왜 실패하는지 별도 네트워크/Windows 방화벽/브로드캐스트 경로로 분리해서 확인하는 것이다.

## 32. marker가 있는 교차점에서 L 판별과 line tracking 중심 보정

### 문제 상황

지연 문제를 해결한 뒤, marker가 교차점 위에 있을 때 다음 문제가 남아 있었다.

- white line에서는 `+`와 `T` 교차점은 잘 잡히지만 marker가 놓인 `L` 교차점을 제대로 판별하지 못하는 경우가 있었다.
- black line에서는 교차점 모양은 잘 판단하지만 ArUco marker가 다시 검출되지 않는 경우가 있었다.
- line tracing overlay에서 marker가 감지된 상황에도 line mask의 무게중심이나 branch contour에 끌려 tracking X가 marker 중심과 어긋날 수 있었다.

### 원인

white `L` 문제는 marker occlusion이 line mask를 지우면서 실제 `L` 중심 후보보다 다른 blob 중심 후보가 더 좋은 후보로 선택될 수 있었기 때문이다. marker가 교차점 중심에 있다는 사실을 intersection center 후보 선택에서 충분히 강하게 반영하지 못했다.

black marker 문제는 marker 외곽의 검정 영역이 black line과 붙어 기본 ArUco contour 후보가 무너지는 경우였다. 이때 교차점 mask topology는 유지되므로 `L`/`T`/`+` 판단은 가능하지만, marker ID는 나오지 않았다.

line tracking 중심 문제는 marker가 검출된 상황에서도 line detector가 기존 projection run 기준 tracking point를 유지했기 때문이다. 교차점 marker 위에서는 실제 제어 기준으로 marker 중심 X가 더 의미 있다.

### 해결

적용 커밋:

```text
f47e30e Improve marker-centered line and intersection handling
```

수정 내용:

- `IntersectionDetector`에서 marker occlusion 중심 후보를 더 강하게 반영했다.
- marker 중심 후보에서 나온 `L`/`T`/`+` type에 bonus를 부여해, marker가 교차점 중앙에 있을 때 실제 marker 주변 branch topology가 선택되도록 했다.
- `ArucoDetector`에 live 중앙부 template fallback을 추가했다. 기본 OpenCV ArUco 검출이 실패해도 중앙부의 ArUco binary pattern과 template을 비교해 black line 위 marker를 복구한다.
- `LineDetector`에 `applyMarkerCenterTracking()`을 추가했다. marker가 검출되면 tracking point의 X는 marker center X를 사용하고, Y는 기존 overlay/제어 기준처럼 camera center Y를 유지한다.
- stabilizer가 marker 중심 보정을 다시 완화하지 않도록 raw line과 filtered line 모두에 marker center 보정을 적용했다.

### 검증

사용자 제공 캡처 기준:

```text
C:\Users\mseoky\Pictures\Screenshots\white.png
C:\Users\mseoky\Pictures\Screenshots\black.png
```

`white.png` 결과:

```text
markers=1
marker_id=3 marker_center=(631.50,449.00)
tracking_point=(631.50,418.50)
ix_type=L
ix_valid=true
ix_detected=true
```

`black.png` 결과:

```text
markers=1
marker_id=1 marker_center=(582.00,386.00)
tracking_point=(582.00,420.00)
ix_type=L
ix_valid=true
ix_detected=true
```

전체 grid replay 회귀:

- white: node/event/coord `16/16`, line segment `15/15`, marker 4개 저장 유지.
- black: node/event/coord `16/16`, line segment `15/15`, marker 4개 저장 유지.

Pi 실시간 검증:

```text
uav-gcs/logs/20260510-054105-marker-centered-verify/
uav-gcs/logs/20260510-054244-marker-centered-verify/
```

- `dark_on_light`: 약 93초, stdout `12.02fps`, video drop `0`, skip `0`.
- `light_on_dark`: 약 93초, stdout `12.02fps`, video drop `0`, skip `0`.

### 남은 이슈

black line에서 ArUco marker 인식이 깜빡이면서 됐다 안 됐다를 반복하거나, 잠깐만 인식되는 문제가 아직 남아 있다. 현재 단계에서는 교차점 판단과 line tracing은 유지되지만, marker ID 검출 결과의 temporal 안정화는 다음 작업으로 남긴다.

## 33. `dark_on_light`에서 ArUco marker ID가 짧게 깜빡이는 문제

### 문제 상황

`dark_on_light`에서는 검정 라인과 ArUco marker의 검정 외곽이 붙어 보인다. 이 경우 OpenCV ArUco 검출이 marker quiet zone과 외곽 contour를 안정적으로 분리하지 못해, marker가 실제로 화면 중앙에 있어도 ID가 한두 detection 주기마다 사라질 수 있다.

ROI를 더 넓히거나 fallback ROI 개수를 늘리면 31번 문제처럼 ArUco generic fallback 비용이 다시 커질 수 있으므로, CPU 사용량을 크게 늘리는 방식은 피해야 한다.

### 해결

온보드에 `MarkerStabilizer`를 추가했다.

- ArUco가 fresh detection을 낸 프레임에서는 기존처럼 즉시 marker를 갱신한다.
- detection 주기 사이 또는 `dark_on_light`에서 일시적으로 미검출된 프레임에서는 최근 marker를 짧게 hold한다.
- fresh marker가 없을 때는 line/intersection mask용 occlusion에는 넘기지 않는다. 즉, 오래된 marker box가 line mask를 계속 지우지는 않는다.
- line tracking X 보정에는 기존 detection interval 안의 marker만 사용한다. 더 오래 hold된 marker는 marker ID telemetry 안정화용으로만 남긴다.
- hold 길이는 `[aruco] hold_frames`로 설정한다. 기본값은 `9`이며, 12FPS 기준 약 0.75초다.

핵심은 “더 많이 찾는 것”이 아니라 “이미 맞게 잡힌 ID를 짧게 안정화하는 것”이다. 따라서 ArUco ROI budget, fallback component 수, camera ROI는 늘리지 않는다.

### 검증

Windows OpenCV test build 기준:

```powershell
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
cmake --build build
ctest --test-dir build --output-on-failure
.\build\aruco_detector_tester.exe --config config --image ..\grid_images\black_grid_with_aruco_marker.png
.\build\aruco_detector_tester.exe --config config --image ..\grid_images\white_grid_with_aruco_marker.png
```

결과:

- `marker_stabilizer` 포함 onboard OpenCV tests `6/6` 통과.
- `black_grid_with_aruco_marker.png`: marker 4개 검출.
- `white_grid_with_aruco_marker.png`: marker 4개 검출.

### 현장 튜닝 기준

- marker ID가 1-2 detection 주기만 비는 정도면 `hold_frames = 9`를 유지한다.
- 드론 이동 속도가 빨라 stale marker가 오래 남는 느낌이면 `hold_frames = 6`으로 낮춘다.
- marker가 거의 한 번도 fresh detection되지 않는다면 hold로는 해결되지 않는다. 이 경우에는 ROI를 넓히기보다 marker 주변에 흰색 quiet zone/plate를 물리적으로 확보하거나, 중앙부 template fallback을 더 자주 돌리는 별도 튜닝을 검토한다.

## 34. `line_follow_node`가 `vx=0.25`를 찍는데 Gazebo 기체가 멈춘 것처럼 보임

### 문제 상황

`line_follow_node` control log에는 다음처럼 전진 명령이 계속 출력됐다.

```text
state=LINE_FOLLOW mode=GUIDED ... vx=0.25 vy=0 vz_down=0 yaw_rate=0
```

하지만 Gazebo camera 화면에서는 기체가 라인 중간이나 ArUco marker 근처에서 더 이상 앞으로 가지 않는 것처럼 보였다.

### 원인

초기 MAVLink UDP transport가 “마지막으로 packet을 보낸 peer”를 다음 command 송신 대상으로 사용했다. SITL, GCS, Mission Planner/MAVProxy가 같은 주변 port를 쓰면 GCS heartbeat가 마지막 packet이 될 수 있고, 이때 `SET_POSITION_TARGET_LOCAL_NED` command가 ArduPilot이 아니라 GCS 쪽으로 나갈 수 있었다.

겉으로는 onboard log에 `vx`가 찍히지만 실제 ArduPilot에는 command가 들어가지 않으므로, 화면에서는 멈춘 것처럼 보인다.

### 해결

- `MavlinkTransport::pinPeerFromLastMessage()` 경로를 추가했다.
- `AutopilotMavlinkAdapter`는 target system/component의 autopilot heartbeat를 확인한 뒤 command peer를 고정한다.
- GCS/Mission Planner heartbeat는 command peer를 가로채지 못하게 했다.
- control log에 `mode`, `local_xy`, `vel_ned`를 추가해 “명령을 보냈는지”와 “실제 local position이 움직였는지”를 같이 본다.
- `test_udp_mavlink_transport`에 GCS heartbeat가 peer를 hijack하지 않는 회귀 테스트를 추가했다.

### 검증

SITL에서 `local_xy`가 0m 부근에서 약 3m 부근까지 증가하고, `MARKER_APPROACH -> MARKER_HOVER -> LAND -> COMPLETE`까지 진행되는 것을 확인했다.

### 재발 확인법

- `vx=0.25`인데 `local_xy`가 거의 변하지 않으면 MAVLink command peer, SITL routing, heartbeat source를 먼저 본다.
- `mode=GUIDED`가 풀렸거나 `vel_ned`가 0으로 붙어 있으면 ArduPilot이 command를 거부하거나 받지 못하는 상태로 본다.
- 로그에 아직 `line_ahead`가 출력되면 오래된 binary를 실행 중인 것이다. `cmake --build build` 후 다시 실행한다.

## 35. 화면 위쪽 line이 끊겼다고 판단해 line-follow가 너무 일찍 멈춤

### 문제 상황

하향 camera 화면에서 현재 중심 아래/주변에는 라인이 선명하게 보이는데, 화면 위쪽이나 marker 주변이 끊긴 것처럼 보이면 기체가 더 이상 전진하지 않는 경우가 있었다.

### 원인

초기 상태머신은 `line_ahead` 같은 전방 라인 존재 조건을 별도로 보려 했다. marker나 화면 crop 때문에 위쪽 line segment가 부분적으로 끊겨 보이면, 실제로는 계속 따라갈 라인이 남아 있어도 “앞에 라인이 없다”고 판단할 수 있었다.

### 해결

라인 추종 중에는 `line_detected`가 true인 동안 무조건 계속 전진/추종하도록 단순화했다.

- `line_ahead` gating 제거.
- 라인이 완전히 잡히지 않을 때만 line lost timer를 진행.
- ArUco marker가 보이면 `MARKER_APPROACH`로 들어가되, marker가 잠깐 사라지고 line은 보이면 line-follow fallback으로 계속 진행.
- line이 더 이상 잡히지 않고 marker도 없으면 안전 착륙한다.

### 판단

현재 축소 MVP에서는 “전방에 라인이 이어지는지”보다 “하향 camera에서 추종 가능한 라인이 현재 보이는지”가 더 안정적인 기준이다. full grid/snake 단계에서 분기점 판단이 필요해지면 line/intersection detector 출력으로 별도 state를 추가한다.

## 36. `vision_debug_node`와 `line_follow_node`를 동시에 실행해 GCS telemetry/video가 꼬임

### 문제 상황

`vision_debug_node --video`와 `line_follow_node --video`를 같이 실행하면 GCS 화면/telemetry가 중간에 멈추거나, 서로 다른 source의 overlay가 섞여 보이는 것처럼 혼란스러운 현상이 있었다.

### 원인

두 프로그램 모두 Gazebo camera frame을 읽고 같은 GCS telemetry/video port로 송신할 수 있다. GCS는 mission source를 독점적으로 구분해 합성하는 구조가 아니므로, 두 onboard-like sender를 동시에 띄우면 frame/vision log/telemetry가 경쟁한다.

### 해결

운영 기준을 분리했다.

- 비행 없는 vision-only smoke: `vision_debug_node --target sitl --vision gazebo --video`
- 실제 SITL 비행 smoke: `line_follow_node --target sitl --vision gazebo --video`

`line_follow_node --video`는 camera capture, vision 처리, GCS telemetry, GCS video 송신을 모두 포함한다. 비행 중에는 `vision_debug_node`를 별도로 실행하지 않는다.

## 37. `LAND` 진입 후 GCS 영상이 멈춤

### 문제 상황

ArUco marker에서 3초 hover 후 `LAND`로 들어가면 GCS 영상이 멈췄다. 착륙 시작부터 바닥에 닿기 전까지의 상황을 GCS에서 볼 수 없었다.

### 원인

초기 구현은 mission state가 `LAND/COMPLETE`로 넘어가면 control loop가 종료되면서 frame capture와 debug video publish도 함께 끝났다. 따라서 ArduPilot은 착륙 중이어도 GCS 영상 송신은 이미 중단됐다.

### 해결

`LAND` command를 보낸 뒤에도 autopilot disarm 또는 timeout까지 frame을 계속 읽고 GCS video/telemetry를 송신하는 landing video drain 구간을 추가했다.

정상 로그 예:

```text
[mission] LAND
[mission] LAND reason=marker hover complete
[gcs] landing_video_frames=84
[mission] COMPLETE
```

### 판단

Debug video는 mission-critical 경로는 아니지만, 착륙 관제에는 필요하다. 앞으로도 state machine 종료와 video publisher 생명주기는 분리해서 본다.

## 38. 이륙 전에는 GCS 영상이 안 나오고 takeoff 후에야 비전이 시작됨

### 문제 상황

`line_follow_node` 실행 후 heartbeat, GUIDED, arm, takeoff가 진행되는 동안에는 GCS camera 화면이 갱신되지 않고, 이륙 후 line-follow control이 시작될 때부터 영상이 나왔다.

### 원인

초기 `line_follow_node`는 takeoff sequence 이후 control loop에 들어가면서 frame capture와 GCS publish를 시작했다. 라인 추종에는 이륙 후 영상만 필요하지만, 운용 관점에서는 시동 직후부터 camera/GCS link 상태를 확인하는 편이 맞다.

### 해결

GCS publisher를 연 직후 startup video streaming 구간을 추가했다. 이 구간은 MAVLink heartbeat/arm/takeoff가 진행되는 동안에도 frame을 읽고 GCS로 보낸다.

정상 로그 예:

```text
[gcs] startup video streaming until line-follow control starts
[gcs] startup_video_frames=200
[mission] LINE_FOLLOW
```

### 주의

startup video는 관제 편의 기능이다. 이륙 전부터 비전 결과가 보이더라도 mission state는 takeoff 완료 후 `LINE_FOLLOW`부터 제어에 사용한다.

## 39. Gazebo camera zoom/FOV 튜닝 위치

### 문제 상황

실기체 IMX519 camera는 같은 고도에서도 Gazebo 기본 camera보다 라인이 더 굵게 보였다. 반대로 zoom-in을 너무 많이 하면 line은 크게 보이지만 ArUco marker 접근/landing 상황에서 시야가 좁아져 운용이 불편했다.

### 설정 위치

Gazebo Iris 하향 camera FOV는 다음 파일에서 조정한다.

```text
uav-onboard/sim/gazebo/models/iris_with_downward_camera/model.sdf
```

핵심 값:

```xml
<horizontal_fov>1.15</horizontal_fov>
```

튜닝 기준:

- 값이 작을수록 zoom-in. 라인/marker가 더 크게 보인다.
- 값이 클수록 zoom-out. 더 넓은 영역이 보이고 라인/marker는 작게 보인다.
- line/marker 물리 크기는 바꾸지 않는다. 실기체 camera에 맞출 때는 camera FOV만 조정한다.
- Gazebo model SDF 변경 후에는 Gazebo/SITL을 재시작한다. C++ rebuild는 FOV-only 변경에는 필요 없다.

## 40. Raspberry Pi 4 + Pixhawk1에서 `line_follow_node`를 바로 쓸 수 있는지

### 현재 판단

이 항목은 초기 판단 기록이며, 바로 다음 41번 항목의 serial bring-up 작업으로
상태가 바뀌었다. 현재 기준으로는 native `SerialMavlinkTransport`가 구현되어
있고, `line_follow_node --target pixhawk1`는 no-arm smoke와 명시적
`--allow-arm-takeoff` guard를 가진다.

가능한 것:

- Raspberry Pi 4 + IMX519 `rpicam-vid` 기반 camera capture.
- onboard line/ArUco/intersection vision 처리.
- GCS telemetry/video 송신.
- `line_follow_node --target sitl --vision gazebo --video`로 검증된 mission/control 구조.
- Pixhawk1 USB/serial MAVLink heartbeat/probe/smoke path.

여전히 바로 하면 안 되는 것:

- 실제 arm/takeoff는 `--allow-arm-takeoff` 없이는 열리지 않는다.
- RC takeover, battery, local estimate, motor order, prop direction, manual hover
  gate를 통과하지 않은 상태에서 자동비행을 시작하지 않는다.
- `grid_mission_node --target pixhawk1`는 현재 real arm/takeoff grid mission
  path가 아니며 `--no-arm` smoke만 기준으로 본다.

현재 smoke 예시:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

### 실기체 전 필수 gate

- MTF-01 optical flow/range가 ArduPilot EKF local estimate에 안정적으로 들어오는지 확인한다.
- GUIDED mode에서 `LOCAL_POSITION_NED`가 정상 갱신되는지 확인한다.
- props-off 상태에서 heartbeat, mode change, arm inhibit, land, RC takeover를 확인한다.
- GCS video는 broadcast보다 unicast `--gcs-ip <laptop-ip>`를 우선 사용한다.
- 실기체 IMX519 시야에 맞게 camera focus/lens/FOV와 line width를 다시 튜닝한다.

결론: vision/GCS와 native Pixhawk serial smoke는 이어서 쓸 수 있다. 다만
실제 자동비행은 여전히 explicit arm/takeoff flag와 모든 실기체 gate를
통과한 뒤에만 진행한다.

## 41. Pixhawk USB serial bench bring-up과 MTF-01 local estimate 확인

### 문제 상황

실제 Pixhawk1은 Raspberry Pi 4의 USB-A 포트와 Pixhawk USB 포트로 연결되어 있었다. 프로펠러는 제거되어 있었지만, 모터 전원은 인가된 상태였다.

초기 조사에서는 Python MAVLink probe로 heartbeat와 parameter list는 읽혔지만, 다음 메시지가 확인되지 않아 GUIDED velocity 기반 line-follow gate를 통과하지 못한 상태로 보였다.

```text
LOCAL_POSITION_NED
DISTANCE_SENSOR
OPTICAL_FLOW
OPTICAL_FLOW_RAD
```

또한 `line_follow_node`는 serial target을 인식하더라도 serial transport가 미구현이라 종료했고, serial이 열리면 기존 mission path가 GUIDED/arm/takeoff로 바로 이어질 위험이 있었다.

### 원인

직접적인 코드 원인은 native serial MAVLink transport와 no-arm probe 경로가 없었던 것이다.

설정 면에서는 Pixhawk EKF source가 no-GPS optical-flow MVP에 맞지 않았다.

관찰된 차이:

```text
EK3_SRC1_POSXY=3
EK3_SRC1_VELZ=3
```

no-GPS optical-flow bench target으로는 다음 값이 필요했다.

```text
EK3_SRC1_POSXY=0
EK3_SRC1_VELZ=0
```

### 해결

적용 commit:

```text
030a476 Add Pixhawk serial MAVLink bench tools
```

구현 내용:

- `SerialMavlinkTransport` 추가.
- `mavlink_probe` no-arm bench tool 추가.
- `mavlink_motor_test` 저출력 motor test tool 추가.
- `line_follow_node`에 real serial safety guard 추가.
- `line_follow_node --mavlink-smoke` 추가.
- `runtime.pixhawk1.toml`을 stable USB by-id path로 변경.
- `config/pixhawk1_usb.params`, `config/pixhawk1_usb.expected.toml` 추가.

Pi 배포/검증:

```bash
cd /home/astroquad/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

결과:

```text
13/13 tests passed
```

Pixhawk parameter 적용:

```bash
./build/mavlink_probe --config config --target pixhawk1 --duration-ms 12000 \
  --param-file config/pixhawk1_usb.params \
  --apply-params --i-understand-this-writes-pixhawk-params
```

적용된 값:

```text
EK3_SRC1_POSXY: 3 -> 0
EK3_SRC1_VELZ: 3 -> 0
```

### 검증

No-arm probe:

```bash
./build/mavlink_probe --config config --target pixhawk1 --duration-ms 12000
```

대표 결과:

```text
heartbeat: system=1 component=1 mode=STABILIZE(0) armed=false
local_position_ned: x≈-1.97 y≈-2.79 z≈101.7 vx≈0 vy≈0 vz≈0.005
range: distance_sensor_m≈1.54 rangefinder_m≈1.54
optical_flow: quality≈139 ground_distance_m≈1.54
rc: channels=0 rssi=255
ekf: flags=0x16f
```

Strict local estimate gate:

```bash
./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 12000 --strict-local-estimate
```

결과:

```text
exit status 0
```

`line_follow_node` serial smoke:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

결과:

```text
[mavlink] safety smoke mode: no mode/arm/takeoff/velocity commands
[mavlink] heartbeat ok system=1 component=1 mode=STABILIZE armed=false
[mavlink-smoke] ... local_xy=(...) vel_ned=(...) range=...
```

### 남은 이슈

- `RC_CHANNELS count=0`: RC receiver input/takeover는 아직 MAVLink에서 확인되지 않았다.
- `BATT_MONITOR=0`: battery voltage/current telemetry가 없다.
- `ARMING_CHECK=0`: bench에서는 편하지만 flight-ready safety 상태가 아니다.
- `mavlink_motor_test`는 구현됐지만 실제 모터 회전은 물리 안전 확인이 필요하므로 자동 실행하지 않았다.

### 다음 확인

모터 회전 테스트는 사용자가 기체 주변을 정리하고 손/케이블을 치운 뒤 직접 실행한다.

```bash
cd /home/astroquad/astroquad/uav-onboard
./build/mavlink_motor_test --config config --target pixhawk1 \
  --motor 1 --percent 5 --seconds 1 --props-removed
```

예상:

- Pixhawk heartbeat가 출력된다.
- motor 1이 약 1초 동안 낮은 출력으로 돈다.
- `COMMAND_ACK result=0`이면 ArduPilot이 motor test command를 accepted한 것이다.

모터가 돌지 않으면 확인할 것:

- Pixhawk safety switch 상태.
- ESC signal/power wiring.
- MAIN OUT motor order.
- ESC calibration 여부.
- ArduPilot `MOT_PWM_TYPE`, `MOT_PWM_MIN/MAX`, frame class/type.
- IOMCU/safety 관련 statustext.

## 42. TELEM2 MAVLink 구성에서 `RC_CHANNELS`가 보이지 않음

### 문제 상황

Pixhawk1 TELEM2에는 조종기/안테나 계통이 연결되어 있고 Mission Planner와 조종기 조작으로 모터 반응은 확인되었다. 하지만 companion computer에서 MAVLink로 확인한 RC 상태는 계속 비어 있었다.

```text
rc: channels=0 rssi=0
```

`line_follow_node`의 RC gate도 같은 이유로 통과하지 못했다.

### 원인 판단

TELEM2는 현재 MAVLink 용도로 쓰는 설정이 맞고, 조종기 입력은 일반적인 Pixhawk `RCIN` 입력이 아니라 Mission Planner/RC 시스템을 통한 MAVLink override 또는 별도 경로로 들어오는 것으로 보인다. 이 경우 Pixhawk가 companion link로 `RC_CHANNELS`를 안정적으로 내보내지 않을 수 있다.

### 임시 해결

실비행 직전 테스트를 위해 `line_follow_node`에 다음 강제 옵션을 추가했다.

```bash
--unsafe-assume-rc-present
```

이 옵션은 MAVLink RC gate만 우회한다. 실제 serial arm/takeoff는 여전히 `--allow-arm-takeoff`가 있어야 실행된다.

### 상태

미해결. 조종기 수동 개입 자체는 가능해 보였지만, companion software가 `RC_CHANNELS`로 조종기 존재를 검증하는 문제는 남아 있다. 추후 RC receiver 연결 방식, ArduPilot RC input/override 설정, `RC_CHANNELS`/`RC_CHANNELS_OVERRIDE` 경로를 별도로 정리해야 한다.

## 43. 배터리 telemetry가 잡히지 않음

### 문제 상황

초기 Pixhawk probe에서 배터리 voltage/current가 정상적으로 나오지 않았고, strict battery check를 통과하지 못했다.

### 원인

Pixhawk parameter에서 battery monitor가 꺼져 있었다.

```text
BATT_MONITOR=0
```

### 해결

Pixhawk1 analog power module 기준으로 battery monitor parameter를 적용하고 reboot했다.

```text
BATT_MONITOR=4
BATT_VOLT_PIN=2
BATT_CURR_PIN=3
BATT_VOLT_MULT=10.1
BATT_AMP_PERVLT=17
BATT_AMP_OFFSET=0
```

### 결과

`mavlink_probe --strict-battery`가 통과했고, bench 상태에서 다음처럼 전압/전류/잔량이 확인되었다.

```text
battery: voltage_v=11.42 current_a=0.04 remaining_pct=99
```

### 상태

해결됨. 단, 실제 배터리 잔량 퍼센트와 전류 정확도는 power module 실측 캘리브레이션으로 추후 보정할 수 있다.

## 44. `line_follow_node`에서 라인 색상 모드를 직접 지정하지 못함

### 문제 상황

`vision_debug_node`는 `--line-mode light_on_dark|dark_on_light|auto`를 지원했지만, 실비행에 사용할 `line_follow_node`는 같은 옵션이 없었다. 콘크리트 바닥/테이프 조건에서는 라인 색상 모드 전환이 즉시 필요했다.

### 해결

`line_follow_node`에 `--line-mode <mode>` 옵션을 추가하고, runtime log에 적용된 mode가 출력되도록 했다.

예:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --vision rpicam --line-mode dark_on_light
```

### 상태

해결됨.

## 45. IMX519 초점이 흐릿함

### 문제 상황

첫 실비행 후 GCS 영상과 vision debug 화면에서 카메라 초점이 흐릿해 라인/마커 인식 신뢰도가 낮아질 수 있었다.

### 확인

Pi에서 IMX519 focus sweep을 수행했고 Laplacian sharpness 기준으로 `focus_absolute=4095`가 가장 선명했다.

대표 결과:

```text
4095: sharpness 6.96
3600: sharpness 6.08
3200: sharpness 3.45
1984: sharpness 1.29
```

### 해결

`config/vision.toml`의 focus 값을 `4095`로 조정했다. 또한 focus motor 설정 직후 바로 `rpicam-vid`를 실행하면 camera timeout이 발생할 수 있어, `RpicamMjpegSource`에서 focus 설정 후 500ms 대기하도록 수정했다.

### 상태

해결됨. Pi에서 `vision_debug_node --vision-smoke-count`와 전체 test가 통과했다.

## 46. 검정 절연 테이프 라인이 햇빛 반사 때문에 잘 잡히지 않음

### 문제 상황

실외 회색 콘크리트 바닥에 검정 절연 테이프를 붙여 테스트했지만, 테이프 표면이 매끄러워 햇빛을 강하게 반사했다. 결과적으로 회색 바닥과 검정 테이프가 둘 다 밝게 보이며 라인 트레이싱이 제대로 시작되지 않았다.

### 원인 판단

소프트웨어 threshold 문제만이 아니라 라인 재질/조명 조건 문제다. 반사광이 들어오면 `dark_on_light` 조건에서도 검정 테이프가 밝은 영역으로 인식될 수 있다.

### 조치

코드 쪽에서는 라인 confidence가 낮거나 라인을 잃었을 때 전진 명령을 끊고 XY hold로 들어가도록 보강했다. 물리 환경 쪽에서는 무광 재질, 더 넓은 폭, 바닥과 확실히 대비되는 색상으로 라인을 다시 준비해야 한다.

### 상태

부분 해결. 소프트웨어 fallback은 보강됐지만, 실제 라인 재질/색상 문제는 미해결이며 다음 실비행 전 물리 라인을 바꿔 검증해야 한다.

## 47. 라인 미검출 후 착륙이 사선으로 진행됨

### 문제 상황

실비행에서 기체는 목표 고도 약 2m까지 이륙했지만 라인을 잡지 못했고, 약 3초 후 landing으로 넘어갔다. 이때 제자리 착륙이 아니라 한쪽 방향으로 사선 이동하며 내려왔다.

### 원인

기존 구현은 라인 추종 중에는 body velocity setpoint를 보냈지만, 이륙 후 라인이 없거나 착륙 중인 상태에서는 "속도 0"에 가깝게만 명령했다. 이는 Loiter 같은 위치 hold가 아니며, optical-flow local estimate가 일시적으로 흔들리는 상황에서는 XY 위치를 적극적으로 붙잡지 못한다.

비행 로그에서도 다음 흐름이 보였다.

```text
EKF3 IMU1 stopped aiding
EKF3 IMU1 started relative aiding
EKF3 IMU1 fusing optical flow
```

### 해결

`line_follow_node`에 LOCAL_NED XY hold anchor를 추가했다.

- 실기체 serial target에서는 arm/takeoff 전에 local position, range, optical-flow quality, EKF relative aiding이 1.5초 이상 안정적이어야 한다.
- 이륙 중에도 takeoff 위치의 XY anchor를 유지한다.
- 목표 고도 도달 후 기본 2초간 XY hold settle을 수행한다.
- 라인이 감지되지 않거나 confidence가 낮으면 현재 위치를 anchor로 잡고 body velocity 대신 LOCAL_NED XY hold setpoint를 보낸다.
- 마커 hover도 XY hold로 수행한다.
- 착륙도 우선 GUIDED descent + XY hold로 내려가며, 실패하면 LAND mode로 fallback한다.

### 상태

구현 완료, 로컬 빌드/테스트 통과. 실제 Pi pull/build 및 실비행 재검증은 아직 필요하다.

## 48. 착륙 후 모터가 자동으로 멈추지 않음

### 문제 상황

첫 실비행에서 기체가 착륙했지만 모터가 계속 돌았고, Mission Planner에서 수동 disarm을 수행해야 했다.

### 원인

기존 landing flow는 `LAND` mode 진입 후 disarm 완료를 충분히 보장하지 않았다. Rangefinder 기준으로 지면에 닿았다고 판단할 수 있는 상태에서도 별도 disarm retry가 약했다.

### 해결

착륙 후 다음 로직을 추가했다.

- rangefinder가 0.30m 이하이고 local velocity가 충분히 작으면 near-ground stable 상태로 판단한다.
- 해당 상태가 2초 이상 유지되면 `MAV_CMD_COMPONENT_ARM_DISARM`으로 disarm을 명시적으로 요청한다.
- 최종 near-ground 상태에서 추가 disarm attempt를 수행한다.
- 최신 구현에서는 GUIDED descent 중 near-ground stable 상태가 되면 descent 속도를 0으로 보내고 disarm한다.

### 상태

구현 완료, 로컬 테스트 통과. 실기체에서 착륙 후 자동 disarm까지 재검증 필요.

## 49. `line_follow_node` GCS 영상이 이륙 후 멈추거나 끊김

### 문제 상황

`line_follow_node --video` 실행 시 처음 1-2초 정도는 GCS 영상이 나오다가 이륙 후 화면이 멈춘 것처럼 보였다. `vision_debug_node --fps 12`에서는 onboard 송신 로그는 계속 나오지만 GCS 쪽 frame이 뚝뚝 끊겼다.

### 원인 판단

복합 이슈다.

- 기존 `line_follow_node`는 takeoff/control 전후 video 송신 생명주기가 짧아 landing 구간에서 화면이 끊길 수 있었다.
- GCS video는 UDP MJPEG chunk 기반이라 Wi-Fi 품질, broadcast/unicast, FPS, JPEG 크기에 민감하다.
- 실비행 중 control loop와 vision publish가 같은 프로세스에서 돌기 때문에 FPS를 과하게 잡으면 frame skip/drop이 생길 수 있다.

### 해결/개선

- `line_follow_node`에 `--fps <n>` 옵션을 추가해 GCS debug video FPS를 직접 낮출 수 있게 했다.
- startup video streaming을 추가해 heartbeat/arm/takeoff 중에도 영상 송신을 시작한다.
- landing 중에도 disarm 또는 timeout까지 video drain을 계속 수행한다.
- mission/landing log에 `video_sent`, `video_drop`, `video_skip`, `video_fail` counters를 출력한다.

### 상태

부분 해결. `--fps 12` 기준 송신 경로는 동작하지만, 실제 GCS stutter는 네트워크/FPS/JPEG 품질 영향을 계속 받는다. 다음 실비행에서는 `--fps 8` 또는 `--fps 10`, 가능하면 GCS IP unicast 지정으로 재검증한다.

## 50. `ARMING_CHECK=0` 상태

### 문제 상황

Mission Planner 로그에 다음 warning이 남았다.

```text
Warning: Arming Checks Disabled
```

### 원인

Bench bring-up과 초기 테스트를 위해 ArduPilot arming check가 꺼져 있다.

### 상태

미해결. 당장 실험 속도는 빠르지만 실비행 안전 관점에서는 좋지 않다. 이번 MVP 비행 이후에는 최소한 필요한 arming check를 다시 켜고, 어떤 check를 의도적으로 끄는지 별도 목록으로 관리해야 한다.


## 50. Gazebo SITL 라인 추종 중 좌우 헤드쉐이크 진동

최종 업데이트: 2026-05-17

### 문제 상황

Gazebo SITL 라인 추종 테스트에서 드론이 라인 위에 정렬된 상태에서도 1–2초 주기로 좌우 진동("헤드쉐이크")이 발생했다. 전진 속도가 높을수록 진동이 커졌고, 라인 검출 각도가 목표값 근처에서 약간 흔들릴 때 특히 두드러졌다.

### 원인 분석

세 가지 독립적인 원인이 복합적으로 작용했다.

**1. `angle_yaw_kp` 과대 (기존 기본값: 1.0)**

라인 각도 오차에 대한 비례 게인이 yaw_rate 명령으로 직접 연결된다. ±5°(≈0.087 rad) 수준의 미세한 각도 흔들림이 ±0.087 rad/s 수준의 yaw 명령을 만들어 냈다. 제어 루프 20 Hz에서 기체가 돌면 카메라 각도가 바뀌고, 컨트롤러가 반대 방향으로 보정하면서 전형적인 P게인 헌팅이 발생했다.

**2. `offset_yaw_kp` 교차 결합 (기존 기본값: 0.25)**

드론이 옆으로 밀릴 때 이 게인이 횡방향 오차에 비례하는 추가 yaw 성분을 더했다. 의도는 "라인 쪽으로 기수를 돌린다"였지만 실제 동작은 두 번째 피드백 경로를 형성했다: 횡방향 오차 → 추가 yaw → 카메라 각도 변화 → 반대 방향 각도 오차 → 반대 yaw 명령 → 진동. `offset_kp`가 `vy`로 횡방향 드리프트를 이미 보정하는 상황에서 이 결합 항은 잉여이며 루프를 조건부 불안정하게 만들었다.

**3. 출력 스무딩 없음**

EMA alpha 기본값이 1.0(스무딩 없음)이었다. 비전 15 fps에서 두 프레임 연속으로 각도 잡음이 나타나면 다음 틱에서 full-magnitude yaw 명령 반전이 그대로 비행 컨트롤러에 전달됐다.

### 해결 방법

모든 변경은 하위 호환성을 유지한다. 기존 Gazebo 동작은 기본값 베이스라인으로 보존되고, 새 기능은 TOML config 옵트인으로만 활성화된다.

**게인 감소 (`config/mission.toml`, `config/runtime.sitl.toml`)**

```toml
[line_controller]
angle_yaw_kp       = 0.5    # 기존 1.0 → 절반으로 축소, 헌팅 진폭 감소
offset_yaw_kp      = 0.0    # 기존 0.25 → 교차 결합 비활성화 (vy만 사용)
max_yaw_rate_rad_s = 0.35   # 기존 0.6 → 최악의 경우 명령 상한 축소
```

**EMA 출력 스무딩** (`GuidedVelocityController` — 새 상태 필드 추가)

```toml
output_ema_alpha = 0.4   # 0 = 고정, 1.0 = 스무딩 없음(레거시 기본값)
```

최종 `vy`와 `yaw_rate` 명령에 저역 통과 필터를 적용한다. alpha=0.4, 20 Hz 기준으로 2 Hz 이상의 고주파 반전이 약 80% 감쇠된다. 컨트롤러 C++ 기본값은 1.0으로 유지되어 TOML 옵트인 없이는 동작이 바뀌지 않는다.

**레이트 리미팅** (기본값 비활성, 실제 비행용)

```toml
max_lateral_rate_mps        = 0.025   # 50 ms 스텝당 vy 변화 상한 → ~0.5 m/s²
max_yaw_rate_change_rad_s   = 0.025   # 스텝당 yaw_rate 변화 상한
```

EMA 필터와 독립적으로 기체 프레임 가속도를 제한한다. 기본값 0.0(비활성)으로 SITL에 영향 없음.

**실비행 보수적 프로파일** (`config/runtime.pixhawk1.toml`)

추가로 게인을 낮추고, 레이트 리미팅을 활성화하고, 전진 속도를 줄인 전용 블록을 별도 파일로 관리한다.

### 코드 변경 요약

| 파일 | 변경 내용 |
|---|---|
| `src/control/GuidedVelocityController.hpp` | config 구조체에 `output_ema_alpha`, `max_lateral_rate_mps`, `max_yaw_rate_change_rad_s`, `forward_confidence_scale` 추가; `updateLine`/`update`/`stop`을 non-`const`로 변경; `resetSmoothing()`, `smoothOutput()` 추가 |
| `src/control/GuidedVelocityController.cpp` | `smoothOutput()` 구현 (EMA + 레이트 리미팅); 라인 소실 시 스무더 상태를 0으로 감쇠; 컨피던스 스케일 전진 속도 |
| `tools/line_follow_node.cpp` | `[line_controller]` 4개 새 키 TOML 파싱 추가 (기존에는 묵묵히 무시됨); 기동 시 게인 배너 출력; `--profile-vision` 플래그 추가; 시뮬/실기 추정기 경로 주석 추가 |
| `config/mission.toml` | `[line_controller]` 기본값 업데이트, 모든 튜닝 필드 인라인 주석 추가 |
| `config/runtime.sitl.toml` | 시뮬 전용 오버라이드 (`angle_yaw_kp=0.5`, `offset_yaw_kp=0.0`, `output_ema_alpha=0.4`) |
| `config/runtime.pixhawk1.toml` | 보수적 실비행 전용 블록 추가 |
| `src/vision/LineMaskBuilder.cpp` | 지연 `ensure_hsv()` 클로저로 프레임당 중복 `cvtColor` 호출 최대 2회 제거 |
| `tests/test_guided_velocity_controller.cpp` | 5개 명명 서브테스트 추가 (레거시 수학, EMA 감쇠, 레이트 리미팅, 컨피던스 스케일링, 소실 시 스무더 감쇠) |

### 발견된 버그: TOML 키가 실제로는 읽히지 않았음

이전 WIP diff에서 config 구조체에 필드를 추가하고 TOML 파일에 값을 썼지만, `loadRuntimeConfig()`와 `applyRuntimeOverlay()` 양쪽에 `.value_or()` 호출을 추가하는 것을 빠뜨렸다. `output_ema_alpha`, `max_lateral_rate_mps`, `max_yaw_rate_change_rad_s`, `forward_confidence_scale` 네 키가 런타임에 묵묵히 무시되어 TOML에 뭘 써도 C++ 기본값이 유지됐다. 두 로더 위치에 파싱 블록을 추가해서 수정했다.

### 검증 결과

- `cmake --build build-codex --target onboard_core` — 경고 0개, 클린 빌드
- `cmake --build build-codex --target test_guided_velocity_controller` — 클린 빌드
- `./build-codex/tests/test_guided_velocity_controller` — 5개 서브테스트 모두 통과 (exit 0)
- `LineMaskBuilder.cpp` OpenCV 헤더 포함 컴파일 — 오류 없음

### 미해결 / 알려진 리스크

**`LineStabilizer` 180° 각도 플립**: 스태빌라이저가 프레임 간 라인 각도를 EMA 보간한다. 라인 검출이 ~90°에서 ~270°(축 대칭 앨리어스)로 점프하면 보간이 0°를 지나면서 대규모 순간 각도 오차가 생기고, 짧은 yaw 스파이크가 발생한다. 주요 진동 원인은 아니지만 실비행 적극 튜닝 전에 수정할 것. 완화책: 블렌딩 전 각도 차이를 [−90°, +90°] 범위로 클램프.

**MTF-01 광학 흐름 품질**: 컨트롤러의 `min_confidence` 게이팅은 비행 컨트롤러의 광학 흐름 품질 플래그와 별개다. 실기에서는 `OPTICAL_FLOW_RAD.quality`가 운용 환경에서 안정적으로 ≥ 50을 유지하는지 `forward_mps`를 높이기 전에 벤치에서 검증할 것.

**부호 규약**: MAVLink `SET_POSITION_TARGET_LOCAL_NED` 기체 프레임 부호가 펌웨어 버전에 따라 다를 수 있다. 첫 야외 테스트 전 벤치에서 `invert_lateral`과 `invert_yaw` 플래그를 검증할 것.

---

## 51. `grid_mission_node` Cycle 8 — `alt` 표시 혼란 + LINE_ENTER timeout / yaw drift

**증상**:
- log의 `alt`가 3.3~3.48 m로 표시되어 알고리즘 결함처럼 보임 (실제 비행 고도는 ground 기준 ~2 m).
- vertiport→첫 grid 교차점 진입 도중 yaw가 천천히 회전(1.57→1.40)하면서 라인 이탈, LINE_ENTER 5 s timeout → EmergencyLand.

**원인**:
- `alt` 필드가 LOCAL_NED z(takeoff origin 기준)였고, 컨트롤러는 `distance_sensor.value_or(local_altitude)`로 rangefinder 우선 사용. 단상(0.7 m)에서 이륙했으므로 단상 벗어나면 LOCAL_NED z만 점프해 표시상 ↑.
- LineDetector contour 전체에 `cv::fitLine`을 적용하므로 T-shape 교차점이 프레임 상단에 들어오는 partial-Unknown 구간 동안 fused 각도가 반환 → LineFollow가 그 각도를 따라가 yaw drift.

**해결**:
- log 포맷: `agl=2.00 lz=3.48` (rangefinder=AGL, local_z 분리). ceiling 체크도 rangefinder 우선 + LOCAL_NED fallback.
- `populateLineInputs`에서 `intersection.valid`(Unknown/Straight/L/T/+ 어떤 타입이든)이면 `line_angle_error_rad=0`, `line_center_error_norm=0`로 freeze → yaw가 fused angle을 따라가는 것을 차단.
- `IntersectionDecision`에 `node_record_y_min/max` (0.40 / 0.65) 게이트 추가 — 교차점이 정확히 카메라 중앙 zone에 들어왔을 때만 NodeRecord 발사.
- LineEnter timeout 5 s → 10 s, idle counter는 라인이 잡힌 마지막 시점부터 측정.

---

## 52. Tracker가 동일 (0,0) 교차점을 4번 advance해 가짜 노드 생성

**증상**:
- 드론은 (0,0)에서 약 1.5 m만 전진했는데 GCS 그리드에 `nodes=5 current=(0,-4)` 표시.
- 로그에 `idec=node_record` 이벤트가 t=20.82/22.41/24.01에 약 1.5 s 간격으로 동일 시그니처(type=T cy=0.40~0.61 bm=0x0D)로 반복 발사됨.

**원인**:
- `IntersectionDecisionEngine.record_node_once_frames`(18 frames, ~1.5 s)는 frame-level dedup이지 거리 dedup이 아님.
- `GridCoordinateTracker.update()`는 `decision.event_ready`만 보고 `node_advance_min_frames`(4 frames) 후엔 무조건 `current_coord_` advance. mission 레이어의 distance gate는 `intersections_recorded_`만 막을 뿐 tracker coord는 자유롭게 advance했음.

**해결 — peek/commit 분리**:
- `tracker.update(...)`는 event 메타데이터만 채워 반환하고 `current_coord_` advance를 보류 (peek).
- 새 `tracker.commitAdvance(event)`를 mission이 distance gate + lockouts 모두 통과했을 때만 호출.
- `GridMissionOutput`에 `commit_tracker_advance` 플래그 추가. mission이 "진짜 노드"로 판정 시 true.

**부수 변경 — handleSnakeRecordNode stale node_event**:
- dwell 시간 후 boundary 판정에 `in.node_event.grid_branch_mask`를 매 tick 다시 읽었으나 lockout 활성 후 valid=false라 mask=0 → 항상 boundary로 오판 → SnakeStopAtCenter → SnakeComplete → LAND.
- 해결: `handleSnakeForward`에서 SnakeRecordNode 진입 직전 `last_node_grid_branch_mask_`, `last_node_front_open_`을 latch하고 dwell 종료 후 cached 값으로 판정.

---

## 53. 마커가 1-2 frame false-positive로 commit되는 문제

**증상**:
- (0,0) 근처에서 mks=1 (id=7) 한순간 잡힘 → `MarkerRegistry`에 commit. 실제로는 교차점 십자 패턴의 ArUco 오인식.
- `markers_expected` 카운트 오류, `marker_hover_active_` 잘못 트리거.

**해결 변천**:
- Cycle 9: mission 레이어에 `marker_candidate_count_` 추가, 같은 ID가 연속 N 프레임 보일 때만 commit (`marker_observation_min_frames=3`).
- Cycle 11: 임계값 부족 — 3 frames(150 ms)는 OpenCV ArUco가 교차점 십자/T를 일시 안정 인식하는 시간보다 짧음. 6 frames(300 ms)로 상향 + `min_marker_perimeter_rate` 0.03→0.05로 보수화 (이미지 둘레의 5% 미만 후보 거부).
- Cycle 16: sliding window로 교체 — 최근 N(8) frames 중 동일 ID가 M(4) 이상 등장하면 commit, 윈도우 내 다른 ID 섞이면 flush.

---

## 54. Boundary 워치독이 너무 일찍 발동해 교차점 중심 도달 전 정지

**증상**:
- 워치독이 `TurnConfirm` (cy=0.34) 단계에 트리거 → 교차점이 프레임 상단에 막 보이기 시작한 시점에 SnakeStopAtCenter → 드론이 실제 교차점에서 1+m 못 도달한 상태에서 정지.

**원인**:
- `TurnConfirm`: cy가 `turn_zone_y_min` 아래(접근 중)일 때 발사.
- `TurnReady`: cy ∈ [turn_zone_y_min, turn_zone_y_max] (중심 위치).
- 둘 다 받으면 cy=0.34 시점에 정지.

**해결**:
- 워치독을 `TurnReady` 단계에만 발동하도록 좁힘. SnakeForward는 그 사이 `forward_speed_advance_mps`로 계속 전진 → 자연스럽게 교차점 중심 도달 시점에 워치독 트리거.

---

## 55. SnakeStopAtCenter가 stale grid_branch_mask로 SnakeComplete 오판

**증상**:
- SnakeStopAtCenter → SnakeTurn90으로 가야 하는데 SnakeComplete로 직행, LAND.

**원인**:
- `handleSnakeStopAtCenter`가 `sp_in.grid_branch_mask = in.node_event.grid_branch_mask`로 SnakePlanner에 전달. 이 시점엔 peek valid=false → mask=0 → SnakePlanner는 모든 branch 없음 → `Complete` 반환.

**해결**:
- 워치독에서 latch한 fresh `last_node_grid_branch_mask_`을 SnakePlanner에 전달.

---

## 56. 좌표계 north 방향 결정

**경과**:
- Cycle 12: `north = +y` (math convention) 채택 — `s`가 화면 아래쪽, `+`가 위로 누적.
- Cycle 13: 사용자 mental model과 충돌해 `north = -y` (screen convention) 원복. `advance()`, `headingVector()`, `canvasRow()` 모두 일관성 있게 revert.
- 좌회전(negative x) 케이스는 `canvasCol = (x - min_x) * 4`가 동적 계산이라 영향 없음.

**교훈**: 좌표계 선택은 컴포넌트 간 일관성보다 사용자 시각 모델과의 일치가 우선. revert는 advance/headingVector/canvasRow 세 곳을 동시에 바꿔야 함.

---

## 57. L 코너에서 30-50 cm 일찍 정지

**증상**: 워치독 발동 후 stop ramp-down 시작 위치가 교차점 중심에서 30-50 cm 부족.

**해결 — cy-feedback 감속**:
- `GridControlMapper`의 `StopAndCenter` intent에 cy 기반 점진 감속 추가:
  ```cpp
  if (cy < stop_center_target_cy) {
      gap = stop_center_target_cy - cy;
      scale = min(1.0, gap / 0.30);
      vx = stop_center_max_vx_mps * scale;
  } else { vx = 0; }
  ```
- Cycle 12: target_cy=0.55 — 여전히 부족.
- Cycle 13: target_cy=0.65 — 카메라 중심선보다 +0.15 아래까지 전진 후 정지. Gazebo Iris 하향 카메라 principal point 오프셋 보정.

---

## 58. 회전 후 U-turn (이전 corridor의 라인 재추종)

**증상**: SnakeTurn90 후 SnakeAdvanceOneCell이 즉시 LineFollow 활성화. 드론이 중심에서 30-50 cm 떨어진 위치에 있어 카메라가 column 1의 OLD 라인을 봄 → "다음 노드 도착" 오인식 → SnakeTurn90Again 또 좌회전 → 결국 180° U-turn.

**해결 — Blind forward window**:
- SnakeTurn90 / SnakeTurn90Again 완료 시 `snake_post_turn_blind_until_s_ = now + snake_post_turn_blind_s` latch.
- 그 기간 동안 `intent=ForwardBlind` (yaw freeze, body forward 0.40 m/s) — LineFollow 비활성, distance gate skip, `nodeJustRecorded` 무시.
- Cycle 12: 2.5 s (1.0 m blind) → 너무 김.
- Cycle 13: 1.5 s (0.6 m blind) — 새 corridor 진입 + 이전 corridor 라인 잔상 클리어 충분.

---

## 59. SnakeAdvanceOneCell `advance_timeout` EmergencyLand

**증상**: SnakeAdvanceOneCell 진입 후 8 s 내 다음 노드 못 도달 → `advance_timeout` → EmergencyLand.

**원인**:
- `forward_speed_advance_mps = 0.18` (hardcoded 기본값, mission.toml에서 로드 안 됨).
- 0.18 m/s × (8 − 2.5) s = 0.99 m → cell 3 m 미달성.

**해결**:
- mission.toml에서 `forward_speed_advance_mps` 로드 추가 + 0.30 m/s로 상향.
- `snake_advance_timeout_s` 8 → 15 s.
- 산수: blind 1.5 s × 0.40 + (15 − 1.5) s × 0.30 = 4.65 m → cell 3 m + 마진.

---

## 60. `populateLineInputs` freeze가 lateral 보정도 차단

**증상**: 회전 후 카메라가 라인을 우측에 두고 진행해도 vy_right_mps가 0. yaw만 정렬되고 lateral 보정 안 됨.

**원인**: Cycle 8에서 추가한 `if (intersection.valid) { line_center_error_norm=0; }`가 SnakeAdvanceOneCell에서도 발동해 lateral 입력 차단.

**해결 변천**:
- Cycle 13: state-aware freeze — SnakeForward / SnakeRecordNode에서만 freeze, LineEnter / SnakeAdvanceOneCell / GridOriginLock에선 비활성.
- Cycle 14 (롤백): `best_observed_type ∈ {L,T,Cross}` 기반 cross_potential로 재설계 시도, LineEnter 동안 type=Straight 유지로 yaw drift 차단 못 함 → 전면 롤백.
- Cycle 15: `accepted_type=Straight` AND raw branches Front+Back angle 차이 ∈ [170°, 190°] 일 때만 보정 허용. T 시작 transient에서 contour merging이 angle을 흔드는 경우 차단.

---

## 61. Column 전환 시 tracker가 commit 안 되어 column 2/3가 column 1을 덮어씀

**증상**:
- 첫 column 10 노드 이후 GCS 그리드 갱신 안 됨. onboard 로그는 `nodes=11,12,...18` 증가하지만 모두 `coord=(0,*)` — column 2 진입 후 x가 -1로 안 바뀜.
- column 2 끝에서 잘못된 turn → trace 꼬임 → fallsafe.

**원인**:
- `handleSnakeAdvanceOneCell`에서 다음 노드 도착 시 `intersections_recorded_++` 후 SnakeTurn90Again으로 전이하지만 `out.commit_tracker_advance`를 켜지 않음 + node_event를 GCS로 latch도 안 함.
- 결과: tracker의 `current_coord_`는 column 1 끝 좌표 유지. 다음 commit의 peek event = `advance((0,-9), south)` = `(0,-8)` → column 1과 같은 x 덮어쓰기.

**해결**:
- `handleSnakeAdvanceOneCell`의 노드 도착 분기에 `out.commit_tracker_advance = true;` + `last_node_local_x_/y_` 갱신 추가. SnakeTurn90이 `notifyTurnCompleted(west)`를 이미 호출했으므로 peek event는 정확히 `(-1, -9)` (새 column 진입점).

---

## 62. 마커 hover race — mks=0 frame 한 번으로 hover 해제

**증상**: 마커 commit은 되지만 호버 안 됨. mks=1 frame 사이 mks=0이 한 번이라도 끼면 다음 tick의 `focus_marker_id` 재선택에서 -1 → MarkerHover intent 해제.

**원인**: `handleSnakeRecordNode`가 **매 tick** `focus_marker_id`를 다시 선택. 현재 frame `in.vision->markers`만 보고 candidate count 체크.

**해결 — hover latch**:
- `marker_hover_active_`가 latch된 후엔 `last_recorded_marker_id_`로 fix → 다음 tick에 마커가 안 보여도 hover 유지.
- `populateMarkerInputs`가 marker_detected=false로 처리하지만 intent=MarkerHover면 `GuidedVelocityController::updateMarker`가 vy=0 안정 호버 명령 출력.

---

## 63. ArUco 마커가 검정 grid line에 닿아 인식 불가

**증상**: 흰 배경 + 검정 라인 arena로 변경 후 ArUco DICT_4X4_50 마커 외곽 검정 quiet zone과 라인이 시각적으로 이어져 OpenCV가 contour 분리 실패.

**해결**: 마커 plane(0.50×0.50 m) 아래에 더 큰 흰색 plane(0.60×0.60 m)을 z=0.001에 추가. 5 cm 흰색 margin이 라인과 마커 quiet zone을 명확히 구분. PNG는 건드리지 않고 SDF만 수정.

---

## 64. Snake alternation이 깨짐 — `직진→좌좌→직진→좌좌`

**증상**: 정상은 `직진→좌좌→직진→우우→직진→좌좌...`이나 실제로 좌회전만 반복. column 2 끝 boundary에서 우회전 대신 좌회전.

**원인**:
- `SnakePlanner::planAtBoundary`에서 `snake_dir_ = chosen;`로 chosen-flipped 값을 덮어씀.
- snake_dir_의 의미는 "snake의 alternation 기준 방향" — boundary에서 primary 방향에 branch가 없어 일시 flip한 chosen 값을 snake_dir_에 저장하면 다음 `notifySecondTurnCompleted()`의 flip이 잘못된 base에서 일어남.

**해결**: `snake_dir_ = chosen;` 라인 제거. snake_dir_은 `notifySecondTurnCompleted`에서만 mutate되도록.

---

## 65. LineFollow / LaunchAlign이 yaw를 누적 drift시킴

**증상**: 셀당 ±0.02-0.04 rad yaw drift 누적, 7-8셀 후 라인 이탈 → fallsafe.

**원인**:
- `GridControlMapper`의 LineFollow / LaunchAlign case가 `line_controller_->updateLine()` 결과를 그대로 사용.
- `updateLine()`은 `yaw_rate = angle_yaw_kp · angle_error + offset_yaw_kp · offset_error` — LineDetector angle bias를 그대로 따라감.
- LaunchAlign 1-2 s × line_controller yaw_rate → 한 사이클 동안 ±0.1-0.2 rad 흔들림. ForwardBlind가 복귀시키기엔 시간 부족.

**해결 변천**:
- Cycle 17: SnakeLaunchAlign 종료 시 `yaw_align_target_rad_ = wrap(*attitude_yaw)` re-latch 제거 (drift된 yaw가 새 target으로 freeze되는 것 차단). SnakeTurn90 / SnakeAdvanceOneCell 90° target을 `latched_yaw + dir·π/2`로 계산 (live attitude 아님). Turn90Again 완료 시 `yaw_align_target_rad_ = yaw_target_rad_` 갱신.
- Cycle 18 (핵심): `GridControlMapper`의 LineFollow / LaunchAlign case에서 line_controller가 출력한 `sp.yaw_rate_rad_s`를 `computeYawRate(current, target)`으로 덮어쓰기. vy는 line_controller 그대로, yaw만 mission target lock. **이게 진짜 fix** — Cycle 17만으로는 LineFollow window에서 잔여 drift 누적 지속.

**교훈**: line-following 컨트롤러의 yaw output을 mission 레이어가 신뢰하지 말 것. column heading 같은 mission-level invariant는 mission 자신이 lock해야 함.

---

## 66. Boundary 노드가 GCS에 commit 안 됨

**증상**: 각 열의 마지막 (boundary) 노드 1개씩 GCS 그리드에 누락. 9-cell column에서 8개만 표시.

**원인**:
- `IntersectionDecision`이 `required_turn=true`인 boundary 교차점에선 state를 `NodeRecord` 거치지 않고 직접 `TurnConfirm/TurnReady`로 routing.
- `handleSnakeForward`의 정규 arrival 경로(`nodeJustRecorded` 체크)는 false → commit_tracker_advance 미설정.
- 워치독은 state=TurnReady에서 발사되지만 commit 블록이 없음.

**해결**: 워치독 분기에도 정규 arrival과 동일한 commit 블록 추가. `in.node_event.valid`이면 peek event 사용, 아니면 synthetic GridNodeEvent를 합성 (cy가 turn_zone에는 들어왔지만 node_zone overlap 못 잡은 케이스 fallback).

---

## 67. 회전 직후 mapper의 LineFollow가 lateral bias를 그대로 따라감

**증상**: ForwardBlind intent에서도 LineDetector의 작은 center bias 때문에 N/S 방향에 따라 일관된 좌/우 tilt 발생.

**원인**: ForwardBlind는 yaw freeze + body forward만 명령했지만 lateral은 0 (보정 없음). LineDetector의 grid-frame bias 방향에 따라 라인 중심이 카메라 중심에서 벗어남.

**해결**: ForwardBlind에서도 라인이 잡혔으면 `line_controller_->updateLine()`을 호출해 vy만 사용 (yaw_rate는 무시, mission이 직접 lock). `angle_error_rad=0`을 명시적으로 넘겨 yaw 영향 0. `populateLineInputs`가 이미 교차점 근처에서 line 입력을 null로 만들기 때문에 confident straight line일 때만 작동.

---

## 68. GCS 화살표가 cell 사이에서 표시 정확도 / s 위치 문제

**경과**:
- Cycle 13: 드론 sub-cell 위치 telemetry 송신 + GCS가 `canvasRow + round(frac_y * 2)` 형태로 화살표를 cell 사이에 표시. 자세한 진행도 표시.
- Cycle 16: arena 변경(vertiport 진입 라인 제거)으로 `s` 표시 의미 없어짐 + sub-cell arrow 정확도가 LOCAL_NED drift에 민감 → 화살표는 항상 `current_coord_`의 캔버스 좌표에 표시. `s` 표시 제거.

**교훈**: telemetry 표시 정확도는 보내는 신호의 안정성을 넘어설 수 없음. LOCAL_NED 1-cell 진행도는 분당 누적 optical flow drift 안에서 신뢰 가능한 신호.

---

## 69. GCS에 vertiport id=23이 미검출 상태에서 바로 표시됨

**증상**: onboard 프로그램 시작 직후 GCS에 `[vertiport] id=23  vertiport (pending)`가 표시됨. 실제 버티포트 마커를 보기 전인데도 23이 먼저 뜸.

**원인**:
- onboard의 내부 미션 로직은 latched vertiport id가 없을 때 `config_.vertiport_marker_id`를 fallback으로 사용해야 안전함.
- 그런데 telemetry용 public getter도 같은 fallback 함수를 타면서, 미검출 상태의 설정값 23이 GCS로 새어 나감.

**해결**:
- 내부 로직은 `effectiveVertiportMarkerId()` fallback 유지.
- telemetry getter는 fallback 없이 실제 latched 값만 반환하도록 분리. 아직 본 적 없으면 `-1`.
- GCS 표시는 `pending` / `verified` 문구를 제거하고, 실제 첫 vertiport detection id만 표시하도록 정리.

---

## 70. GCS Vision Log 3분할 창 크기 조절 + Windows `LoadCursorW` 빌드 실패

**증상**:
- Vision Log의 grid / marker / mission 로그 3칸 크기가 고정되어 현장 디버깅 시 불편.
- Windows MinGW 빌드에서 `LoadCursorW(nullptr, IDC_SIZEWE)`가 `LPSTR` → `LPCWSTR` 변환 실패로 컴파일 에러.

**원인**:
- 기존 Win32 UI가 splitter drag 상태를 갖고 있지 않음.
- MinGW 환경에서 `IDC_SIZEWE`, `IDC_SIZENS`, `IDC_ARROW` 매크로 타입이 wide API 인자와 맞지 않음.

**해결**:
- `VisionLogWindow`에 vertical / horizontal splitter drag 상태와 hit test 추가.
- 마우스 드래그로 좌우 패널 폭과 상하 로그 높이를 조절.
- cursor 로딩은 integer resource id + `MAKEINTRESOURCEW(...)`를 사용해 `LoadCursorW` 타입 문제 해결.

---

## 71. Snake 완료 시 남은 열 synth가 현재 위치까지 이동시킴

**증상**: 마지막 마커를 예를 들어 `(-4,-7)`에서 발견했는데, grid 완성을 위해 남은 열을 채운 뒤 GCS current가 `(-4,-8)`처럼 열 끝으로 이동해 보임. 실제 드론은 움직이지 않았음.

**원인**:
- 닫힌 직사각형 grid를 만들기 위한 synthetic node가 실제 tracker advance처럼 처리되어 current pose까지 갱신.
- grid map completion과 drone current pose가 같은 이벤트 경로를 공유하고 있었음.

**해결**:
- 남은 열을 채우는 synthetic `GridNodeEvent`에 `updates_current=false`, `origin_local_only=true`를 부여.
- `SnakeComplete`에서 pending synthetic event를 한 tick씩 drain하되, tracker의 현재 좌표/방향은 마지막 실제 위치로 유지.

**교훈**: map topology 보강 이벤트와 vehicle pose 이벤트는 같은 grid node라도 의미가 다르다.

---

## 72. ForwardBlind 중 yaw는 고정되지만 N/S 방향별 lateral drift 발생

**증상**: yaw는 mission target에 잘 lock되어 있는데, north/south 진행 방향에 따라 라인 중심에서 한쪽으로 비껴가는 경향이 생김.

**원인**:
- `ForwardBlind`는 vx + yaw lock만 내고 `vy_right_mps=0`.
- LineDetector의 body-frame center bias가 heading에 따라 grid frame에서 반대 방향 lateral drift로 나타남.
- 기존의 짧은 LineFollow window만으로는 셀 전체 구간의 lateral 오차를 충분히 누르지 못함.

**해결**:
- `ForwardBlind`에서도 라인이 검출되면 line controller를 호출하되 `angle_error_rad=0.0`으로 고정.
- line controller 결과 중 `vy_right_mps`만 사용하고 yaw는 mission-level `target_yaw_rad`로 계속 lock.
- 교차점 근처는 기존 `populateLineInputs` freeze가 line error를 0으로 만들기 때문에 옆 가지에 끌려갈 위험을 낮춤.

---

## 73. 마커 hover가 마커 중심에서 충분히 맞지 않고 착륙 위치가 밀림

**증상**: 마지막 마커 발견 후 3초 hover 후 착륙하는데, 체감상 마커 중심보다 50 cm 정도 벗어나서 내려앉음.

**원인**:
- 마커 hover는 이미 3초 타이머로 유지되고 있었지만, hover 중 marker center error를 적극적으로 줄이는 품질이 중요했음.
- yaw는 mission target으로 유지해야 하고, 마커 방향/회전각을 따라 yaw를 돌리면 실전에서 마커 부착 방향에 의존하게 됨.

**해결**:
- `MarkerHover`는 ArUco center error x/y로 평행 이동 보정.
- yaw는 마커 orientation이 아니라 mission target yaw로 고정.
- `markerHoverComplete()`는 `snake_marker_hover_s` 기준으로 3초 유지. 마지막 발견 마커가 재방문 1순위여도 revisit leg에서 다시 한 번 target marker로 3초 hover.

**교훈**: 마커 hover align은 marker orientation이 아니라 marker center 기준이어야 실전 배치 방향에 독립적이다.

---

## 74. 탐색 속도 상향 후 일반 노드 dwell 때문에 overshoot / align 시간이 늘어남

**증상**:
- forward 속도를 올리면 전체 시간은 줄지만, 매 일반 노드마다 정지 dwell을 하면서 overshoot가 커지고 다시 교차점 align에 시간이 듦.
- marker 없는 중간 노드에서 멈추는 시간이 탐색 시간의 큰 비중을 차지.

**원인**:
- 모든 노드를 같은 방식으로 `SnakeRecordNode` dwell 처리.
- 일반 통과 노드와 boundary / turn / marker node를 구분하지 않음.

**해결**:
- 기존 dwell 경로는 유지해 현장 fallback 가능하게 둠.
- `snake_passthrough_regular_nodes` 분기 추가: marker hint가 없고, turn 후보가 아니고, forward branch가 열린 일반 노드는 정지 없이 통과.
- marker가 보이거나 boundary / turn node면 기존처럼 정지, branch 판별, marker hover를 수행.

**교훈**: 속도 개선은 forward m/s보다 "정지해야 하는 노드"를 줄이는 효과가 더 크다.

---

## 75. Snake 이후 마커 재방문을 별도 프로그램으로 나눌지 여부

**문제**: line tracing / snake / revisit를 별도 프로그램으로 쪼개면 테스트는 쉬울 수 있지만, 실제 운용 목표는 onboard + GCS 두 프로그램만 사용하는 것.

**결론**:
- 새 프로그램을 만들지 않고 `grid_mission_node`를 확장.
- snake가 만든 `MarkerRegistry`와 `GridMapTracker`를 그대로 재사용.
- `MarkerRevisitPlanner`로 asc/desc target order와 Manhattan leg를 생성.
- 회전, passthrough, marker hover는 snake 단계의 기존 state / helper를 최대한 재사용.

**GCS 표시**:
- marker list 시간을 `found=` / `revisited=`로 구분.
- 현재 재방문 target은 `[>]`, 완료 marker는 `[x]`로 표시.
- Mission summary에 revisit 성공 여부를 `marker_revisit`로 추가.

---

## 76. Revisit 중 경로 위의 다른 마커 처리

**질문**: 현재 target이 아닌 마커가 재방문 경로 위에 있으면 hover하는가?

**코드 기준 동작**:
- `RevisitForward`는 교차점 통과와 좌표 advance만 처리.
- `RevisitMarkerHover`에서만 `revisit_current_marker_id_`를 `focus_id`로 잡고 marker hover / markRevisited를 수행.
- 따라서 경로 중간에 보이는 다른 ArUco는 telemetry에는 보일 수 있지만 미션 제어상 무시하고 지나감.

**의도**: desc/asc 순서 보장을 위해 target marker에 도착했을 때만 hover/visited 처리.

---

## 77. 재방문 완료 후 버티포트 복귀 / 착륙 단계

**요구**: 전체 미션을 `snake 탐색 -> marker revisit -> (0,0) 복귀 -> vertiport 복귀 -> 착륙 -> 종료`로 확장.

**구현 방향**:
- revisit planner의 Manhattan leg 생성 로직을 재사용해 현재 위치에서 `(0,0)`까지 복귀.
- `(0,0)`에서 grid 기준 south를 바라보도록 회전.
- grid를 벗어나는 순간 GCS grid map의 current arrow는 숨김.
- 라인 추적은 끄고 yaw freeze + forward blind로 vertiport marker id가 보일 때까지 전진.
- vertiport marker가 잡히면 marker center hover 후 land.

**Failsafe**:
- 버티포트 복귀 forward / hover timeout은 진입 단계와 같은 `entry_forward_timeout_s` 계열을 사용.
- active vertiport id가 없거나 제한 시간 안에 찾지 못하면 emergency land.

---

## 78. 새 edge-case world에서 Gazebo camera frame timeout

**증상**: `grid_arena_edge_case_test.sh`로 새 world를 띄운 뒤 기존 명령 `--world grid`로 실행하면 `[vision] read failed: timed out waiting for Gazebo camera frame` 반복.

**원인**:
- onboard의 Gazebo camera source는 `--world grid`일 때 기존 `/world/grid_arena_test_world/...` topic을 구독.
- 새 SDF 내부 world name을 새 이름으로 바꾸면 Gazebo camera topic도 달라져 기존 `--world grid` 경로와 불일치.

**해결**:
- course file / script 이름은 `grid_arena_edge_case_test_world`로 새로 두되, SDF 내부 `<world name>`은 기존 topic과 맞게 유지.
- 새 3x3 edge-case arena는 기존 camera / drone 조건을 보존하고 marker 배치만 변경.

**교훈**: Gazebo의 file name과 internal world name은 별개다. runtime topic은 internal world name을 따른다.

---

## 79. `(0,0)` 원점에 있는 ArUco marker가 저장되지 않음

**증상**: edge-case arena에서 id=3 marker를 `(0,0)`에 배치하면 grid 진입과 snake 출발은 정상인데, id=3이 registry에 저장되지 않음.

**원인**:
- grid entry flow는 `ENTRY_CENTER_ORIGIN -> latchGridOrigin -> SNAKE_LAUNCH_ALIGN`으로 바로 넘어감.
- 원점은 이미 "시작 노드"로 처리되므로 일반 `SnakeRecordNode` marker registration 경로를 타지 않음.

**해결**:
- `EntryOriginMarkerHover` 상태 추가.
- `ENTRY_CENTER_ORIGIN`에서 원점 정렬이 끝난 직후 stable non-vertiport marker가 있으면 `(0,0)` marker로 register.
- 원점을 다음 노드로 착각하지 않도록 node advance는 하지 않고, marker만 registry에 붙인 뒤 3초 hover 후 `SnakeLaunchAlign`로 진행.
- hover 전후 `marker_window_`를 clear해 원점 marker가 다음 노드에 stale하게 붙지 않도록 함.

---

## 80. 두 번째 회전 노드 marker hover 중 yaw가 이전 방향으로 돌아감

**증상**: 두 번째 회전 노드에 marker가 있으면 marker hover 중 yaw가 이전 column 방향으로 돌아갔다가, 이후 `SNAKE_TURN_90_AGAIN`에서 다시 180도 가까이 도는 이상 동작.

**원인**:
- `handleTurnNodeMarkerHover()`가 항상 `yaw_align_target_rad_`를 hover target yaw로 사용.
- 두 번째 turn-cell marker hover 시점에는 connector heading을 유지해야 하는데, `yaw_align_target_rad_`에는 이전 column heading이 남아있음.
- `MarkerHover` intent는 yaw도 target으로 제어하므로 hover 중 실제로 기체가 돌아감.

**해결**:
- marker hover 진입 시점에 `pending_post_hover_yaw_target_rad_`를 latch.
- 첫 boundary marker hover는 기존 column yaw를 유지.
- 두 번째 turn-cell marker hover는 second-turn target을 계산하기 전 connector yaw를 latch해 hover 동안 유지.
- hover 종료 후에만 다음 yaw turn 단계로 진행.

---

## 81. Marker orientation을 align 기준으로 쓰면 실전 배치 방향에 의존

**질문**: 마커 hover / align이 marker 방향 기준이면, 현장에 마커가 아무 방향으로 붙어 있을 때 문제가 되지 않는가?

**확인 결과**:
- 현재 marker hover 제어는 ArUco orientation을 yaw 기준으로 쓰지 않음.
- `populateMarkerInputs`는 marker center pixel error x/y만 control input으로 전달.
- yaw는 mission target yaw로 lock된다.
- registry에는 orientation을 기록하지만, 이동/hover 제어의 기준은 아님.

**교훈**: orientation은 telemetry/debug metadata로만 두고, 제어는 center + mission yaw 기준으로 유지하는 편이 실전 배치에 강하다.

---

## 82. Revisit order 기본값과 CLI 옵션

**요구**: 앞으로 항상 marker revisit까지 수행하므로 `--revisit-order none`은 제거하고, 옵션 생략 시 desc가 기본이 되게 함.

**해결**:
- config 기본값을 `RevisitOrder::Desc`로 유지.
- CLI는 `--revisit-order asc|desc`만 받도록 정리.
- README에 기본 desc 실행 예시와 asc 예시를 추가.

**운용 메모**:
- 기본 실행은 `--revisit-order` 생략 가능.
- 오름차순 테스트 때만 `--revisit-order asc`를 명시.

---

## 83. [백필] 2026-05-23 이후 공백 요약 — 하드웨어 전면 업그레이드와 플랫폼 정합화

**배경**: 개발로그가 2026-05-23에 멈춘 사이 코드 저장소(`uav-onboard`)는
2026-07-03까지 계속 진행됐다(실비행 EKF/GUIDED 브링업, safety, PI control,
RC-override 실험). 이 엔트리는 그 공백과 2026-07-04 정합화 작업을 요약한다.

**그 사이 일어난 일**:
- 1차 예선 통과. 지원금으로 하드웨어 전면 교체:
  Pixhawk 1(fmuv2) → **Pixhawk 6C Mini**, Pi 4 → **Pi 5 (4GB)**,
  IMX519 → **IMX296 모노 글로벌셔터(고정초점 CS-mount)**,
  3S/1.5kg → **4S/약 2.1kg** (S500 V2, X2212 980KV, 9450 프롭, 35A BLHeli_S
  OPTO, BEC12S-PRO). MTF-01은 유지.
- 경기장 확정: 실내, 잔디 위 10cm 흰 라인 3m 그리드 (= 시뮬 기본 월드와
  동일 극성). 마커는 50cm, 흰 패드 위 배치.
- 실기체에서 발견된 비전 문제: 잔디 위 ArUco 오검출, 마커 주변 흰 패드가
  라인 contour로 유입. → 원인 분석 결과 (a) 절대 HSV 임계값(white_fill)의
  조명 취약성, (b) 마커 차폐가 흰 패드(quiet zone)를 안 덮음, (c) ArUco
  템플릿/밝은-blob 폴백 경로의 약한 수락 기준이 주범으로 특정됨.

**2026-07-04 정합화 (코드/설정/문서)**:
- 설정 우선순위 버그 수정: `AstroquadOnboardApp.cpp`의 하드코딩 게인 블록
  삭제 → `mission.toml [line_controller]/[altitude_hold]`가 실제 소스가 됨
  (우선순위: 컴파일 기본 < mission.toml < 런타임 프로파일 < CLI).
- `config/pixhawk6c_indoor_flow.params` + `.expected.toml` 신규 작성
  (구 fmuv2 파일은 SUPERSEDED 표기). 4S 배터리 임계값, ELRS CRSF 네이티브
  RC(RC_CHANNELS 게이트), MTF-01 TELEM1, 컴퍼스 리스크 완화 절차 포함.
- `vision.toml` IMX296 1단계 전환: 센서명/해상도(1456×1088), 포커스 제어
  삭제, AE 잠금 절차 주석, `marker_size_mm` 80→500.
- rpicam 디코드 `IMREAD_GRAYSCALE` 전환 + Gazebo L8 모노 패스스루 —
  파이프라인 그레이 네이티브화(검출기는 1채널 이미 지원).
- 아키텍처 결정 기록(`ARCHITECTURE_OVERVIEW.md`): GUIDED 단일 제어 경로,
  RC-override 폐기, PX4/ROS 2/BT 마이그레이션 기각과 재검토 조건.
- 문서 전면 갱신: README/PROJECT_SPEC×2/PROTOCOL v1.9×2/SYSTEM_SPEC/
  sim README(SITL 역학 fidelity 주의).

**남은 리스크 (신규 식별)**: 렌즈 FOV 미실측, Pi5 UART 경로 미확정,
BEC 5V 전원 예산 미검증, 외장 컴퍼스 부재(최대 추정 리스크), AE 헌팅,
마커 상실 후 패드 잔존 차폐, 재방문 순서 규정 확인 필요.

---

## 84. 비전 강건화 + 안전/구조 정리 (2026-07-04, 실촬영 데이터 확보 전 데스크 작업)

**작업 내용**:

1. **마커 차폐 확장 (`LineMaskBuilder`)** — 마커 주변 흰 패드(quiet zone)가
   라인/교차점 검출 오염원이 되는 문제의 수정:
   - 차폐 폴리곤을 중심 기준 `marker_occlusion_scale`(기본 1.7, vision.toml)로
     팽창해 패드 링까지 소거.
   - 차폐 입력을 프레임별 신규 검출에서 **stabilizer 유지 마커**(age ≤
     detect_interval×2)로 교체 — ArUco 스킵 프레임(3프레임 간격)과 프레임
     가장자리 이탈 직후에도 차폐 유지(TTL).
   - 마커 폴리곤 차폐를 전 극성에 적용(기존엔 dark_on_light에서 통째 비활성).
     휴리스틱 사각형 후보 검출기만 dark_on_light에서 계속 비활성.
   - dark 마스크에도 차폐 적용(마커 검은 본체는 dark 마스크의 오염원).
2. **ArUco 오검출 게이팅 (`ArucoDetector`)**:
   - 템플릿 매칭 폴백과 밝은-blob ROI 스윕을 config 플래그 뒤로 이동,
     **기본 off** (`template_fallback_enabled`, `bright_roi_fallback_enabled`).
     잔디 텍스처 오검출의 주범. 실측 코퍼스에서 기본 검출기가 실마커를
     놓칠 때만 재활성.
   - 기본(primary) 검출 경로에 형태/명암 타당성 검사 추가
     (`markerContrastLooksPlausible`: 종횡비 + 어두운 비트 비율 ≥0.35).
     크기 밴드는 의도적으로 제외 — vertiport 마커는 호버 고도에서 프레임을
     가득 채우므로 기존 `markerShapeLooksPlausible`의 max-size 검사를 primary에
     그대로 적용하면 MARKER_LOCK_YAW가 깨짐.
   - **잠복 버그 수정**: `isLiveFrameSize` 임계 1000 → 1200. IMX296 네이티브
     1456×1088이 "recorded"로 오분류되면 무제한 폴백 스윕이 매 프레임 돌아
     Pi에서 CPU 폭주였음.
3. **line-lost 감시 재활성 시도 → 근거 기반 회귀(revert)**:
   - SNAKE/REVISIT/RETURN 전진 상태에서 line-lost 타임아웃(2s)을 상태 인지형으로
     강제해 봄 → SITL 2회 연속, 첫 홉 진입 ~2s 시점(t≈25.8s)에서 **결정적으로**
     EMERGENCY_LAND. 그리드 홉은 설계상 `fwd_blind`(요 고정 + 블라인드 전진 +
     기회적 라인 센터링)라 홉 초반 수 초의 라인 부재가 정상.
   - 결론: 그리드 임무에서 라인 상실의 방어선은 hop-distance 페일세이프
     (`hop_max_distance_m`)이며 line-lost 타임아웃은 연속 라인 추종 임무
     전용. 강제는 되돌리고, "무언의 비활성"이던 것을 코드 주석으로 명시적
     설계 결정으로 문서화.
4. **죽은 코드 삭제**: `grid_mission_node`(바이트 동일 중복 바이너리) 제거,
   `LineDetector.cpp`의 레거시 마스크 빌더 클러스터(`buildMasks` + 전용 헬퍼
   ~160줄, LineMaskBuilder와 auto 모드 처리가 달랐던 중복 구현) 삭제.
   마스크 구현은 LineMaskBuilder 단일화.
5. **회귀 인프라**: `scripts/sitl_mission_regression.sh` 신규 — 헤드리스
   Gazebo+SITL+onboard 전체 임무 자동 실행 + 기계 판정(MISSION_COMPLETE 도달,
   EMERGENCY_LAND 0회, regids 완비, 정상 disarm). 단위 테스트
   `test_line_mask_strategies` 신규(모노 white_fill 폴백, 조명 그래디언트에서
   local_contrast 우위, 패드 차폐 스케일 검증). 총 18/18 통과.

**중요 관찰 — SITL 회귀는 비결정적**: 동일 환경에서 클린 HEAD도 실패
양상이 달랐음(런1: 51노드까지 진행 후 max_intersections + 마커 2,3 미발견 /
런2: 첫 홉 hop_distance_exceeded). EKF 광류 드리프트·타이밍 변동이 원인으로
추정. 단일 런 실패로 회귀 판정하지 말고 복수 런으로 비교할 것. GUI 환경
(팀원 로컬)과 헤드리스 환경의 통과율 차이도 확인 필요.
