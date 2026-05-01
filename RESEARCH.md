# Astroquad 프로젝트 리서치

최종 업데이트: 2026-05-01

범위: 현재 작업 디렉토리의 `development-log`, `uav-onboard`, `uav-gcs` 구조, 파일 역할, 구현 상태, 런타임 기본값, 검증 이력, 남은 리스크를 최신화한다. `build/`, `.git/` 같은 생성/관리 산출물은 구조 설명에서 제외하되, 2026-05-01 grid smoke 결과는 decision/grid 검증 산출물이므로 별도 검증 이력에 기록한다.

## 1. 현재 목표와 설계 원칙

Astroquad는 Raspberry Pi 4 + IMX519-78 onboard와 Windows 노트북 GCS를 나누어, 라인 기반 격자 경기장에서 UAV가 ArUco marker와 line tracing 정보를 기반으로 임무를 수행하도록 만드는 C++ 프로젝트다. Raspberry Pi Zero 2 W + ZeroCam에서 시작한 저부하 원칙은 유지하지만, 현재 기본 hardware target은 Raspberry Pi 4 + IMX519-78이다.

핵심 원칙:

- Onboard는 camera capture, ArUco/line detection, telemetry 생성, 추후 mission 판단과 MAVLink 제어를 담당한다.
- GCS는 telemetry/video 수신, log 표시, raw camera 영상 위 overlay drawing을 담당한다.
- ArUco box, line contour, tracking point, label 같은 overlay는 절대 onboard에서 그리지 않는다.
- Debug video는 관제/튜닝용 best-effort channel이다. 지연되거나 drop되어도 mission-critical vision/control loop를 막으면 안 된다.
- 최종 시연/비행은 GCS 없이 onboard 단독 실행도 가능해야 하므로, mission 판단에 필요한 값은 telemetry 이전에 onboard에서 계산되어야 한다.
- Protocol 문서는 v1.7이며 JSON top-level `protocol_version`은 호환성을 위해 여전히 integer `1`이다.

현재 구현된 영역:

- Raspberry Pi `rpicam-vid` 기반 MJPEG camera frame capture
- UDP JSON telemetry 송신/수신
- UDP MJPEG chunk debug video 송신/수신
- GCS discovery beacon과 video unicast 전환
- GCS video display, Win32/WIC fallback, OpenCV backend
- GCS-side ArUco/line/intersection/intersection-decision overlay
- Onboard ArUco detection
- Onboard line tracing MVP와 stabilizer
- Onboard 교차점 분류 `+`, `T`, `L`, `straight`와 temporal stabilizer
- Onboard `IntersectionDecisionEngine`: 최근 branch evidence window 기반 `L`/`T`/`+` decision, turn candidate, approach phase, overshoot risk telemetry
- Onboard `GridCoordinateTracker`: 첫 안정 node를 local `(0,0)`으로 두는 상대 좌표 기록, camera-relative branch mask의 grid-relative 변환
- Pi 4 + IMX519 camera/focus/exposure config
- Onboard system/camera/debug telemetry v1.7
- GCS vision log window와 packet/video/line/intersection/intersection-decision/grid-node/marker/system/camera 로그
- GCS-side intersection decision overlay label

아직 구현되지 않았거나 placeholder인 영역:

- Pixhawk/MAVLink 실제 제어 loop
- Mission command TCP channel
- 전체 grid snake mission policy와 official competition coordinate 변환
- marker 발견 위치 기반 최종 임무 판단
- 안전/배터리/heartbeat 기반 fail-safe 로직

## 2. 저장소 구성

루트 `astroquad/` 자체는 git 저장소가 아니며, 하위 3개 폴더가 각각 독립 git 저장소다.

```text
astroquad/
├─ development-log/
│  ├─ grid-smoke-20260501/
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
  -> ArucoDetector and/or shared LineMaskBuilder
  -> LineDetector + IntersectionDetector
  -> LineStabilizer
  -> IntersectionStabilizer
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> TelemetryMessage JSON
  -> UDP telemetry
  -> optional raw MJPEG debug video
  -> GCS video receive thread
  -> GCS TelemetryStore frame_seq matching
  -> GCS MarkerOverlay/LineOverlay/IntersectionOverlay drawing
  -> GCS camera window and vision log window
```

현재 `vision_debug_node` 기본값은 metadata-only다. `config/vision.toml`의 `debug_video.enabled = false`이므로 `--video`를 붙이지 않으면 GCS camera window는 `waiting for video stream...` 상태일 수 있지만, telemetry/log는 정상으로 볼 수 있다. 이 동작은 onboard 부담을 줄이기 위한 의도된 기본값이다.

## 4. `development-log` 파일 역할

```text
development-log/
├─ grid-smoke-20260501/
├─ PLAN.md
├─ RESEARCH.md
└─ TROUBLESHOOTING.md
```

| 파일 | 역할 |
|---|---|
| `PLAN.md` | Pi 4 + IMX519 전환 계획과 단계별 작업 계획. 기존 계획은 완료되었으며 다음 작업 시 덮어써도 되는 작업 문서 |
| `RESEARCH.md` | 현재 구조, 파일 역할, 구현 상태, 설계 원칙, 런타임 기본값 최신 문서 |
| `TROUBLESHOOTING.md` | bring-up 중 발생한 문제, 원인, 해결, 설계 판단 누적 로그 |
| `grid-smoke-20260501/` | 2026-05-01 연습용 격자 이미지 2장 기반 section crop, snake 확대 crop, CSV 검증 산출물 |

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
| `config/vision.toml` | IMX519 camera, legacy video, opt-in debug video, ArUco, line detector/stabilizer, intersection decision runtime config |
| `config/autopilot.toml` | 추후 Pixhawk/MAVLink serial/system/component 설정 placeholder |
| `config/mission.toml` | 추후 grid/marker/takeoff mission 설정 placeholder |
| `config/safety.toml` | 추후 line lost, GCS lost, Pixhawk heartbeat, low voltage timeout 설정 placeholder |
| `docs/PROTOCOL.md` | onboard/GCS 공통 protocol v1.7 명세. `uav-gcs/docs/PROTOCOL.md`와 동일해야 함 |
| `scripts/setup_rpi_dependencies.sh` | Raspberry Pi OS 의존성 설치. CMake/build tools/rpicam/OpenCV contrib/dev tools 확인 |
| `logs/.gitkeep` | runtime log 폴더 placeholder |
| `test_data/*/.gitkeep` | 이미지/로그/영상 테스트 데이터 폴더 placeholder |

### 5.2 Onboard Source

| 파일 | 현재 역할 |
|---|---|
| `src/main.cpp` | 기본 telemetry bring-up sender `uav_onboard` entry point |
| `src/app/VisionDebugPipeline.hpp` | vision debug pipeline options와 class 선언 |
| `src/app/VisionDebugPipeline.cpp` | rpicam frame read, JPEG decode, ArUco/line/intersection/decision/grid update 실행, telemetry 생성/송신, optional latest-frame video worker, Pi system telemetry 수집 |
| `src/camera/CameraFrame.hpp` | camera frame id/timestamp/size/JPEG payload 구조체 |
| `src/camera/RpicamMjpegSource.hpp` | rpicam MJPEG source options와 class 선언 |
| `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid --codec mjpeg -o -` 실행, stdout JPEG SOI/EOI frame extraction, focus/exposure/AWB/denoise/orientation option 전달 |
| `src/common/NetworkConfig.hpp` | GCS IP/port/telemetry interval config 구조체 |
| `src/common/NetworkConfig.cpp` | `config/network.toml` 파싱 |
| `src/common/Time.hpp` | timestamp utility 선언 |
| `src/common/Time.cpp` | Unix timestamp milliseconds utility |
| `src/common/VisionConfig.hpp` | camera/video/debug_video/aruco/line/intersection_decision config 구조체 |
| `src/common/VisionConfig.cpp` | `config/vision.toml` 파싱과 기본값 적용. `[intersection_decision]` 포함 |
| `src/network/UdpTelemetrySender.hpp` | UDP telemetry sender class 선언 |
| `src/network/UdpTelemetrySender.cpp` | UDP socket, broadcast option, telemetry datagram send |
| `src/mission/IntersectionDecision.hpp` | decision state/action, branch evidence, decision sample/result 구조체와 `IntersectionDecisionEngine` 선언 |
| `src/mission/IntersectionDecision.cpp` | 짧은 branch evidence window 기반 topology accept, `+` false-upgrade guard, node event, turn candidate, approach phase, overshoot risk 계산 |
| `src/mission/GridCoordinateTracker.hpp` | local grid coord, heading, node event 구조체와 `GridCoordinateTracker` 선언 |
| `src/mission/GridCoordinateTracker.cpp` | 첫 node local `(0,0)` 기록, heading 기반 좌표 증가, camera branch mask의 grid-relative mask 변환 |
| `src/protocol/TelemetryMessage.hpp` | telemetry field 구조체: system, camera, vision, line, intersection, intersection decision, grid node, markers, grid, debug |
| `src/protocol/TelemetryMessage.cpp` | protocol v1 JSON serialization. 문서 버전 v1.7의 `vision.intersection_decision`과 `vision.grid_node` 포함 |
| `src/video/UdpMjpegStreamer.hpp` | UDP MJPEG streamer class 선언 |
| `src/video/UdpMjpegStreamer.cpp` | JPEG frame을 `AQV1` chunk packet으로 나눠 송신, chunk pacing 적용 |
| `src/video/VideoPacket.hpp` | `AQV1` header 구조와 encode/decode 선언 |
| `src/video/VideoPacket.cpp` | big-endian UDP video packet serialize/parse |
| `src/vision/ArucoDetector.hpp` | ArUco detector 선언 |
| `src/vision/ArucoDetector.cpp` | OpenCV ArUco dictionary mapping, marker id/corners/center/orientation 계산 |
| `src/vision/LineMaskBuilder.hpp` | line/intersection이 공유하는 ROI geometry, polarity mask, source/work 좌표 변환 선언 |
| `src/vision/LineMaskBuilder.cpp` | ROI resize, grayscale, threshold/local contrast mask, morphology clean 공통 구현 |
| `src/vision/LineDetector.hpp` | line detector 선언 |
| `src/vision/LineDetector.cpp` | shared line mask 또는 image 입력 기반 contour scoring, lookahead band tracking point, workload counter |
| `src/vision/LineStabilizer.hpp` | line stabilizer 선언 |
| `src/vision/LineStabilizer.cpp` | EMA smoothing, jump rejection, velocity limit, hold/reacquire logic |
| `src/vision/IntersectionDetector.hpp` | 교차점 classifier 선언. `LineMaskFrame`과 raw `LineDetection`을 입력으로 받음 |
| `src/vision/IntersectionDetector.cpp` | mask bridge, largest-blob/center 후보, 4방향 ray scoring, branch count와 angle 기반 `+`/`T`/`L`/`straight` 분류 |
| `src/vision/IntersectionStabilizer.hpp` | 교차점 temporal stabilizer 선언 |
| `src/vision/IntersectionStabilizer.cpp` | type majority/reacquire, short hold, center/score EMA, raw/stable field 보존 |
| `src/vision/OpenCvCameraSource.hpp` | OpenCV camera preview source 선언 |
| `src/vision/OpenCvCameraSource.cpp` | legacy/local camera preview용 OpenCV `VideoCapture` wrapper |
| `src/vision/VisionTypes.hpp` | detector 내부 point/marker/line/intersection/vision result type |
| `src/autopilot/.gitkeep` | 추후 MAVLink/Pixhawk adapter 자리 |
| `src/control/.gitkeep` | 추후 control loop 자리 |
| `src/logging/.gitkeep` | 추후 onboard logging 자리 |
| `src/mission/.gitkeep` | legacy placeholder. 실제 mission/debug decision 코드는 `IntersectionDecision.*`, `GridCoordinateTracker.*`에 있음 |
| `src/safety/.gitkeep` | 추후 failsafe 자리 |

### 5.3 Onboard Tests and Tools

| 파일 | 현재 역할 |
|---|---|
| `tests/CMakeLists.txt` | onboard unit test target 등록 |
| `tests/test_line_stabilizer.cpp` | line stabilizer hold/jump/reacquire 동작 회귀 테스트 |
| `tests/test_intersection_stabilizer.cpp` | 교차점 type smoothing, type 전환, short hold 회귀 테스트 |
| `tests/test_intersection_decision.cpp` | branch evidence aggregation, downgrade/false-upgrade guard, turn-zone/overshoot, cooldown, local grid coordinate 회귀 테스트 |
| `tests/test_intersection_detector.cpp` | synthetic image 기반 `+`, `T`, `L`, `straight`, dark/light polarity 분류 테스트. OpenCV target이 있을 때만 등록 |
| `tests/test_telemetry_line_json.cpp` | line/intersection/intersection_decision/grid_node/debug/video telemetry JSON serialization 회귀 테스트 |
| `tools/vision_debug_node.cpp` | 현재 주 실행 파일. camera override, detector toggle, line tuning, opt-in `--video`, `--fps`, GCS discovery 처리 |
| `tools/video_streamer.cpp` | standalone rpicam/test-pattern/image MJPEG UDP streamer |
| `tools/line_detector_tuner.cpp` | 이미지 파일 기반 line detector 튜닝/출력 tool |
| `tools/grid_image_smoke.cpp` | 연습용 격자 이미지 기반 section crop, intersection decision, full/entry snake 좌표 저장 smoke tool |
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
mask_strategy = "white_fill"
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
lookahead_band_ratio = 0.06
local_contrast_blur = 31
local_contrast_threshold = 10
white_v_min = 145
white_s_max = 90
min_area_px = 250
morph_open_kernel = 1
morph_close_kernel = 7
morph_dilate_kernel = 1
fill_close_kernel = 11
fill_dilate_kernel = 3
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
intersection_threshold = 0.8

[intersection_decision]
enabled = true
fps_assumption = 12
cruise_window_frames = 6
turn_confirm_frames = 8
cooldown_frames = 8
min_cross_branch_frames = 2
min_t_branch_frames = 2
min_l_branch_frames = 3
min_branch_score = 0.72
high_confidence_score = 0.85
candidate_min_frames = 2
turn_confirm_required = 3
record_node_once_frames = 18
turn_zone_y_min = 0.42
turn_zone_y_max = 0.68
late_zone_y = 0.78
min_prearm_frames = 2
front_missing_frames = 2
node_advance_min_frames = 4

[aruco]
dictionary = "DICT_4X4_50"
marker_size_mm = 80.0
```

해석:

- Camera capture는 `960x720 @ 12FPS, JPEG quality 45`가 현재 기본이다.
- `debug_video.enabled = false`라서 기본 실행은 telemetry-only다.
- GCS raw video/overlay 확인이 필요할 때만 `vision_debug_node --video`를 사용한다.
- Debug video 송신은 detector frame loop와 분리되어 `5FPS`로 decimation되고 `150us` chunk pacing을 적용한다.
- 디버깅 중 GCS 화면이 답답하면 `vision_debug_node --video --fps 12`처럼 송신 FPS만 올릴 수 있다. `--fps`는 camera capture FPS가 아니라 GCS debug video send FPS다.
- Line/intersection 처리는 원본 960x720 전체가 아니라 ROI를 `line.process_width = 480`으로 줄인 work image에서 수행한다.
- `line.mask_strategy = "white_fill"`은 넓은 흰색 line을 얇은 edge strip이 아니라 채워진 blob으로 잡기 위한 현재 기본값이다. `local_contrast`는 밝기 변화 비교 실험용으로 계속 사용할 수 있다.
- `intersection_threshold = 0.8`은 front/right/back/left branch ray가 present로 판정되기 위한 기본 score threshold다.
- `intersection_decision.min_branch_score = 0.72`는 decision layer의 branch evidence 기준이다. 단, `+` 채택은 4방향 모두 `high_confidence_score = 0.85`를 넘어야 하므로 `T -> +` false upgrade를 억제한다.
- `cruise_window_frames = 6`은 12FPS 기준 약 0.5초 window다. 교차점마다 장시간 정지하지 않고 `L`/`T`/`+`를 node event로 기록하는 것이 목적이다.
- `record_node_once_frames = 18`과 `node_advance_min_frames = 4`는 같은 node 중복 저장과 너무 빠른 좌표 증가를 막기 위한 lockout이다.
- `turn_zone_y_min/max`, `late_zone_y`는 실제 제어 명령이 아니라 turn candidate와 overshoot risk telemetry를 위한 image-space gate다.
- `1280x960`/고품질 JPEG 실험은 ArUco bench 화질 개선 효과 대비 onboard 부담이 커서 기본값에서 제외했다.
- 대회 조건의 50cm x 50cm ArUco marker를 약 2m 고도에서 본다는 가정에서는 현재 보수적 capture 설정을 우선한다.

성능 해석:

- 카메라 해상도를 640x480으로 낮추면 JPEG decode, camera payload, optional debug video 대역폭은 줄어든다.
- 하지만 `line.process_width`를 480으로 그대로 두면 line/intersection mask 처리 크기는 크게 줄지 않는다.
- Line/intersection CPU를 직접 줄이려면 camera 해상도뿐 아니라 `line.process_width = 320`, `local_contrast_blur = 21~25` 같은 detector work image 설정을 함께 실험해야 한다.
- ArUco 원거리 인식까지 같이 봐야 하면 960x720 유지가 유리하고, line/intersection만 보는 성능 우선 실험에서는 640x480 + `process_width=320` profile이 합리적인 후보이다.

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
| `README.md` | build/run, video/vision debug receiver, intersection decision/grid-node log와 overlay, Pi bring-up 순서, latency 표시 제거 이유 |
| `config/network.toml` | onboard IP/port와 timeout 설정. 현재 telemetry 수신은 UDP port 중심 |
| `config/ui.toml` | window/layout/video window 설정. `show_latency = false`로 latency overlay 비활성 의도를 문서화 |
| `docs/PROTOCOL.md` | onboard/GCS 공통 protocol v1.7 명세. onboard protocol 문서와 동일해야 함 |
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
| `src/app/VisionDebugApp.cpp` | telemetry thread, video receive thread, TelemetryStore sync, GCS-side marker/line/intersection/decision overlay, log window 구동 |
| `src/common/NetworkConfig.hpp` | GCS network config 구조체 |
| `src/common/NetworkConfig.cpp` | `config/network.toml` 파싱 |
| `src/network/UdpTelemetryReceiver.hpp` | UDP telemetry receiver class 선언 |
| `src/network/UdpTelemetryReceiver.cpp` | telemetry UDP bind/receive, timeout 처리 |
| `src/protocol/TelemetryMessage.hpp` | protocol v1.7 parser-side telemetry 구조체와 stats. intersection decision과 grid node 구조 포함 |
| `src/protocol/TelemetryMessage.cpp` | telemetry JSON parse, legacy field compatibility, structured intersection/decision/grid-node parse, stats update |
| `src/overlay/OverlayPrimitive.hpp` | backend-independent line/circle/text overlay primitive |
| `src/overlay/LineOverlay.hpp` | line overlay builder 선언 |
| `src/overlay/LineOverlay.cpp` | magenta contour, green tracking point, line label primitive 생성 |
| `src/overlay/IntersectionOverlay.hpp` | intersection/decision overlay builder 선언 |
| `src/overlay/IntersectionOverlay.cpp` | cyan center/type label, yellow present branch ray, branch score label, green decision/local coordinate label primitive 생성 |
| `src/overlay/MarkerOverlay.hpp` | marker overlay builder 선언 |
| `src/overlay/MarkerOverlay.cpp` | marker box, corners, center, direction arrow, label primitive 생성 |
| `src/telemetry/TelemetryStore.hpp` | frame_seq 기반 최근 vision frame store 선언. line/intersection/decision/grid-node metadata 포함 |
| `src/telemetry/TelemetryStore.cpp` | telemetry observe, video frame과 frame_seq/timestamp matching, decision/grid-node 보관 |
| `src/telemetry/VisionLogFormatter.hpp` | vision log formatter 선언 |
| `src/telemetry/VisionLogFormatter.cpp` | onboard processing timing, packet/video/system/camera/line/intersection/intersection-decision/grid-node/marker log 문자열 생성 |
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
| `tests/test_intersection_overlay.cpp` | intersection + decision overlay primitive 회귀 테스트 |
| `tests/test_telemetry_line_parse.cpp` | telemetry v1.7 line/intersection/decision/grid-node/debug parse 회귀 테스트 |
| `tests/test_video_reassembler.cpp` | MJPEG chunk reassembler/drop stats 회귀 테스트 |
| `tools/mock_onboard.cpp` | GCS telemetry receiver용 mock telemetry sender |
| `tools/log_replayer.cpp` | 아직 log replay 미구현 placeholder |

Tracked placeholder files with no runtime logic: `src/app/.gitkeep`, `src/common/.gitkeep`, `src/network/.gitkeep`, `src/protocol/.gitkeep`, `src/ui/.gitkeep`.

## 8. GCS 현재 동작

- `uav_gcs`: UDP telemetry 수신과 basic console 출력.
- `uav_gcs_video`: raw MJPEG video 수신/표시.
- `uav_gcs_vision_debug`: raw MJPEG video와 telemetry를 동시에 수신하고 GCS에서 marker/line/intersection overlay를 그리며 별도 vision log window를 연다.
- Intersection overlay에는 기존 cyan/yellow classifier 정보와 함께 green `DEC ...` label이 추가되어 decision state, accepted topology, local coordinate, turn zone/overshoot hint를 볼 수 있다.
- Vision log에는 `[intersection-decision]`과 `[grid-node]` line이 추가되어 detector type, decision type, branch evidence, local coordinate를 구분해 확인할 수 있다.
- Video 수신은 background thread에서 socket을 계속 drain하고, UI는 최신 complete JPEG만 표시한다.
- GCS camera overlay는 `frame N`만 표시한다. Video latency/age 표시는 제거했다.
- 음수 latency 또는 20-30ms처럼 체감과 맞지 않는 값은 Pi/Windows clock sync 부재와 best-effort buffering 때문에 오해를 만들 수 있다.
- 성능 판단은 `processing/read/decode/aruco/line/ix/json/tsend/vsubmit/vsend`, `capture_fps`, `processing_fps`, `completed/incomplete`, `video_sent/skipped/dropped/failures`, `cpu_temp_c`, `throttled_raw` 중심으로 본다.

## 9. Protocol v1.7 핵심

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
| `vision.intersection_detected`, `vision.intersection_score` | backward-compatible intersection summary |
| `vision.intersection.*` | valid/detected/type/raw_type/stable/held/center/raw_center/score/raw_score/branch_mask/branch_count/stable_frames/radius/selected_mask/branches |
| `vision.intersection.branches[]` | front/right/back/left branch별 present, score, endpoint, angle |
| `vision.intersection_decision.*` | decision state/action, accepted/best type, node event 여부, turn candidate, front availability, center_y_norm, approach_phase, overshoot risk |
| `vision.intersection_decision.branches[]` | front/right/back/left branch별 decision window evidence: present_frames, max_score, average_score |
| `vision.intersection_decision.node`, `vision.grid_node` | local grid node event. `local_coord`는 official origin이 아닌 탐색 시작 기준 local 좌표 |
| `grid.*` | legacy top-level grid summary. grid node event가 있으면 row/col에 local y/x를 채울 수 있음 |
| `debug.processing_latency_ms` | frame read 이후 onboard decode+detector 처리 시간 |
| `debug.read_frame_ms`, `debug.jpeg_decode_ms` | capture wait/read와 JPEG decode 시간 |
| `debug.aruco_latency_ms`, `debug.line_latency_ms`, `debug.intersection_latency_ms`, `debug.intersection_decision_latency_ms` | detector/decision별 처리 시간 |
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
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --fps 12
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
- Windows local `uav-gcs` tests 통과: `telemetry_line_parse`, `video_reassembler`, `line_overlay`, `intersection_overlay`, `grid_map_tracker`.
- Local OpenCV build에서 onboard vision tools target build 성공. `vision_debug_node`까지 build 확인.
- Onboard 기본 tests 통과: `telemetry_line_json`, `line_stabilizer`, `intersection_stabilizer`, `intersection_decision`.
- Onboard OpenCV tests 통과: `telemetry_line_json`, `line_stabilizer`, `intersection_stabilizer`, `intersection_decision`, `intersection_detector`.
- GCS parser/overlay tests는 `vision.intersection_decision`과 `vision.grid_node` parse, line/intersection overlay primitive, live grid map 누적 렌더링까지 포함한다.
- Onboard telemetry JSON test는 `vision.intersection_decision`, `vision.grid_node`, `debug.intersection_decision_latency_ms` serialization을 포함한다.
- `uav-onboard/docs/PROTOCOL.md`와 `uav-gcs/docs/PROTOCOL.md` hash 동일성 확인 완료.
- `vision_debug_node --help`에서 `--fps` option 노출 확인.
- `--fps 0`, `--no-video --fps 12` option error 처리 확인.
- Raspberry Pi 4 + IMX519에서 `vision_debug_node` build/run smoke 확인.
- Metadata-only 실행에서 telemetry packet은 정상 수신되고 video counter는 `video_sent=0`, `chunks_last=0`, `last_bytes=0`으로 표시됨을 확인.
- `--video`를 켠 경우 GCS camera window/overlay 확인 가능.
- 고품질 camera 설정 실험 후 성능 우선 기본값 `960x720`, `jpeg_quality=45`, debug video `5FPS`로 되돌림.
- GCS video latency/age 표시 제거 후 overlay에는 `frame N`만 표시.
- 2026-05-01 `grid_image_smoke`로 연습용 격자 이미지 2장 검증 완료. `*_final3` 산출물은 snake 이동 중 카메라 yaw 회전을 반영하지 않아 최종 기준에서 제외하고, 드론이 항상 정면 이동하도록 각 이동 구간의 camera heading에 맞춰 crop을 회전한 `*_rotated_centered` 산출물을 기준으로 삼는다.
- 중간 진입 이미지 `mid_entry_rotated_centered`: grid `7 x 5`, entry column `1`, section crop `35/35` event 저장 및 `35/35` expected type 일치, full-field snake 좌표 `35/35` 일치, entry-start snake 좌표 `34/34` 일치.
- 끝점 진입 이미지 `edge_entry_rotated_centered`: grid `7 x 6`, entry column `0`, section crop `42/42` event 저장 및 `42/42` expected type 일치, full-field snake 좌표 `42/42` 일치, entry-start snake 좌표 `42/42` 일치.
- 회전 snake smoke에서는 가장자리 노드가 이미지 경계에서 잘리며 화면 중심에서 밀리는 문제를 피하기 위해 고정 크기 중심 crop을 만들고 이미지 밖은 어두운 배경으로 padding한다. 이는 실제 카메라가 노드를 화면 중심에 둔 상태로 yaw만 회전하는 상황에 더 가깝다.
- Smoke 과정에서 약한 네 번째 branch가 edge `T`를 `+`로 올리는 false upgrade가 확인되어 `+` 채택 조건을 4방향 모두 `high_confidence_score` 이상으로 강화했다.
- 2026-05-01 추가 vision/grid smoke에서 사용자 제공 4개 라인 캡처 모두 `white_fill` mask로 검출 성공. 산출물은 `uav-gcs/logs/20260501-214828-vision-grid-smoke/line_screenshots/`에 저장되며, `logs/*`는 `.gitignore`로 추적 제외된다.
- 같은 smoke에서 연습용 격자 이미지는 `7 x 6`, bottom entry, `entry_col=0`, `entry_row=5`로 검출되었고 `snake_from_entry` node `42/42`, section topology `42/42`가 모두 통과했다. 단계별 crop과 ASCII grid log는 `uav-gcs/logs/20260501-214828-vision-grid-smoke/grid_snake/` 아래에 저장된다.
- 커밋 전 검증 기준 명령: `ctest --test-dir uav-gcs/build-tests --output-on-failure`, `ctest --test-dir uav-onboard/build-opencv-tests --output-on-failure`(Windows에서는 OpenCV DLL 경로로 `C:\msys64\ucrt64\bin`을 `PATH`에 추가).

문서 최신화 중 확인한 불일치:

- 과거 문서에는 protocol v1.6과 grid placeholder 설명이 남아 있었다. 현재는 protocol v1.7의 `vision.intersection_decision`, `vision.grid_node`, GCS decision overlay/log가 기준이다.

## 12. 2026-04-30 교차점 GCS 캡처 관찰

사용자가 GCS에서 캡처한 교차점 테스트 이미지는 어두운 배경 위의 밝은 선과 상대적으로 밝은 나무/흙색 배경 위의 밝은 선을 모두 포함한다. 관찰 결과는 다음과 같다.

이 절의 원인 분석은 `local_contrast`가 기본이던 시점의 관찰이다. 2026-05-01 작업 이후 현재 기본값은 `white_fill`이며, 넓은 흰색 라인을 edge strip이 아니라 채워진 blob으로 잡도록 `white_v_min`, `white_s_max`, `fill_close_kernel`, `fill_dilate_kernel`을 추가했다. `local_contrast`는 비교 실험용 fallback으로 유지한다.

### 12.1 어두운 배경에서 더 잘 되는 이유

- 당시 `line.mode = light_on_dark`, `mask_strategy = local_contrast` 조합은 밝은 선이 어두운 배경 위에 있을 때 foreground recall이 높았다.
- 어두운 천/그림자 배경에서는 선 내부와 배경의 local contrast가 커서 ray scoring의 front/right/back/left hit가 길게 이어진다.
- 밝은 나무/흙색 배경에서는 선과 배경의 명도 차가 작고, 조명 gradient/JPEG block/바닥 무늬가 선과 비슷한 local contrast fragment를 만든다. 이때 branch ray score는 일부 방향만 높거나 center density가 낮아져 `unknown`이 자주 나온다.
- 같은 흰색 선이라도 배경이 밝으면 threshold mask가 선 내부 전체를 채우기보다 선 가장자리나 반쪽 폭만 foreground로 남기 쉽다.

### 12.2 "라인 전체를 감싸지 않고 얇은 구간을 인식"하는 이유

현재 line contour와 intersection classifier는 같은 mask를 공유하지만 목적이 다르다.

- `LineDetector`는 line following용 tracking point 하나를 안정적으로 찾기 위해 selected contour 하나를 고른다.
- `LineDetector`의 contour는 `cv::findContours(..., RETR_EXTERNAL, ...)` 결과 중 scoring이 좋은 외곽 contour이다. 교차점 전체 topology를 완전하게 감싸는 것이 목적이 아니다.
- `local_contrast_blur = 31`은 배경 추정 창이 선 폭과 비슷하거나 작은 조건에서 선 내부를 균일하게 채우기보다 양쪽 edge contrast를 강조할 수 있다.
- 폭이 절반 정도인 얇은 선은 blur window 대비 선 폭이 작아 밝은 선 전체가 하나의 contrast blob으로 남기 쉽다. 반대로 폭이 넓은 휴지/테이프는 내부 중앙부가 local background와 비슷해져 edge/얇은 strip만 foreground로 남는 경향이 있다.
- 밝은 배경에서는 foreground fill이 더 약해져 contour가 한 branch의 얇은 테두리만 따라가고, 이 때문에 magenta line contour가 실제 선 폭보다 얇게 보인다.

### 12.3 `+`가 `L`/`T`로, `T`가 `L`로 흔들리는 이유

현재 `IntersectionDetector`는 skeleton graph가 아니라 center 후보 주변의 4방향 ray score를 계산한다. 분류 규칙은 단순하다.

- present branch 4개 이상: `+`
- present branch 3개: `T`
- present branch 2개: angle difference가 약 180도면 `straight`, 약 90도면 `L`
- 그 외: `unknown`

따라서 오분류는 대부분 "branch 하나가 threshold를 못 넘는 문제"로 설명된다.

- `+`에서 한 방향 branch가 약하면 branch count가 3이 되어 `T`가 된다.
- `+`에서 두 방향 branch가 약하면 branch count가 2가 되어 `L` 또는 `straight`가 된다.
- `T`에서 한 방향 branch가 약하면 branch count가 2가 되어 `L` 또는 `straight`가 된다.
- 캡처에서 branch ray score가 0.75~0.84 근처로 threshold 0.8 주변에 걸리는 경우가 많다. 이 구간은 frame-to-frame threshold crossing이 생기기 쉬워 `T`, `L`, `unknown`이 섞인다.
- center 후보가 실제 교차점 중심보다 한쪽 branch로 밀리면 반대쪽/수직 branch의 ray가 inner radius부터 충분히 foreground를 밟지 못한다. 이때 실제로는 존재하는 branch도 absent가 된다.
- Ray는 현재 front/right/back/left 축 방향에 고정되어 있다. 카메라 yaw나 기울기로 격자가 약간 회전하면 긴 branch라도 ray strip을 벗어나 score가 낮아질 수 있다.

### 12.4 `L`이 가장 잘 되는 이유

- `L`은 필요한 branch가 2개뿐이고, 보통 두 branch가 화면 중심 부근에서 길고 선명하게 보인다.
- 한 방향이 빠져도 `L`은 다른 복잡한 타입보다 classification ambiguity가 적다.
- 현재 algorithm은 branch count 기반이라, 4개/3개 branch를 모두 안정적으로 채워야 하는 `+`/`T`보다 2개 branch만 맞으면 되는 `L`이 구조적으로 유리하다.

### 12.5 `straight`가 `unknown`과 반반 뜨는 이유

- `straight`는 branch 2개가 present이고 angle difference가 충분히 180도에 가까워야 한다.
- 한쪽 끝이 frame 밖으로 잘리거나 중심 후보가 선 한쪽으로 밀리면 branch count가 1이 되어 `unknown`이 된다.
- 밝은 배경에서는 straight 선 내부 fill이 약해 ray continuity가 떨어지고, 한 방향 score만 threshold를 넘는 상황이 생긴다.
- 화면에서 line following contour가 한쪽 edge만 잡히면 intersection center와 branch ray가 실제 선 중앙이 아니라 edge 위에 놓여 반대 방향 ray가 불안정해진다.

### 12.6 지금 당장 의미 있는 개선 후보

당시 기준의 개선 후보는 다음과 같았다. 이 중 넓은 흰색 선 fill 보강은 2026-05-01 `white_fill` 기본값으로 반영되었다.

1. 밝은 배경에서 `line.process_width = 320`, `local_contrast_blur = 21~25`, `morph_close_kernel = 5~7` 조합을 비교한다. 넓은 선 내부가 edge만 남는지, branch score가 threshold 주변에서 덜 흔들리는지 본다.
2. `intersection_threshold = 0.8`은 현재 캡처 기준으로 다소 빡빡하다. `0.70~0.75` 후보를 테스트해 `+`/`T` 누락이 줄어드는지 확인하되, 바닥 무늬 false positive가 늘어나는지도 같이 본다.
3. 카메라 해상도를 640x480으로 낮출 때는 `line.process_width`도 320으로 같이 낮춰야 line/intersection CPU 비용 감소가 의미 있다.
4. Algorithm 개선을 한다면 1순위는 ray 각도를 고정 축이 아니라 dominant line angle 또는 detected branch endpoint 방향에 맞춰 약간 회전 허용하는 것이다.
5. 2순위는 branch present 판정을 단일 threshold가 아니라 hysteresis/temporal branch EMA로 바꾸는 것이다. 현재 type stabilizer는 최종 type을 안정화하지만 branch 하나가 threshold 주변에서 깜빡이는 근본 원인은 남아 있다.
6. 3순위는 넓은 테이프/휴지 선을 위해 local contrast mask 이후 fill/close를 선 폭 기준으로 보강하거나, distance transform/skeleton 보조 feature를 붙이는 것이다.

## 13. 2026-05-01 Decision/Grid 및 Vision/Grid smoke 구현 결과

`PLAN.md`의 다음 단계 작업은 detector를 크게 바꾸지 않고, `IntersectionDetector`/`IntersectionStabilizer` 위에 얇은 mission/debug decision layer를 얹는 방향으로 진행했다.

구현된 것:

- `LineMaskBuilder`의 현재 기본 전략은 `white_fill`이다. HSV low-saturation/high-value 조건으로 흰색 라인을 검출하고 close/dilate로 넓은 선 내부를 채워, 사용자 캡처처럼 두꺼운 흰색 라인이 얇은 edge만 남는 문제를 줄였다.
- `LineDetector`는 전체 contour centroid 대신 anchor/lookahead band의 X projection을 사용한다. L 교차점의 가로 branch가 라인 추적 중심을 끌어당기는 영향을 줄이기 위한 변경이다.
- `line_detector_tuner`는 `--output`을 받아 `mask_0.png`, `line_overlay.png`, `summary.txt`를 저장하고, `--white-v-min`, `--white-s-max`, `--fill-close`, `--fill-dilate`로 `white_fill` 튜닝값을 재현할 수 있다.
- `IntersectionDecisionEngine`은 최근 frame queue에서 branch evidence를 누적한다.
- `+ > T > L > straight > unknown` 우선순위는 적용하되, 단일 frame의 상위 타입 튐을 바로 채택하지 않는다.
- `+`는 4방향 branch가 모두 `high_confidence_score` 이상일 때만 채택한다. 이는 연습용 edge 이미지에서 확인된 `T -> +` false upgrade를 막기 위한 보수적 guard다.
- `L`/`T`/`+`는 모두 grid node event로 기록된다.
- `GridCoordinateTracker`는 첫 안정 node를 local `(0,0)`으로 두고, heading이 알려진 경우 다음 node마다 local coordinate를 한 칸씩 증가시킨다.
- camera-relative branch mask는 heading 기준 grid-relative branch mask로 변환된다.
- `front_available`, `required_turn`, `turn_candidate`, `center_y_norm`, `approach_phase`, `overshoot_risk`, `too_late_to_turn`이 telemetry/log에 나온다.
- 실제 Pixhawk stop/turn 명령은 아직 없다. `TurnReady`/`PrepareTurn`/`HoldPosition`은 telemetry/debug action hint다.

GCS 반영:

- `vision.intersection_decision`과 `vision.grid_node`를 parser/store/log/overlay가 처리한다.
- Vision log에는 `[intersection-decision]`, `[grid-node]`, `[grid-map]`이 표시된다.
- `GridMapTracker`는 `vision.grid_node` telemetry를 받아 방문 좌표, 현재 heading marker, 인접 edge를 ASCII grid로 누적 렌더링한다. 처음에는 entry line과 `(0,0)` 주변만 보이고, 새 node가 저장될 때마다 동적으로 확장된다.
- Line overlay는 magenta contour와 camera-center Y에 고정된 red line-center point, 그리고 camera center에서 line-center point까지의 green horizontal offset line만 기본 표시한다.
- Intersection overlay는 화면 상단/현재 접근 영역의 교차점 branch 방향만 compact하게 보여준다. branch score text와 긴 decision label은 live overlay에서 제거하고 log에 남긴다.

Smoke 산출물:

- `development-log/grid-smoke-20260501/mid_entry_rotated_centered/`
- `development-log/grid-smoke-20260501/edge_entry_rotated_centered/`

이 산출물에는 `sections.csv`, `snake_full_field.csv`, `snake_from_entry.csv`, `sections/` crop PNG, `snake_full_field_zoom/` 확대 이동 crop PNG, `snake_from_entry_zoom/` 확대 이동 crop PNG가 포함된다. Snake crop은 현재 이동 구간의 camera heading에 맞춰 회전되어, 드론이 yaw를 바꾼 뒤 항상 정면으로 이동하는 운용 가정을 반영한다. 두 이미지 모두 구간별 교차점 판단과 회전 snake 방식 좌표 저장이 최종 smoke 기준으로 통과했다.

추가로 `uav-gcs/logs/20260501-214828-vision-grid-smoke/`에는 이번 작업의 실제 검증 산출물이 저장되었다. 라인 캡처 4장은 모두 검출되었고, 연습용 격자 이미지는 bottom entry 기준 `7 x 6` 전체 `42/42` node와 `42/42` topology 검증을 통과했다. 최종 grid log는 `grid_snake/snake_from_entry_grid_log/step_041_r0_c6_heading_north.txt`이다. 이 폴더는 재현/디버깅용 로컬 산출물이며 git에는 추적하지 않는다.

중요한 해석:

- 첫 node local `(0,0)`은 official origin이 아니다.
- 첫 이미지처럼 entry가 grid 중간 column으로 들어오면 entry-start snake는 official 전체 좌표와 다를 수 있다. 현재는 local 탐색 좌표 저장만 검증한다.
- 두 번째 이미지처럼 entry가 grid 끝점으로 들어오는 경우에는 local `(0,0)`이 실제 외곽 시작점 후보와 더 잘 맞는다.
- `*_final3`처럼 경기장 이미지를 고정한 채 crop만 이동하는 방식은 실제 snake flight 검증으로 부정확하다. 드론은 회전 후 카메라 정면으로 이동하므로 검증 이미지도 camera heading 기준으로 회전해야 한다.
- official coordinate 변환은 경기장 규칙, 시작 마커, 탐색된 bounding box를 이용한 후속 단계로 남아 있다.

## 14. 현재 리스크와 다음 작업

현재 리스크:

- Mission/MAVLink/control loop가 아직 없으므로 실제 비행 판단/제어는 구현 전이다.
- 교차점 decision layer는 구현되었지만 실제 드론 이동 속도에서는 0.5초 window 동안 이동 거리가 생긴다. 속도/FPS/고도 기준으로 window frame 수와 turn zone을 보정해야 한다.
- 밝은 배경과 넓은 흰색 선 검출은 `white_fill`로 크게 개선되었지만, 실제 조명/노출이 달라지면 `white_v_min`, `white_s_max`, fill morphology를 다시 보정해야 한다.
- threshold 주변 branch score와 회전된 격자에서 detector raw `unknown`/type downgrade는 여전히 남을 수 있다.
- Local grid coordinate는 저장되지만 official competition origin/axis 변환은 아직 구현되지 않았다.
- Full snake search policy는 아직 구현되지 않았다. 현재 snake 검증은 camera heading 회전을 반영한 smoke tool의 가정 경로를 tracker에 먹인 것이다.
- IMX519 focus/exposure는 실제 고도, 속도, 조명에서 다시 calibration해야 한다.
- Debug video 기본값은 5FPS라 GCS 관제 화면은 답답할 수 있다. 디버깅 때는 `--fps 12`로 올릴 수 있지만, 기본값은 onboard 제어 여유와 telemetry 안정성을 위해 유지한다.
- Wi-Fi UDP video는 packet loss가 정상적으로 발생할 수 있으므로 frame drop을 오류로 보지 말고 counters로 판단해야 한다.
- GCS video latency는 정확히 측정하지 않는다. 필요하면 clock sync가 가능한 별도 측정 설계를 추가해야 한다.

추천 다음 단계:

1. 실제 경기장과 유사한 밝은/어두운 배경에서 `intersection_threshold`, `process_width`, `local_contrast_blur`, `morph_close_kernel` 후보를 같은 frame set으로 비교한다.
2. `+`/`T` downgrade가 branch 누락 때문인지 center 후보 오차 때문인지 branch score log와 overlay를 함께 기록한다.
3. `grid_image_smoke`를 실제 캡처 frame set/replay 입력까지 확장해, 단일 이미지 crop이 아니라 연속 이동 frame에서 decision/grid event를 검증한다.
4. Local grid map을 탐색된 bounding box와 시작 entry topology로 official coordinate 후보에 변환한다.
5. Snake search policy를 mission layer에 추가하고 `turn_expected`를 decision engine에 실제로 공급한다.
6. 같은 조건에서 `--video --fps 12`를 켠 관제용 성능과 metadata-only 성능을 비교하되, 제어 loop 추가 전 기본값은 metadata-only/5FPS video cap으로 유지한다.
7. ArUco 50cm x 50cm marker를 실제 2m 거리에서 `aruco-only`, `aruco+line+intersection+decision` 각각 확인한다.
8. IMX519 `lens_position`, `autofocus_mode`, `exposure`, `shutter_us`, `gain`을 mission-like 조건에서 고정값 후보로 정한다.
9. 실제 고도/속도에서 `turn_zone_y_min/max`, `late_zone_y`, `record_node_once_frames`를 보정한다.
10. 그 다음 Pixhawk/MAVLink control loop를 onboard critical path로 구현한다.
