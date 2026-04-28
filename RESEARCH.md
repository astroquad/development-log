# Astroquad 프로젝트 리서치

최종 업데이트: 2026-04-29

범위: 현재 작업 디렉토리의 tracked 파일 기준 `development-log`, `uav-onboard`, `uav-gcs` 구조, 파일 역할, 구현 상태, 런타임 기본값, 검증 이력, 남은 리스크를 최신화한다. `build/`, `.git/`, 캡처 산출물처럼 tracked가 아닌 생성물은 구조 설명에서 제외했다.

## 1. 현재 목표와 설계 원칙

Astroquad는 Raspberry Pi 4 + IMX519-78 onboard와 Windows 노트북 GCS를 나누어, 라인 기반 격자 경기장에서 UAV가 ArUco marker와 line tracing 정보를 기반으로 임무를 수행하도록 만드는 C++ 프로젝트다. Raspberry Pi Zero 2 W + ZeroCam에서 시작한 저부하 원칙은 유지하지만, 현재 기본 hardware target은 Raspberry Pi 4 + IMX519-78이다.

핵심 원칙:

- Onboard는 camera capture, ArUco/line detection, telemetry 생성, 추후 mission 판단과 MAVLink 제어를 담당한다.
- GCS는 telemetry/video 수신, log 표시, raw camera 영상 위 overlay drawing을 담당한다.
- ArUco box, line contour, tracking point, label 같은 overlay는 절대 onboard에서 그리지 않는다.
- Debug video는 관제/튜닝용 best-effort channel이다. 지연되거나 drop되어도 mission-critical vision/control loop를 막으면 안 된다.
- 최종 시연/비행은 GCS 없이 onboard 단독 실행도 가능해야 하므로, mission 판단에 필요한 값은 telemetry 이전에 onboard에서 계산되어야 한다.
- Protocol 문서는 v1.5이며 JSON top-level `protocol_version`은 호환성을 위해 여전히 integer `1`이다.

현재 구현된 영역:

- Raspberry Pi `rpicam-vid` 기반 MJPEG camera frame capture
- UDP JSON telemetry 송신/수신
- UDP MJPEG chunk debug video 송신/수신
- GCS discovery beacon과 video unicast 전환
- GCS video display, Win32/WIC fallback, OpenCV backend
- GCS-side ArUco/line overlay
- Onboard ArUco detection
- Onboard line tracing MVP와 stabilizer
- Pi 4 + IMX519 camera/focus/exposure config
- Onboard system/camera/debug telemetry v1.5
- GCS vision log window와 packet/video/line/marker/system/camera 로그

아직 구현되지 않았거나 placeholder인 영역:

- Pixhawk/MAVLink 실제 제어 loop
- Mission command TCP channel
- 교차점 `+`, `T`, `L` 분류
- 격자 좌표 저장과 이동 판단
- marker 발견 위치 기반 최종 임무 판단
- 안전/배터리/heartbeat 기반 fail-safe 로직

## 2. 저장소 구성

루트 `astroquad/` 자체는 git 저장소가 아니며, 하위 3개 폴더가 각각 독립 git 저장소다.

```text
astroquad/
├─ development-log/
│  ├─ PLAN.md
│  ├─ RESEARCH.md
│  └─ TROUBLESHOOTING.md
├─ uav-onboard/
└─ uav-gcs/
```

| 저장소 | 역할 |
|---|---|
| `development-log` | 계획, 리서치, 트러블슈팅 및 설계 판단 기록 |
| `uav-onboard` | Raspberry Pi runtime, IMX519 camera, onboard vision, telemetry/debug video 송신 |
| `uav-gcs` | 노트북 GCS, telemetry/debug video 수신, GCS-side overlay, log window |

## 3. Runtime Architecture

```text
IMX519 camera
  -> rpicam-vid MJPEG stdout
  -> RpicamMjpegSource frame extraction
  -> one JPEG decode for onboard detectors
  -> ArucoDetector and/or LineDetector
  -> LineStabilizer
  -> TelemetryMessage JSON
  -> UDP telemetry
  -> optional raw MJPEG debug video
  -> GCS video receive thread
  -> GCS TelemetryStore frame_seq matching
  -> GCS MarkerOverlay/LineOverlay drawing
  -> GCS camera window and vision log window
```

현재 `vision_debug_node` 기본값은 metadata-only다. `config/vision.toml`의 `debug_video.enabled = false`이므로 `--video`를 붙이지 않으면 GCS camera window는 `waiting for video stream...` 상태일 수 있지만, telemetry/log는 정상으로 볼 수 있다. 이 동작은 onboard 부담을 줄이기 위한 의도된 기본값이다.

## 4. `development-log` 파일 역할

```text
development-log/
├─ PLAN.md
├─ RESEARCH.md
└─ TROUBLESHOOTING.md
```

| 파일 | 역할 |
|---|---|
| `PLAN.md` | Pi 4 + IMX519 전환 계획과 단계별 작업 계획. 기존 계획은 완료되었으며 다음 작업 시 덮어써도 되는 작업 문서 |
| `RESEARCH.md` | 현재 구조, 파일 역할, 구현 상태, 설계 원칙, 런타임 기본값 최신 문서 |
| `TROUBLESHOOTING.md` | bring-up 중 발생한 문제, 원인, 해결, 설계 판단 누적 로그 |

## 5. `uav-onboard` 구조와 파일 역할

```text
uav-onboard/
├─ .gitignore
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
├─ logs/.gitkeep
├─ scripts/
│  ├─ .gitkeep
│  └─ setup_rpi_dependencies.sh
├─ src/
│  ├─ app/
│  ├─ autopilot/
│  ├─ camera/
│  ├─ common/
│  ├─ control/
│  ├─ logging/
│  ├─ mission/
│  ├─ network/
│  ├─ protocol/
│  ├─ safety/
│  ├─ video/
│  └─ vision/
├─ test_data/
│  ├─ images/.gitkeep
│  ├─ logs/.gitkeep
│  └─ videos/.gitkeep
├─ tests/
└─ tools/
```

### 5.1 Root, Config, Docs, Scripts

| 파일 | 현재 역할 |
|---|---|
| `.gitignore` | CMake build 산출물, 임시 파일, local output 제외 |
| `CMakeLists.txt` | C++17, `nlohmann/json`, `tomlplusplus`, OpenCV optional target, tests/tools target 정의 |
| `PROJECT_SPEC.md` | 장기 시스템 설계 문서. MissionManager, MAVLink, grid 등 일부 항목은 아직 구현 전인 목표 구조 |
| `README.md` | Pi 4 + IMX519 bring-up, build, 실행 예시, metadata-only/debug-video 운용 원칙 |
| `config/network.toml` | GCS broadcast IP, telemetry/video/command port, telemetry interval |
| `config/vision.toml` | IMX519 camera, legacy video, opt-in debug video, ArUco, line detector/stabilizer runtime config |
| `config/autopilot.toml` | 추후 Pixhawk/MAVLink serial/system/component 설정 placeholder |
| `config/mission.toml` | 추후 grid/marker/takeoff mission 설정 placeholder |
| `config/safety.toml` | 추후 line lost, GCS lost, Pixhawk heartbeat, low voltage timeout 설정 placeholder |
| `docs/PROTOCOL.md` | onboard/GCS 공통 protocol v1.5 명세. `uav-gcs/docs/PROTOCOL.md`와 동일해야 함 |
| `scripts/setup_rpi_dependencies.sh` | Raspberry Pi OS 의존성 설치. CMake/build tools/rpicam/OpenCV contrib/dev tools 확인 |
| `logs/.gitkeep` | runtime log 폴더 placeholder |
| `test_data/*/.gitkeep` | 이미지/로그/영상 테스트 데이터 폴더 placeholder |

### 5.2 Onboard Source

| 파일 | 현재 역할 |
|---|---|
| `src/main.cpp` | 기본 telemetry bring-up sender `uav_onboard` entry point |
| `src/app/VisionDebugPipeline.hpp` | vision debug pipeline options와 class 선언 |
| `src/app/VisionDebugPipeline.cpp` | rpicam frame read, JPEG decode, ArUco/line 실행, telemetry 생성/송신, optional latest-frame video worker, Pi system telemetry 수집 |
| `src/camera/CameraFrame.hpp` | camera frame id/timestamp/size/JPEG payload 구조체 |
| `src/camera/RpicamMjpegSource.hpp` | rpicam MJPEG source options와 class 선언 |
| `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid --codec mjpeg -o -` 실행, stdout JPEG SOI/EOI frame extraction, focus/exposure/AWB/denoise/orientation option 전달 |
| `src/common/NetworkConfig.hpp` | GCS IP/port/telemetry interval config 구조체 |
| `src/common/NetworkConfig.cpp` | `config/network.toml` 파싱 |
| `src/common/Time.hpp` | timestamp utility 선언 |
| `src/common/Time.cpp` | Unix timestamp milliseconds utility |
| `src/common/VisionConfig.hpp` | camera/video/debug_video/aruco/line config 구조체 |
| `src/common/VisionConfig.cpp` | `config/vision.toml` 파싱과 기본값 적용 |
| `src/network/UdpTelemetrySender.hpp` | UDP telemetry sender class 선언 |
| `src/network/UdpTelemetrySender.cpp` | UDP socket, broadcast option, telemetry datagram send |
| `src/protocol/TelemetryMessage.hpp` | telemetry field 구조체: system, camera, vision, line, markers, grid, debug |
| `src/protocol/TelemetryMessage.cpp` | protocol v1 JSON serialization |
| `src/video/UdpMjpegStreamer.hpp` | UDP MJPEG streamer class 선언 |
| `src/video/UdpMjpegStreamer.cpp` | JPEG frame을 `AQV1` chunk packet으로 나눠 송신, chunk pacing 적용 |
| `src/video/VideoPacket.hpp` | `AQV1` header 구조와 encode/decode 선언 |
| `src/video/VideoPacket.cpp` | big-endian UDP video packet serialize/parse |
| `src/vision/ArucoDetector.hpp` | ArUco detector 선언 |
| `src/vision/ArucoDetector.cpp` | OpenCV ArUco dictionary mapping, marker id/corners/center/orientation 계산 |
| `src/vision/LineDetector.hpp` | line detector 선언 |
| `src/vision/LineDetector.cpp` | ROI resize, local/global threshold mask, morphology, contour scoring, lookahead band tracking point, workload counter |
| `src/vision/LineStabilizer.hpp` | line stabilizer 선언 |
| `src/vision/LineStabilizer.cpp` | EMA smoothing, jump rejection, velocity limit, hold/reacquire logic |
| `src/vision/OpenCvCameraSource.hpp` | OpenCV camera preview source 선언 |
| `src/vision/OpenCvCameraSource.cpp` | legacy/local camera preview용 OpenCV `VideoCapture` wrapper |
| `src/vision/VisionTypes.hpp` | detector 내부 point/marker/line/vision result type |
| `src/autopilot/.gitkeep` | 추후 MAVLink/Pixhawk adapter 자리 |
| `src/control/.gitkeep` | 추후 control loop 자리 |
| `src/logging/.gitkeep` | 추후 onboard logging 자리 |
| `src/mission/.gitkeep` | 추후 mission state/grid/marker map 자리 |
| `src/safety/.gitkeep` | 추후 failsafe 자리 |

### 5.3 Onboard Tests and Tools

| 파일 | 현재 역할 |
|---|---|
| `tests/CMakeLists.txt` | onboard unit test target 등록 |
| `tests/test_line_stabilizer.cpp` | line stabilizer hold/jump/reacquire 동작 회귀 테스트 |
| `tests/test_telemetry_line_json.cpp` | line/debug/video telemetry JSON serialization 회귀 테스트 |
| `tools/vision_debug_node.cpp` | 현재 주 실행 파일. camera override, detector toggle, line tuning, opt-in `--video`, GCS discovery 처리 |
| `tools/video_streamer.cpp` | standalone rpicam/test-pattern/image MJPEG UDP streamer |
| `tools/line_detector_tuner.cpp` | 이미지 파일 기반 line detector 튜닝/출력 tool |
| `tools/aruco_detector_tester.cpp` | 이미지 파일 기반 ArUco detector smoke test |
| `tools/camera_preview.cpp` | OpenCV camera preview/capture smoke tool |
| `tools/mock_gcs_command.cpp` | 아직 command channel 미구현 placeholder |
| `tools/replay_vision.cpp` | 아직 vision replay 미구현 placeholder |

Tracked placeholder files with no runtime logic: `src/app/.gitkeep`, `src/common/.gitkeep`, `src/network/.gitkeep`, `src/protocol/.gitkeep`, `src/vision/.gitkeep`, `test_data/images/.gitkeep`, `test_data/logs/.gitkeep`, `test_data/videos/.gitkeep`.

## 6. `uav-onboard` 현재 런타임 기본값

`config/vision.toml`의 핵심값:

```toml
[camera]
sensor_model = "imx519"
width = 960
height = 720
fps = 12
codec = "mjpeg"
jpeg_quality = 45
autofocus_mode = "manual"
lens_position = 0.67
exposure = "sport"
awb = "auto"
denoise = "cdn_fast"

[video]
width = 640
height = 480
fps = 12
jpeg_quality = 45
send_fps = 5
chunk_pacing_us = 150
port = 5600

[debug_video]
enabled = false
send_fps = 5
jpeg_quality = 40
chunk_pacing_us = 150
send_width = 0
send_height = 0

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
morph_open_kernel = 1
morph_close_kernel = 7
morph_dilate_kernel = 1
line_run_merge_gap_px = 16
max_contour_points = 48
confidence_min = 0.25
min_line_width_px = 8
max_line_width_ratio = 0.22
process_width = 480
max_candidates = 8
filter_enabled = true
filter_ema_alpha = 0.35
filter_max_offset_jump_ratio = 0.16
filter_max_offset_velocity_ratio = 0.08
filter_hold_frames = 3
filter_reacquire_frames = 3

[aruco]
dictionary = "DICT_4X4_50"
marker_size_mm = 80.0
```

해석:

- Camera capture는 `960x720 @ 12FPS, JPEG quality 45`가 현재 기본이다.
- `debug_video.enabled = false`라서 기본 실행은 telemetry-only다.
- GCS raw video/overlay 확인이 필요할 때만 `vision_debug_node --video`를 사용한다.
- Debug video 송신은 detector frame loop와 분리되어 `5FPS`로 decimation되고 `150us` chunk pacing을 적용한다.
- `1280x960`/고품질 JPEG 실험은 ArUco bench 화질 개선 효과 대비 onboard 부담이 커서 기본값에서 제외했다.
- 대회 조건의 50cm x 50cm ArUco marker를 약 2m 고도에서 본다는 가정에서는 현재 보수적 capture 설정을 우선한다.

## 7. `uav-gcs` 구조와 파일 역할

```text
uav-gcs/
├─ .gitignore
├─ CMakeLists.txt
├─ PROJECT_SPEC.md
├─ README.md
├─ config/
│  ├─ network.toml
│  └─ ui.toml
├─ docs/
│  └─ PROTOCOL.md
├─ logs/.gitkeep
├─ scripts/.gitkeep
├─ src/
│  ├─ app/
│  ├─ common/
│  ├─ logging/
│  ├─ network/
│  ├─ overlay/
│  ├─ protocol/
│  ├─ state/
│  ├─ telemetry/
│  ├─ ui/
│  └─ video/
├─ test_data/telemetry/.gitkeep
├─ tests/
├─ third_party/.gitkeep
└─ tools/
```

### 7.1 Root, Config, Docs

| 파일 | 현재 역할 |
|---|---|
| `.gitignore` | CMake build 산출물, local output 제외 |
| `CMakeLists.txt` | C++17, dependencies, OpenCV/Win32 video backend, apps/tools/tests target 정의 |
| `PROJECT_SPEC.md` | GCS 장기 설계 문서. 명령 UI, mission state 등 일부는 목표 구조 |
| `README.md` | build/run, video/vision debug receiver, Pi bring-up 순서, latency 표시 제거 이유 |
| `config/network.toml` | onboard IP/port와 timeout 설정. 현재 telemetry 수신은 UDP port 중심 |
| `config/ui.toml` | window/layout/video window 설정. `show_latency = false`로 latency overlay 비활성 의도를 문서화 |
| `docs/PROTOCOL.md` | onboard/GCS 공통 protocol v1.5 명세. onboard protocol 문서와 동일해야 함 |
| `logs/.gitkeep` | runtime log 폴더 placeholder |
| `scripts/.gitkeep` | helper script 폴더 placeholder |
| `third_party/.gitkeep` | 필요 시 vendored dependency 자리 |
| `test_data/telemetry/.gitkeep` | telemetry replay/smoke 데이터 자리 |

### 7.2 GCS Source

| 파일 | 현재 역할 |
|---|---|
| `src/main.cpp` | 기본 telemetry receiver `uav_gcs` entry point |
| `src/app/VideoViewerApp.hpp` | video-only viewer options/class 선언 |
| `src/app/VideoViewerApp.cpp` | `uav_gcs_video` app, GCS beacon 송신, MJPEG 수신/표시 |
| `src/app/VisionDebugApp.hpp` | vision debug app options/class 선언 |
| `src/app/VisionDebugApp.cpp` | telemetry thread, video receive thread, TelemetryStore sync, GCS-side overlay, log window 구동 |
| `src/common/NetworkConfig.hpp` | GCS network config 구조체 |
| `src/common/NetworkConfig.cpp` | `config/network.toml` 파싱 |
| `src/network/UdpTelemetryReceiver.hpp` | UDP telemetry receiver class 선언 |
| `src/network/UdpTelemetryReceiver.cpp` | telemetry UDP bind/receive, timeout 처리 |
| `src/protocol/TelemetryMessage.hpp` | protocol v1.5 parser-side telemetry 구조체와 stats |
| `src/protocol/TelemetryMessage.cpp` | telemetry JSON parse, legacy field compatibility, stats update |
| `src/overlay/OverlayPrimitive.hpp` | backend-independent line/circle/text overlay primitive |
| `src/overlay/LineOverlay.hpp` | line overlay builder 선언 |
| `src/overlay/LineOverlay.cpp` | magenta contour, green tracking point, line label primitive 생성 |
| `src/overlay/MarkerOverlay.hpp` | marker overlay builder 선언 |
| `src/overlay/MarkerOverlay.cpp` | marker box, corners, center, direction arrow, label primitive 생성 |
| `src/telemetry/TelemetryStore.hpp` | frame_seq 기반 최근 vision frame store 선언 |
| `src/telemetry/TelemetryStore.cpp` | telemetry observe, video frame과 frame_seq/timestamp matching |
| `src/telemetry/VisionLogFormatter.hpp` | vision log formatter 선언 |
| `src/telemetry/VisionLogFormatter.cpp` | onboard processing timing, packet/video/system/camera/line/marker log 문자열 생성 |
| `src/telemetry/MarkerLogFormatter.hpp` | marker-only log formatter 선언 |
| `src/telemetry/MarkerLogFormatter.cpp` | marker 중심 log 문자열 생성 |
| `src/ui/VideoWindow.hpp` | video window backend 공통 interface |
| `src/ui/VideoWindow.cpp` | OpenCV `imshow` backend, JPEG decode, GCS-side overlay drawing |
| `src/ui/VideoWindowWin32.cpp` | Windows OpenCV 미설치용 WIC JPEG decode + GDI double-buffered drawing backend |
| `src/ui/VisionLogWindow.hpp` | vision log window interface |
| `src/ui/VisionLogWindow.cpp` | Windows log window 또는 stdout fallback |
| `src/video/GcsDiscoveryBeacon.hpp` | GCS discovery beacon class 선언 |
| `src/video/GcsDiscoveryBeacon.cpp` | UDP 5601 `AQGCS1 video_port=5600` beacon broadcast |
| `src/video/JpegFrameReassembler.hpp` | JPEG chunk reassembler와 stats 선언 |
| `src/video/JpegFrameReassembler.cpp` | `AQV1` chunk 재조립, incomplete/old/mismatch stats |
| `src/video/UdpMjpegReceiver.hpp` | UDP MJPEG receiver class와 stats 선언 |
| `src/video/UdpMjpegReceiver.cpp` | video UDP bind/receive/reassembler 연동 |
| `src/video/VideoPacket.hpp` | `AQV1` packet 구조 선언 |
| `src/video/VideoPacket.cpp` | packet serialize/parse |
| `src/video_main.cpp` | `uav_gcs_video` CLI entry point |
| `src/vision_debug_main.cpp` | `uav_gcs_vision_debug` CLI entry point |
| `src/logging/.gitkeep` | 추후 GCS log subsystem 자리 |
| `src/state/.gitkeep` | 추후 mission/GCS state model 자리 |

### 7.3 GCS Tests and Tools

| 파일 | 현재 역할 |
|---|---|
| `tests/CMakeLists.txt` | GCS unit test target 등록 |
| `tests/test_line_overlay.cpp` | line overlay primitive 회귀 테스트 |
| `tests/test_telemetry_line_parse.cpp` | telemetry v1.5 line/debug parse 회귀 테스트 |
| `tests/test_video_reassembler.cpp` | MJPEG chunk reassembler/drop stats 회귀 테스트 |
| `tools/mock_onboard.cpp` | GCS telemetry receiver용 mock telemetry sender |
| `tools/log_replayer.cpp` | 아직 log replay 미구현 placeholder |

Tracked placeholder files with no runtime logic: `src/app/.gitkeep`, `src/common/.gitkeep`, `src/network/.gitkeep`, `src/protocol/.gitkeep`, `src/ui/.gitkeep`.

## 8. GCS 현재 동작

- `uav_gcs`: UDP telemetry 수신과 basic console 출력.
- `uav_gcs_video`: raw MJPEG video 수신/표시.
- `uav_gcs_vision_debug`: raw MJPEG video와 telemetry를 동시에 수신하고 GCS에서 overlay를 그리며 별도 vision log window를 연다.
- Video 수신은 background thread에서 socket을 계속 drain하고, UI는 최신 complete JPEG만 표시한다.
- GCS camera overlay는 `frame N`만 표시한다. Video latency/age 표시는 제거했다.
- 음수 latency 또는 20-30ms처럼 체감과 맞지 않는 값은 Pi/Windows clock sync 부재와 best-effort buffering 때문에 오해를 만들 수 있다.
- 성능 판단은 `processing/read/decode/aruco/line/json/tsend/vsubmit/vsend`, `capture_fps`, `processing_fps`, `completed/incomplete`, `video_sent/skipped/dropped/failures`, `cpu_temp_c`, `throttled_raw` 중심으로 본다.

## 9. Protocol v1.5 핵심

문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

주요 telemetry top-level:

- `protocol_version`, `type`, `seq`, `timestamp_ms`
- `mission`
- `system`
- `camera`
- `vision`
- `grid`
- `debug`

현재 중요한 field:

| Field | 의미 |
|---|---|
| `system.board_model`, `system.os_release` | Pi board/OS 확인 |
| `system.cpu_temp_c`, `system.throttled_raw` | 발열/throttling 확인 |
| `system.cpu_load_1m`, `system.mem_available_kb` | onboard 여유 확인 |
| `system.wifi_signal_dbm`, `system.wifi_tx_bitrate_mbps` | Wi-Fi 상태 확인 |
| `camera.sensor_model`, `camera.camera_index` | IMX519/rpicam index 확인 |
| `camera.width`, `camera.height`, `camera.configured_fps` | capture config 확인 |
| `camera.measured_capture_fps`, `debug.capture_fps` | 실제 capture rate 확인 |
| `camera.autofocus_mode`, `camera.lens_position`, `camera.exposure_mode` | focus/exposure 상태 확인 |
| `vision.markers[]` | marker id/corners/center/orientation |
| `vision.line.*` | detected/raw/filtered/held/rejected/tracking point/offset/angle/confidence/contour |
| `grid.*` | 현재 placeholder. row/col 기본값은 `-1` |
| `debug.processing_latency_ms` | frame read 이후 onboard decode+detector 처리 시간 |
| `debug.read_frame_ms`, `debug.jpeg_decode_ms` | capture wait/read와 JPEG decode 시간 |
| `debug.aruco_latency_ms`, `debug.line_latency_ms` | detector별 처리 시간 |
| `debug.telemetry_build_ms`, `debug.telemetry_send_ms` | JSON serialization/send call 시간 |
| `debug.video_submit_ms`, `debug.video_send_ms` | optional debug video queue submit/send 시간 |
| `debug.debug_video_send_fps`, `debug.video_chunk_pacing_us` | debug video pacing config |
| `debug.video_jpeg_bytes`, `debug.video_chunk_count` | optional video payload 크기 |
| `debug.video_sent_frames`, `debug.video_skipped_frames`, `debug.video_dropped_frames`, `debug.video_send_failures` | optional debug video health |
| `debug.line_*` counters | line detector workload/count diagnostics |

Video packet:

- UDP port `5600`
- `AQV1` 28-byte header + JPEG chunk payload
- maximum payload per datagram `1200` bytes
- GCS는 모든 chunk를 받은 frame만 표시하고 incomplete frame은 drop한다.
- `timestamp_ms`는 frame/telemetry correlation용이며 GCS에서 latency로 표시하지 않는다.

Discovery:

- GCS가 UDP `5601`에 `AQGCS1 video_port=5600` beacon을 broadcast한다.
- Onboard video sender는 `--video`처럼 video가 실제로 켜진 경우에만 discovery를 기다린다.
- Metadata-only 실행에서는 discovery 대기 없이 바로 vision loop를 시작한다.

## 10. 권장 실행 조합

GCS:

```powershell
cd uav-gcs
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi, metadata-only line tracing:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Raspberry Pi, GCS camera/overlay까지 확인:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
```

ArUco만 bench tuning:

```bash
./build/vision_debug_node --config config --aruco-only
./build/vision_debug_node --config config --aruco-only --video
./build/vision_debug_node --config config --aruco-only --camera-quality 90 --lens-position 1.0
```

네트워크 discovery/broadcast가 막히면:

```bash
./build/vision_debug_node --config config --gcs-ip <laptop-ip>
./build/vision_debug_node --config config --video --gcs-ip <laptop-ip>
```

## 11. 검증 이력

최근 작업 기준으로 확인된 검증:

- Windows local `uav-gcs` Release build 성공.
- Windows local `uav-gcs` tests 통과: `telemetry_line_parse`, `video_reassembler`, `line_overlay`.
- Local OpenCV build에서 onboard vision tools target build 성공.
- Raspberry Pi 4 + IMX519에서 `vision_debug_node` build/run smoke 확인.
- Metadata-only 실행에서 telemetry packet은 정상 수신되고 video counter는 `video_sent=0`, `chunks_last=0`, `last_bytes=0`으로 표시됨을 확인.
- `--video`를 켠 경우 GCS camera window/overlay 확인 가능.
- 고품질 camera 설정 실험 후 성능 우선 기본값 `960x720`, `jpeg_quality=45`, debug video `5FPS`로 되돌림.
- GCS video latency/age 표시 제거 후 overlay에는 `frame N`만 표시.

문서 최신화 중 확인한 불일치:

- `uav-gcs/docs/PROTOCOL.md`에 이전 고화질 실험값 `1280x960`, `video_chunk_pacing_us=250`이 남아 있었다. 현재 기본값은 onboard protocol/config와 같은 `960x720`, `150us`가 맞다.

## 12. 현재 리스크와 다음 작업

현재 리스크:

- Mission/MAVLink/control loop가 아직 없으므로 실제 비행 판단/제어는 구현 전이다.
- 교차점과 grid state가 아직 placeholder라 line tracing은 경로 판단 전 단계다.
- IMX519 focus/exposure는 실제 고도, 속도, 조명에서 다시 calibration해야 한다.
- Debug video는 5FPS라 GCS 관제 화면은 답답할 수 있다. 하지만 현재 단계에서는 onboard 제어 여유와 telemetry 안정성이 더 중요하다.
- Wi-Fi UDP video는 packet loss가 정상적으로 발생할 수 있으므로 frame drop을 오류로 보지 말고 counters로 판단해야 한다.
- GCS video latency는 정확히 측정하지 않는다. 필요하면 clock sync가 가능한 별도 측정 설계를 추가해야 한다.

추천 다음 단계:

1. 실제 경기장과 유사한 흙색/무광 배경에서 1.5m, 1.8m, 2.0m line-only metadata 성능을 기록한다.
2. 같은 조건에서 `--video`를 켠 관제용 성능을 비교하되, 제어 loop 추가 전 기본값은 metadata-only로 유지한다.
3. ArUco 50cm x 50cm marker를 실제 2m 거리에서 `aruco-only`, `aruco+line` 각각 확인한다.
4. IMX519 `lens_position`, `autofocus_mode`, `exposure`, `shutter_us`, `gain`을 mission-like 조건에서 고정값 후보로 정한다.
5. `IntersectionCandidate`를 `vision.line`과 분리된 telemetry로 추가해 line following contour와 교차점 판단을 동시에 보존한다.
6. 그 다음 grid coordinate, mission state, Pixhawk/MAVLink control loop를 onboard critical path로 구현한다.
