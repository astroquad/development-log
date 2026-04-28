# Astroquad 프로젝트 리서치

최종 업데이트: 2026-04-28

범위: `development-log`, `uav-gcs`, `uav-onboard`의 최신 구조, 주요 파일 역할, 구현 상태, 검증 결과, 남은 리스크와 추천 다음 단계.

## 1. 현재 목표

Astroquad는 Raspberry Pi Zero 2 W onboard와 노트북 GCS를 나누어, 라인 기반 격자 경기장에서 UAV가 임무를 수행하도록 만드는 C++ 프로젝트다.

현재까지 구현/개선된 영역:

- Raspberry Pi camera MJPEG capture
- UDP telemetry
- UDP MJPEG debug video streaming
- GCS video display
- GCS-side ArUco/line overlay
- onboard ArUco detection
- onboard line tracing MVP
- line tracing stabilizer
- 고도 증가 시 line detection 개선
- GCS video receive thread 분리
- video frame drop/FPS/chunk 계측 강화
- Pi CPU temperature telemetry

아직 남은 핵심 기능:

- Pixhawk/MAVLink 기반 실제 기체 제어
- 교차점 `+`, `T`, `L` 판단
- 격자 좌표 저장 및 이동 판단
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

## 3. 전체 아키텍처

```text
Pi camera
  -> rpicam-vid MJPEG capture
  -> onboard JPEG decode for detectors
  -> ArUcoDetector
  -> LineDetector
  -> LineStabilizer
  -> telemetry JSON
  -> raw MJPEG debug video
  -> GCS video receive thread
  -> GCS frame/telemetry sync
  -> GCS marker/line overlay
  -> GCS vision log
```

중요 원칙:

- 실제 임무 판단과 제어에 필요한 값은 onboard에서 계산한다.
- GCS video/overlay/log는 관찰용 best-effort debug channel이다.
- debug video가 밀리면 오래된 frame을 버리고 mission-critical vision loop를 막지 않는다.
- protocol 문서는 v1.4이며, JSON top-level `protocol_version` integer는 여전히 `1`이다.

## 4. `uav-onboard` 최신 구조와 역할

```text
uav-onboard/
├─ config/
│  ├─ network.toml
│  └─ vision.toml
├─ docs/
│  └─ PROTOCOL.md
├─ src/
│  ├─ app/VisionDebugPipeline.cpp
│  ├─ camera/RpicamMjpegSource.cpp
│  ├─ common/VisionConfig.cpp
│  ├─ protocol/TelemetryMessage.cpp
│  ├─ video/UdpMjpegStreamer.cpp
│  └─ vision/
│     ├─ ArucoDetector.cpp
│     ├─ LineDetector.cpp
│     ├─ LineStabilizer.cpp
│     └─ VisionTypes.hpp
├─ tests/
│  ├─ test_line_stabilizer.cpp
│  └─ test_telemetry_line_json.cpp
└─ tools/
   ├─ line_detector_tuner.cpp
   ├─ video_streamer.cpp
   └─ vision_debug_node.cpp
```

주요 파일:

| 파일 | 역할 |
|---|---|
| `src/app/VisionDebugPipeline.cpp` | camera read, JPEG decode, ArUco/line 실행, telemetry/video 송신 |
| `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid --codec mjpeg` stdout frame source |
| `src/common/VisionConfig.cpp` | `config/vision.toml` 파싱 |
| `src/protocol/TelemetryMessage.cpp` | telemetry JSON serialization |
| `src/video/UdpMjpegStreamer.cpp` | MJPEG frame UDP chunk 송신, chunk pacing |
| `src/vision/LineDetector.cpp` | local contrast mask, morphology, contour scoring, lookahead band projection |
| `src/vision/LineStabilizer.cpp` | EMA, hold, jump rejection, velocity limit |
| `tools/line_detector_tuner.cpp` | 이미지 파일 기반 line detector 튜닝 |
| `tools/vision_debug_node.cpp` | 현재 주 테스트 실행 파일 |

### 4.1 최신 video/line 설정

`uav-onboard/config/vision.toml` 주요값:

```toml
[video]
width = 640
height = 480
fps = 12
jpeg_quality = 45
send_fps = 10
chunk_pacing_us = 150

[line]
enabled = true
mode = "light_on_dark"
mask_strategy = "local_contrast"
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
lookahead_band_ratio = 0.06
local_contrast_blur = 31
local_contrast_threshold = 10
min_area_px = 250
morph_kernel = 5
morph_open_kernel = 1
morph_close_kernel = 7
morph_dilate_kernel = 1
line_run_merge_gap_px = 16
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
```

이번 변경의 핵심:

- video capture 기본값 `15FPS/quality50`에서 `12FPS/quality45`로 낮춤
- debug video send FPS를 `10`으로 제한
- UDP chunk 사이 `150us` pacing 추가
- local contrast threshold `12 -> 10`
- morphology open/close kernel 분리
- close kernel을 키워 라인이 두 edge로 갈라지는 문제 완화
- projection run merge gap 추가
- onboard telemetry에 video send/chunk/skip/cpu temperature 지표 추가

## 5. `uav-gcs` 최신 구조와 역할

```text
uav-gcs/
├─ config/
│  ├─ network.toml
│  └─ ui.toml
├─ docs/
│  └─ PROTOCOL.md
├─ src/
│  ├─ app/
│  │  ├─ VideoViewerApp.cpp
│  │  └─ VisionDebugApp.cpp
│  ├─ overlay/
│  │  ├─ LineOverlay.cpp
│  │  └─ MarkerOverlay.cpp
│  ├─ protocol/TelemetryMessage.cpp
│  ├─ telemetry/
│  │  ├─ TelemetryStore.cpp
│  │  └─ VisionLogFormatter.cpp
│  ├─ ui/
│  │  ├─ VideoWindow.cpp
│  │  ├─ VideoWindowWin32.cpp
│  │  └─ VisionLogWindow.cpp
│  └─ video/
│     ├─ GcsDiscoveryBeacon.cpp
│     ├─ JpegFrameReassembler.cpp
│     └─ UdpMjpegReceiver.cpp
└─ tests/
   ├─ test_line_overlay.cpp
   ├─ test_telemetry_line_parse.cpp
   └─ test_video_reassembler.cpp
```

주요 파일:

| 파일 | 역할 |
|---|---|
| `src/app/VisionDebugApp.cpp` | telemetry thread와 video receive thread를 구동하고 최신 complete frame 표시 |
| `src/video/UdpMjpegReceiver.cpp` | UDP video packet 수신 및 packet 통계 |
| `src/video/JpegFrameReassembler.cpp` | chunk를 JPEG frame으로 재조립하고 incomplete/drop 통계 |
| `src/telemetry/VisionLogFormatter.cpp` | latency, video stats, line stats, CPU temperature 표시 |
| `src/overlay/LineOverlay.cpp` | line contour와 tracking point overlay primitive 생성 |
| `tests/test_video_reassembler.cpp` | incomplete frame stats 회귀 테스트 |

이번 변경의 핵심:

- `uav_gcs_vision_debug`에서 UDP video 수신을 UI draw loop와 분리
- UI thread는 최신 complete JPEG frame만 표시
- video receiver 통계 추가: packets, malformed, completed, incomplete, old packets, mismatch resets, last chunk count, last frame bytes
- displayed FPS 표시 추가

## 6. Protocol v1.4 상태

문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

새 debug fields:

| Field | 의미 |
|---|---|
| `debug.video_send_ms` | onboard video worker의 최근 UDP chunk 송신 시간 |
| `debug.cpu_temp_c` | Pi CPU 온도. 사용 불가 시 `0.0` |
| `debug.video_skipped_frames` | video send FPS 제한 때문에 의도적으로 video 송신을 생략한 frame 수 |
| `debug.video_chunks_sent` | onboard가 보낸 누적 UDP video chunk 수 |
| `debug.video_chunk_count` | 최근 video frame의 chunk 수 |

기존 주요 fields:

- `vision.line.detected`
- `vision.line.raw_detected`
- `vision.line.filtered`
- `vision.line.held`
- `vision.line.rejected_jump`
- `vision.line.tracking_point_px`
- `vision.line.raw_tracking_point_px`
- `vision.line.center_offset_px`
- `vision.line.confidence`
- `vision.line.contour_px`
- `debug.line_latency_ms`
- `debug.processing_latency_ms`
- `debug.video_dropped_frames`

## 7. 현재 실행 조합

GCS 노트북:

```powershell
cd uav-gcs
git pull
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

검정 라인 확인:

```bash
./build/vision_debug_node --config config --line-only --line-mode dark_on_light
```

네트워크 discovery가 막히면:

```bash
./build/vision_debug_node --config config --gcs-ip <laptop-ip>
```

## 8. 로컬 검증 결과

빌드/테스트:

```powershell
cmake --build uav-onboard/build
cmake --build uav-onboard/build-tests
ctest --test-dir uav-onboard/build-tests --output-on-failure

cmake --build uav-gcs/build
cmake --build uav-gcs/build-tests
ctest --test-dir uav-gcs/build-tests --output-on-failure
```

결과:

- onboard `telemetry_line_json`: passed
- onboard `line_stabilizer`: passed
- GCS `telemetry_line_parse`: passed
- GCS `video_reassembler`: passed
- GCS `line_overlay`: passed

OpenCV local tuner:

```powershell
cmake -S uav-onboard -B uav-onboard/build-opencv-local -DCMAKE_BUILD_TYPE=Release -DOpenCV_DIR=C:/msys64/ucrt64/lib/cmake/opencv4 -DBUILD_TESTS=OFF
cmake --build uav-onboard/build-opencv-local --target line_detector_tuner vision_debug_node
```

GCS screenshot 기반 v3 smoke 결과:

| 이미지 | 결과 | 비고 |
|---|---|---|
| `v3_line1.png` | detected | screenshot에는 overlay/주변 물체가 포함되어 있어 raw camera 판정과 다를 수 있음 |
| `v3_line2_1.png` | detected | 검은 천 위 흰 라인 유지 |
| `v3_line2_2.png` | detected | 검은 천 위 흰 라인 유지 |
| `v3_line3.png` | detected | 십자형 구조 회귀 없음 |
| `v3_line4.png` | detected | L자형 구조 회귀 없음 |

주의: 위 이미지는 GCS 화면 캡처라 기존 magenta/green overlay와 텍스트가 포함된다. 실제 검출 성능은 Pi raw camera frame으로 다시 확인해야 한다.

## 9. 현재 리스크

- Pi 실기에서 video complete/display FPS가 실제로 개선됐는지 아직 확인 필요
- chunk pacing은 packet loss를 줄일 수 있지만 send time을 늘릴 수 있으므로 현장 수치 확인 필요
- `v3_line1` 같은 마루+휴지 hard case는 실제 경기장 대표 조건이 아닐 수 있음
- `morph_close_kernel = 7`은 라인 edge 갈라짐을 줄이지만 강한 반사 환경에서는 false positive를 키울 수 있음
- Pi Zero 2 W 발열이 이미 심하므로 fan/heatsink와 `cpu_temp_c`, `vcgencmd get_throttled` 확인이 필요
- ArUco + line + video를 모두 항상 켜면 실제 mission logic과 Pixhawk 제어까지 얹었을 때 여유가 작을 수 있음

## 10. 추천 다음 단계

1. Pi에서 `vision_debug_node --line-only --line-mode light_on_dark`로 150cm, 180cm, 190cm, 200cm 테스트
2. GCS log에서 `display_fps`, `completed`, `incomplete`, `video_chunk_count`, `video_send_ms`, `cpu_temp_c` 기록
3. `line-only no-video`, `line-only video`, `aruco+line video` 조합으로 CPU/온도/프레임 비교
4. 실제 흙색 배경 + 흰색 무광 테이프 또는 송진가루 테스트 샘플 확보
5. 검정 라인 가능성에 대비해 같은 샘플에서 `dark_on_light` 테스트
6. line tracking이 안정화된 뒤 `IntersectionCandidate`를 별도 경로로 구현
7. 그 다음 grid 좌표 저장, 이동 판단, Pixhawk/MAVLink 제어 loop 구현
