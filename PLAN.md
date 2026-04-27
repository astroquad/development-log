# Astroquad Next Vision Milestone Plan

작성일: 2026-04-27  
상태: 구현 전 계획 문서  
원칙: 이 문서는 작업 계획만 정의한다. 이번 요청에서는 소스 코드를 수정하지 않는다.

## 1. 현재 프로젝트 진행 상황 요약

`RESEARCH.md`와 현재 파일 구조 기준으로 Astroquad는 기본 bring-up 단계를 넘어 다음 흐름이 검증된 상태다.

```text
Pi camera
  -> onboard ArUco detection
  -> onboard marker telemetry
  -> onboard raw MJPEG streaming
  -> GCS video reception
  -> GCS-side marker overlay
  -> GCS console marker logs
```

현재 핵심 설계 원칙은 유지한다.

| 책임 | Onboard | GCS |
|---|---|---|
| 카메라 frame 획득 | 수행 | 수행하지 않음 |
| 미션 판단에 필요한 비전 연산 | 수행 | 수행하지 않음 |
| ArUco 마커 인식 | 수행 | 수행하지 않음 |
| 라인트레이싱 인식 | 수행 예정 | 수행하지 않음 |
| 영상 송신 | 원본 JPEG best-effort 송신 | 수신 및 표시 |
| 오버레이 drawing | 수행하지 않음 | 수행 |
| GUI 창 | 수행하지 않음 | 수행 |
| 로그/관제 표시 | telemetry로 전달 | 표시 및 기록 |

즉, 온보드는 숫자화된 비전 결과만 만들고, GCS는 그 결과를 사람이 보기 좋은 형태로 그린다.

## 2. 현재 전체 디렉터리 구조

```text
astroquad/
├── development-log/
│   ├── .git/
│   ├── PLAN.md
│   ├── RESEARCH.md
│   └── TROUBLESHOOTING.md
├── uav-gcs/
│   ├── .git/
│   ├── CMakeLists.txt
│   ├── PROJECT_SPEC.md
│   ├── README.md
│   ├── config/
│   ├── docs/
│   ├── logs/
│   ├── scripts/
│   ├── src/
│   ├── test_data/
│   ├── tests/
│   ├── third_party/
│   └── tools/
└── uav-onboard/
    ├── .git/
    ├── CMakeLists.txt
    ├── PROJECT_SPEC.md
    ├── README.md
    ├── config/
    ├── docs/
    ├── logs/
    ├── scripts/
    ├── src/
    ├── test_data/
    ├── tests/
    └── tools/
```

루트 `astroquad/` 자체는 Git 저장소가 아니다. `development-log`, `uav-gcs`, `uav-onboard`가 각각 독립적인 Git 저장소다.

## 3. 주요 파일 역할 파악

### 3.1 `development-log`

| 파일 | 역할 |
|---|---|
| `PLAN.md` | 현재 문서. 다음 구현 단계의 기술 계획. |
| `RESEARCH.md` | 프로젝트 구조, 구현 상태, 검증 결과, 다음 단계 요약. |
| `TROUBLESHOOTING.md` | 실제 개발 중 발생한 문제와 해결 과정 기록. |

### 3.2 `uav-gcs` 핵심 구조

| 영역 | 핵심 파일 | 현재 역할 |
|---|---|---|
| 빌드 | `CMakeLists.txt` | `gcs_core`, `gcs_video`, `uav_gcs`, `uav_gcs_video`, `uav_gcs_vision_debug` target 정의. OpenCV가 없으면 Windows WIC/GDI backend 사용. |
| 설정 | `config/network.toml`, `config/ui.toml` | telemetry/video port, timeout, video window title 설정. |
| 실행 진입점 | `src/main.cpp` | 기본 telemetry console receiver. |
| 실행 진입점 | `src/video_main.cpp` | 원본 MJPEG video viewer. |
| 실행 진입점 | `src/vision_debug_main.cpp` | video + telemetry + overlay vision debug viewer. |
| 앱 orchestration | `src/app/VisionDebugApp.*` | video 수신, telemetry thread, telemetry store, marker overlay, console marker log를 하나의 GCS process 안에서 조합. |
| 앱 orchestration | `src/app/VideoViewerApp.*` | overlay 없는 raw video viewer. |
| telemetry 수신 | `src/network/UdpTelemetryReceiver.*` | UDP telemetry receive. |
| telemetry parsing | `src/protocol/TelemetryMessage.*` | JSON telemetry parse. `vision.markers[]`와 legacy marker fields를 처리. |
| telemetry store | `src/telemetry/TelemetryStore.*` | frame sequence/timestamp 기준으로 marker telemetry를 video frame에 매칭. 현재 이름은 marker 중심이지만 역할은 vision frame store에 가깝다. |
| log formatting | `src/telemetry/MarkerLogFormatter.*` | marker telemetry를 console 문자열로 formatting. |
| overlay model | `src/overlay/OverlayPrimitive.hpp` | line, circle, text primitive 정의. 라인트레이싱 오버레이도 이 모델을 그대로 사용할 수 있다. |
| marker overlay | `src/overlay/MarkerOverlay.*` | marker corners/center/orientation을 overlay primitive로 변환. |
| video UI | `src/ui/VideoWindow.*` | OpenCV backend 또는 Win32/WIC/GDI backend로 JPEG decode 및 overlay drawing. |
| video 수신 | `src/video/UdpMjpegReceiver.*`, `JpegFrameReassembler.*`, `VideoPacket.*` | `AQV1` UDP chunk 수신 및 JPEG frame 재조립. |
| discovery | `src/video/GcsDiscoveryBeacon.*` | GCS video receiver discovery beacon broadcast. |
| tools | `tools/mock_onboard.cpp`, `tools/log_replayer.cpp` | 개발용 mock/replay placeholder. |

### 3.3 `uav-onboard` 핵심 구조

| 영역 | 핵심 파일 | 현재 역할 |
|---|---|---|
| 빌드 | `CMakeLists.txt` | `onboard_core`, `onboard_video`, 조건부 `onboard_vision`, `vision_debug_node`, `video_streamer`, tool target 정의. |
| 설정 | `config/network.toml` | GCS IP/port, telemetry interval 설정. discovery fallback과 호환. |
| 설정 | `config/vision.toml` | camera/video/ArUco 설정. 이미 `[line]` section placeholder가 있다. |
| 실행 진입점 | `src/main.cpp` | 기본 bring-up telemetry sender. |
| camera | `src/camera/RpicamMjpegSource.*`, `CameraFrame.hpp` | `rpicam-vid` stdout에서 MJPEG frame 추출. |
| video 송신 | `src/video/UdpMjpegStreamer.*`, `VideoPacket.*` | JPEG frame을 `AQV1` UDP chunk로 GCS에 송신. |
| telemetry 송신 | `src/network/UdpTelemetrySender.*` | UDP telemetry sender. |
| telemetry build | `src/protocol/TelemetryMessage.*` | onboard telemetry JSON 생성. marker 배열과 line placeholder field가 이미 있다. |
| vision type | `src/vision/VisionTypes.hpp` | `MarkerObservation`, `VisionResult` 정의. 현재 line은 bool placeholder만 있다. |
| ArUco | `src/vision/ArucoDetector.*` | OpenCV ArUco detector. id/corners/center/orientation 계산, drawing은 하지 않음. |
| live debug | `tools/vision_debug_node.cpp` | Pi camera capture, JPEG decode, ArUco detect, telemetry 송신, 원본 video 송신. 현재 동기 loop 구조. |
| raw video | `tools/video_streamer.cpp` | ArUco 없이 raw camera/test-pattern/image video 송신. |
| line tool | `tools/line_detector_tuner.cpp` | 현재 scaffold. 라인트레이싱 튜닝 도구로 쓰기 좋다. |
| replay tool | `tools/replay_vision.cpp` | 현재 scaffold. recorded frame 기반 regression 확인 도구 후보. |

## 4. 핵심 설계 판단: ArUco와 라인트레이싱은 같은 live workflow에 통합한다

이번 라인트레이싱 MVP는 **기존 `vision_debug_node` + `uav_gcs_vision_debug` 흐름에 통합하는 것이 최적**이다.

결론:

```text
Live 실행 파일은 분리하지 않는다.

유지/확장:
  Raspberry Pi: ./build/vision_debug_node --config config
  GCS:          .\build\uav_gcs_vision_debug.exe --config config

별도 도구:
  line_detector_tuner, replay_vision은 튜닝/테스트용으로만 사용한다.
```

### 4.1 통합하는 이유

| 판단 기준 | 통합 workflow | 별도 live 실행 파일 |
|---|---|---|
| 최종 미션 구조 | ArUco와 line은 같은 카메라 frame에서 나온 mission vision 결과이므로 자연스럽다. | 나중에 다시 합쳐야 한다. |
| video/telemetry 동기화 | 같은 `frame_seq`로 marker와 line을 동시에 매칭할 수 있다. | 두 실행 파일이면 frame 기준이 갈라진다. |
| 카메라 점유 | Pi camera를 한 process가 소유한다. | 두 live process가 동시에 camera를 쓰기 어렵다. |
| UDP port/firewall | 기존 GCS port와 방화벽 규칙을 재사용한다. | 새 실행 파일/port/firewall 문제가 늘어난다. |
| 코드 중복 | discovery, video 송신, telemetry 송신, overlay loop를 재사용한다. | 거의 같은 코드를 한 벌 더 만들 위험이 있다. |
| 디버깅 | 한 영상에서 marker와 line을 같이 보며 frame alignment를 확인할 수 있다. | 각각은 쉬워도 통합 문제를 늦게 발견한다. |

### 4.2 단, detector enable/disable 옵션은 둔다

통합 실행 파일을 쓰되, 기능은 config 또는 CLI로 끄고 켤 수 있어야 한다.

권장 옵션:

```bash
./build/vision_debug_node --config config
./build/vision_debug_node --config config --line-only
./build/vision_debug_node --config config --aruco-only
./build/vision_debug_node --config config --disable-aruco
./build/vision_debug_node --config config --disable-line
```

GCS도 overlay 표시 옵션을 둘 수 있다.

```powershell
.\build\uav_gcs_vision_debug.exe --config config
.\build\uav_gcs_vision_debug.exe --config config --show-line
.\build\uav_gcs_vision_debug.exe --config config --hide-markers
```

초기 구현에서는 옵션 수를 최소화해도 된다. 중요한 것은 live 실행 파일을 새로 나누는 것이 아니라, 같은 workflow 안에서 detector와 overlay를 선택 가능하게 만드는 것이다.

## 5. 라인트레이싱 MVP 범위

이번 라인트레이싱 단계에서 구현할 것:

- 온보드에서 카메라 frame 기반 라인 검출
- 라인 contour 또는 edge polyline 추출
- 라인 tracking point 계산
- line offset, angle, confidence 계산
- GCS telemetry로 line metadata 송신
- GCS 영상 창에 line overlay 표시
- 라인 contour는 분홍색/magenta로 표시
- tracking point는 초록색/green 점으로 표시
- ArUco marker overlay와 line overlay를 같은 영상 창에 동시에 표시 가능하게 구성

이번 단계에서 구현하지 않을 것:

- 교차점 판단
- grid row/col 갱신
- visited coordinate 저장
- marker map 저장
- Pixhawk/MAVLink 제어
- 라인 기반 실제 비행 제어 출력
- mission command channel

## 6. 라인트레이싱 데이터 정의

`VisionTypes.hpp`에는 marker와 같은 레벨의 line 결과 타입을 둔다.

권장 onboard type:

```cpp
struct LineDetection {
    bool detected = false;
    Point2f tracking_point_px;
    Point2f centroid_px;
    float center_offset_px = 0.0f;
    float angle_deg = 0.0f;
    float confidence = 0.0f;
    std::vector<Point2f> contour_px;
};
```

필드 의미:

| 필드 | 의미 |
|---|---|
| `detected` | 유효한 라인을 찾았는지 여부. |
| `tracking_point_px` | 라인트레이싱 제어에 사용할 대표 점. GCS에서는 초록색 점으로 표시한다. |
| `centroid_px` | 검출된 contour의 무게중심. tracking point 계산 실패 시 fallback으로 쓸 수 있다. |
| `center_offset_px` | image center x 기준 tracking point의 좌우 오프셋. |
| `angle_deg` | 라인의 image-plane 방향. `fitLine` 또는 contour principal direction 기준. |
| `confidence` | contour 면적, 길이, mask 품질 등을 합친 0.0~1.0 신뢰도. |
| `contour_px` | GCS에서 분홍색 테두리로 그릴 contour/polyline 좌표. |

### 6.1 tracking point 정의

사용자가 말한 “카메라가 라인을 인식하는 점”은 단순 centroid보다 **lookahead row에서의 line center**로 정의하는 것이 라인트레이싱에 더 유용하다.

권장 방식:

```text
1. frame 하단 또는 중앙 하단에 ROI를 잡는다.
2. ROI 안에서 가장 신뢰도 높은 line contour를 찾는다.
3. configurable lookahead_y_ratio 위치의 가로 scan line과 contour가 만나는 x 범위를 구한다.
4. 그 x 범위의 중앙을 tracking_point_px로 둔다.
5. 실패하면 contour centroid를 fallback tracking point로 둔다.
```

초기값 예:

```toml
[line]
enabled = true
roi_top_ratio = 0.35
lookahead_y_ratio = 0.70
min_area_px = 250
max_contour_points = 80
```

이렇게 하면 초록색 점이 “라인 전체의 중심”이 아니라 실제 제어 입력에 쓸 관측점이 된다.

## 7. Telemetry schema 확장 계획

현재 schema에는 legacy line fields가 이미 있다.

```json
{
  "vision": {
    "line_detected": false,
    "line_offset": 0.0,
    "line_angle": 0.0
  }
}
```

이번 단계에서는 backward compatibility를 위해 위 필드는 유지하고, 상세 line 객체를 추가한다.

권장 schema:

```json
{
  "vision": {
    "line_detected": true,
    "line_offset": -18.5,
    "line_angle": 2.4,
    "line": {
      "detected": true,
      "tracking_point_px": { "x": 304.0, "y": 336.0 },
      "centroid_px": { "x": 310.2, "y": 258.6 },
      "center_offset_px": -16.0,
      "angle_deg": 2.4,
      "confidence": 0.86,
      "contour_px": [
        { "x": 282.0, "y": 120.0 },
        { "x": 350.0, "y": 120.0 },
        { "x": 365.0, "y": 470.0 },
        { "x": 260.0, "y": 470.0 }
      ]
    },
    "marker_detected": true,
    "marker_count": 1,
    "markers": []
  }
}
```

프로토콜 문서는 구현 단계에서 `uav-gcs/docs/PROTOCOL.md`와 `uav-onboard/docs/PROTOCOL.md` 양쪽을 동일하게 갱신한다. 문서 버전은 `v1.2` 후보로 둔다.

## 8. GCS 오버레이 설계

라인 오버레이는 기존 `OverlayPrimitive`만으로 구현 가능하다.

| 시각 요소 | 색상 | primitive |
|---|---|---|
| line contour / border | magenta `RGB(255, 0, 255)` | contour point 사이 `OverlayLine` |
| tracking point | green `RGB(0, 255, 0)` | filled `OverlayCircle` |
| optional line direction | yellow/cyan 계열 | `OverlayLine` arrow |
| optional label | green 또는 white | `OverlayText` |

권장 drawing order:

1. line contour를 먼저 그림
2. line tracking point를 그림
3. ArUco marker box/corner/center/text를 그림
4. frame latency text는 기존처럼 상단에 표시

이 순서가 좋은 이유는 line contour가 화면의 큰 면적을 차지할 수 있기 때문이다. marker overlay를 나중에 그리면 marker 정보가 line contour에 묻히지 않는다.

추가할 GCS module 후보:

```text
uav-gcs/src/overlay/LineOverlay.hpp
uav-gcs/src/overlay/LineOverlay.cpp
```

`VisionDebugApp`에서는 다음처럼 overlay를 합친다.

```text
overlays = buildLineOverlays(frame.line)
overlays += buildMarkerOverlays(frame.markers)
window.showFrame(frame, overlays)
```

## 9. 최종 단계별 실행 계획

### 1단계: GCS 전용 로그 창 추가

### 목표

현재 marker log는 PowerShell console에만 출력된다. 다음 단계에서는 별도 log window를 추가해 video window와 분리된 관측성을 확보한다.

### 설계 판단

`MarkerLogWindow`라는 이름으로 시작하기보다, 라인트레이싱까지 고려해 **`VisionLogWindow`** 또는 이에 준하는 일반 이름을 쓰는 편이 낫다.

이유:

- 4단계에서 marker뿐 아니라 line detection 상태도 표시해야 한다.
- 지금 marker 전용 이름으로 만들면 곧바로 rename/refactor가 필요하다.
- 최종 GCS는 marker, line, packet stats, latency, event를 함께 보여줘야 한다.

### 구현 후보

GCS:

```text
uav-gcs/src/ui/VisionLogWindow.hpp
uav-gcs/src/ui/VisionLogWindow.cpp
uav-gcs/src/telemetry/VisionLogFormatter.hpp
uav-gcs/src/telemetry/VisionLogFormatter.cpp
```

초기에는 `MarkerLogFormatter`의 문자열을 재사용해도 된다. 다만 파일/클래스 이름은 line 확장에 맞게 일반화한다.

### 완료 기준

- `uav_gcs_vision_debug` 실행 시 video window와 별도 log window가 열린다.
- marker id, center, orientation, ArUco latency, packet stats가 1~2초 주기로 갱신된다.
- log window가 실패하거나 Windows backend 이슈가 있으면 console log fallback이 유지된다.
- 아직 command UI는 구현하지 않는다.

### 2단계: `vision_debug_node`를 non-blocking vision pipeline 방향으로 리팩터링

### 목표

현재 `vision_debug_node`는 한 loop에서 다음 작업을 순서대로 수행한다.

```text
camera read
  -> JPEG decode
  -> ArUco detect
  -> telemetry send
  -> video send
```

라인 검출까지 추가되면 detection 비용이 늘어난다. 또한 video send가 느려질 때 mission-critical vision loop가 막히면 안 된다. 따라서 line 구현 전 또는 line 구현과 동시에 pipeline 경계를 정리한다.

### 권장 구조

```text
RpicamMjpegSource
  -> CaptureWorker
  -> LatestFrameSlot
  -> VisionWorker
       -> JPEG decode once
       -> enabled detectors: ArUco, Line
       -> VisionResult
  -> TelemetryPublisher
  -> DebugVideoStreamer
```

핵심 정책:

- JPEG decode는 frame당 한 번만 한다.
- ArUco와 Line detector는 같은 decoded `cv::Mat`을 사용한다.
- telemetry는 latest vision result 기준으로 송신한다.
- debug video는 best-effort다.
- video send가 느려지면 오래된 debug frame은 버린다.
- `frame_seq`는 video frame id와 telemetry `camera.frame_seq`에서 동일해야 한다.

### 구현 후보

Onboard:

```text
uav-onboard/src/app/VisionDebugPipeline.hpp
uav-onboard/src/app/VisionDebugPipeline.cpp
uav-onboard/src/vision/VisionTypes.hpp
uav-onboard/tools/vision_debug_node.cpp
```

처음부터 과도한 framework를 만들 필요는 없다. 다만 `vision_debug_node.cpp`에 모든 로직이 계속 쌓이지 않도록 pipeline class로 분리한다.

### 완료 기준

- 기존 ArUco live debug 기능이 유지된다.
- video stream이 잠깐 느려져도 capture/detection loop가 멈추지 않는다.
- `--count`, `--no-video`, `--no-telemetry` 옵션이 계속 동작한다.
- frame sequence 동기화가 유지된다.

### 3단계: ArUco test data와 자동화 테스트 추가

### 목표

라인 기능을 붙이기 전에 이미 성공한 ArUco/overlay/protocol 동작을 테스트로 고정한다.

### 추천 테스트

Onboard:

```text
uav-onboard/tests/test_aruco_detector.cpp
uav-onboard/tests/test_telemetry_marker_json.cpp
uav-onboard/tests/test_vision_types.cpp
```

GCS:

```text
uav-gcs/tests/test_marker_telemetry_parse.cpp
uav-gcs/tests/test_telemetry_store.cpp
uav-gcs/tests/test_marker_overlay_mapping.cpp
uav-gcs/tests/test_video_frame_marker_sync.cpp
```

Test data:

```text
uav-onboard/test_data/images/aruco_marker_*.png
uav-gcs/test_data/telemetry/marker_*.json
```

### 구현 방향

- ArUco marker image는 OpenCV로 생성하거나 repo에 작은 PNG로 저장한다.
- marker 0개, 1개, 여러 개 telemetry JSON을 모두 테스트한다.
- `TelemetryStore`의 exact `frame_seq` match와 timestamp fallback을 테스트한다.
- `MarkerOverlay`가 marker 1개당 기대 개수의 primitive를 만드는지 확인한다.

### 완료 기준

- ArUco marker 검출 결과가 deterministic하게 확인된다.
- marker telemetry build/parse round-trip이 깨지지 않는다.
- GCS overlay primitive 생성이 테스트로 보호된다.
- line schema 추가 전에 기존 marker 기능 regression 위험이 줄어든다.

### 4단계: 라인트레이싱 MVP 구현

### 목표

교차점 판단이나 grid 저장 없이, 라인 인식과 GCS overlay만 구현한다.

목표 영상:

```text
raw camera frame
  + magenta line contour/border
  + green tracking point
  + optional ArUco marker overlay
```

### 4.1 Onboard detector

추가 후보:

```text
uav-onboard/src/vision/LineDetector.hpp
uav-onboard/src/vision/LineDetector.cpp
```

권장 초기 알고리즘:

1. decoded BGR frame 입력
2. `vision.toml`의 `[line]` config 로드
3. ROI crop 적용
4. grayscale 또는 HSV 기반 threshold/mask 생성
5. morphology open/close로 noise 제거
6. 가장 큰 valid contour 선택
7. contour area, length, bounding box로 confidence 계산
8. `fitLine` 또는 contour principal axis로 `angle_deg` 계산
9. lookahead row 기준 `tracking_point_px` 계산
10. contour point를 적당히 simplify해서 telemetry 크기 제한

초기 config 후보:

```toml
[line]
enabled = true
mode = "dark_on_light"
roi_top_ratio = 0.35
lookahead_y_ratio = 0.70
threshold = 90
min_area_px = 250
morph_kernel = 5
max_contour_points = 80
confidence_min = 0.30
```

실제 경기장 라인 색과 조명에 따라 `mode`, `threshold`, `roi_top_ratio`는 조정될 수 있다. 따라서 `line_detector_tuner`를 먼저 살려서 이미지 파일 기반으로 threshold를 조정할 수 있게 만든다.

### 4.2 Onboard telemetry

`VisionResult`에 line result를 추가한다.

```cpp
struct VisionResult {
    std::uint32_t frame_seq = 0;
    std::int64_t timestamp_ms = 0;
    int width = 0;
    int height = 0;
    std::vector<MarkerObservation> markers;
    LineDetection line;
};
```

`BringupTelemetry.vision`에는 legacy fields와 상세 `line` object를 함께 채운다.

```text
vision.line_detected = result.line.detected
vision.line_offset = result.line.center_offset_px
vision.line_angle = result.line.angle_deg
vision.line = detailed line object
```

### 4.3 GCS parser/store

현재 `TelemetryStore`는 marker 중심의 `MarkerFrame`을 저장한다. 라인까지 들어오면 다음 중 하나를 선택한다.

권장:

```text
MarkerFrame -> VisionFrame으로 일반화
TelemetryStore -> VisionTelemetryStore 또는 이름 유지 후 내부 type만 일반화
```

최소 변경 경로:

- class 이름 `TelemetryStore`는 유지한다.
- 저장 struct만 `VisionFrame`으로 확장한다.
- 기존 marker API는 가능하면 wrapper로 유지한다.

`VisionFrame` 후보:

```cpp
struct VisionFrame {
    std::uint32_t frame_seq = 0;
    std::int64_t timestamp_ms = 0;
    int width = 0;
    int height = 0;
    double processing_latency_ms = 0.0;
    double aruco_latency_ms = 0.0;
    double line_latency_ms = 0.0;
    std::vector<protocol::MarkerTelemetry> markers;
    std::optional<protocol::LineTelemetry> line;
};
```

### 4.4 GCS line overlay

추가 후보:

```text
uav-gcs/src/overlay/LineOverlay.hpp
uav-gcs/src/overlay/LineOverlay.cpp
```

`LineOverlay`는 detector가 아니라 telemetry-to-primitive 변환기다.

입력:

- `LineTelemetry`

출력:

- contour point 사이 magenta line segments
- tracking point green filled circle
- optional angle/offset text

색상:

```cpp
magenta = {255, 0, 255}
green   = {0, 255, 0}
```

### 4.5 GCS live overlay 통합

`VisionDebugApp`의 기존 marker overlay 흐름을 다음처럼 확장한다.

```text
vision_frame = telemetry_store.findForFrame(video_frame.frame_id, video_frame.timestamp_ms)

overlays = []
if vision_frame.line.detected:
    overlays += buildLineOverlays(vision_frame.line)
overlays += buildMarkerOverlays(vision_frame.markers)

video_window.showFrame(video_frame, overlays)
```

기본값은 ArUco와 line을 동시에 표시하는 것이다. 단, 성능이나 화면 혼잡이 문제되면 overlay toggle을 둔다.

### 4.6 라인트레이싱 튜닝 도구

live 실행 파일을 새로 만들지는 않지만, tuning tool은 별도로 둔다.

`line_detector_tuner`의 권장 역할:

```bash
./build/line_detector_tuner --image test_data/images/line_sample_01.jpg --config config
./build/line_detector_tuner --image sample.jpg --threshold 90 --roi-top 0.35
```

출력:

```text
detected=true
tracking_point=(304.0,336.0)
offset=-16.0px
angle=2.4deg
confidence=0.86
contour_points=42
```

OpenCV GUI가 가능한 환경에서는 mask/contour preview를 띄울 수 있다. Pi headless 환경에서는 숫자 출력과 output image 저장만으로 시작해도 된다.

### 4.7 완료 기준

- Pi에서 `vision_debug_node --config config` 실행 시 line telemetry가 송신된다.
- GCS `uav_gcs_vision_debug` 영상 창에 분홍색 line contour가 표시된다.
- GCS 영상 창에 초록색 tracking point가 표시된다.
- ArUco marker가 보이면 기존 marker overlay도 함께 표시된다.
- `--line-only` 또는 equivalent config로 ArUco 없이 line만 확인할 수 있다.
- marker가 없어도 line overlay는 정상 표시된다.
- line이 없으면 stale line overlay를 그리지 않는다.
- 교차점 판단, grid 좌표 저장, marker map 저장은 구현하지 않는다.

## 10. 권장 작업 순서

실제 구현 시 순서는 다음이 가장 안전하다.

1. `PLAN.md` 기준으로 1단계 log window를 generic vision log window로 구현한다.
2. 기존 ArUco live flow가 그대로 동작하는지 GCS/Pi에서 확인한다.
3. `vision_debug_node` 내부 구조를 pipeline class로 분리하되 behavior는 바꾸지 않는다.
4. ArUco/protocol/GCS store/overlay 테스트를 추가한다.
5. `LineDetection` type과 line config를 추가한다.
6. `line_detector_tuner`를 image 기반으로 먼저 구현한다.
7. `LineDetector`를 `vision_debug_node` pipeline에 붙인다.
8. telemetry schema에 `vision.line`을 추가하고 GCS parser를 확장한다.
9. `LineOverlay`를 추가하고 GCS video window에 magenta/green overlay를 표시한다.
10. Pi live camera로 line-only, ArUco-only, combined mode를 각각 확인한다.

## 11. 주요 리스크와 대응

| 리스크 | 영향 | 대응 |
|---|---|---|
| line detection threshold가 조명에 민감함 | 라인 contour가 끊기거나 바닥을 라인으로 오인할 수 있음 | `line_detector_tuner`, config 기반 threshold/ROI/morphology 조정 |
| contour point가 너무 많음 | telemetry packet이 커지고 UDP 손실 가능성 증가 | contour simplify, `max_contour_points` 제한 |
| ArUco와 line을 동시에 돌리면 Pi CPU 부담 증가 | FPS 저하 가능 | detector enable/disable 옵션, ROI 제한, video best-effort drop |
| video와 telemetry frame mismatch | overlay 위치가 늦게 보일 수 있음 | 기존 `frame_seq` match 유지, stale threshold 적용 |
| marker와 line overlay 색이 헷갈림 | 화면 해석 어려움 | line은 magenta/green 고정, marker overlay는 기존 색 유지하되 필요 시 후속 조정 |
| log window가 marker 전용으로 굳어짐 | line log 추가 때 refactor 필요 | 1단계부터 `VisionLogWindow` 이름으로 일반화 |
| live 실행 파일을 분리함 | 나중에 통합 비용 증가 | live는 통합, tuning/replay만 별도 tool 유지 |

## 12. 최종 결론

이번 1~4단계의 최적 방향은 **기존 vision debug workflow를 일반화해서 ArUco와 라인트레이싱을 같은 video/telemetry/GCS overlay 흐름에 태우는 것**이다.

따라서 live 실행 파일은 새로 나누지 않는다.

```text
Raspberry Pi: vision_debug_node
GCS:          uav_gcs_vision_debug
```

대신 다음은 별도 도구로 유지한다.

```text
line_detector_tuner: line threshold/ROI/contour 튜닝
replay_vision: recorded frame 기반 regression 확인
aruco_detector_tester: ArUco detector smoke test
```

라인트레이싱 MVP의 화면 목표는 명확하다.

```text
GCS video window
  -> 원본 카메라 영상
  -> 분홍색 라인 contour/border overlay
  -> 초록색 tracking point overlay
  -> 필요 시 기존 ArUco marker overlay 동시 표시
```

교차점 판단과 grid state 저장은 이번 milestone의 범위 밖으로 둔다. 라인 검출 결과가 안정적으로 보이고 telemetry/overlay 구조가 굳어진 뒤 다음 단계에서 교차점 판단과 grid state를 붙이는 것이 맞다.
