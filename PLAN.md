# Raspberry Pi 4 + IMX519-78 전환 계획

작성일: 2026-04-28
상태: 1차 구현 진행. Pi 4 + IMX519 camera config, rpicam option, system/camera telemetry, GCS log/parser 확장 적용.

## 1. 이번 계획의 범위

이번 문서는 다음 세 가지를 기준으로 작성한다.

- `development-log/RESEARCH.md`, `uav-gcs/README.md`, `uav-onboard/README.md`를 기준으로 현재 구조와 구현 상태를 정리한다.
- 기존 Raspberry Pi Zero 2 W + ZeroCam 계열 운용을 Raspberry Pi 4 + IMX519-78 16MP AF CSI 카메라 운용으로 바꾸기 위해 수정할 범위를 식별한다.
- Pi 4와 IMX519-78로 생기는 여유를 이용해 onboard가 GCS로 송신할 정보를 어떻게 개선할지 계획한다.

참조한 외부 문서:

- Raspberry Pi 4 공식 사양: https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/
- Raspberry Pi camera software/rpicam 공식 문서: https://www.raspberrypi.com/documentation/computers/camera_software.html
- Waveshare IMX519-78 16MP AF Camera 문서: https://www.waveshare.com/wiki/IMX519-78_16MP_AF_Camera

핵심 외부 정보:

- Raspberry Pi 4는 1.8GHz quad-core Cortex-A72, LPDDR4 RAM, 2.4/5GHz 802.11ac Wi-Fi, Gigabit Ethernet, 2-lane MIPI CSI, H.264 1080p30 encode를 제공한다.
- IMX519-78은 Sony IMX519 기반 16MP AF CSI 카메라이고, 78도 FOV와 autofocus를 지원한다.
- Raspberry Pi OS Bookworm 이후 camera app 이름은 `rpicam-*`이며, `rpicam-vid`는 MJPEG 출력, autofocus, lens position, exposure/gain, metadata 출력 옵션을 제공한다.

## 2. 현재 저장소 구조

루트 `astroquad/`는 작업 폴더이고, 하위 3개 디렉토리가 실질적인 프로젝트 단위다.

```text
astroquad/
├─ development-log/
│  ├─ PLAN.md
│  ├─ RESEARCH.md
│  └─ TROUBLESHOOTING.md
├─ uav-gcs/
└─ uav-onboard/
```

| 디렉토리 | 역할 |
|---|---|
| `development-log/` | 리서치, 계획, 트러블슈팅 기록 |
| `uav-onboard/` | Raspberry Pi에서 실행되는 camera, vision, telemetry/video 송신 코드 |
| `uav-gcs/` | 노트북 GCS에서 실행되는 telemetry/video 수신, overlay, log 표시 코드 |

현재 전체 데이터 흐름은 다음과 같다.

```text
Pi camera
  -> rpicam-vid MJPEG stdout
  -> RpicamMjpegSource JPEG frame extraction
  -> OpenCV JPEG decode
  -> onboard ArUcoDetector
  -> onboard LineDetector
  -> onboard LineStabilizer
  -> UDP telemetry JSON
  -> UDP MJPEG debug video chunks
  -> GCS video receive thread
  -> GCS telemetry store/frame matching
  -> GCS-side marker/line overlay
  -> GCS video/log windows
```

중요한 설계 원칙:

- mission-critical 판단값은 onboard에서 계산한다.
- GCS overlay는 onboard metadata를 그릴 뿐, 현재 GCS에서 vision detection을 다시 수행하지 않는다.
- debug video는 best-effort이며, 오래된 frame은 버려도 vision loop가 막히면 안 된다.
- telemetry JSON receiver는 unknown field를 무시하도록 설계되어 있어 optional field 확장에 유리하다.

## 3. `development-log` 파일 역할

| 파일 | 현재 역할 |
|---|---|
| `RESEARCH.md` | 최신 구현 상태, 구조, protocol v1.5, 테스트 결과, 남은 리스크 요약 |
| `PLAN.md` | 이번 Raspberry Pi 4 + IMX519-78 전환 계획. 기존 완료 계획을 대체 |
| `TROUBLESHOOTING.md` | telemetry 수신 실패, OpenCV/Win32 fallback, rpicam stdout, GCS discovery, firewall, line/video/frame drop/thermal 이슈 기록 |

## 4. `uav-onboard` 구조와 파일 역할

```text
uav-onboard/
├─ CMakeLists.txt
├─ PROJECT_SPEC.md
├─ README.md
├─ config/
├─ docs/
├─ logs/
├─ scripts/
├─ src/
├─ test_data/
├─ tests/
└─ tools/
```

### 4.1 루트 문서와 빌드

| 파일 | 역할 |
|---|---|
| `CMakeLists.txt` | C++17, nlohmann/json, toml++, OpenCV optional targets, onboard libraries/tools/tests 구성 |
| `PROJECT_SPEC.md` | 온보드 최종 목표 문서. 현재 hardware 항목은 Pi 4 + IMX519-78 기준으로 갱신됨 |
| `README.md` | Pi build, rpicam camera bring-up, video streaming, vision debug, telemetry 실행 안내 |

### 4.2 `config/`

| 파일 | 역할 |
|---|---|
| `network.toml` | GCS IP/telemetry/video/command port, telemetry interval 설정. 기본 telemetry는 broadcast |
| `vision.toml` | camera/video/line/aruco runtime parameter. 현재 640x480, 12FPS, quality 45, send_fps 10 |
| `autopilot.toml` | Pixhawk UART device와 MAVLink system/component id 설정. 아직 실제 MAVLink 구현 전 |
| `mission.toml` | grid 크기, marker_count, snake 방향, takeoff altitude placeholder |
| `safety.toml` | line lost, GCS/Pixhawk timeout, mission timeout, battery low voltage placeholder |

### 4.3 `docs/`

| 파일 | 역할 |
|---|---|
| `PROTOCOL.md` | onboard-GCS protocol v1.5. telemetry JSON, system/camera debug fields, UDP MJPEG chunk header, GCS discovery beacon, reserved command channel 명세 |

`uav-onboard/docs/PROTOCOL.md`와 `uav-gcs/docs/PROTOCOL.md`는 동일하게 유지해야 한다.

### 4.4 `scripts/`

| 파일 | 역할 |
|---|---|
| `setup_rpi_dependencies.sh` | Raspberry Pi OS Lite 64-bit 기준 build/OpenCV/rpicam/libcamera 관련 apt package 설치 |

Pi 4 + IMX519-78 전환 시 이 스크립트에는 dependency 설치뿐 아니라 `rpicam-hello --list-cameras`, `rpicam-hello --version`, OS/kernel 확인, IMX519 driver/focus tool 안내를 추가할 필요가 있다.

### 4.5 `src/common/`

| 파일 | 역할 |
|---|---|
| `NetworkConfig.hpp/.cpp` | `network.toml` 파싱. GCS IP와 telemetry/video/command port, interval 저장 |
| `VisionConfig.hpp/.cpp` | `vision.toml` 파싱. camera/video/aruco/line 설정 저장 |
| `Time.hpp/.cpp` | Unix timestamp ms 유틸리티 |

Pi 4 + IMX519-78 전환의 핵심 수정 지점은 `VisionConfig`다. autofocus, lens position, exposure/gain, AWB, denoise, camera index, tuning file, sensor mode 등을 config로 끌어올려야 한다.

### 4.6 `src/camera/`

| 파일 | 역할 |
|---|---|
| `CameraFrame.hpp` | frame id, timestamp, width/height, JPEG byte vector |
| `RpicamMjpegSource.hpp/.cpp` | `rpicam-vid --codec mjpeg -o -` stdout에서 JPEG SOI/EOI를 찾아 frame 생성 |

현재 `RpicamMjpegSource`는 width/height/fps/jpeg_quality만 command에 반영한다. IMX519-78에서는 아래 옵션 확장이 필요하다.

- `--camera`
- `--autofocus-mode`
- `--autofocus-range`
- `--autofocus-speed`
- `--autofocus-window`
- `--lens-position`
- `--shutter`
- `--gain`
- `--ev`
- `--awb`
- `--awbgains`
- `--metering`
- `--exposure`
- `--denoise`
- `--sharpness`
- `--contrast`
- `--brightness`
- `--saturation`
- `--roi`
- `--hflip`, `--vflip`, `--rotation`
- `--tuning-file`
- `--metadata`, `--metadata-format json`는 별도 검토

### 4.7 `src/network/`

| 파일 | 역할 |
|---|---|
| `UdpTelemetrySender.hpp/.cpp` | UDP telemetry 송신. broadcast 허용. JSON payload 송신 |

새 telemetry field가 늘어도 이 계층은 payload string 송신만 담당하므로 큰 변경은 없다.

### 4.8 `src/protocol/`

| 파일 | 역할 |
|---|---|
| `TelemetryMessage.hpp/.cpp` | onboard telemetry 구조체와 JSON serialization |

Pi 4 + IMX519-78 전환에서 protocol 확장 후보가 가장 많다. camera/system/health/autofocus/exposure 관련 optional field를 추가하고 GCS parser/log도 맞춰야 한다.

### 4.9 `src/video/`

| 파일 | 역할 |
|---|---|
| `VideoPacket.hpp/.cpp` | `AQV1` UDP video header serialization/parsing. GCS 쪽 동일 파일과 형식 일치 필요 |
| `UdpMjpegStreamer.hpp/.cpp` | JPEG frame을 1200-byte payload chunk로 쪼개 UDP 송신. chunk pacing 지원 |

IMX519-78 고해상도 JPEG는 chunk 수가 증가한다. 단순히 해상도를 올리면 complete frame 확률이 낮아질 수 있으므로 debug video 송신량을 별도 제어해야 한다.

### 4.10 `src/vision/`

| 파일 | 역할 |
|---|---|
| `VisionTypes.hpp` | marker/line/vision result 구조체 |
| `ArucoDetector.hpp/.cpp` | OpenCV ArUco detector, marker id/corners/center/orientation 계산 |
| `LineDetector.hpp/.cpp` | ROI resize, grayscale/local contrast mask, morphology, contour scoring, lookahead projection으로 line 검출 |
| `LineStabilizer.hpp/.cpp` | line result EMA smoothing, hold, jump rejection, velocity limit |
| `OpenCvCameraSource.hpp/.cpp` | OpenCV `VideoCapture` camera source. 현재 Pi CSI main path는 아니고 `camera_preview`에서 사용 |

IMX519-78 변경은 line/ArUco algorithm 자체보다 입력 영상 특성에 영향을 준다.

- FOV가 바뀌므로 `roi_top_ratio`, `lookahead_y_ratio`, `max_line_width_ratio`, `min_area_px` 재튜닝 필요
- 16MP 센서에서 960x720/1280x960 입력을 쓰면 2m 고도 line/marker pixel 수가 늘어남
- autofocus나 auto exposure가 흔들리면 line mask가 흔들릴 수 있으므로 고정 focus/exposure 정책 검토 필요

### 4.11 `src/app/`

| 파일 | 역할 |
|---|---|
| `VisionDebugPipeline.hpp/.cpp` | 현재 vision debug 핵심 실행 경로. camera read, JPEG decode, ArUco/line, telemetry build/send, video submit 담당 |

Pi 4 + IMX519-78에서 다음 변경이 필요하다.

- camera command option 전달 확장
- per-frame camera metadata 수집 가능성 검토
- capture FPS, processing FPS, debug video FPS 분리 계측
- 고해상도 capture와 저해상도 debug video를 분리할지 판단
- CPU 온도 외 throttling, CPU load, memory, Wi-Fi 상태 송신

### 4.12 `src/main.cpp`

| 파일 | 역할 |
|---|---|
| `src/main.cpp` | 기본 telemetry bring-up sender. camera/vision 없이 주기적 JSON 송신 |

### 4.13 `tools/`

| 파일 | 역할 |
|---|---|
| `vision_debug_node.cpp` | 현재 주 실행 도구. GCS discovery, CLI override, `VisionDebugPipeline` 구동 |
| `video_streamer.cpp` | rpicam/test-pattern/image source를 UDP MJPEG로 송신하는 video smoke tool |
| `camera_preview.cpp` | OpenCV `VideoCapture` 기반 camera smoke. Pi CSI에서는 rpicam 경로가 더 안정적이라 역할 재검토 필요 |
| `line_detector_tuner.cpp` | 이미지 파일 기반 line detector parameter tuning |
| `aruco_detector_tester.cpp` | 이미지 파일 기반 ArUco detector smoke |
| `replay_vision.cpp` | placeholder scaffold |
| `mock_gcs_command.cpp` | placeholder scaffold |

Pi 4 + IMX519-78에서는 `camera_preview`를 rpicam 기반 smoke로 바꾸거나, README에서 보조/비권장 도구로 명확히 분리하는 것이 좋다.

### 4.14 `tests/`, `test_data/`, `logs/`

| 경로 | 역할 |
|---|---|
| `tests/CMakeLists.txt` | onboard test target 등록 |
| `tests/test_telemetry_line_json.cpp` | telemetry JSON에 line/debug field가 들어가는지 확인 |
| `tests/test_line_stabilizer.cpp` | line stabilizer hold/jump/reacquire 동작 확인 |
| `test_data/images/.gitkeep` | 이미지 샘플 저장 위치 |
| `test_data/videos/.gitkeep` | 영상 샘플 저장 위치 |
| `test_data/logs/.gitkeep` | 로그 샘플 저장 위치 |
| `logs/.gitkeep` | runtime log 위치 placeholder |

새 telemetry field를 추가하면 onboard JSON test를 확장해야 한다.

## 5. `uav-gcs` 구조와 파일 역할

```text
uav-gcs/
├─ CMakeLists.txt
├─ PROJECT_SPEC.md
├─ README.md
├─ config/
├─ docs/
├─ logs/
├─ src/
├─ tests/
└─ tools/
```

### 5.1 루트 문서와 빌드

| 파일 | 역할 |
|---|---|
| `CMakeLists.txt` | gcs core/video libraries, OpenCV 또는 Win32/WIC video backend, executables/tools/tests 구성 |
| `PROJECT_SPEC.md` | GCS 최종 목표 문서. 시스템 그림은 Pi Zero 2 W 기준이라 Pi 4로 갱신 필요 |
| `README.md` | telemetry receiver, video receiver, vision debug receiver, local mock, tests 실행 안내 |

### 5.2 `config/`

| 파일 | 역할 |
|---|---|
| `network.toml` | onboard IP placeholder, telemetry/video/command port, timeout/retry 설정 |
| `ui.toml` | window size, layout flag, video title/timeout 설정 |

### 5.3 `docs/`

| 파일 | 역할 |
|---|---|
| `PROTOCOL.md` | onboard와 동일한 protocol v1.5 문서 |

### 5.4 `src/common/`, `src/network/`

| 파일 | 역할 |
|---|---|
| `common/NetworkConfig.hpp/.cpp` | GCS network config 파싱 |
| `network/UdpTelemetryReceiver.hpp/.cpp` | UDP telemetry receiver. timeout 기반 수신 |

### 5.5 `src/protocol/`

| 파일 | 역할 |
|---|---|
| `TelemetryMessage.hpp/.cpp` | telemetry JSON parser, backward compatibility, packet seq stats |

새 telemetry field를 onboard에 추가하면 이 parser와 tests를 같이 확장해야 한다.

### 5.6 `src/video/`

| 파일 | 역할 |
|---|---|
| `VideoPacket.hpp/.cpp` | onboard와 동일한 `AQV1` UDP video header |
| `JpegFrameReassembler.hpp/.cpp` | chunk를 JPEG frame으로 재조립. incomplete/old/mismatch stats 유지 |
| `UdpMjpegReceiver.hpp/.cpp` | UDP packet 수신, header parse, reassembler로 전달 |
| `GcsDiscoveryBeacon.hpp/.cpp` | GCS video receiver가 `AQGCS1 video_port=...` beacon을 broadcast |

고해상도 IMX519 debug video를 계속 MJPEG chunk 방식으로 보낼 경우, GCS는 complete/incomplete/frame byte/chunk count 지표를 계속 핵심 판단 기준으로 써야 한다.

### 5.7 `src/app/`

| 파일 | 역할 |
|---|---|
| `VideoViewerApp.hpp/.cpp` | video-only receiver. GCS discovery beacon 송신, MJPEG 표시 |
| `VisionDebugApp.hpp/.cpp` | telemetry thread + video receive thread + UI draw loop. 최신 complete frame과 telemetry를 frame id 기준으로 매칭 |

새 system/camera telemetry를 log window에 표시하려면 `VisionDebugApp` 자체보다 `TelemetryStore`와 `VisionLogFormatter` 수정이 중심이다.

### 5.8 `src/telemetry/`

| 파일 | 역할 |
|---|---|
| `TelemetryStore.hpp/.cpp` | 최근 frame telemetry를 저장하고 video frame id/timestamp에 맞춰 찾음 |
| `VisionLogFormatter.hpp/.cpp` | vision/debug/video/system stats를 사람이 읽을 text로 포맷 |
| `MarkerLogFormatter.hpp/.cpp` | 기존 marker log formatter |

Pi 4 + IMX519-78 telemetry 개선 시 GCS에서 가장 많이 바뀔 영역이다.

### 5.9 `src/overlay/`

| 파일 | 역할 |
|---|---|
| `OverlayPrimitive.hpp` | line/circle/text overlay primitive 공통 구조 |
| `LineOverlay.hpp/.cpp` | line contour와 tracking point를 magenta/green overlay로 변환 |
| `MarkerOverlay.hpp/.cpp` | marker corners/center/orientation arrow/label overlay 생성 |

새 field 중 `line_width`, `intersection_type`, `camera ROI` 등을 표시하려면 이 영역 확장 가능성이 있다.

### 5.10 `src/ui/`

| 파일 | 역할 |
|---|---|
| `VideoWindow.hpp` | video display 공통 인터페이스 |
| `VideoWindow.cpp` | OpenCV highgui backend. JPEG decode, latency text, overlay drawing |
| `VideoWindowWin32.cpp` | OpenCV 없는 Windows용 WIC/Win32 fallback backend |
| `VisionLogWindow.hpp/.cpp` | Windows에서는 별도 log window, 그 외 환경에서는 terminal fallback |

고해상도 영상에서는 OpenCV `WINDOW_AUTOSIZE`가 화면보다 커질 수 있으므로 `WINDOW_NORMAL` 또는 display scaling 검토가 필요하다.

### 5.11 `src/main.cpp`, entry points, tools

| 파일 | 역할 |
|---|---|
| `src/main.cpp` | basic telemetry receiver |
| `src/video_main.cpp` | `uav_gcs_video` CLI/config entry |
| `src/vision_debug_main.cpp` | `uav_gcs_vision_debug` CLI/config entry |
| `tools/mock_onboard.cpp` | GCS 없이 mock telemetry를 UDP로 송신 |
| `tools/log_replayer.cpp` | placeholder scaffold |

### 5.12 `tests/`, `logs/`

| 경로 | 역할 |
|---|---|
| `tests/CMakeLists.txt` | GCS test target 등록 |
| `tests/test_telemetry_line_parse.cpp` | telemetry line/debug parser 회귀 테스트 |
| `tests/test_line_overlay.cpp` | line overlay primitive 생성 테스트 |
| `tests/test_video_reassembler.cpp` | incomplete video frame stats 테스트 |
| `logs/.gitkeep` | runtime log 위치 placeholder |

새 telemetry field를 추가하면 GCS parse/store/log tests를 추가해야 한다.

## 6. Raspberry Pi 4 + IMX519-78 전환 영향

### 6.1 하드웨어/OS bring-up

필수 확인:

1. Raspberry Pi OS Lite 64-bit Bookworm 계열로 시작한다.
2. Pi 4는 USB-C 5V 3A급 전원과 방열판/팬을 기본 전제로 한다.
3. Pi 4의 CSI 커넥터에 IMX519-78 FFC 케이블 방향을 맞춘다. Waveshare 문서 기준 Pi 4 계열은 FFC 금속면이 HDMI 쪽을 향한다.
4. `rpicam-hello --list-cameras`에서 IMX519가 보이는지 확인한다.
5. camera가 안 보이면 OS/kernel/rpicam 버전과 IMX519 vendor driver 설치 상태를 먼저 확인한다.
6. `rpicam-still -t 1000 --nopreview -o test_data/images/imx519_smoke.jpg`로 still capture 확인.
7. `rpicam-vid -t 5000 --nopreview --codec mjpeg --width 640 --height 480 --framerate 12 -o /tmp/imx519_test.mjpeg`로 MJPEG path 확인.

주의:

- IMX519-78 autofocus는 driver와 software support가 필요하다.
- `vcgencmd get_camera` 같은 legacy 확인보다 `rpicam-hello --list-cameras`를 기준으로 삼는다.
- vendor driver 설치가 필요한 경우 스크립트가 자동으로 무조건 덮어쓰게 만들기보다, 감지와 안내를 먼저 구현한다.

### 6.2 문서 수정 범위

수정 대상:

| 파일 | 수정 내용 |
|---|---|
| `uav-onboard/PROJECT_SPEC.md` | Companion Computer를 Raspberry Pi 4, camera를 IMX519-78로 갱신 |
| `uav-gcs/PROJECT_SPEC.md` | 시스템 다이어그램의 Pi Zero 2 W 표기를 Pi 4로 갱신 |
| `uav-onboard/README.md` | Pi 4/IMX519 camera bring-up, autofocus/focus calibration, Pi 4 power/cooling, test matrix 추가 |
| `uav-gcs/README.md` | 고해상도 debug video 운용 주의, GCS log에서 확인할 새 지표 추가 |
| `development-log/RESEARCH.md` | 최신 기준 hardware와 리스크 갱신 |
| `uav-onboard/docs/PROTOCOL.md` | telemetry field 추가 시 v1.5 문서화 |
| `uav-gcs/docs/PROTOCOL.md` | onboard protocol 문서와 동일하게 유지 |
| `development-log/TROUBLESHOOTING.md` | IMX519 camera detection/focus/driver 문제 해결 절차 추가 |

### 6.3 dependency/script 수정 범위

`uav-onboard/scripts/setup_rpi_dependencies.sh` 수정 후보:

- `rpicam-hello --version` 존재 여부 확인
- `rpicam-hello --list-cameras`를 안내 또는 optional check로 추가
- `python3`, `python3-pip`, `i2c-tools` 등 focus tool 확인용 package 필요성 검토
- Bookworm/Bullseye에 따라 `rpicam-apps`와 `libcamera-apps` 이름 차이 안내
- vendor IMX519 driver 설치는 자동 실행보다 문서화 또는 explicit option으로 분리

권장 정책:

- 기본 dependency script는 안전한 apt package 설치만 수행한다.
- IMX519 vendor driver는 `--with-imx519-driver` 같은 명시 option을 별도 설계하기 전까지 문서 안내로 둔다.

### 6.4 camera config/code 수정 범위

현재 `vision.toml`의 `[video]`가 capture와 debug video를 동시에 의미한다. Pi 4 + IMX519 전환에서는 이 의미를 분리하는 것이 좋다.

권장 config 방향:

```toml
[camera]
index = 0
width = 960
height = 720
fps = 12
codec = "mjpeg"
jpeg_quality = 45
autofocus_mode = "manual"        # continuous, auto, manual
autofocus_range = "normal"
autofocus_speed = "normal"
lens_position = 0.67             # 1 / distance_m, 예: 1.5m 근처
exposure = "sport"
shutter_us = 0                   # 0이면 auto
gain = 0.0                       # 0이면 auto
awb = "auto"
denoise = "cdn_fast"
tuning_file = ""
hflip = false
vflip = false
rotation = 0

[debug_video]
enabled = true
send_fps = 8
jpeg_quality = 40
chunk_pacing_us = 150
send_width = 0                   # 0이면 source JPEG 그대로
send_height = 0
```

초기 구현에서는 backward compatibility를 위해 기존 `[video]`를 유지하고, 새 `[camera]`, `[debug_video]`를 단계적으로 도입한다.

수정 대상:

| 파일 | 수정 내용 |
|---|---|
| `src/common/VisionConfig.hpp/.cpp` | camera/rpicam/debug_video 설정 구조체와 TOML parser 추가 |
| `src/camera/RpicamMjpegSource.hpp/.cpp` | rpicam command builder 확장 |
| `tools/vision_debug_node.cpp` | 핵심 camera override CLI 추가 또는 config 중심으로 정리 |
| `tools/video_streamer.cpp` | 동일 camera option 반영 |
| `src/app/VisionDebugPipeline.cpp` | capture 설정과 debug video 송신 설정 분리 |
| `uav-onboard/config/vision.toml` | Pi 4 + IMX519 기준 default 후보 반영 |

### 6.5 autofocus/focus 정책

IMX519-78은 AF camera이지만, 드론 하향 카메라에서는 continuous AF가 항상 좋은 선택은 아니다.

검증해야 할 모드:

| 모드 | 장점 | 리스크 |
|---|---|---|
| `continuous` | 고도 변화에 자동 대응 | 이동 중 focus hunting으로 line confidence가 흔들릴 수 있음 |
| `auto` | 시작 시 한 번 focus sweep | 고도 변화가 크면 재초점 안 됨 |
| `manual + lens_position` | 반복성과 latency 안정성 좋음 | 고도별 calibration 필요 |

초기 권장:

- bench test는 `continuous`, `auto`, `manual`을 모두 비교한다.
- 실제 mission-like 주행은 `manual + lens_position`을 우선 후보로 둔다.
- lens position은 dioptre라 대략 `1 / distance_m`로 시작한다. 예: 1.2m는 0.83, 1.5m는 0.67, 2.0m는 0.50. 단, 렌즈 calibration 오차가 있으므로 실측으로 보정한다.

### 6.6 resolution/FPS 정책

Pi 4와 IMX519 덕분에 검출 입력 해상도를 올릴 수 있다. 다만 UDP MJPEG complete FPS는 chunk 수가 늘면 악화될 수 있다.

초기 테스트 matrix:

| 후보 | capture | capture FPS | JPEG quality | debug send FPS | line process_width | 목적 |
|---|---:|---:|---:|---:|---:|---|
| A | 640x480 | 12 | 45 | 10 | 480 | 기존 baseline |
| B | 960x720 | 12 | 45 | 8 | 640 | Pi 4 1차 후보 |
| C | 1280x960 | 10 | 40 | 5 | 640 또는 720 | line/marker pixel 증가 확인 |
| D | 1280x720 | 12 | 40 | 8 | 640 | 16:9 crop 영향 비교 |
| E | 960x720 | 15 | 40 | 10 | 640 | Pi 4 여유 확인 |

판단 기준:

- line-only no-video에서 processing FPS가 안정적인가
- aruco+line no-video에서 latency가 허용되는가
- aruco+line video에서 GCS complete/display FPS가 관제 가능한가
- high resolution에서 UDP incomplete frame이 늘지 않는가
- line confidence와 ArUco 검출률이 실제로 좋아지는가

### 6.7 exposure/white balance 정책

라인 검출은 밝기 기반이므로 auto exposure/gain/AWB 변화가 mask를 흔들 수 있다.

검증 항목:

- default auto exposure
- `--exposure sport`
- fixed `--shutter` + auto gain
- fixed `--shutter` + fixed `--gain`
- `--ev` 보정
- `--awb daylight/cloudy/auto`
- fixed `--awbgains`
- `--denoise off/cdn_fast`
- `--contrast`, `--sharpness`

초기 권장:

- 실내 bench에서는 auto와 sport를 비교한다.
- 움직임이 생기는 실제 비행에서는 짧은 shutter를 우선한다.
- 색 기반 line 확장을 하기 전까지는 AWB보다 grayscale contrast 안정성이 우선이다.

## 7. 장치 업그레이드로 개선 가능한 onboard 송신 정보

현재 telemetry는 vision/debug 중심이다. Pi 4 전환 후에는 CPU/RAM/network 여유가 커지므로 mission-critical loop를 막지 않는 선에서 더 많은 상태 정보를 보낼 수 있다.

### 7.1 즉시 추가 가치가 큰 field

권장 추가 field:

```json
{
  "system": {
    "board_model": "Raspberry Pi 4 Model B",
    "os_release": "Raspberry Pi OS ...",
    "uptime_s": 1234.0,
    "cpu_temp_c": 58.2,
    "throttled_raw": "0x0",
    "cpu_load_1m": 0.72,
    "mem_available_kb": 3120000,
    "wifi_signal_dbm": -48,
    "wifi_tx_bitrate_mbps": 72.2
  },
  "camera": {
    "sensor_model": "imx519",
    "camera_index": 0,
    "capture_width": 960,
    "capture_height": 720,
    "configured_fps": 12.0,
    "measured_capture_fps": 11.8,
    "autofocus_mode": "manual",
    "lens_position": 0.67,
    "exposure_mode": "sport",
    "shutter_us": 0,
    "gain": 0.0,
    "awb": "auto"
  }
}
```

초기 구현에서는 실제 per-frame metadata가 아니라 configured value와 OS/sysfs/command output 기반 health부터 추가한다.

### 7.2 IMX519 metadata 기반 확장 후보

`rpicam-*`는 metadata output을 지원한다. 다만 현재 코드는 stdout을 JPEG stream으로 사용하므로 metadata를 같은 stdout에 섞으면 frame parser가 깨질 수 있다.

검토 후보:

- `--metadata <file>` 또는 별도 fd/file로 metadata sidecar를 쓰고 onboard가 읽는다.
- `--metadata-format json`을 사용해 exposure/gain/lens/AF state를 주기적으로 파싱한다.
- 안정성이 낮으면 rpicam stdout 방식 대신 libcamera/Picamera2 기반 capture abstraction을 후속 검토한다.

metadata로 얻고 싶은 값:

- actual exposure time
- analogue/digital gain
- lens position
- AF state
- focus measure
- frame duration
- sensor timestamp
- colour gains

### 7.3 vision telemetry 개선 후보

Pi 4/IMX519로 입력 해상도가 올라가면 다음 정보를 보내는 가치가 커진다.

| Field 후보 | 이유 |
|---|---|
| `vision.line.center_offset_norm` | 해상도 변경 후에도 controller/GCS가 일관된 값 사용 |
| `vision.line.tracking_point_norm` | overlay와 제어 입력을 해상도 독립적으로 비교 |
| `vision.line.estimated_width_px` | 고도/초점/threshold 문제 진단 |
| `vision.line.polarity` | `light_on_dark`, `dark_on_light`, `auto-selected` 상태 표시 |
| `vision.line.mask_strategy` | field log 해석성 개선 |
| `vision.line.lookahead_y_px` | tuning 값과 실제 tracking point 위치 확인 |
| `vision.intersection.type` | `none`, `cross`, `T`, `L`로 확장 준비 |
| `vision.intersection.center_px` | 교차점 진입 판단과 GCS overlay 개선 |
| `vision.marker.pose` | camera calibration 이후 marker 거리/방향 추정 |

### 7.4 video/network telemetry 개선 후보

| Field 후보 | 이유 |
|---|---|
| `debug.capture_fps` | rpicam read가 실제로 설정 FPS를 만족하는지 확인 |
| `debug.processing_fps` | detector loop 성능 확인 |
| `debug.video_bytes_per_sec` | debug video bandwidth 추정 |
| `debug.video_chunk_pacing_us` | 현장 log에서 config 확인 |
| `debug.debug_video_send_fps` | capture FPS와 video FPS 분리 표시 |
| `debug.socket_send_failures` | Wi-Fi/UDP 송신 문제 진단 |
| `debug.rpicam_restarts` | camera process 재시작 감지 시 기록 |

GCS에서만 알 수 있는 값:

- completed frames
- incomplete frames
- displayed FPS
- old packets
- malformed packets
- latest overwritten frames

이 값은 onboard telemetry가 아니라 GCS log에 유지한다.

### 7.5 Pixhawk/MAVLink 연동 이후 송신할 정보

Pi 4 전환 자체와 별개지만, 성능 여유가 생기므로 이후 protocol에 넣을 후보:

- armed
- flight mode
- altitude
- attitude roll/pitch/yaw
- battery voltage/current/percent
- Pixhawk heartbeat age
- failsafe state
- current mission state
- current grid coordinate
- visited cells
- marker map
- current control command summary

## 8. 구현 우선순위

### 1단계: 문서와 bring-up 절차 갱신

목표:

- 팀원이 Pi 4 + IMX519-78을 같은 절차로 켤 수 있게 한다.

작업:

- README/PROJECT_SPEC/RESEARCH hardware 표기 갱신
- IMX519-78 camera detection/focus smoke command 추가
- Pi 4 power/cooling/CSI cable 방향/OS 기준 문서화
- `setup_rpi_dependencies.sh`에 rpicam version/list-cameras 확인 안내 추가

완료 기준:

- 새 Pi에서 `rpicam-hello --list-cameras`와 still/MJPEG smoke test 절차가 문서만 보고 가능하다.

### 2단계: rpicam option config화

목표:

- autofocus, focus, exposure, gain, AWB, denoise, orientation을 재빌드 없이 바꾼다.

작업:

- `VisionConfig`에 camera/rpicam options 추가
- `RpicamMjpegSource` command builder 확장
- `vision_debug_node`, `video_streamer`가 새 config를 사용하도록 정리
- 기존 `[video]` 설정과 backward compatibility 유지

완료 기준:

- `vision.toml`만 수정해서 AF mode, lens position, exposure mode를 바꿀 수 있다.

### 3단계: Pi 4 + IMX519 baseline 측정

목표:

- 업그레이드가 실제로 line/ArUco/video에 주는 영향을 숫자로 확인한다.

테스트 조합:

```bash
./build/vision_debug_node --config config --line-only --no-video --count 300
./build/vision_debug_node --config config --line-only --count 300
./build/vision_debug_node --config config --count 300
./build/vision_debug_node --config config --aruco-only --count 300
```

기록:

- `read_frame_ms`
- `jpeg_decode_ms`
- `line_latency_ms`
- `aruco_latency_ms`
- `processing_latency_ms`
- `video_send_ms`
- `video_chunk_count`
- `video_jpeg_bytes`
- GCS `completed/incomplete/display_fps`
- `cpu_temp_c`
- `vcgencmd get_throttled`

완료 기준:

- Pi Zero 2 W 기준 결과와 Pi 4 결과를 같은 matrix로 비교할 수 있다.

### 4단계: telemetry v1.5 확장

목표:

- hardware/camera/system 상태를 GCS에서 바로 판단할 수 있게 한다.

작업:

- onboard `TelemetryMessage`에 optional `system` 또는 확장 `debug` field 추가
- GCS parser/store/log formatter 확장
- protocol 문서 양쪽 갱신
- tests 갱신

초기 field 우선순위:

1. `system.board_model`
2. `system.throttled_raw`
3. `system.cpu_load_1m`
4. `system.mem_available_kb`
5. `camera.sensor_model`
6. `camera.measured_capture_fps`
7. `camera.autofocus_mode`
8. `camera.lens_position`
9. `camera.exposure_mode`
10. `debug.capture_fps`, `debug.processing_fps`

완료 기준:

- GCS vision log에서 Pi 4/IMX519 운용 상태를 영상 없이도 판단할 수 있다.

### 5단계: resolution/focus/exposure tuning

목표:

- 2m 내외 고도에서 line과 marker 검출 안정성을 높인다.

작업:

- 640x480 baseline, 960x720, 1280x960 비교
- `process_width` 480/640/720 비교
- AF continuous/auto/manual 비교
- lens position 고도별 calibration
- exposure sport/fixed shutter 비교
- line config 재튜닝

완료 기준:

- line confidence와 tracking point가 Pi Zero 2 W 대비 좋아졌는지 raw camera frame 기준으로 확인된다.
- focus hunting이나 exposure hunting이 line stabilizer를 흔들지 않는다.

### 6단계: debug video 송신량 분리

목표:

- 검출은 더 좋은 입력을 쓰되 GCS debug video complete FPS를 유지한다.

선택지:

1. 현재처럼 source JPEG를 그대로 보낸다. 가장 단순하지만 고해상도에서 chunk loss 위험이 크다.
2. Pi 4에서 decoded frame을 저해상도 JPEG로 재인코딩해 debug video만 낮춰 보낸다.
3. H.264/GStreamer/MediaMTX 계열로 debug video transport를 별도 설계한다.

초기 권장:

- 1번으로 baseline을 측정한다.
- incomplete frame이 커지면 2번을 구현한다.
- 3번은 protocol 변경 폭이 크므로 후속 단계로 둔다.

완료 기준:

- line/ArUco는 높은 입력 품질을 쓰면서 GCS displayed FPS가 8-12FPS 이상 안정적이다.

### 7단계: GCS 표시 개선

목표:

- 새 field와 고해상도 video를 GCS에서 읽기 좋게 표시한다.

작업:

- `VisionLogFormatter`에 system/camera/focus/exposure/processing FPS 표시
- OpenCV video window scaling 검토
- latency는 clock offset 때문에 절대값만 믿지 않도록 표시 보정 또는 주의 문구 추가
- GCS log에 resolution/fps/config snapshot 출력

완료 기준:

- 현장 테스트 중 GCS 창만 보고 camera/focus/exposure/system/video 병목을 구분할 수 있다.

## 9. 검증 체크리스트

### 9.1 camera bring-up

```bash
rpicam-hello --version
rpicam-hello --list-cameras
rpicam-still -t 1000 --nopreview -o test_data/images/imx519_smoke.jpg
rpicam-vid -t 5000 --nopreview --codec mjpeg --width 640 --height 480 --framerate 12 -o /tmp/imx519_test.mjpeg
```

### 9.2 focus 비교

```bash
rpicam-still -t 1000 --nopreview --autofocus-mode continuous -o test_data/images/focus_continuous.jpg
rpicam-still -t 1000 --nopreview --autofocus-mode auto -o test_data/images/focus_auto.jpg
rpicam-still -t 1000 --nopreview --autofocus-mode manual --lens-position 0.67 -o test_data/images/focus_manual_067.jpg
```

### 9.3 onboard/GCS 통합

GCS:

```powershell
cd uav-gcs
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Pi:

```bash
cd ~/astroquad/uav-onboard
cmake --build build
./build/vision_debug_node --config config --line-only
./build/vision_debug_node --config config
```

### 9.4 성공 기준

| 항목 | 기준 |
|---|---|
| camera detection | `rpicam-hello --list-cameras`에서 IMX519 확인 |
| line-only no-video | processing loop가 설정 FPS 근처 유지 |
| line-only video | GCS display FPS 8-12 이상, incomplete 증가 과도하지 않음 |
| aruco+line no-video | line latency와 aruco latency가 mission loop에 허용 |
| aruco+line video | debug video가 밀려도 vision telemetry는 계속 송신 |
| thermal | `get_throttled`가 정상이고 cpu temp가 장시간 급상승하지 않음 |
| focus | manual/auto/continuous 중 line confidence가 가장 안정적인 정책 선정 |
| exposure | 밝기 변화에도 line mask false positive가 크게 증가하지 않음 |

## 10. 리스크와 판단 기준

| 리스크 | 대응 |
|---|---|
| IMX519 driver/autofocus 미동작 | OS/kernel/rpicam version, vendor driver 설치 여부, `rpicam-hello --list-cameras` 기준으로 분리 진단 |
| continuous AF focus hunting | manual lens position 또는 auto-on-start로 전환 |
| 고해상도 MJPEG UDP chunk loss | debug video FPS/quality 낮추기, 저해상도 재인코딩, H.264 후속 검토 |
| exposure hunting으로 line confidence 흔들림 | sport/fixed shutter/gain/AWB gains 테스트 |
| OpenCV `camera_preview` 혼란 | Pi CSI main path는 rpicam이라고 README에서 명확화하거나 rpicam smoke tool로 교체 |
| protocol field 증가로 GCS parser 누락 | unknown field compatibility 유지, tests 동시 수정 |
| Pi 4 발열/전원 | 5V 3A 전원, fan/heatsink, `get_throttled` telemetry/log 추가 |

## 11. 결론

Raspberry Pi 4 + IMX519-78 전환은 단순히 기존 코드를 더 빠른 보드에서 돌리는 작업이 아니다. IMX519-78은 autofocus, 더 높은 해상도, 다른 FOV, 노출/게인 제어 특성이 들어오므로 camera 설정을 config로 끌어올리고, focus/exposure/resolution을 측정 가능한 방식으로 튜닝해야 한다.

가장 먼저 해야 할 일은 문서와 bring-up 절차를 Pi 4 + IMX519-78 기준으로 갱신하고, `RpicamMjpegSource`가 rpicam camera control option을 받을 수 있게 만드는 것이다. 그 다음 Pi 4 baseline을 측정한 뒤 telemetry v1.5로 system/camera/focus/exposure 상태를 추가한다. 고해상도 검출이 실제로 좋아지더라도 debug video는 mission-critical이 아니므로, UDP MJPEG complete FPS가 흔들리면 저해상도 재인코딩 또는 H.264 계열 transport를 별도 단계로 검토한다.
