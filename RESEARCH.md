# Astroquad 프로젝트 리서치

최종 업데이트: 2026-04-27

범위: 루트 문서, `uav-gcs`, `uav-onboard`, 현재 소스 구조, 구현 상태, 검증된 bring-up 결과, 추천 다음 단계.

관련 문서:

- `PLAN.md`: ArUco 인식 및 GCS 오버레이 단계의 구현 계획 기록. 현재 주요 항목은 대부분 구현 완료됨.
- `TROUBLESHOOTING.md`: 개발 중 실제로 발생한 통신, 카메라, 영상 스트리밍, 방화벽, 오버레이 설계 판단 관련 문제와 해결 기록.
- `uav-gcs/docs/PROTOCOL.md`
- `uav-onboard/docs/PROTOCOL.md`

현재 저장소 최신 커밋:

| 저장소 | 최신 로컬 커밋 |
|---|---|
| `uav-gcs` | `dcb8093 Document vision debug firewall note` |
| `uav-onboard` | `17bec5c Fix OpenCV ArUco dictionary type` |

두 저장소 모두 `origin/main`과 동기화되어 있으며 작업 트리는 깨끗한 상태다.

## 1. 프로젝트 목표

Astroquad는 실내 격자 경기장에서 동작하는 C++ 기반 UAV 시스템이다. 최종 미션 목표는 다음과 같다.

1. 이륙한다.
2. 라인을 따라 격자 경기장에 진입한다.
3. snake 패턴으로 격자를 탐색한다.
4. 교차점 근처에서 ArUco 마커를 인식한다.
5. 마커 번호와 발견 위치를 저장한다.
6. 대회 규칙에 맞게 마커 위치를 재방문하거나 결과를 활용한다.
7. 출발 지점으로 복귀하고 착륙한다.

현재는 완성된 드론 기체와 Pixhawk가 없는 상태다. 따라서 현재 개발의 중심은 Raspberry Pi 카메라를 손에 들고 격자 경기장을 탑다운뷰로 촬영하면서 비전 기능을 먼저 완성하는 것이다.

현재 개발 원칙은 다음과 같다.

| 책임 | `uav-onboard` | `uav-gcs` |
|---|---|---|
| Pi 카메라 캡처 | 수행 | 수행하지 않음 |
| 미션 수행에 필요한 비전 연산 | 수행 | 수행하지 않음 |
| ArUco 마커 인식 | 수행 | 수행하지 않음 |
| 라인트레이싱 및 교차점 판단 | 예정 | 수행하지 않음 |
| 격자 좌표 및 마커 상태 저장 | 예정 | 표시만 담당 |
| 원본 카메라 영상 송신 | best-effort 디버그 출력 | 수신 및 표시 |
| ArUco 오버레이 그리기 | 수행하지 않음 | 수행 |
| 디버그 로그 | telemetry로 전송 | 표시 및 기록 |
| GUI 창 표시 | 수행하지 않음 | 수행 |
| 미션 시작, 중단, 비상 착륙 명령 | 추후 수신 | 추후 송신 |

핵심 아키텍처 결정은 유지된다. 온보드는 미션 수행에 필요한 최소한의 계산만 수행하고, GCS는 관제용 시각화, 오버레이, 로그, 명령 UI를 담당한다.

## 2. 현재 실행 모드

현재 실행 조합은 세 가지로 분리되어 있다.

| 목적 | GCS 명령 | Raspberry Pi 명령 | 결과 |
|---|---|---|---|
| 텔레메트리만 확인 | `.\build\uav_gcs.exe --config config` | `./build/uav_onboard --config config --count 10` | 콘솔 텔레메트리만 출력. 카메라 창은 뜨지 않음. |
| 원본 카메라 영상만 확인 | `.\build\uav_gcs_video.exe --config config` | `./build/video_streamer --source rpicam --config config` | 영상 창 1개 표시. ArUco 오버레이 없음. |
| 현재 ArUco 디버그 워크플로 | `.\build\uav_gcs_vision_debug.exe --config config` | `./build/vision_debug_node --config config` | 영상 창 1개에 GCS 측 ArUco 오버레이 표시. 마커 로그는 GCS 콘솔에 출력. |

중요: `uav_gcs_vision_debug`는 현재 별도의 로그 GUI 창을 만들지 않는다. 마커 로그는 해당 실행 파일을 실행한 PowerShell 콘솔에 주기적으로 출력된다.

## 3. 현재 검증된 기능

### 3.1 텔레메트리

- `uav_onboard`는 UDP 포트 `14550`으로 JSON 텔레메트리를 송신한다.
- `uav_gcs`는 텔레메트리를 수신하고 JSON을 파싱하며 packet loss 통계를 계산한다.
- 텔레메트리 스키마에는 camera frame sequence, marker 배열, ArUco latency 필드가 추가되었다.
- GCS parser는 초기 bring-up 시 사용했던 이전 telemetry 구조도 일부 backward-compatible하게 처리한다.

### 3.2 영상 스트리밍

- `RpicamMjpegSource`는 `rpicam-vid --codec mjpeg -o -`의 stdout에서 MJPEG frame을 읽는다.
- JPEG frame은 `AQV1` header를 가진 UDP chunk로 분할되어 전송된다.
- `uav_gcs_video`와 `uav_gcs_vision_debug`는 UDP chunk를 재조립해 complete JPEG frame만 표시한다.
- incomplete frame은 버리고, 다음 complete frame이 도착하기 전까지 마지막 정상 frame을 유지한다. 이 방식으로 검은 화면 깜빡임을 줄였다.
- Windows GCS는 OpenCV가 없어도 Win32/WIC backend로 JPEG decode와 화면 표시가 가능하다.

### 3.3 GCS 자동 discovery

GCS는 UDP 포트 `5601`로 다음 discovery beacon을 1초마다 broadcast한다.

```text
AQGCS1 video_port=5600
```

`video_streamer`와 `vision_debug_node`는 `config/network.toml`의 GCS IP가 `255.255.255.255` 또는 `0.0.0.0`이고 `--gcs-ip` override가 없을 때 이 beacon을 기다린다. beacon을 받으면 송신자 IP를 실제 GCS IP로 사용하고, 영상은 broadcast가 아니라 unicast로 송신한다.

이 구조 덕분에 노트북 IP가 네트워크마다 바뀌어도 매번 Pi 설정 파일을 수정하지 않아도 된다.

### 3.4 ArUco 비전 디버그

현재 확인된 동작:

- `vision_debug_node`가 Pi camera MJPEG frame을 캡처한다.
- JPEG를 `cv::Mat`으로 decode한다.
- OpenCV ArUco detector로 온보드에서 ArUco 인식을 수행한다.
- 원본 JPEG frame을 GCS로 전송한다.
- marker id, corners, center, orientation을 telemetry로 GCS에 보낸다.
- Raspberry Pi에서는 오버레이를 그리지 않는다.
- `uav_gcs_vision_debug`는 영상과 marker telemetry를 수신한다.
- GCS는 `frame_seq` 기준으로 영상 frame과 marker telemetry를 맞춘다.
- GCS에서만 marker box, corner point, center point, 방향 화살표, 텍스트 label을 그린다.

사용자 실기 테스트에서 ArUco 오버레이가 GCS 영상 창에 정상 표시되는 것을 확인했다.

### 3.5 알려진 환경 이슈

Windows에서는 새로 빌드된 실행 파일이 Windows Defender Firewall에 의해 자동으로 차단될 수 있다. `uav_gcs_vision_debug.exe`도 inbound UDP 허용 규칙이 필요하다.

Pi에서 `discovered GCS video receiver at <laptop-ip>:5600`가 출력되는데 GCS가 packet을 받지 못한다면, 우선 Windows 방화벽을 확인해야 한다.

## 4. 루트 디렉토리 구조

```text
astroquad/
├── PLAN.md
├── RESEARCH.md
├── TROUBLESHOOTING.md
├── uav-gcs/
└── uav-onboard/
```

현재 루트 디렉토리 자체는 git 저장소가 아니며, `uav-gcs`와 `uav-onboard`가 각각 독립적인 git 저장소다.

| 루트 파일 | 역할 |
|---|---|
| `PLAN.md` | ArUco/GCS overlay milestone 구현 계획 기록. 현재 대부분 구현 완료. |
| `RESEARCH.md` | 현재 프로젝트 구조, 구현 상태, 검증 결과, 다음 단계 정리 문서. |
| `TROUBLESHOOTING.md` | 실제 개발 중 발생한 문제와 해결 방법을 보고서용 개발로그로 기록한 문서. |

## 5. `uav-gcs` 구조와 파일 역할

### 5.1 디렉토리 개요

```text
uav-gcs/
├── .gitignore
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
├── scripts/
│   └── .gitkeep
├── src/
│   ├── main.cpp
│   ├── video_main.cpp
│   ├── vision_debug_main.cpp
│   ├── app/
│   ├── common/
│   ├── logging/
│   ├── network/
│   ├── overlay/
│   ├── protocol/
│   ├── state/
│   ├── telemetry/
│   ├── ui/
│   └── video/
├── test_data/
│   └── telemetry/.gitkeep
├── tests/
│   └── CMakeLists.txt
├── third_party/
│   └── .gitkeep
└── tools/
    ├── log_replayer.cpp
    └── mock_onboard.cpp
```

### 5.2 Build target

| Target | 상태 | 설명 |
|---|---|---|
| `gcs_dependencies` | 구현됨 | `nlohmann/json`, `toml++` header-only dependency include target. |
| `gcs_core` | 구현됨 | network config, UDP telemetry receiver, telemetry parser/statistics. |
| `gcs_video` | 구현됨 | 영상 수신, discovery beacon, 영상 창, marker overlay, marker telemetry store/log formatter. |
| `uav_gcs` | 구현됨 | 기본 telemetry console receiver. |
| `uav_gcs_video` | 구현됨 | marker overlay 없는 원본 영상 viewer. |
| `uav_gcs_vision_debug` | 구현됨 | GCS 측 ArUco overlay와 console marker log가 포함된 vision debug viewer. |
| `mock_onboard` | 구현됨 | GCS telemetry receiver 테스트용 mock sender. |
| `log_replayer` | scaffold | 향후 log replay용 placeholder. |

GCS video UI backend는 두 가지다.

- OpenCV backend: `src/ui/VideoWindow.cpp`
- Windows fallback backend: `src/ui/VideoWindowWin32.cpp`, WIC로 JPEG decode, GDI로 drawing 수행

### 5.3 `uav-gcs` 파일별 역할

| 파일 | 현재 역할 |
|---|---|
| `.gitignore` | build 결과물과 runtime 파일 제외. |
| `CMakeLists.txt` | dependency, `gcs_core`, `gcs_video`, 실행 파일, tools, optional tests 정의. |
| `PROJECT_SPEC.md` | GCS 원본 프로젝트 명세. |
| `README.md` | 빌드/실행 가이드. Ninja와 multi-config generator의 실행 파일 경로 차이 및 방화벽 주의사항 포함. |
| `config/network.toml` | GCS network config. onboard IP, telemetry port, command port, video port, timeout 설정. |
| `config/ui.toml` | 영상 창 title, timeout 등 UI 설정. |
| `docs/PROTOCOL.md` | protocol v1.1 문서. onboard copy와 반드시 동일해야 함. |
| `logs/.gitkeep` | runtime log 디렉토리를 git에 유지하기 위한 placeholder. |
| `scripts/.gitkeep` | 향후 script용 placeholder. |
| `src/main.cpp` | `uav_gcs` entrypoint. UDP telemetry 수신, packet 파싱, 통계 출력. |
| `src/video_main.cpp` | `uav_gcs_video` entrypoint. config를 읽고 raw video viewer 실행. |
| `src/vision_debug_main.cpp` | `uav_gcs_vision_debug` entrypoint. config를 읽고 vision debug viewer 실행. |
| `src/app/.gitkeep` | app 디렉토리 placeholder. |
| `src/app/VideoViewerApp.cpp` | raw MJPEG video receive, discovery beacon, video window display loop. |
| `src/app/VideoViewerApp.hpp` | raw video viewer option과 class 선언. |
| `src/app/VisionDebugApp.cpp` | video receive, telemetry receive thread, telemetry store, marker overlay, console marker log orchestration. |
| `src/app/VisionDebugApp.hpp` | vision debug viewer option과 class 선언. |
| `src/common/.gitkeep` | common 디렉토리 placeholder. |
| `src/common/NetworkConfig.cpp` | `toml++` 기반 `network.toml` parser. |
| `src/common/NetworkConfig.hpp` | GCS network config 구조체와 loader 선언. |
| `src/logging/.gitkeep` | 향후 logging module placeholder. |
| `src/network/.gitkeep` | network 디렉토리 placeholder. |
| `src/network/UdpTelemetryReceiver.cpp` | cross-platform UDP bind/select/receive 구현. |
| `src/network/UdpTelemetryReceiver.hpp` | UDP telemetry receiver interface. |
| `src/overlay/OverlayPrimitive.hpp` | line, circle, text, color, point로 구성된 overlay primitive model. |
| `src/overlay/MarkerOverlay.cpp` | marker telemetry를 box, corner, center, arrow, label primitive로 변환. |
| `src/overlay/MarkerOverlay.hpp` | marker overlay builder 선언. |
| `src/protocol/.gitkeep` | protocol 디렉토리 placeholder. |
| `src/protocol/TelemetryMessage.cpp` | JSON telemetry parser. `vision.markers[]`, `debug.aruco_latency_ms`, sequence stats 처리. |
| `src/protocol/TelemetryMessage.hpp` | GCS telemetry data structure, marker structure, parser/stat declarations. |
| `src/state/.gitkeep` | 향후 application state module placeholder. |
| `src/telemetry/TelemetryStore.cpp` | 최근 marker telemetry를 frame sequence 기준으로 저장하고 timestamp fallback matching 제공. |
| `src/telemetry/TelemetryStore.hpp` | `MarkerFrame`, `TelemetryStore` 선언. |
| `src/telemetry/MarkerLogFormatter.cpp` | marker telemetry를 주기적 console log 문자열로 formatting. |
| `src/telemetry/MarkerLogFormatter.hpp` | marker log formatter 선언. |
| `src/ui/.gitkeep` | UI backend 디렉토리 placeholder. |
| `src/ui/VideoWindow.hpp` | video window abstraction. `showFrame(frame, overlays)` 지원. |
| `src/ui/VideoWindow.cpp` | OpenCV video window backend. frame text와 overlay primitive를 OpenCV로 drawing. |
| `src/ui/VideoWindowWin32.cpp` | Windows WIC/GDI backend. OpenCV 없이 JPEG decode 및 overlay drawing. |
| `src/video/GcsDiscoveryBeacon.cpp` | UDP `5601`에 `AQGCS1 video_port=...` discovery beacon broadcast. |
| `src/video/GcsDiscoveryBeacon.hpp` | discovery beacon interface. |
| `src/video/JpegFrameReassembler.cpp` | UDP chunk를 JPEG frame으로 재조립하고 incomplete/outdated data drop. |
| `src/video/JpegFrameReassembler.hpp` | `JpegFrame`, reassembler 선언. |
| `src/video/UdpMjpegReceiver.cpp` | `AQV1` UDP chunk를 수신해 complete JPEG frame 반환. |
| `src/video/UdpMjpegReceiver.hpp` | UDP MJPEG receiver interface. |
| `src/video/VideoPacket.cpp` | big-endian `AQV1` header serialize/parse. |
| `src/video/VideoPacket.hpp` | `AQV1` 상수와 header 구조체. |
| `test_data/telemetry/.gitkeep` | telemetry sample data placeholder. |
| `tests/CMakeLists.txt` | test subtree placeholder. 실제 테스트는 아직 거의 없음. |
| `third_party/.gitkeep` | vendored dependency placeholder. |
| `tools/log_replayer.cpp` | 향후 log replay utility placeholder. |
| `tools/mock_onboard.cpp` | local GCS test용 mock telemetry sender. |

### 5.4 현재 GCS 한계

- 별도 log GUI window가 없다. marker log는 `uav_gcs_vision_debug`를 실행한 terminal에 출력된다.
- command/control UI가 없다.
- mission dashboard가 없다.
- telemetry/event file logging이 없다.
- overlay mapping, telemetry store, video receive behavior에 대한 자동화 테스트가 없다.

## 6. `uav-onboard` 구조와 파일 역할

### 6.1 디렉토리 개요

```text
uav-onboard/
├── .gitignore
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
├── logs/
│   └── .gitkeep
├── scripts/
│   ├── .gitkeep
│   └── setup_rpi_dependencies.sh
├── src/
│   ├── main.cpp
│   ├── app/
│   ├── autopilot/
│   ├── camera/
│   ├── common/
│   ├── control/
│   ├── logging/
│   ├── mission/
│   ├── network/
│   ├── protocol/
│   ├── safety/
│   ├── video/
│   └── vision/
├── test_data/
│   ├── images/.gitkeep
│   ├── logs/.gitkeep
│   └── videos/.gitkeep
├── tests/
│   └── CMakeLists.txt
└── tools/
    ├── aruco_detector_tester.cpp
    ├── camera_preview.cpp
    ├── line_detector_tuner.cpp
    ├── mock_gcs_command.cpp
    ├── replay_vision.cpp
    ├── video_streamer.cpp
    └── vision_debug_node.cpp
```

### 6.2 Build target

| Target | 상태 | 설명 |
|---|---|---|
| `onboard_dependencies` | 구현됨 | `nlohmann/json`, `toml++` header-only dependency include target. |
| `onboard_core` | 구현됨 | config loader, time helper, telemetry sender, telemetry JSON builder. |
| `onboard_video` | 구현됨 | Pi camera MJPEG source, UDP MJPEG streamer, video packet support. |
| `onboard_vision` | 조건부 | OpenCV와 `aruco` module이 있을 때만 build. ArUco detector와 OpenCV camera wrapper 포함. |
| `uav_onboard` | 구현됨 | 기본 telemetry bring-up sender. |
| `video_streamer` | 구현됨 | raw camera/test-pattern/image MJPEG streamer. |
| `vision_debug_node` | 구현됨 | Pi camera capture, ArUco detection, raw video streaming, marker telemetry. |
| `camera_preview` | 조건부 | OpenCV camera preview/smoke tool. Pi CSI camera에서는 rpicam 경로가 더 안정적. |
| `aruco_detector_tester` | 구현됨 | image file 기반 ArUco detector smoke tool. |
| `line_detector_tuner` | scaffold | line tracing 작업용 placeholder. |
| `replay_vision` | scaffold | 향후 replay 기반 vision test용 placeholder. |
| `mock_gcs_command` | scaffold | command channel test용 placeholder. |

### 6.3 `uav-onboard` 파일별 역할

| 파일 | 현재 역할 |
|---|---|
| `.gitignore` | build 결과물과 runtime 파일 제외. |
| `CMakeLists.txt` | core/video/vision library, 실행 파일, conditional OpenCV tools, optional tests 정의. |
| `PROJECT_SPEC.md` | onboard 원본 프로젝트 명세. |
| `README.md` | build/run guide, Pi dependency setup, camera/video/ArUco debug instructions. |
| `config/autopilot.toml` | 향후 autopilot/Pixhawk 설정. 현재 runtime에서 적극 사용하지 않음. |
| `config/mission.toml` | 향후 mission 설정. 현재 runtime에서 적극 사용하지 않음. |
| `config/network.toml` | GCS 목적지 IP/port와 telemetry interval. 기본값은 broadcast/discovery 호환 구조. |
| `config/safety.toml` | 향후 safety 설정. 현재 runtime에서 적극 사용하지 않음. |
| `config/vision.toml` | camera/video 설정과 ArUco detector parameter. |
| `docs/PROTOCOL.md` | protocol v1.1 문서. GCS copy와 반드시 동일해야 함. |
| `logs/.gitkeep` | runtime log 디렉토리 placeholder. |
| `scripts/.gitkeep` | scripts 디렉토리 placeholder. |
| `scripts/setup_rpi_dependencies.sh` | Raspberry Pi build/runtime dependency 설치. camera package와 OpenCV contrib package 포함. |
| `src/main.cpp` | `uav_onboard` entrypoint. 기본 telemetry packet 송신. |
| `src/app/.gitkeep` | 향후 onboard application orchestration placeholder. |
| `src/autopilot/.gitkeep` | 향후 Pixhawk/MAVLink integration placeholder. |
| `src/camera/CameraFrame.hpp` | camera frame 구조체. frame id, timestamp, width, height, JPEG bytes 포함. |
| `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid`를 실행하고 stdout에서 JPEG SOI/EOI frame 추출. |
| `src/camera/RpicamMjpegSource.hpp` | `RpicamOptions`와 source interface. |
| `src/common/.gitkeep` | common 디렉토리 placeholder. |
| `src/common/NetworkConfig.cpp` | `network.toml`의 `[gcs]`, `[telemetry]` parser. |
| `src/common/NetworkConfig.hpp` | onboard network config 구조체와 loader 선언. |
| `src/common/Time.cpp` | Unix timestamp milliseconds helper. |
| `src/common/Time.hpp` | time helper 선언. |
| `src/common/VisionConfig.cpp` | `vision.toml`의 camera/video/ArUco section parser. |
| `src/common/VisionConfig.hpp` | camera, video, ArUco, aggregate vision config 구조체. |
| `src/control/.gitkeep` | 향후 control logic placeholder. |
| `src/logging/.gitkeep` | 향후 onboard logging placeholder. |
| `src/mission/.gitkeep` | 향후 mission state machine placeholder. |
| `src/network/.gitkeep` | network 디렉토리 placeholder. |
| `src/network/UdpTelemetrySender.cpp` | cross-platform UDP telemetry sender. broadcast mode 지원. |
| `src/network/UdpTelemetrySender.hpp` | UDP telemetry sender interface. |
| `src/protocol/.gitkeep` | protocol 디렉토리 placeholder. |
| `src/protocol/TelemetryMessage.cpp` | `vision.markers[]`, ArUco latency를 포함한 telemetry JSON builder. |
| `src/protocol/TelemetryMessage.hpp` | onboard telemetry data structure와 JSON builder 선언. |
| `src/safety/.gitkeep` | 향후 safety/failsafe logic placeholder. |
| `src/video/UdpMjpegStreamer.cpp` | JPEG frame을 `AQV1` UDP chunk로 송신. |
| `src/video/UdpMjpegStreamer.hpp` | UDP MJPEG streamer interface. |
| `src/video/VideoPacket.cpp` | big-endian `AQV1` header serialize/parse. |
| `src/video/VideoPacket.hpp` | `AQV1` 상수와 header 구조체. |
| `src/vision/.gitkeep` | vision 디렉토리 placeholder. |
| `src/vision/ArucoDetector.cpp` | OpenCV ArUco detector. id, corner, center, image-plane orientation 계산. drawing은 하지 않음. |
| `src/vision/ArucoDetector.hpp` | ArUco detector interface. |
| `src/vision/OpenCvCameraSource.cpp` | `camera_preview`에서 쓰는 OpenCV `VideoCapture` wrapper. |
| `src/vision/OpenCvCameraSource.hpp` | OpenCV camera source interface. |
| `src/vision/VisionTypes.hpp` | onboard vision result type. 현재 marker observation과 향후 line/intersection placeholder 포함. |
| `test_data/images/.gitkeep` | image test data placeholder. |
| `test_data/logs/.gitkeep` | log test data placeholder. |
| `test_data/videos/.gitkeep` | video test data placeholder. |
| `tests/CMakeLists.txt` | test subtree placeholder. 실제 테스트는 아직 거의 없음. |
| `tools/aruco_detector_tester.cpp` | image를 load해 `ArucoDetector`를 실행하고 marker id/center/corners/orientation 출력. |
| `tools/camera_preview.cpp` | OpenCV camera preview/smoke tool. |
| `tools/line_detector_tuner.cpp` | 향후 line detector tuning placeholder. |
| `tools/mock_gcs_command.cpp` | 향후 GCS command test placeholder. |
| `tools/replay_vision.cpp` | 향후 offline vision replay placeholder. |
| `tools/video_streamer.cpp` | rpicam/test-pattern/image source를 GCS로 streaming. discovery와 `--gcs-ip` override 지원. |
| `tools/vision_debug_node.cpp` | 현재 live ArUco debug node. Pi camera capture, ArUco detection, raw MJPEG, marker telemetry 송신. |

### 6.4 현재 onboard 한계

- vision loop가 아직 final mission application이 아니라 debug executable에 머물러 있다.
- ArUco detector가 camera read, telemetry send, video send와 같은 loop에서 동기적으로 실행된다.
- line tracing 구현이 없다.
- intersection detection과 grid state update가 없다.
- persistent marker map이 없다.
- command receiver가 없다.
- Pixhawk/MAVLink integration이 없다.
- ArUco detector, telemetry JSON builder, video chunking 자동화 테스트가 없다.

## 7. 프로토콜 요약

Protocol version: `1`

Protocol 문서 version: `v1.1`

`uav-gcs/docs/PROTOCOL.md`와 `uav-onboard/docs/PROTOCOL.md`는 동일한 내용을 담고 있다.

| Channel | 방향 | Transport | 기본 포트 |
|---|---|---|---:|
| Telemetry | onboard -> GCS | UDP JSON | 14550 |
| Command | GCS -> onboard | TCP JSON | 14551 |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 |

현재 telemetry 주요 필드:

- `protocol_version`
- `type`
- `seq`
- `timestamp_ms`
- `mission.state`
- `camera.status`
- `camera.width`
- `camera.height`
- `camera.fps`
- `camera.frame_seq`
- `vision.line_detected`
- `vision.line_offset`
- `vision.line_angle`
- `vision.intersection_detected`
- `vision.intersection_score`
- `vision.marker_detected`
- `vision.marker_id`
- `vision.marker_count`
- `vision.markers[]`
- `grid.row`
- `grid.col`
- `grid.heading_deg`
- `debug.processing_latency_ms`
- `debug.aruco_latency_ms`
- `debug.note`

`vision.markers[]` entry는 다음 정보를 담는다.

- `id`
- `center_px`
- `corners_px[4]`
- `orientation_deg`

GCS overlay는 이 marker telemetry를 그대로 사용한다. GCS는 ArUco 인식을 다시 수행하지 않는다.

## 8. 현재 빌드 및 실행 메모

### 8.1 GCS build

```powershell
cd uav-gcs
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

현재 로컬 generator는 Ninja이므로 실행 파일은 `build/Release/`가 아니라 `build/` 바로 아래에 있다.

```powershell
.\build\uav_gcs.exe --config config
.\build\uav_gcs_video.exe --config config
.\build\uav_gcs_vision_debug.exe --config config
```

### 8.2 Raspberry Pi onboard build

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

깨끗한 Raspberry Pi에서는 먼저 다음을 실행한다.

```bash
bash scripts/setup_rpi_dependencies.sh
```

### 8.3 현재 ArUco debug 실행

GCS:

```powershell
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi:

```bash
./build/vision_debug_node --config config
```

정상 동작 기준:

- Pi가 `discovered GCS video receiver at <ip>:5600`를 출력한다.
- Pi가 `frame=N markers=M jpeg_bytes=...`를 출력한다.
- GCS에 영상 창 1개가 열린다.
- marker가 보이면 GCS 영상 창에 marker overlay가 그려진다.
- marker log는 PowerShell console에 출력된다.

## 9. 검증된 테스트와 관찰 결과

현재 milestone 중 수행한 확인:

| 확인 항목 | 결과 |
|---|---|
| 로컬 `uav-gcs` CMake build | 성공 |
| 로컬 `uav-onboard` CMake build without OpenCV | non-OpenCV target 성공, OpenCV tools skip 정상 |
| 로컬 `uav_gcs_vision_debug --help` | 성공 |
| test-pattern video + generated marker telemetry local smoke test | 성공. GCS가 marker log를 출력하고 telemetry 30개 수신 |
| Pi OpenCV ArUco availability | 확인. Pi OpenCV 4.10.0 + ArUco module 사용 가능 |
| Pi `uav-onboard` build after pull | 성공. `vision_debug_node`, `aruco_detector_tester` build됨 |
| Pi `vision_debug_node --count 5 --no-video` | 성공. camera frame capture 및 ArUco 처리 loop 수행 |
| Pi/GCS discovery | 성공. Pi가 GCS beacon으로 laptop IP 발견 |
| Pi/GCS vision debug integration | Windows firewall이 허용된 실행 파일 경로에서는 성공 |
| 사용자 live ArUco overlay test | 성공. GCS 영상 창에 overlay 정상 표시 |

주의 사항:

- Windows가 새 실행 파일에 Block firewall rule을 만들 수 있다.
- marker log는 별도 GUI 창이 아니라 console output이다.
- Pi와 laptop clock이 완전히 동기화되어 있지 않으면 log의 `age`가 약간 음수로 보일 수 있다.

## 10. 현재 risk register

| Risk | 현재 영향 | 대응 |
|---|---|---|
| Windows firewall이 새 executable을 차단 | Pi가 GCS를 발견해도 video/telemetry packet이 안 들어올 수 있음 | `uav_gcs_vision_debug.exe` inbound allow rule 추가 |
| UDP video packet loss | incomplete frame drop 발생 | 마지막 complete frame 유지, 추후 FPS/resolution/quality 조정 |
| vision debug loop가 동기 구조 | video send 또는 telemetry send가 mission vision에 영향을 줄 수 있음 | final mission loop 전 capture/vision/debug streaming을 bounded queue 또는 latest-frame slot으로 분리 |
| 자동화 테스트 부족 | 기능 추가 시 regression 위험 증가 | protocol, packet, reassembler, store, overlay mapping 테스트 추가 |
| 별도 log GUI 없음 | 관제자가 로그를 독립 창에서 확인할 수 없음 | `MarkerLogWindow` 또는 ImGui log panel 구현 |
| line/intersection logic 없음 | 최종 grid mission 진행 불가 | ArUco debug 안정화 후 line tracing MVP 구현 |
| grid/marker state persistence 없음 | marker는 표시되지만 mission knowledge로 저장되지 않음 | marker map과 grid state module 추가 |
| command channel 없음 | GCS에서 start/abort/reset 불가 | vision debug/log UI 안정화 후 구현 |

## 11. 추천 다음 단계

### 1단계: GCS 전용 로그 창 추가

현재 ArUco milestone은 기능적으로 동작하지만 marker log가 terminal에만 출력된다. 최종 GCS 목표는 video 창, log 창, command 창 총 3개다. 따라서 다음 UI 단계는 단순한 별도 log window를 추가하는 것이다.

추천 범위:

- `src/ui/MarkerLogWindow.hpp/.cpp` 추가
- Windows에서는 우선 가벼운 Win32 text/list window로 구현
- 초기에는 `MarkerLogFormatter` 출력 재사용
- terminal output은 fallback으로 유지
- 아직 full dashboard까지 확장하지 않음

완료 기준:

- `uav_gcs_vision_debug` 실행 시 video window와 별도 marker/log window가 열린다.
- log window가 1-2초 주기로 갱신된다.
- latest frame id, marker count, marker id, center, orientation, ArUco latency, packet stats가 표시된다.

### 2단계: `vision_debug_node`를 non-blocking vision pipeline 방향으로 리팩터링

현재 `vision_debug_node`는 debug 용도로는 충분하지만, 최종 mission code에서는 GCS 영상 송신이 mission-critical vision을 막으면 안 된다.

권장 구조:

```text
RpicamMjpegSource
  -> latest captured frame
  -> VisionPipeline
  -> VisionState
  -> TelemetryPublisher
  -> optional VideoDebugStreamer
```

구현 방향:

- ArUco detection은 onboard에 유지
- overlay drawing은 GCS에 유지
- debug video는 bounded queue 또는 latest-frame slot 사용
- video send가 느려지면 debug frame을 버리고 vision loop를 우선
- 단계별 latency 측정

### 3단계: ArUco test data와 자동화 테스트 추가

다음 vision logic을 붙이기 전에 현재 동작을 테스트로 고정해야 한다.

추천 테스트:

- `uav-onboard/test_data/images/`에 sample ArUco image 저장 또는 생성
- known marker ID로 `aruco_detector_tester` 검증
- telemetry JSON build/parse round-trip test
- GCS marker telemetry parse test
- `TelemetryStore` frame matching test
- `MarkerOverlay` primitive generation test

### 4단계: 라인트레이싱 MVP 구현

ArUco debug가 안정화되면 live mission 통합 전에 image file과 recorded frame 기반으로 line tracing을 시작한다.

추천 초기 data model:

```cpp
struct LineDetection {
    bool detected;
    float center_offset_px;
    float angle_deg;
    float confidence;
    std::vector<Point2f> contour_px;
};
```

추천 접근:

- threshold/mask + contour 또는 `fitLine`으로 시작
- `line_detected`, offset, angle, confidence 출력
- 동일한 overlay primitive system으로 GCS에 line contour/center line overlay 추가

### 5단계: 교차점 판단 및 grid state 구현

line detection이 안정화된 뒤 다음 기능을 붙인다.

- intersection candidate와 score 계산
- duplicate event 방지를 위한 hysteresis/cooldown
- 현재 grid row/col 및 heading 추적
- visited intersection 저장
- grid state telemetry 송신
- 향후 log window에 현재 grid 위치 표시

### 6단계: marker map 저장 구현

현재 marker detection은 동작하지만, 최종 mission에는 persistent marker knowledge가 필요하다.

추천 구조:

```cpp
struct MarkerRecord {
    int id;
    int row;
    int col;
    std::int64_t first_seen_timestamp_ms;
};
```

저장할 정보:

- marker id
- first confirmed 시점의 grid coordinate
- first seen time
- optional confidence 또는 repeated detection count

### 7단계: vision debug UI 안정화 후 command channel 구현

command channel은 최종 설계에서 제외된 것이 아니라 현재 우선순위가 아닐 뿐이다.

초기 command는 비행 명령보다 vision debug command부터 시작하는 것이 좋다.

- `reset_vision_state`
- `set_initial_grid`
- `set_heading`
- `pause_vision`
- `resume_vision`
- `request_status`

비행 관련 command는 이후 실제 기체와 Pixhawk 준비 후 붙인다.

- `start_mission`
- `abort_mission`
- `emergency_land`

## 12. 현재 결론

프로젝트는 기본 bring-up 단계를 넘어 다음 흐름이 검증된 상태다.

```text
Pi camera
  -> onboard ArUco detection
  -> onboard marker telemetry
  -> onboard raw MJPEG streaming
  -> GCS video reception
  -> GCS-side marker overlay
  -> GCS console marker logs
```

이는 현재 설계 원칙과 일치한다. 온보드는 인식만 수행하고, GCS는 overlay와 관제용 시각화를 담당한다.

다음에 가장 유용한 작업은 Pixhawk 연동이 아니다. 먼저 전용 log window를 추가해 debug 관측성을 높이고, 그다음 onboard vision loop를 mission-safe한 구조로 리팩터링해야 한다. 이후 line tracing, intersection detection, marker map, grid state를 차례로 붙이는 것이 적절하다.
