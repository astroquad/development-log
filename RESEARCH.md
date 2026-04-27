# Astroquad 프로젝트 리서치

최종 업데이트: 2026-04-27

범위: `development-log`, `uav-gcs`, `uav-onboard`의 최신 디렉터리 구조, 주요 파일 역할, 현재 구현 상태, 검증 결과, 남은 리스크, 추천 다음 단계.

작성 기준 코드:

| 저장소 | 기준 커밋 |
|---|---|
| `uav-gcs` | `dca728a Document connected line contour overlay` |
| `uav-onboard` | `ae8022f Restore aggressive line contour detection` |

## 1. 현재 프로젝트 목표

Astroquad는 Raspberry Pi Zero 2 W와 GCS 노트북을 중심으로, 라인 기반 격자 경기장에서 UAV가 탐색 임무를 수행하도록 만드는 C++ 프로젝트다.

최종 임무 흐름:

1. UAV 이륙
2. 라인 트레이싱으로 경기장 격자 진입
3. 교차점 기반 grid 이동
4. ArUco marker 인식
5. marker id와 발견 grid 좌표 저장
6. 대회 규칙에 맞는 marker 재방문 또는 결과 활용
7. 출발 지점 복귀 및 착륙

현재 기체/Pixhawk 제어는 아직 붙이지 않았다. 지금까지의 개발 초점은 camera, telemetry, video streaming, GCS overlay, ArUco detection, line tracing MVP를 먼저 안정화하는 것이다.

## 2. 핵심 아키텍처 판단

| 책임 | `uav-onboard` | `uav-gcs` |
|---|---|---|
| Pi camera capture | 수행 | 수행하지 않음 |
| ArUco detection | 수행 | 수행하지 않음 |
| line detection | 수행 | 수행하지 않음 |
| intersection 판단 | 다음 단계 | 수행하지 않음 |
| grid/marker state | 다음 단계 | 표시 및 로그 예정 |
| raw MJPEG debug stream | best-effort 송신 | 수신 및 표시 |
| overlay drawing | 수행하지 않음 | 수행 |
| vision log 표시 | telemetry 송신 | 별도 vision log window 표시 |
| mission command UI | 수신 예정 | 송신 예정 |
| Pixhawk/MAVLink 제어 | 예정 | 수행하지 않음 |

가장 중요한 원칙은 온보드가 mission-critical 계산에 집중하고, GCS video/overlay/log는 관제용 debug channel로만 유지하는 것이다. 따라서 온보드는 원본 JPEG와 vision metadata를 보내고, marker box나 line contour drawing은 GCS에서 한다.

## 3. 루트 디렉터리 구조

```text
astroquad/
├── development-log/
│   ├── PLAN.md
│   ├── RESEARCH.md
│   └── TROUBLESHOOTING.md
├── uav-gcs/
└── uav-onboard/
```

루트 자체는 git 저장소가 아니다. 세 폴더가 각각 독립적인 git 저장소다.

| 저장소 | 역할 |
|---|---|
| `development-log` | 리서치, 구현 계획, 트러블슈팅, 보고서용 개발 이력 |
| `uav-gcs` | Windows/Linux GCS, telemetry/video 수신, overlay, log window |
| `uav-onboard` | Raspberry Pi onboard runtime, camera, vision, telemetry, video sender |

## 4. `development-log` 문서 역할

| 파일 | 역할 |
|---|---|
| `PLAN.md` | 라인트레이싱/ArUco/GCS overlay milestone 계획과 설계 판단 기록 |
| `RESEARCH.md` | 현재 프로젝트 구조, 파일 역할, 구현 상태, 검증 결과, 다음 단계 정리 |
| `TROUBLESHOOTING.md` | 실제 문제 발생, 원인 분석, 해결 과정, 현재 판단을 시간순으로 기록 |

현재 `TROUBLESHOOTING.md`에는 telemetry, video, rpicam, OpenCV, Windows firewall, GCS overlay 설계, line false positive, ROI/lookahead, branch filtering rollback까지 정리되어 있다.

## 5. 현재 실행 조합

| 목적 | GCS | Raspberry Pi |
|---|---|---|
| telemetry만 확인 | `.\build\uav_gcs.exe --config config` | `./build/uav_onboard --config config --count 10` |
| raw camera video만 확인 | `.\build\uav_gcs_video.exe --config config` | `./build/video_streamer --source rpicam --config config` |
| ArUco + line vision debug | `.\build\uav_gcs_vision_debug.exe --config config` | `./build/vision_debug_node --config config` |
| line만 확인 | `.\build\uav_gcs_vision_debug.exe --config config` | `./build/vision_debug_node --config config --line-only --line-mode light_on_dark` |
| ArUco만 확인 | `.\build\uav_gcs_vision_debug.exe --config config` | `./build/vision_debug_node --config config --aruco-only` |

흰색 라인을 테스트할 때는 `--line-mode light_on_dark`가 가장 직접적이다. 라인 색이 확정되지 않았으면 `--line-mode auto`로 시작한 뒤 현장에서 고정한다.

## 6. Protocol 상태

현재 protocol 문서 버전은 `v1.2`다. `uav-gcs/docs/PROTOCOL.md`와 `uav-onboard/docs/PROTOCOL.md`는 동일해야 하며, 현재 동일한 내용으로 유지되어 있다.

채널:

| Channel | 방향 | Transport | 기본 포트 |
|---|---|---|---:|
| Telemetry | onboard -> GCS | UDP JSON | 14550 |
| Command | GCS -> onboard | TCP JSON | 14551 |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 |

주요 telemetry:

- `camera.frame_seq`: MJPEG frame id와 매칭되는 frame sequence
- `vision.markers[]`: ArUco id, corners, center, orientation
- `vision.line`: line detected, tracking point, centroid, offset, angle, confidence, contour
- `debug.processing_latency_ms`
- `debug.aruco_latency_ms`
- `debug.line_latency_ms`

GCS는 `frame_seq`와 timestamp fallback으로 video frame과 telemetry를 맞춘다.

## 7. `uav-gcs` 최신 구조

```text
uav-gcs/
├── CMakeLists.txt
├── PROJECT_SPEC.md
├── README.md
├── config/
│   ├── network.toml
│   └── ui.toml
├── docs/
│   └── PROTOCOL.md
├── logs/
│   └── .gitkeep
├── src/
│   ├── main.cpp
│   ├── video_main.cpp
│   ├── vision_debug_main.cpp
│   ├── app/
│   │   ├── VideoViewerApp.cpp/.hpp
│   │   └── VisionDebugApp.cpp/.hpp
│   ├── common/
│   │   └── NetworkConfig.cpp/.hpp
│   ├── network/
│   │   └── UdpTelemetryReceiver.cpp/.hpp
│   ├── overlay/
│   │   ├── LineOverlay.cpp/.hpp
│   │   ├── MarkerOverlay.cpp/.hpp
│   │   └── OverlayPrimitive.hpp
│   ├── protocol/
│   │   └── TelemetryMessage.cpp/.hpp
│   ├── telemetry/
│   │   ├── MarkerLogFormatter.cpp/.hpp
│   │   ├── TelemetryStore.cpp/.hpp
│   │   └── VisionLogFormatter.cpp/.hpp
│   ├── ui/
│   │   ├── VideoWindow.cpp
│   │   ├── VideoWindow.hpp
│   │   ├── VideoWindowWin32.cpp
│   │   └── VisionLogWindow.cpp/.hpp
│   └── video/
│       ├── GcsDiscoveryBeacon.cpp/.hpp
│       ├── JpegFrameReassembler.cpp/.hpp
│       ├── UdpMjpegReceiver.cpp/.hpp
│       └── VideoPacket.cpp/.hpp
├── tests/
│   ├── CMakeLists.txt
│   ├── test_line_overlay.cpp
│   └── test_telemetry_line_parse.cpp
└── tools/
    ├── log_replayer.cpp
    └── mock_onboard.cpp
```

### 7.1 `uav-gcs` build target

| Target | 상태 | 설명 |
|---|---|---|
| `gcs_dependencies` | 구현됨 | `nlohmann/json`, `toml++` include target |
| `gcs_core` | 구현됨 | network config, telemetry receiver, telemetry parser |
| `gcs_video` | 구현됨 | video receiver, discovery beacon, overlay, telemetry store, UI |
| `uav_gcs` | 구현됨 | telemetry console receiver |
| `uav_gcs_video` | 구현됨 | raw MJPEG video viewer |
| `uav_gcs_vision_debug` | 구현됨 | video + marker/line overlay + vision log window |
| `mock_onboard` | 구현됨 | local telemetry mock sender |
| `log_replayer` | scaffold | 향후 replay tool |
| `test_telemetry_line_parse` | 구현됨 | line telemetry parser test |
| `test_line_overlay` | 구현됨 | line overlay primitive generation test |

### 7.2 `uav-gcs` 주요 파일 역할

| 파일 | 역할 |
|---|---|
| `CMakeLists.txt` | GCS target, optional OpenCV backend, Win32 fallback, tests 정의 |
| `README.md` | build/run/test/firewall/vision debug 사용법 |
| `config/network.toml` | telemetry/video/command port와 timeout |
| `config/ui.toml` | video window title, timeout 등 UI 설정 |
| `docs/PROTOCOL.md` | onboard와 공유하는 protocol v1.2 문서 |
| `src/main.cpp` | `uav_gcs` telemetry-only entrypoint |
| `src/video_main.cpp` | `uav_gcs_video` entrypoint |
| `src/vision_debug_main.cpp` | `uav_gcs_vision_debug` entrypoint, CLI option parsing |
| `src/app/VideoViewerApp.cpp` | raw video receive loop, GCS discovery beacon, video window 표시 |
| `src/app/VisionDebugApp.cpp` | video receive, telemetry receive thread, overlay 생성, vision log window 갱신 |
| `src/common/NetworkConfig.cpp` | `network.toml` parser |
| `src/network/UdpTelemetryReceiver.cpp` | cross-platform UDP telemetry receive |
| `src/protocol/TelemetryMessage.cpp` | telemetry JSON parser, legacy line fields와 detailed line object 동시 처리 |
| `src/telemetry/TelemetryStore.cpp` | 최근 marker/line telemetry를 frame sequence 기준으로 저장하고 video frame에 매칭 |
| `src/telemetry/VisionLogFormatter.cpp` | marker/line/latency/packet stats를 vision log text로 formatting |
| `src/telemetry/MarkerLogFormatter.cpp` | marker 중심 log formatter. 현재는 vision log formatter가 더 최신 흐름 |
| `src/overlay/OverlayPrimitive.hpp` | line, circle, text primitive 모델 |
| `src/overlay/MarkerOverlay.cpp` | marker corners/center/orientation을 overlay primitive로 변환 |
| `src/overlay/LineOverlay.cpp` | line contour를 magenta polyline, tracking point를 green circle/text로 변환 |
| `src/ui/VideoWindow.cpp` | OpenCV video window backend |
| `src/ui/VideoWindowWin32.cpp` | Windows WIC/GDI video window backend. OpenCV 없이 JPEG decode/drawing |
| `src/ui/VisionLogWindow.cpp` | Windows에서는 별도 log window, 비지원 환경에서는 console fallback |
| `src/video/GcsDiscoveryBeacon.cpp` | UDP 5601로 `AQGCS1 video_port=...` broadcast |
| `src/video/UdpMjpegReceiver.cpp` | AQV1 UDP chunk 수신 |
| `src/video/JpegFrameReassembler.cpp` | chunk를 complete JPEG frame으로 재조립, incomplete frame drop |
| `src/video/VideoPacket.cpp` | AQV1 header serialize/parse |
| `tools/mock_onboard.cpp` | GCS 테스트용 mock telemetry 송신 |
| `tools/log_replayer.cpp` | 향후 log replay placeholder |
| `tests/test_telemetry_line_parse.cpp` | `vision.line` JSON parsing 검증 |
| `tests/test_line_overlay.cpp` | line overlay 색/primitive 생성 검증 |

### 7.3 GCS 현재 기능

- UDP telemetry 수신
- UDP MJPEG chunk 수신 및 JPEG 재조립
- GCS discovery beacon 송신
- raw video viewer
- vision debug viewer
- ArUco marker overlay
- line contour/tracking point overlay
- 별도 vision log window
- OpenCV 없는 Windows fallback UI
- line telemetry parse test와 line overlay test

### 7.4 GCS 남은 한계

- command/control UI 없음
- mission dashboard 없음
- file logging 없음
- marker map/grid state 표시 없음
- video/telemetry replay tool 미완성
- non-Windows log window는 console fallback 중심

## 8. `uav-onboard` 최신 구조

```text
uav-onboard/
├── CMakeLists.txt
├── PROJECT_SPEC.md
├── README.md
├── config/
│   ├── autopilot.toml
│   ├── mission.toml
│   ├── network.toml
│   ├── safety.toml
│   └── vision.toml
├── docs/
│   └── PROTOCOL.md
├── scripts/
│   └── setup_rpi_dependencies.sh
├── src/
│   ├── main.cpp
│   ├── app/
│   │   └── VisionDebugPipeline.cpp/.hpp
│   ├── camera/
│   │   ├── CameraFrame.hpp
│   │   └── RpicamMjpegSource.cpp/.hpp
│   ├── common/
│   │   ├── NetworkConfig.cpp/.hpp
│   │   ├── Time.cpp/.hpp
│   │   └── VisionConfig.cpp/.hpp
│   ├── network/
│   │   └── UdpTelemetrySender.cpp/.hpp
│   ├── protocol/
│   │   └── TelemetryMessage.cpp/.hpp
│   ├── video/
│   │   ├── UdpMjpegStreamer.cpp/.hpp
│   │   └── VideoPacket.cpp/.hpp
│   └── vision/
│       ├── ArucoDetector.cpp/.hpp
│       ├── LineDetector.cpp/.hpp
│       ├── OpenCvCameraSource.cpp/.hpp
│       └── VisionTypes.hpp
├── test_data/
│   ├── images/.gitkeep
│   ├── logs/.gitkeep
│   └── videos/.gitkeep
├── tests/
│   ├── CMakeLists.txt
│   └── test_telemetry_line_json.cpp
└── tools/
    ├── aruco_detector_tester.cpp
    ├── camera_preview.cpp
    ├── line_detector_tuner.cpp
    ├── mock_gcs_command.cpp
    ├── replay_vision.cpp
    ├── video_streamer.cpp
    └── vision_debug_node.cpp
```

### 8.1 `uav-onboard` build target

| Target | 상태 | 설명 |
|---|---|---|
| `onboard_dependencies` | 구현됨 | `nlohmann/json`, `toml++` include target |
| `onboard_core` | 구현됨 | config, time, telemetry sender, telemetry JSON builder |
| `onboard_video` | 구현됨 | rpicam MJPEG source, UDP MJPEG streamer |
| `onboard_vision` | 조건부 구현 | OpenCV가 있을 때 ArUco/Line/OpenCV camera source build |
| `uav_onboard` | 구현됨 | telemetry bring-up sender |
| `video_streamer` | 구현됨 | rpicam/test-pattern/image source debug streamer |
| `vision_debug_node` | 구현됨 | rpicam capture, ArUco/line detection, telemetry, raw video |
| `camera_preview` | 조건부 | OpenCV camera smoke test |
| `aruco_detector_tester` | 조건부 | image 기반 ArUco detector smoke test |
| `line_detector_tuner` | 조건부 | image 기반 line detector tuning |
| `replay_vision` | scaffold | 향후 recorded frame replay |
| `mock_gcs_command` | scaffold | 향후 command channel test |
| `test_telemetry_line_json` | 구현됨 | onboard line telemetry JSON generation test |

### 8.2 `uav-onboard` 주요 파일 역할

| 파일 | 역할 |
|---|---|
| `CMakeLists.txt` | core/video/vision target, tools, tests 정의. OpenCV 없으면 vision tools skip |
| `README.md` | Pi dependency, build, camera, video, vision debug, line tuning 사용법 |
| `config/network.toml` | GCS IP/port, telemetry interval. 기본은 discovery/broadcast 호환 |
| `config/vision.toml` | camera/video/ArUco/line detector 설정 |
| `docs/PROTOCOL.md` | GCS와 공유하는 protocol v1.2 문서 |
| `scripts/setup_rpi_dependencies.sh` | Raspberry Pi build/runtime dependency 설치 |
| `src/main.cpp` | `uav_onboard` telemetry bring-up entrypoint |
| `src/app/VisionDebugPipeline.cpp` | vision debug runtime. rpicam frame decode, ArUco/line detection, telemetry/video send |
| `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid --codec mjpeg -o -` stdout에서 JPEG SOI/EOI frame 추출 |
| `src/camera/CameraFrame.hpp` | frame id, timestamp, size, JPEG bytes |
| `src/common/NetworkConfig.cpp` | `network.toml` parser |
| `src/common/VisionConfig.cpp` | `vision.toml` parser |
| `src/common/VisionConfig.hpp` | camera/video/aruco/line config 구조체. line default는 `auto`, ROI `0.08`, lookahead `0.55` |
| `src/network/UdpTelemetrySender.cpp` | UDP telemetry sender, broadcast 지원 |
| `src/protocol/TelemetryMessage.cpp` | marker/line/debug latency 포함 telemetry JSON builder |
| `src/video/UdpMjpegStreamer.cpp` | JPEG frame을 AQV1 UDP chunk로 송신. best-effort latest-frame 방식 |
| `src/video/VideoPacket.cpp` | AQV1 header serialize/parse |
| `src/vision/ArucoDetector.cpp` | OpenCV ArUco detection. id, corners, center, orientation 계산. drawing 없음 |
| `src/vision/LineDetector.cpp` | grayscale threshold/morphology/contour 기반 line detection |
| `src/vision/VisionTypes.hpp` | marker, line, vision result type |
| `tools/vision_debug_node.cpp` | CLI, config override, GCS discovery, pipeline 실행 |
| `tools/video_streamer.cpp` | raw video stream smoke tool |
| `tools/line_detector_tuner.cpp` | image 파일로 line mode/threshold/ROI/lookahead 튜닝 |
| `tools/aruco_detector_tester.cpp` | image 파일로 ArUco detector 확인 |
| `tests/test_telemetry_line_json.cpp` | `vision.line` telemetry JSON 생성 검증 |

### 8.3 LineDetector 현재 동작

현재 라인 검출은 Pi Zero 2 W 부담을 줄이기 위해 가벼운 OpenCV pipeline으로 유지한다.

동작 요약:

1. `roi_top_ratio`로 상단 일부를 제외한다. 기본값은 `0.08`.
2. ROI를 grayscale로 변환한다.
3. `mode=auto`면 bright-line mask와 dark-line mask를 둘 다 생성한다.
4. `light_on_dark`는 밝은 라인, `dark_on_light`는 어두운 라인만 threshold한다.
5. `threshold=0`이면 Otsu 자동 threshold를 사용한다.
6. morphology open/close로 작은 노이즈를 줄인다.
7. contour 후보를 찾는다.
8. `lookahead_y_ratio=0.55` 위치 행을 지나는 contour만 후보로 본다.
9. 면적, 길이 span, 폭, 화면 중앙성, edge contact penalty로 후보를 scoring한다.
10. 선택된 connected contour 전체를 `contour_px`로 단순화해서 telemetry로 보낸다.

중요한 현재 판단:

- branch filtering은 되돌렸다.
- 십자형 교차부가 connected component로 잡히면 GCS magenta contour도 십자 모양으로 보이게 한다.
- 교차점 판단은 아직 구현하지 않는다.
- 반사광은 `line-mode`, `line-threshold`, `camera angle`, `roi/lookahead`로 우선 튜닝한다.

### 8.4 VisionDebugPipeline 현재 동작

`vision_debug_node`는 다음 순서로 동작한다.

```text
rpicam MJPEG frame
  -> JPEG decode to cv::Mat
  -> ArUco detector optional
  -> LineDetector optional
  -> telemetry JSON build/send
  -> raw JPEG frame best-effort video send
```

온보드에서는 overlay drawing을 하지 않는다. GCS가 telemetry를 받아 marker/line overlay를 그린다.

## 9. 현재 검증된 기능

| 항목 | 상태 |
|---|---|
| Windows GCS build without OpenCV | 성공. Win32/WIC fallback 사용 |
| GCS video window | 성공 |
| GCS vision debug window | 성공 |
| GCS separate vision log window | 구현됨 |
| GCS line telemetry parser test | 성공 |
| GCS line overlay test | 성공 |
| onboard build without OpenCV | non-OpenCV target 성공, vision tools skip 정상 |
| onboard telemetry line JSON test | 성공 |
| Pi OpenCV ArUco build | 이전 Pi 테스트에서 성공 |
| Pi camera rpicam MJPEG source | 이전 Pi 테스트에서 성공 |
| GCS discovery 후 unicast video | 성공 |
| ArUco overlay | 사용자 live test에서 성공 |
| line overlay | 사용자 live test에서 동작 확인 |
| line ROI/lookahead 개선 | tracking point 중앙화, 상단 절단 완화 확인 |
| connected cross contour | branch filtering rollback 후 목표 동작으로 복구 |

최근 로컬 검증 명령:

```powershell
cd uav-onboard
cmake --build build
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure

cd ..\uav-gcs
cmake --build build
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

주의: Windows 로컬 환경에는 OpenCV가 없으므로 `vision_debug_node`, `LineDetector`, `ArucoDetector`가 포함된 onboard vision target은 로컬에서 빌드되지 않는다. Pi에서 OpenCV/aruco가 있는 환경으로 최종 빌드를 확인해야 한다.

## 10. 현재 문제 해결 이력 요약

자세한 내용은 `TROUBLESHOOTING.md`에 있다.

| 번호 | 이슈 | 현재 결론 |
|---:|---|---|
| 1 | GCS telemetry 미수신 | broadcast + `SO_BROADCAST`로 해결 |
| 2 | Windows OpenCV 없음 | Win32/WIC fallback video backend로 해결 |
| 3 | Pi `rpicam-vid` 시작 실패 | POSIX `popen("r")`로 해결 |
| 4 | `--count` 후 rpicam abort 로그 | stderr log 분리, 정상 종료 부작용으로 판단 |
| 5 | broadcast video 불안정 | GCS discovery 후 unicast video 전송 |
| 6 | GCS 검은 화면 flicker | 마지막 complete frame 유지, double buffering |
| 7 | OpenCV VideoCapture와 CSI camera 불안정 | Pi camera는 rpicam 경로 사용 |
| 8 | ArUco enum build error | Pi OpenCV 4.10 enum type에 맞춤 |
| 9 | GCS vision debug packet 0개 | Windows firewall inbound allow 필요 |
| 10 | overlay를 어디서 그릴지 | GCS-side overlay로 확정 |
| 13 | 라인 상단 절단/낮은 point/반사광 | ROI/lookahead 개선, line mode/threshold override |
| 14 | branch filtering이 십자 인식을 방해 | branch filtering rollback, connected contour 우선 |

## 11. 현재 리스크

| 리스크 | 영향 | 대응 |
|---|---|---|
| Pi Zero 2 W CPU 여유 부족 | ArUco+line+video 동시 실행 시 FPS 저하 가능 | video는 best-effort, 필요 시 `--no-video`, FPS/quality 축소 |
| line threshold가 조명에 민감 | 반사광/그림자/바닥 무늬 오검출 | `--line-mode`, `--line-threshold`, `--line-roi-top`, `--line-lookahead` 현장 튜닝 |
| 교차점 판단 미구현 | grid mission 진행 불가 | 다음 단계에서 intersection telemetry 추가 |
| line branch와 intersection contour 목적 충돌 | 주행용 중심선과 교차점 검출 요구가 다름 | line branch와 intersection candidate를 별도 result로 분리 |
| Windows firewall | 새 GCS exe가 packet을 못 받을 수 있음 | inbound UDP allow rule 점검 |
| 자동화 테스트 부족 | vision regression 위험 | captured image 기반 line/ArUco 테스트 추가 |
| command channel 없음 | GCS에서 mode/threshold를 실행 중 바꾸기 어려움 | command channel 또는 debug TCP command 추가 |
| Pixhawk/MAVLink 미연동 | 실제 비행 제어 불가 | vision 안정화 후 붙이는 것이 합리적 |

## 12. 추천 다음 단계

### 1단계: Pi에서 최신 line/ArUco 동시 빌드 및 실행 확인

가장 먼저 Pi에서 최신 코드를 pull하고 OpenCV target이 실제로 빌드되는지 확인한다.

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-mode light_on_dark
```

확인 기준:

- Pi console에 `aruco: on`, `line: on` 표시
- GCS video window 표시
- GCS vision log window 표시
- ArUco marker가 보이면 marker overlay 표시
- 라인이 보이면 magenta contour와 green tracking point 표시
- 십자 라인이 connected contour로 표시

### 2단계: line detector 현장 튜닝 preset 정리

현재 line detector는 실행 전 CLI override로 대응한다. 경기장 전까지 다음 preset을 문서화하고 테스트한다.

흰색/밝은 라인:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

어두운 라인:

```bash
./build/vision_debug_node --config config --line-only --line-mode dark_on_light
```

반사광이 심할 때:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-threshold 180
```

tracking point 위치 조정:

```bash
./build/vision_debug_node --config config --line-only --line-lookahead 0.45
./build/vision_debug_node --config config --line-only --line-lookahead 0.60
```

### 3단계: 테스트 이미지/영상 데이터셋 확보

현재 가장 큰 위험은 실제 경기장 환경이 아직 불확실하다는 점이다. 다음 데이터를 `uav-onboard/test_data/` 또는 별도 large artifact 정책으로 모아야 한다.

- 흰색 라인, 어두운 라인
- 밝은 자연광
- 그림자
- 반사광
- 흙바닥/운동장 질감
- 십자 교차점
- ArUco marker와 라인이 동시에 보이는 장면

최소한 정지 이미지 기반으로 `line_detector_tuner`를 반복 실행할 수 있어야 한다.

### 4단계: intersection detection MVP

라인 contour가 안정화된 뒤 교차점 판단을 추가한다.

권장 방향:

- `LineDetection`은 주행용 line tracking point/offset을 계속 제공
- 새 `IntersectionDetection` 또는 `intersection_score`를 별도 계산
- connected contour의 shape, bounding box, skeleton/branch count 또는 horizontal/vertical span을 이용
- 교차점 이벤트에는 hysteresis/cooldown 적용
- telemetry에는 `intersection_detected`, `intersection_score`를 먼저 실제 값으로 채움

이 단계에서는 grid 좌표 저장까지 한 번에 넣지 말고, GCS overlay/log로 교차점 검출 안정성을 먼저 본다.

### 5단계: grid state와 marker map

교차점 판단이 안정화되면 grid state를 붙인다.

추천 데이터:

```cpp
struct GridState {
    int row;
    int col;
    float heading_deg;
};

struct MarkerRecord {
    int id;
    int row;
    int col;
    std::int64_t first_seen_timestamp_ms;
    int confirm_count;
};
```

구현 순서:

1. 시작 grid/heading을 config 또는 debug command로 설정
2. intersection event마다 row/col update
3. ArUco marker가 안정적으로 반복 검출되면 현재 grid 좌표에 marker 저장
4. GCS vision log window에 grid/marker map 요약 표시

### 6단계: vision loop를 mission-safe 구조로 리팩터링

현재 `vision_debug_node`는 디버그 실행 파일로는 충분하지만, 최종 mission loop와 완전히 분리된 구조는 아니다.

권장 구조:

```text
Camera capture
  -> latest frame slot
  -> Vision processing
  -> Mission state update
  -> Telemetry publish
  -> optional debug video publish
```

핵심:

- debug video 송신은 bounded queue 또는 latest-frame slot
- video 송신이 밀리면 frame drop
- mission 판단과 control output은 video 송신을 기다리지 않음
- 단계별 latency를 계속 telemetry로 보냄

### 7단계: command channel

비행 제어 command 전에 vision debug command부터 붙이는 것이 안전하다.

초기 command 후보:

- `set_line_mode`
- `set_line_threshold`
- `set_line_roi`
- `reset_vision_state`
- `set_initial_grid`
- `pause_vision`
- `resume_vision`
- `request_status`

실제 비행 command는 Pixhawk/MAVLink 준비 후 추가한다.

## 13. 현재 결론

프로젝트는 다음 흐름까지 구현 및 부분 실기 검증이 진행된 상태다.

```text
Pi camera
  -> onboard ArUco detection
  -> onboard line detection
  -> onboard vision telemetry
  -> onboard raw MJPEG debug streaming
  -> GCS video receive
  -> GCS marker/line overlay
  -> GCS vision log window
```

현재 가장 중요한 다음 작업은 Pixhawk 제어가 아니라 line/intersection vision을 안정화하는 것이다. 특히 교차점 단계에서는 line tracing용 branch와 intersection 판단용 connected contour를 분리해서 설계해야 한다. 이 분리를 하지 않으면 반사광 억제와 십자 교차점 보존 요구가 계속 충돌한다.
