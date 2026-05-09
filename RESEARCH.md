# Astroquad Research Brief

최종 업데이트: 2026-05-09

이 문서는 다음 에이전트가 프로젝트 전반, 현재 진행 상태, 먼저 읽어야 할 파일을 빠르게 판단하도록 만든 요약 문서다. 상세 구현 설명은 각 저장소의 `README.md`, `docs/PROTOCOL.md`, 테스트, 소스 파일을 직접 확인한다.

## 1. 프로젝트 개요

Astroquad는 실내 격자 경기장에서 UAV가 ArUco marker와 line/intersection 정보를 기반으로 탐색 임무를 수행하도록 만드는 C++ 프로젝트다.

- 현재 hardware target은 Raspberry Pi 4 + IMX519-78 onboard와 Windows 노트북 GCS다.
- Onboard는 카메라 캡처, ArUco/line/intersection 검출, grid node 판단, telemetry 생성, 추후 MAVLink 제어를 담당한다.
- GCS는 telemetry/debug video 수신, raw camera 위 overlay drawing, vision log 표시를 담당한다.
- Overlay는 GCS에서만 그린다. Onboard는 mission-critical vision/control path를 위해 raw data와 판단값만 만든다.
- Debug video는 best-effort 관제/튜닝 채널이다. drop이나 지연이 있어도 vision/control loop를 막으면 안 된다.
- Protocol 문서는 v1.7 기준이다. JSON top-level `protocol_version`은 호환성을 위해 integer `1`을 유지한다.

## 2. 저장소 구성

루트 `astroquad/` 자체는 git 저장소가 아니며, 주요 하위 폴더들이 독립적으로 관리된다.

| 경로 | 역할 |
|---|---|
| `development-log/` | 계획, 현재 브리핑, 트러블슈팅, smoke 산출물 기록 |
| `uav-onboard/` | Raspberry Pi runtime, camera, onboard vision, telemetry/debug video 송신 |
| `uav-gcs/` | Windows GCS, telemetry/debug video 수신, overlay, vision log |
| `grid_images/` | 실험용 격자/마커 이미지 |
| `week1_assignments/` | 과제/실험 코드. UAV runtime 작업이 아니면 보통 읽지 않는다 |

`development-log/PLAN.md`는 현재 작업용 메모다. 지금 메모의 핵심은 "ArUco marker가 교차점 중앙에 있을 때 L/T/+ 교차점 판단이 유지되는지 검토"다.

## 3. 현재 구현 상태

구현되어 있는 것:

- `rpicam-vid` 기반 IMX519 MJPEG frame capture
- UDP JSON telemetry 송신/수신
- opt-in UDP MJPEG debug video 송신/수신
- GCS discovery beacon과 video unicast 전환
- Onboard ArUco detection
- Onboard line tracing, line stabilizer
- Onboard intersection classifier와 temporal stabilizer
- `IntersectionDecisionEngine`: 최근 branch evidence window 기반 `L`/`T`/`+` node 판단, turn candidate, approach phase, overshoot risk telemetry
- `GridCoordinateTracker`: 첫 안정 node를 local `(0,0)`으로 두는 상대 좌표 기록, camera-relative branch mask를 grid-relative mask로 변환
- GCS marker/line/intersection/decision overlay
- GCS vision log의 `[intersection-decision]`, `[grid-node]`, `[grid-map]` 표시

아직 구현 전이거나 placeholder인 것:

- Pixhawk/MAVLink 실제 control loop
- Mission command TCP channel
- full snake mission policy
- official competition origin/axis coordinate 변환
- marker 발견 위치 기반 최종 임무 판단
- safety, battery, heartbeat 기반 fail-safe

## 4. 런타임 흐름

```text
IMX519 camera
  -> rpicam-vid MJPEG stdout
  -> RpicamMjpegSource
  -> JPEG decode once for onboard detectors
  -> ArucoDetector / LineMaskBuilder
  -> LineDetector + IntersectionDetector
  -> LineStabilizer + IntersectionStabilizer
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> TelemetryMessage JSON
  -> UDP telemetry
  -> optional raw MJPEG debug video
  -> GCS TelemetryStore frame_seq matching
  -> GCS overlay + vision log
```

`vision_debug_node`의 기본 실행은 metadata-only다. `config/vision.toml`에서 `debug_video.enabled = false`이므로 `--video`를 붙이지 않으면 GCS camera window는 `waiting for video stream...` 상태일 수 있지만, telemetry/log는 정상 동작할 수 있다.

## 5. 먼저 읽을 파일

작업 종류별 시작점:

| 작업 | 먼저 볼 파일 |
|---|---|
| 현재 TODO 확인 | `development-log/PLAN.md` |
| 과거 문제/해결 확인 | `development-log/TROUBLESHOOTING.md` |
| Onboard 실행 흐름 | `uav-onboard/tools/vision_debug_node.cpp`, `uav-onboard/src/app/VisionDebugPipeline.cpp` |
| Onboard 설정 | `uav-onboard/config/vision.toml`, `uav-onboard/config/network.toml` |
| Camera capture | `uav-onboard/src/camera/RpicamMjpegSource.*` |
| ArUco | `uav-onboard/src/vision/ArucoDetector.*`, `uav-onboard/tools/aruco_detector_tester.cpp` |
| Line detection | `uav-onboard/src/vision/LineMaskBuilder.*`, `LineDetector.*`, `LineStabilizer.*` |
| Intersection detection | `uav-onboard/src/vision/IntersectionDetector.*`, `IntersectionStabilizer.*` |
| Decision/grid node | `uav-onboard/src/mission/IntersectionDecision.*`, `GridCoordinateTracker.*` |
| Telemetry schema | `uav-onboard/src/protocol/TelemetryMessage.*`, `uav-gcs/src/protocol/TelemetryMessage.*` |
| Protocol 문서 | `uav-onboard/docs/PROTOCOL.md`, `uav-gcs/docs/PROTOCOL.md` |
| GCS main app | `uav-gcs/src/app/VisionDebugApp.*` |
| GCS overlay | `uav-gcs/src/overlay/MarkerOverlay.*`, `LineOverlay.*`, `IntersectionOverlay.*` |
| GCS log/grid map | `uav-gcs/src/telemetry/VisionLogFormatter.*`, `GridMapTracker.*`, `uav-gcs/src/ui/VisionLogWindow.*` |
| UDP video transport | `uav-onboard/src/video/*`, `uav-gcs/src/video/*` |
| Unit tests | `uav-onboard/tests/`, `uav-gcs/tests/` |

현재 `PLAN.md`의 ArUco-at-intersection 검토를 진행한다면 특히 다음을 같이 읽는다.

- `uav-onboard/src/vision/ArucoDetector.*`
- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/IntersectionDetector.*`
- `uav-onboard/src/mission/IntersectionDecision.*`
- `uav-onboard/tests/test_intersection_detector.cpp`
- `uav-onboard/tests/test_intersection_decision.cpp`
- `grid_images/black_grid_with_aruco_marker.png`
- `grid_images/white_grid_with_aruco_marker.png`

## 6. 중요한 설정 기본값

현재 기준은 `uav-onboard/config/vision.toml`이다.

| 항목 | 기본값/의미 |
|---|---|
| Camera | `960x720 @ 12FPS`, MJPEG quality `45` |
| Focus | `autofocus_mode = "manual"`, `lens_position = 0.67` |
| Exposure | `exposure = "sport"` |
| Debug video | 기본 off, 필요할 때 `vision_debug_node --video` |
| Debug video FPS | 기본 `5FPS`, 관제 튜닝 시 `--fps 12` 가능 |
| Line mode | 기본 `light_on_dark` |
| Line mask | 기본 `white_fill`; `dark_on_light`에서는 `dark_fill` 경로 사용 |
| Line work size | `line.process_width = 480` |
| Intersection threshold | `line.intersection_threshold = 0.8` |
| Decision window | `cruise_window_frames = 6`, 12FPS 기준 약 0.5초 |
| `+` false-upgrade guard | 4방향 branch가 모두 `high_confidence_score = 0.85` 이상이어야 `+` 채택 |
| Node lockout | `record_node_once_frames = 18`, `node_advance_min_frames = 4` |

`--fps`는 camera capture FPS가 아니라 debug video send FPS다. Camera capture FPS를 바꾸려면 config 또는 별도 camera option을 확인한다.

## 7. 권장 실행 조합

GCS vision debug:

```powershell
cd uav-gcs
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Onboard metadata-only line tracing:

```bash
cd ~/astroquad/uav-onboard
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Onboard with GCS camera/overlay:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --fps 12
```

ArUco-only tuning:

```bash
./build/vision_debug_node --config config --aruco-only
./build/vision_debug_node --config config --aruco-only --video
./build/vision_debug_node --config config --aruco-only --camera-quality 90 --lens-position 1.0
```

Network discovery/broadcast가 막히면:

```bash
./build/vision_debug_node --config config --gcs-ip <laptop-ip>
./build/vision_debug_node --config config --video --gcs-ip <laptop-ip>
```

## 8. 검증 이력

최근 기준으로 의미 있는 검증:

- Raspberry Pi 4 + IMX519에서 `vision_debug_node` build/run smoke 확인.
- Metadata-only 실행에서 telemetry packet은 정상 수신되고 video counter는 `video_sent=0`, `chunks_last=0`, `last_bytes=0`으로 표시됨을 확인.
- `--video`를 켠 경우 GCS camera window와 overlay 확인 가능.
- 2026-05-01 `grid_image_smoke`로 연습용 격자 이미지 2장 검증 완료.
- `mid_entry_rotated_centered`: grid `7 x 5`, section/event/full-field snake `35/35`, entry-start snake `34/34`.
- `edge_entry_rotated_centered`: grid `7 x 6`, section/event/full-field/entry-start snake `42/42`.
- 회전 snake smoke는 드론이 yaw 후 정면 이동한다는 가정에 맞춰 crop을 camera heading 기준으로 회전한다. `*_final3`처럼 경기장 이미지를 고정한 채 crop만 이동하는 방식은 현재 기준에서 제외한다.
- 2026-05-01 추가 vision/grid smoke에서 사용자 제공 라인 캡처 4장 모두 `white_fill` mask로 검출 성공.
- 2026-05-03 GCS vision log window는 상단 `[grid-map]` pane과 하단 telemetry pane으로 분리됨.
- 2026-05-03 `dark_on_light`의 굵은 검정 라인 edge-only 검출 문제를 `dark_fill` 경로로 개선.

Smoke 산출물 위치:

- `development-log/grid-smoke-20260501/mid_entry_rotated_centered/`
- `development-log/grid-smoke-20260501/edge_entry_rotated_centered/`
- `uav-gcs/logs/20260501-214828-vision-grid-smoke/`는 로컬 검증 산출물이며 git 추적 대상이 아니다.

## 9. 테스트 명령

GCS:

```powershell
ctest --test-dir uav-gcs/build-tests --output-on-failure
```

Onboard:

```powershell
ctest --test-dir uav-onboard/build-opencv-tests --output-on-failure
```

Windows에서 OpenCV DLL이 필요하면 테스트 전에 `PATH`에 `C:\msys64\ucrt64\bin`을 추가한다.

## 10. 남은 리스크와 다음 작업

중요 리스크:

- 현재 local `(0,0)`은 official competition origin이 아니다.
- Full snake policy와 official coordinate 변환이 아직 없다.
- 실제 이동 속도, 고도, FPS에 따라 decision window와 turn zone을 보정해야 한다.
- IMX519 focus/exposure는 실제 조명, 고도, 속도에서 다시 calibration해야 한다.
- `white_fill`/`dark_fill`은 개선되었지만 실제 배경과 조명에 따라 HSV threshold와 morphology 재튜닝이 필요할 수 있다.
- Branch ray score가 threshold 주변에 있거나 격자가 회전되어 보이면 `+`/`T`/`L` downgrade가 여전히 생길 수 있다.
- Debug video packet loss는 정상적으로 발생할 수 있다. Mission 판단은 telemetry와 onboard state 기준으로 봐야 한다.
- GCS video latency는 clock sync 없이 정확히 측정하지 않는다.

추천 다음 작업:

1. ArUco marker가 교차점 중앙에 있는 이미지에서 `L`/`T`/`+` 분류가 유지되는지 테스트한다.
2. Marker가 line/intersection mask를 가리는 경우, ArUco 영역 masking, marker-aware branch scoring, decision-layer 보정 중 어떤 방식이 필요한지 비교한다.
3. 실제 경기장과 유사한 frame set으로 `intersection_threshold`, `process_width`, white/dark fill threshold를 비교한다.
4. `grid_image_smoke`를 단일 이미지 crop이 아니라 실제 연속 frame/replay 입력까지 확장한다.
5. Local grid map을 탐색된 bounding box와 entry topology 기준으로 official coordinate 후보에 변환한다.
6. Snake search policy를 mission layer에 추가하고 `turn_expected`를 decision engine에 공급한다.
7. 그 다음 Pixhawk/MAVLink control loop를 onboard critical path로 구현한다.

