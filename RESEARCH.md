# Astroquad 프로젝트 리서치

최종 업데이트: 2026-04-28

범위: `development-log`, `uav-gcs`, `uav-onboard`의 최신 디렉터리 구조, 주요 파일 역할, 현재 구현 상태, 검증 결과, 남은 리스크와 추천 다음 단계.

## 1. 현재 목표

Astroquad는 Raspberry Pi Zero 2 W 기반 onboard와 노트북 GCS를 나누어, 라인 기반 격자 경기장에서 UAV가 임무를 수행하도록 만드는 C++ 프로젝트다.

현재까지의 개발 초점:

- Raspberry Pi camera bring-up
- UDP telemetry
- UDP MJPEG debug video streaming
- GCS video display
- GCS-side marker/line overlay
- onboard ArUco detection
- onboard line tracing MVP
- line tracing latency/debug counter 표시
- line tracking point 안정화
- 고도 증가 시 라인 인식률 개선

아직 구현하지 않은 핵심 기능:

- Pixhawk/MAVLink 기반 실제 기체 제어
- 교차점 `+`, `T`, `L` 판단
- 격자 좌표 저장/이동 판단
- mission command TCP channel
- marker 발견 위치와 최종 임무 판단

## 2. 저장소 구성

루트 `astroquad/` 자체는 git 저장소가 아니며, 하위 3개 폴더가 각각 독립 git 저장소다.

```text
astroquad/
├─ development-log/
│  ├─ PLAN.md
│  ├─ RESEARCH.md
│  └─ TROUBLESHOOTING.md
├─ uav-gcs/
└─ uav-onboard/
```

| 저장소 | 역할 |
|---|---|
| `development-log` | 리서치, 구현 계획, 트러블슈팅 기록 |
| `uav-gcs` | 노트북 GCS, telemetry/video 수신, overlay, log window |
| `uav-onboard` | Raspberry Pi runtime, camera, onboard vision, telemetry/video 송신 |

## 3. 전체 아키텍처 판단

현재 설계는 onboard가 vision detection을 수행하고, GCS는 받은 metadata로 overlay만 그린다.

```text
Pi camera
  -> onboard JPEG frame capture
  -> onboard JPEG decode for detectors
  -> ArUcoDetector
  -> LineDetector
  -> LineStabilizer
  -> telemetry JSON
  -> raw MJPEG debug stream
  -> GCS frame sync
  -> GCS marker/line overlay
  -> GCS vision log
```

중요 원칙:

- 실제 임무 판단과 제어에 필요한 값은 onboard에서 계산한다.
- GCS video/overlay/log는 관찰용 best-effort debug channel이다.
- debug video가 밀리면 오래된 frame을 버리고 mission-critical vision loop를 막지 않는다.
- protocol은 v1.3이며, 이번 고고도 라인 개선은 telemetry schema 변경 없이 내부 detector/stabilizer만 개선했다.

## 4. `development-log` 파일 역할

| 파일 | 역할 |
|---|---|
| `PLAN.md` | 고고도 라인 트레이싱 개선의 수행 현황과 남은 작업 |
| `RESEARCH.md` | 현재 프로젝트 구조, 파일 역할, 구현 상태, 검증 결과, 다음 단계 |
| `TROUBLESHOOTING.md` | 실제 문제 발생, 원인 분석, 해결 과정 기록 |

최근 기록된 주요 이슈:

- Wi-Fi/SSH/hotspot 설정 정리
- GCS video stream discovery와 broadcast/unicast 동작
- Windows firewall과 UDP 수신 문제
- ArUco overlay 구현
- line tracing MVP 구현
- 발/검정 물체를 라인처럼 인식한 false positive
- 라인 중심점이 아래로 치우치던 문제
- 교차점 모양을 너무 보수적으로 reject하던 문제
- latency 증가 원인 분석과 video queue/drop 설계
- 고도 증가에 따른 라인 pixel 부족과 local contrast 개선 계획/구현

## 5. `uav-onboard` 최신 구조와 역할

```text
uav-onboard/
├─ CMakeLists.txt
├─ PROJECT_SPEC.md
├─ README.md
├─ config/
│  ├─ autopilot.toml
│  ├─ mission.toml
│  ├─ network.toml
│  ├─ safety.toml
│  └─ vision.toml
├─ docs/
│  └─ PROTOCOL.md
├─ scripts/
│  └─ setup_rpi_dependencies.sh
├─ src/
│  ├─ app/
│  │  ├─ VisionDebugPipeline.cpp
│  │  └─ VisionDebugPipeline.hpp
│  ├─ camera/
│  │  ├─ CameraFrame.hpp
│  │  ├─ RpicamMjpegSource.cpp
│  │  └─ RpicamMjpegSource.hpp
│  ├─ common/
│  │  ├─ NetworkConfig.cpp
│  │  ├─ NetworkConfig.hpp
│  │  ├─ Time.cpp
│  │  ├─ Time.hpp
│  │  ├─ VisionConfig.cpp
│  │  └─ VisionConfig.hpp
│  ├─ network/
│  │  ├─ UdpTelemetrySender.cpp
│  │  └─ UdpTelemetrySender.hpp
│  ├─ protocol/
│  │  ├─ TelemetryMessage.cpp
│  │  └─ TelemetryMessage.hpp
│  ├─ video/
│  │  ├─ UdpMjpegStreamer.cpp
│  │  ├─ UdpMjpegStreamer.hpp
│  │  ├─ VideoPacket.cpp
│  │  └─ VideoPacket.hpp
│  ├─ vision/
│  │  ├─ ArucoDetector.cpp
│  │  ├─ ArucoDetector.hpp
│  │  ├─ LineDetector.cpp
│  │  ├─ LineDetector.hpp
│  │  ├─ LineStabilizer.cpp
│  │  ├─ LineStabilizer.hpp
│  │  ├─ OpenCvCameraSource.cpp
│  │  ├─ OpenCvCameraSource.hpp
│  │  └─ VisionTypes.hpp
│  └─ main.cpp
├─ tests/
│  ├─ CMakeLists.txt
│  ├─ test_line_stabilizer.cpp
│  └─ test_telemetry_line_json.cpp
└─ tools/
   ├─ aruco_detector_tester.cpp
   ├─ camera_preview.cpp
   ├─ line_detector_tuner.cpp
   ├─ mock_gcs_command.cpp
   ├─ replay_vision.cpp
   ├─ video_streamer.cpp
   └─ vision_debug_node.cpp
```

### 5.1 Onboard 핵심 파일

| 파일 | 역할 |
|---|---|
| `src/app/VisionDebugPipeline.*` | camera frame read, JPEG decode, detector 실행, telemetry/video 송신을 묶는 debug pipeline |
| `src/camera/RpicamMjpegSource.*` | Raspberry Pi `rpicam-vid` MJPEG stdout을 frame source로 사용 |
| `src/common/VisionConfig.*` | `config/vision.toml` 파싱. line local contrast, process width, stabilizer 설정 포함 |
| `src/vision/ArucoDetector.*` | OpenCV ArUco marker detection |
| `src/vision/LineDetector.*` | line mask 생성, contour 후보 평가, lookahead band projection으로 tracking point 계산 |
| `src/vision/LineStabilizer.*` | raw line result를 EMA/hold/jump rejection/velocity limit으로 안정화 |
| `src/vision/VisionTypes.hpp` | marker, line, vision result 데이터 구조 |
| `src/protocol/TelemetryMessage.*` | telemetry JSON serialization |
| `src/video/UdpMjpegStreamer.*` | MJPEG frame UDP chunk 송신, GCS discovery beacon 처리 |
| `tools/vision_debug_node.cpp` | 현재 주 테스트 실행 파일. ArUco + line + telemetry + video |
| `tools/line_detector_tuner.cpp` | 이미지 파일 기반 line detector 튜닝 도구 |

### 5.2 최신 line 설정

`uav-onboard/config/vision.toml`의 주요 line 설정:

```toml
[line]
enabled = true
mode = "light_on_dark"
mask_strategy = "local_contrast"
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
lookahead_band_ratio = 0.06
threshold = 0
local_contrast_blur = 31
local_contrast_threshold = 12
min_area_px = 250
morph_kernel = 5
max_contour_points = 48
confidence_min = 0.25
min_line_width_px = 8
max_line_width_ratio = 0.22
process_width = 480
max_candidates = 8
filter_enabled = true
filter_ema_alpha = 0.35
filter_confidence_alpha_min = 0.35
filter_min_confidence = 0.25
filter_max_offset_jump_ratio = 0.16
filter_max_offset_velocity_ratio = 0.08
filter_max_angle_jump_deg = 90.0
filter_hold_frames = 3
filter_reacquire_frames = 3
intersection_threshold = 0.8
```

이번 변경의 핵심:

- `process_width`: `320 -> 480`
- `mask_strategy`: global threshold 대신 `local_contrast`
- `lookahead_band_ratio`: 한 줄 scan 대신 band projection
- `confidence_min`: `0.20 -> 0.25`
- `filter_confidence_alpha_min`: low confidence measurement를 천천히 반영
- `filter_max_offset_velocity_ratio`: accepted measurement도 frame당 이동량 제한

## 6. `uav-gcs` 최신 구조와 역할

```text
uav-gcs/
├─ CMakeLists.txt
├─ PROJECT_SPEC.md
├─ README.md
├─ config/
│  ├─ network.toml
│  └─ ui.toml
├─ docs/
│  └─ PROTOCOL.md
├─ src/
│  ├─ app/
│  │  ├─ VideoViewerApp.cpp
│  │  ├─ VideoViewerApp.hpp
│  │  ├─ VisionDebugApp.cpp
│  │  └─ VisionDebugApp.hpp
│  ├─ common/
│  │  ├─ NetworkConfig.cpp
│  │  └─ NetworkConfig.hpp
│  ├─ network/
│  │  ├─ UdpTelemetryReceiver.cpp
│  │  └─ UdpTelemetryReceiver.hpp
│  ├─ overlay/
│  │  ├─ LineOverlay.cpp
│  │  ├─ LineOverlay.hpp
│  │  ├─ MarkerOverlay.cpp
│  │  ├─ MarkerOverlay.hpp
│  │  └─ OverlayPrimitive.hpp
│  ├─ protocol/
│  │  ├─ TelemetryMessage.cpp
│  │  └─ TelemetryMessage.hpp
│  ├─ telemetry/
│  │  ├─ MarkerLogFormatter.cpp
│  │  ├─ MarkerLogFormatter.hpp
│  │  ├─ TelemetryStore.cpp
│  │  ├─ TelemetryStore.hpp
│  │  ├─ VisionLogFormatter.cpp
│  │  └─ VisionLogFormatter.hpp
│  ├─ ui/
│  │  ├─ VideoWindow.cpp
│  │  ├─ VideoWindow.hpp
│  │  ├─ VideoWindowWin32.cpp
│  │  ├─ VisionLogWindow.cpp
│  │  └─ VisionLogWindow.hpp
│  ├─ video/
│  │  ├─ GcsDiscoveryBeacon.cpp
│  │  ├─ GcsDiscoveryBeacon.hpp
│  │  ├─ JpegFrameReassembler.cpp
│  │  ├─ JpegFrameReassembler.hpp
│  │  ├─ UdpMjpegReceiver.cpp
│  │  ├─ UdpMjpegReceiver.hpp
│  │  ├─ VideoPacket.cpp
│  │  └─ VideoPacket.hpp
│  ├─ main.cpp
│  ├─ video_main.cpp
│  └─ vision_debug_main.cpp
├─ tests/
│  ├─ CMakeLists.txt
│  ├─ test_line_overlay.cpp
│  └─ test_telemetry_line_parse.cpp
└─ tools/
   ├─ log_replayer.cpp
   └─ mock_onboard.cpp
```

### 6.1 GCS 핵심 파일

| 파일 | 역할 |
|---|---|
| `src/app/VisionDebugApp.*` | video frame과 telemetry를 동기화하고 overlay/log window를 구동 |
| `src/overlay/LineOverlay.*` | onboard line contour를 magenta, tracking point를 green으로 변환 |
| `src/overlay/MarkerOverlay.*` | ArUco marker box, center, direction arrow, label primitive 생성 |
| `src/protocol/TelemetryMessage.*` | protocol v1.3 telemetry parse |
| `src/telemetry/VisionLogFormatter.*` | latency, detector state, queue drop, line raw/filtered 상태 표시 |
| `src/video/GcsDiscoveryBeacon.*` | GCS IP discovery beacon broadcast |
| `src/video/JpegFrameReassembler.*` | UDP MJPEG chunk reassembly |
| `src/ui/VideoWindow*` | OpenCV/Win32 video window backend |

이번 onboard 개선은 protocol schema를 바꾸지 않았으므로 GCS 코드는 변경하지 않았다. GCS는 기존 `vision.line.*`와 `debug.line_*` fields를 그대로 표시한다.

## 7. Protocol 상태

현재 protocol 문서는 두 저장소에서 v1.3으로 유지된다.

| 파일 | 상태 |
|---|---|
| `uav-onboard/docs/PROTOCOL.md` | v1.3 |
| `uav-gcs/docs/PROTOCOL.md` | v1.3 |

주요 telemetry:

- `vision.markers[]`: ArUco marker list
- `vision.line.detected`: stabilizer 적용 후 사용 가능한 line 여부
- `vision.line.raw_detected`: 현재 raw detector result 여부
- `vision.line.filtered`: stabilizer 적용 여부
- `vision.line.held`: 이전 line을 hold 중인지
- `vision.line.rejected_jump`: raw jump를 reject했는지
- `vision.line.tracking_point_px`: GCS green point
- `vision.line.raw_tracking_point_px`: filter 전 raw point
- `vision.line.center_offset_px`: 최종 horizontal offset
- `vision.line.raw_center_offset_px`: raw offset
- `vision.line.confidence`: detector confidence
- `vision.line.contour_px`: GCS magenta contour
- `debug.line_latency_ms`: line detector latency
- `debug.processing_latency_ms`: JPEG decode + detector processing latency
- `debug.video_dropped_frames`: video worker가 늦어서 버린 frame 수
- `debug.line_contours_found`, `debug.line_candidates_evaluated`, `debug.line_roi_pixels`, `debug.line_selected_contour_points`

이번 변경은 internal algorithm/config 변경이므로 protocol version을 올리지 않았다.

## 8. 현재 실행 조합

GCS 노트북:

```powershell
cd uav-gcs
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi:

```bash
cd ~/astroquad/uav-onboard
git pull
cmake --build build
./build/vision_debug_node --config config
```

라인만 확인:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

ArUco만 확인:

```bash
./build/vision_debug_node --config config --aruco-only
```

네트워크가 broadcast/discovery를 막는 경우:

```bash
./build/vision_debug_node --config config --gcs-ip <laptop-ip>
```

## 9. 로컬 검증 결과

OpenCV C++ local build:

```powershell
cmake -S . -B build-opencv-local -DCMAKE_BUILD_TYPE=Release -DOpenCV_DIR=C:/msys64/ucrt64/lib/cmake/opencv4 -DBUILD_TESTS=OFF
cmake --build build-opencv-local --target line_detector_tuner vision_debug_node
```

Core/test build:

```powershell
cmake --build build
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

결과:

- `telemetry_line_json`: passed
- `line_stabilizer`: passed
- `line_detector_tuner`: build passed
- `vision_debug_node`: OpenCV local build passed

첨부 이미지 테스트:

| 이미지 | 환경/고도 | 결과 | offset | confidence |
|---|---|---|---:|---:|
| `v2_line1_1.png` | 방 바닥, 약 40cm | detected | `-5.05px` | `0.89` |
| `v2_line1_2.png` | 방 바닥, 약 150cm | detected | `5.02px` | `0.76` |
| `v2_line2_1.png` | 검은 천, 약 100cm | detected | `-133.83px` | `0.84` |
| `v2_line2_2.png` | 검은 천, 약 150cm | detected | `-29.91px` | `0.87` |

연습용 격자 기본/그늘/햇빛 이미지도 모두 detected 상태로 확인했다.

## 10. 현재 리스크

- 로컬 이미지 검증은 정지 이미지 기반이다. 실제 Pi camera의 motion blur, exposure, gain, JPEG 압축, 진동은 아직 반영되지 않았다.
- `process_width = 480`은 고고도 인식에는 유리하지만 Pi Zero 2 W에서 `line_latency_ms`가 증가할 수 있다.
- local contrast는 밝은 라인에는 유리하지만, 강한 직사광 하이라이트가 line-like 구조로 생기면 false positive가 다시 생길 수 있다.
- `+`, `T`, `L` 교차점 판단은 아직 없으므로 현재 contour가 교차 모양으로 보여도 mission logic에는 쓰이지 않는다.
- 현재 stabilizer는 tracking point 안정화용이며, 실제 드론 이동 제어 gains와 결합된 검증은 아직 필요하다.

## 11. 추천 다음 단계

1. Raspberry Pi에서 `vision_debug_node --line-only`로 150cm, 180cm, 190cm, 200cm 고도별 line tracking을 촬영한다.
2. GCS vision log에서 `line_latency_ms`, `processing_latency_ms`, `video_dropped_frames`, `confidence`, `held`, `rejected_jump`를 기록한다.
3. 실기 영상에서 false positive가 남으면 `local_contrast_threshold`, `local_contrast_blur`, `confidence_min`, `max_line_width_ratio`를 조정한다.
4. 고도 180~190cm가 안정화되면 `IntersectionCandidate`를 별도 경로로 추가한다.
5. 교차점 판단이 안정화된 뒤에만 grid 좌표 저장과 이동 판단을 구현한다.
6. 마지막으로 Pixhawk/MAVLink 제어 loop와 line offset을 결합한다.
