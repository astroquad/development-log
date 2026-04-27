# Astroquad ArUco Detection and GCS Overlay Plan

작성일: 2026-04-27  
상태: 구현 전 계획 문서  
원칙: 이 문서는 작업 계획만 정의한다. 현재 단계에서는 소스 코드를 수정하지 않는다.

## 1. 현재 상황 요약

`RESEARCH.md` 기준으로 프로젝트는 다음 상태까지 진행됐다.

- `uav-onboard`
  - Raspberry Pi에서 `rpicam-vid --codec mjpeg -o -` 기반 Pi camera frame capture가 가능하다.
  - `video_streamer --source rpicam`이 MJPEG frame을 UDP chunk로 GCS에 송신한다.
  - GCS discovery beacon을 받아 노트북 IP를 자동으로 찾고, 영상 본 데이터는 unicast로 보낸다.
  - `uav_onboard`는 UDP telemetry JSON을 송신한다.
  - telemetry schema에는 `camera`, `vision`, `grid`, `debug` 기본 객체가 있으나 값 대부분은 placeholder다.

- `uav-gcs`
  - `uav_gcs_video`가 UDP MJPEG stream을 수신해 별도 영상 창에 표시한다.
  - Windows에서 OpenCV가 없어도 Win32/WIC fallback으로 영상 창이 동작한다.
  - 영상 창 깜빡임을 줄이기 위해 마지막 정상 frame 유지와 double buffering이 적용되어 있다.
  - `uav_gcs`는 telemetry를 콘솔로 수신하고 packet loss 통계를 출력한다.

현재 다음 단계의 핵심 목표는 **온보드에서 ArUco marker를 인식하고, GCS에서 그 결과를 로그와 영상 overlay로 확인하는 것**이다.

## 2. 이번 단계 목표

이번 단계의 구현 목표는 다음 하나로 좁힌다.

```text
Pi Camera
  -> onboard frame capture
  -> onboard ArUco marker detection
  -> onboard marker metadata telemetry 송신
  -> GCS video window 수신
  -> GCS marker telemetry 수신
  -> GCS에서 marker box/id/좌표/방향 overlay 표시
  -> GCS에서 marker log를 텍스트로 주기 출력
```

이번 단계에서 구현하지 않을 것:

- 라인트레이싱
- 교차점 판단
- grid 좌표 갱신
- marker map 영구 저장
- Pixhawk/MAVLink 제어
- 미션 시작/강제 중단 command channel
- Dear ImGui 전체 dashboard
- 온보드에서 overlay 이미지 생성
- 온보드에서 GUI 표시

## 3. 핵심 설계 원칙

### 3.1 Onboard와 GCS 책임 분리

가장 중요한 원칙은 다음이다.

| 영역 | Onboard | GCS |
|---|---|---|
| 카메라 frame 획득 | 수행 | 수행하지 않음 |
| ArUco marker 인식 | 수행 | 수행하지 않음 |
| marker id/corner/center 계산 | 수행 | 수신 |
| mission에 필요한 최소 상태 저장 | 수행 | 표시 |
| 영상 송신 | best-effort로 수행 | 수신 |
| overlay 그리기 | 수행하지 않음 | 수행 |
| GUI 창 표시 | 수행하지 않음 | 수행 |
| 로그/관제 표시 | 수행하지 않음 | 수행 |
| 비행 command UI | 수행하지 않음 | 최종 단계에서 수행 |

온보드는 미션 수행에 필요한 최소 연산만 한다. ArUco 인식은 나중에 marker 저장과 경로 판단에 필요하므로 온보드에서 수행한다. 하지만 marker 테두리, 텍스트, 좌표 라벨, 방향 화살표 같은 관제용 시각화는 전부 GCS에서 수행한다.

### 3.2 영상 frame과 marker telemetry 동기화

overlay가 정확하려면 marker telemetry가 어떤 영상 frame에서 나온 결과인지 알아야 한다. 따라서 모든 marker telemetry에는 반드시 영상 frame과 맞출 수 있는 key를 포함한다.

필수 동기화 필드:

- `camera.frame_seq`: video packet의 `frame_id`와 같은 값
- `camera.timestamp_ms`: 해당 frame capture 시각
- `camera.width`, `camera.height`: marker corner 좌표 해석 기준

GCS는 영상 frame을 표시할 때 같은 `frame_seq`의 marker result를 찾는다. 정확히 같은 `frame_seq`가 없으면 timestamp 차이가 작은 최신 result만 사용하고, 너무 오래된 result는 overlay하지 않는다.

### 3.3 좌표와 방향 표시 기준

첨부 이미지처럼 `Pos`, `Rot` 형태의 metric pose를 표시하려면 camera calibration, marker 실제 크기, pose estimation이 필요하다.

이번 단계에서는 다음 순서로 진행한다.

1. **MVP overlay**: marker id, 4개 corner, center, image-space orientation 표시
2. **Calibration 이후 확장**: GCS에서 camera intrinsics와 marker size를 사용해 metric `Pos(mm)`, `Rot(deg)` 계산 및 표시

즉, 온보드는 marker 검출 결과인 id/corners/center만 보낸다. 관제용 pose 계산과 overlay는 GCS가 수행한다. 나중에 미션 수행에 metric pose가 정말 필요하다고 판단되면 그때 온보드 계산 여부를 다시 검토한다.

## 4. 목표 GCS 구조

최종 GCS는 하나의 실행 흐름에서 세 창을 띄우는 구조를 목표로 한다.

| 창 | 최종 목적 | 이번 단계 |
|---|---|---|
| Video Streaming Window | 카메라 영상, ArUco/라인/교차점 overlay | ArUco box/id/좌표/방향 overlay |
| Flight Log Window | marker 목록, 좌표, 고도, 기울기, FPS, 통신 상태, 이벤트 로그 | marker telemetry를 텍스트로 주기 출력 |
| Command Window | 미션 시작, 강제 중단, 비상 착륙, 파라미터 설정 | 구현하지 않음. 구조상 예약 |

중요한 설계 판단:

- 장기적으로는 GCS를 하나의 process가 여러 window를 소유하는 구조로 만든다.
- 영상, telemetry, command를 서로 다른 process가 각각 UDP/TCP port에 붙는 구조는 피한다.
- 이유는 같은 telemetry port를 여러 process가 안정적으로 공유하기 어렵고, overlay를 위해 video frame과 marker telemetry를 같은 app state에서 동기화해야 하기 때문이다.

## 5. 제안 실행 파일 구조

### 5.1 기존 실행 파일 유지

현재 실행 파일은 유지한다.

| 실행 파일 | 유지 이유 |
|---|---|
| `uav_gcs` | 순수 telemetry 수신/디버깅 |
| `uav_gcs_video` | 순수 영상 수신/디버깅 |
| `uav_onboard` | telemetry bring-up |
| `video_streamer` | 순수 영상 stream bring-up |

### 5.2 이번 단계에서 추가할 실행 파일

ArUco 단계에서는 기존 bring-up 도구를 억지로 확장하기보다, 비전 디버깅 전용 실행 파일을 추가하는 편이 낫다.

```text
uav-onboard/build/vision_debug_node
uav-gcs/build/uav_gcs_vision_debug
```

역할:

- `vision_debug_node`
  - Pi camera frame capture
  - ArUco detection
  - raw MJPEG video stream 송신
  - marker telemetry 송신

- `uav_gcs_vision_debug`
  - video stream 수신
  - marker telemetry 수신
  - GCS discovery beacon 송신
  - video overlay 표시
  - marker log를 콘솔 또는 간단한 log window에 출력

기존 `video_streamer`와 `uav_gcs_video`는 문제 분리용 smoke test 도구로 남긴다.

## 6. Telemetry schema 확장 계획

현재 telemetry schema의 `vision` 객체는 단일 marker만 표현할 수 있다.

현재 구조:

```json
{
  "vision": {
    "marker_detected": false,
    "marker_id": -1
  }
}
```

이번 단계에서는 여러 marker를 표현할 수 있도록 `markers` 배열을 추가한다.

제안 schema:

```json
{
  "protocol_version": 1,
  "type": "TELEMETRY",
  "seq": 120,
  "timestamp_ms": 1777199000000,

  "camera": {
    "status": "streaming",
    "width": 640,
    "height": 480,
    "fps": 15.0,
    "frame_seq": 358
  },

  "vision": {
    "marker_detected": true,
    "marker_count": 3,
    "markers": [
      {
        "id": 7,
        "center_px": { "x": 312.4, "y": 188.6 },
        "corners_px": [
          { "x": 280.1, "y": 150.2 },
          { "x": 344.5, "y": 151.0 },
          { "x": 343.8, "y": 222.9 },
          { "x": 279.4, "y": 221.7 }
        ],
        "orientation_deg": 0.7
      }
    ]
  },

  "debug": {
    "processing_latency_ms": 12.4,
    "aruco_latency_ms": 8.1
  }
}
```

호환성 원칙:

- 기존 `marker_detected`, `marker_id`는 당분간 유지한다.
- `marker_id`는 첫 번째 marker id 또는 대표 marker id로 채운다.
- GCS parser는 `vision.markers`가 있으면 배열을 우선 사용한다.
- `vision.markers`가 없으면 기존 단일 `marker_id` fallback을 사용한다.

## 7. uav-onboard 설계 계획

### 7.1 추가/변경 디렉터리 구조

```text
uav-onboard/
├── config/
│   └── vision.toml
├── src/
│   ├── vision/
│   │   ├── VisionTypes.hpp
│   │   ├── ArucoDetector.hpp
│   │   ├── ArucoDetector.cpp
│   │   ├── VisionDebugPipeline.hpp
│   │   └── VisionDebugPipeline.cpp
│   ├── protocol/
│   │   ├── TelemetryMessage.hpp
│   │   └── TelemetryMessage.cpp
│   └── camera/
│       ├── CameraFrame.hpp
│       ├── RpicamMjpegSource.hpp
│       └── RpicamMjpegSource.cpp
└── tools/
    ├── aruco_detector_tester.cpp
    └── vision_debug_node.cpp
```

### 7.2 `VisionTypes.hpp`

비전 기능이 라인, 교차점, ArUco, grid 상태로 확장될 예정이므로 공통 타입을 먼저 둔다.

예상 타입:

```cpp
namespace onboard::vision {

struct Point2f {
    float x = 0.0f;
    float y = 0.0f;
};

struct MarkerObservation {
    int id = -1;
    Point2f center_px;
    std::array<Point2f, 4> corners_px;
    float orientation_deg = 0.0f;
};

struct VisionResult {
    std::uint32_t frame_seq = 0;
    std::int64_t timestamp_ms = 0;
    int width = 0;
    int height = 0;
    std::vector<MarkerObservation> markers;

    // 후속 확장용 placeholder
    bool line_detected = false;
    bool intersection_detected = false;
};

}
```

이 파일은 후속 기능의 중심 타입이 된다. 라인트레이싱, 교차점 판단, grid coordinate, marker map은 여기서 파생되는 별도 타입으로 확장한다.

### 7.3 `ArucoDetector`

역할:

- OpenCV ArUco dictionary 설정
- frame image에서 marker id와 corner 검출
- center와 image-space orientation 계산
- `VisionResult.markers` 반환

중요:

- overlay용 drawing은 절대 하지 않는다.
- `cv::aruco::drawDetectedMarkers` 같은 drawing 함수는 사용하지 않는다.
- detector는 순수하게 숫자 결과만 만든다.

필요 OpenCV component:

- `core`
- `imgproc`
- `imgcodecs`
- `aruco`
- 가능하면 `calib3d`는 GCS pose 확장용이므로 onboard MVP에는 필수로 두지 않는다.

### 7.4 `VisionDebugPipeline`

역할:

```text
RpicamMjpegSource
  -> CameraFrame(JPEG)
  -> cv::imdecode(JPEG -> Mat)
  -> ArucoDetector.detect(Mat)
  -> marker telemetry 송신
  -> 원본 JPEG video stream 송신
```

주의:

- 이번 단계에서는 단순 구현을 우선한다.
- 다만 최종 구조를 고려해 capture, detect, telemetry, video send 단계를 함수 단위로 분리한다.
- 향후에는 capture thread, vision thread, video send thread로 분리할 수 있게 한다.

### 7.5 `vision_debug_node.cpp`

새 onboard 실행 파일.

예상 option:

```bash
./build/vision_debug_node --config config
./build/vision_debug_node --config config --count 100
./build/vision_debug_node --config config --gcs-ip <laptop-ip>
./build/vision_debug_node --config config --no-video
./build/vision_debug_node --config config --no-telemetry
```

동작:

1. `NetworkConfig`, `VisionConfig` 로드
2. GCS discovery 수행
3. `RpicamMjpegSource` open
4. UDP video streamer open
5. UDP telemetry sender open
6. frame loop 시작
7. JPEG decode 후 ArUco detect
8. marker telemetry 송신
9. 원본 JPEG frame 송신

현재 `video_streamer`의 discovery/video 송신 로직과 중복이 생길 수 있다. 구현 시에는 공통 helper로 빼거나, 우선 중복을 허용하되 다음 refactor에서 정리한다.

### 7.6 `aruco_detector_tester.cpp`

기존 scaffold를 실제 도구로 바꾼다.

목적:

- Pi camera 없이 이미지 파일로 ArUco 검출 테스트
- detector 단위 테스트 전 단계의 수동 확인

예상 option:

```bash
./build/aruco_detector_tester --image test_data/images/marker.jpg
./build/aruco_detector_tester --image sample.jpg --dictionary DICT_4X4_50
```

출력 예:

```text
frame=sample.jpg markers=2
id=3 center=(312.4,188.6) orientation=0.7deg
id=8 center=(104.2,420.1) orientation=-12.3deg
```

## 8. uav-onboard config 계획

`config/vision.toml`의 `[aruco]` 섹션을 실제로 사용한다.

제안:

```toml
[aruco]
dictionary = "DICT_4X4_50"
marker_size_mm = 80.0
min_marker_perimeter_rate = 0.03
max_marker_perimeter_rate = 4.0
adaptive_thresh_win_size_min = 3
adaptive_thresh_win_size_max = 23
adaptive_thresh_win_size_step = 10
```

이번 단계에서 `marker_size_mm`는 onboard pose 계산용이 아니라 GCS pose overlay 확장을 위한 metadata로 취급한다. 온보드는 marker id/corners를 보내는 데 집중한다.

## 9. uav-gcs 설계 계획

### 9.1 추가/변경 디렉터리 구조

```text
uav-gcs/
├── src/
│   ├── app/
│   │   ├── VisionDebugApp.hpp
│   │   └── VisionDebugApp.cpp
│   ├── overlay/
│   │   ├── MarkerOverlay.hpp
│   │   └── MarkerOverlay.cpp
│   ├── telemetry/
│   │   ├── TelemetryStore.hpp
│   │   ├── TelemetryStore.cpp
│   │   ├── MarkerLogFormatter.hpp
│   │   └── MarkerLogFormatter.cpp
│   ├── ui/
│   │   ├── VideoWindow.hpp
│   │   ├── VideoWindow.cpp
│   │   ├── VideoWindowWin32.cpp
│   │   ├── MarkerLogWindow.hpp      # 이번 단계에서는 optional
│   │   └── MarkerLogWindow.cpp      # 이번 단계에서는 optional
│   ├── protocol/
│   │   ├── TelemetryMessage.hpp
│   │   └── TelemetryMessage.cpp
│   └── vision_debug_main.cpp
```

### 9.2 `VisionDebugApp`

새 GCS 앱 orchestration class.

역할:

```text
GcsDiscoveryBeacon
UdpMjpegReceiver
UdpTelemetryReceiver
TelemetryStore
MarkerOverlay
VideoWindow
MarkerLogFormatter
```

동작:

1. video UDP `5600` 수신
2. telemetry UDP `14550` 수신
3. discovery beacon 송신
4. 수신한 marker telemetry를 frame_seq 기준으로 저장
5. video frame 표시 시 같은 frame_seq의 marker overlay 조회
6. overlay를 GCS video window에 그림
7. marker log를 1-2초 간격으로 stdout 또는 log window에 출력

### 9.3 `TelemetryStore`

영상 frame과 marker telemetry를 동기화하기 위한 작은 in-memory store.

기능:

- `frame_seq -> marker observations` 저장
- 최근 N개 frame 결과만 유지
- 너무 오래된 telemetry 폐기
- video frame 표시 시 가장 적합한 marker result 반환

정책:

- 같은 `frame_seq`가 있으면 우선 사용
- 없으면 timestamp 차이가 `100-200 ms` 이내인 가장 가까운 result 사용
- 오래된 result는 overlay하지 않음

### 9.4 `MarkerOverlay`

marker telemetry를 화면에 그릴 primitive로 변환한다.

입력:

- video frame width/height
- marker corners/center/orientation
- optional pose result

출력:

- polygon line
- corner point
- center point
- direction arrow
- text label

중요:

- `MarkerOverlay`는 비전 인식 로직을 하지 않는다.
- marker detection은 onboard 결과를 그대로 사용한다.
- GCS pose estimation을 추가하더라도 이는 관제 표시용이며 onboard 미션 판단과 분리한다.

### 9.5 `VideoWindow` 확장

현재 `VideoWindow`는 frame만 표시한다. overlay를 받도록 interface를 확장한다.

예상:

```cpp
struct OverlayPrimitive;

bool showFrame(
    const video::JpegFrame& frame,
    const std::vector<OverlayPrimitive>& overlays);
```

Backend별 처리:

- OpenCV backend
  - `cv::line`, `cv::circle`, `cv::putText`로 overlay 표시
- Win32/WIC backend
  - JPEG decode 후 GDI `MoveToEx`, `LineTo`, `Ellipse`, `DrawTextW`로 overlay 표시

Windows local 환경에 OpenCV가 없으므로 Win32 backend overlay 구현은 필수다.

### 9.6 Marker log

이번 단계에서는 완전한 GUI log window보다 텍스트 출력으로 시작한다.

예상 출력 주기: 1-2초

예:

```text
[marker] frame=358 count=3
  id=7 center=(312.4,188.6) orientation=0.7deg age=21ms
  id=12 center=(108.2,420.1) orientation=-14.2deg age=21ms
  id=3 center=(501.8,244.0) orientation=88.1deg age=21ms
```

후속 단계에서 이 출력을 `MarkerLogWindow` 또는 Dear ImGui log panel로 옮긴다.

## 10. GCS pose/좌표/방향 overlay 계획

### 10.1 이번 단계 MVP

이번 단계에서는 marker의 image-space 정보를 표시한다.

표시 항목:

- marker id
- corner polygon
- corner points
- center point
- marker 방향 arrow
- center pixel coordinate
- orientation angle in image plane

예:

```text
ID: 7
Center: (312.4, 188.6) px
Angle: 0.7 deg
```

### 10.2 metric pose 확장

첨부 이미지처럼 `Pos(mm)`, `Rot(deg)`를 표시하려면 다음이 필요하다.

- camera intrinsic matrix
- distortion coefficients
- marker actual size in mm
- solvePnP 또는 equivalent pose estimation

원칙:

- 이 계산은 GCS에서 한다.
- 온보드는 overlay/pose 표시를 위한 별도 연산을 하지 않는다.
- 온보드는 필요한 raw detection data, 즉 id와 corners만 보낸다.

추가 config 후보:

```toml
[camera_calibration]
fx = 0.0
fy = 0.0
cx = 0.0
cy = 0.0
k1 = 0.0
k2 = 0.0
p1 = 0.0
p2 = 0.0
k3 = 0.0

[overlay]
show_marker_id = true
show_marker_corners = true
show_marker_center = true
show_image_pose = true
show_metric_pose = false
```

GCS에 OpenCV가 있는 환경이면 `solvePnP`를 사용할 수 있다. OpenCV가 없는 Windows fallback에서는 처음에는 image-space overlay만 제공하고, metric pose는 별도 math helper 또는 OpenCV 설치 후 활성화한다.

## 11. 최종 기능으로 확장하기 위한 구조

### 11.1 라인트레이싱 확장

`VisionTypes.hpp`에 다음 타입을 추가할 수 있게 한다.

```cpp
struct LineDetection {
    bool detected;
    float center_offset_px;
    float angle_deg;
    float confidence;
    std::vector<Point2f> contour_px;
};
```

Onboard:

- line detection 수행
- line offset/angle/contour metadata 송신

GCS:

- line contour와 중심선을 overlay로 그림

### 11.2 교차점 판단 확장

```cpp
struct IntersectionDetection {
    bool detected;
    float score;
    Point2f center_px;
    std::string state; // entering, inside, leaving 등
};
```

Onboard:

- 교차점 판단
- hysteresis/cooldown
- grid 좌표 갱신 trigger 생성

GCS:

- 교차점 후보/확정 영역 overlay
- event log 출력

### 11.3 marker map 저장 확장

```cpp
struct MarkerRecord {
    int id;
    int row;
    int col;
    std::int64_t first_seen_timestamp_ms;
};
```

Onboard:

- marker id를 현재 grid coordinate에 저장
- 중복 저장 방지

GCS:

- marker list/log/map 표시

### 11.4 현재 드론 위치 저장 확장

```cpp
struct GridState {
    int row;
    int col;
    float heading_deg;
    std::vector<GridCoord> visited;
};
```

Onboard:

- 교차점 event와 heading 기반으로 grid coordinate 업데이트

GCS:

- 현재 위치와 방문 이력 표시

### 11.5 command window 확장

최종 GCS command window는 다음 기능을 담당한다.

- mission start
- abort mission
- emergency land
- reset vision state
- set initial grid coordinate
- set heading
- pause/resume vision debug

현재 단계에서는 command window를 구현하지 않는다. 다만 GCS 앱 구조는 video/log/command window를 같은 process에서 소유할 수 있게 설계한다.

## 12. 구현 단계 계획

### Phase A: schema와 타입 정의

1. `uav-onboard/src/vision/VisionTypes.hpp` 추가
2. telemetry schema에 `vision.markers[]` 정의
3. `uav-onboard/src/protocol/TelemetryMessage.*`에 marker array build 계획 반영
4. `uav-gcs/src/protocol/TelemetryMessage.*`에 marker array parse 계획 반영
5. `docs/PROTOCOL.md`는 구현 후 양쪽 repo에 동기화

성공 기준:

- marker가 0개, 1개, 여러 개인 JSON을 표현할 수 있다.
- 기존 telemetry parser가 깨지지 않는다.

### Phase B: onboard ArUco detector

1. `ArucoDetector` 구현
2. `aruco_detector_tester`를 image 기반 검출 도구로 구현
3. Pi 또는 로컬 OpenCV 환경에서 sample marker image로 id/corner 출력 확인
4. detector가 drawing을 하지 않는지 확인

성공 기준:

- 이미지 파일에서 marker id와 corner 좌표가 출력된다.
- 검출 결과가 `VisionResult`로 정리된다.

### Phase C: onboard live debug node

1. `vision_debug_node.cpp` 추가
2. `RpicamMjpegSource`에서 JPEG frame 획득
3. JPEG decode 후 ArUco detect
4. marker telemetry 송신
5. 원본 JPEG video stream 송신
6. `frame_seq`를 video frame id와 telemetry camera frame_seq에 동일하게 넣음

성공 기준:

- Pi에서 `vision_debug_node --config config` 실행 시 GCS가 video와 marker telemetry를 동시에 받는다.
- marker가 화면에 없을 때도 marker_count=0 telemetry가 안정적으로 간다.

### Phase D: GCS marker telemetry/log

1. `TelemetryMessage` parser에 `vision.markers[]` 추가
2. `TelemetryStore` 구현
3. `MarkerLogFormatter` 구현
4. `uav_gcs_vision_debug`에서 telemetry 수신 thread 또는 non-blocking loop 추가
5. marker log를 1-2초마다 stdout에 출력

성공 기준:

- GCS console/log에서 현재 보이는 marker id와 center 좌표가 표시된다.
- packet loss가 있어도 앱이 멈추지 않는다.

### Phase E: GCS video overlay

1. `OverlayPrimitive`와 `MarkerOverlay` 구현
2. `VideoWindow` interface를 overlay 입력 가능하게 확장
3. Win32/WIC backend에서 polygon/text/point/arrow drawing 구현
4. OpenCV backend에서도 동일 overlay drawing 구현
5. video frame과 marker telemetry를 `frame_seq` 기준으로 동기화

성공 기준:

- GCS 영상 창에 marker bounding box가 표시된다.
- marker id, center 좌표, orientation이 표시된다.
- overlay는 GCS에서만 그려진다.
- 온보드 송신 영상은 overlay 없는 원본 JPEG다.

### Phase F: 실제 Pi camera 통합 테스트

1. GCS에서 `uav_gcs_vision_debug --config config` 실행
2. Pi에서 `vision_debug_node --config config` 실행
3. ArUco marker 여러 개를 카메라에 보여줌
4. GCS 영상 창 overlay와 marker log 확인
5. marker가 빠르게 움직일 때 frame/telemetry mismatch 여부 확인

성공 기준:

- marker 1개 이상에서 id와 box가 안정적으로 표시된다.
- marker 여러 개가 동시에 보이면 모두 표시된다.
- 영상이 끊기더라도 GCS는 마지막 frame 또는 최신 complete frame 정책을 유지한다.

## 13. 테스트 계획

### 13.1 단위 테스트 후보

```text
uav-onboard/tests/
├── test_aruco_detector.cpp
├── test_telemetry_marker_json.cpp
└── test_vision_types.cpp

uav-gcs/tests/
├── test_marker_telemetry_parse.cpp
├── test_telemetry_store.cpp
├── test_marker_overlay_mapping.cpp
└── test_video_frame_marker_sync.cpp
```

### 13.2 수동 테스트

1. 단일 marker 이미지 검출
2. 여러 marker 이미지 검출
3. marker 없음 상태
4. Pi camera live marker 검출
5. GCS overlay 표시
6. marker log 주기 출력
7. video frame drop 상황에서 overlay stale 처리

### 13.3 성능 측정 항목

온보드:

- frame capture FPS
- ArUco detection latency
- telemetry send rate
- video send FPS
- CPU 사용률

GCS:

- video frame receive FPS
- overlay drawing latency
- video-to-marker telemetry age
- dropped video frame 수
- dropped telemetry seq 수

## 14. 리스크와 대응

| 리스크 | 원인 | 대응 |
|---|---|---|
| Pi에서 OpenCV ArUco module이 없음 | OpenCV 패키지 구성 차이 | CMake에서 `aruco` component 확인, 없으면 dependency 안내 |
| JPEG decode가 온보드 CPU에 부담 | MJPEG를 다시 decode해야 detection 가능 | 640x480 10-15 FPS로 시작, 필요 시 FPS/해상도 조정 |
| video와 telemetry frame이 어긋남 | UDP 채널 분리 | `frame_seq`, timestamp 동기화, stale threshold 적용 |
| marker pose metric 표시가 부정확 | calibration 없음 | MVP는 image-space overlay, metric pose는 calibration 이후 |
| GCS Windows OpenCV 없음 | 현재 로컬 환경 | Win32/GDI overlay를 필수 backend로 구현 |
| telemetry/log와 video process 분리 문제 | 같은 port/state 공유 어려움 | 장기적으로 단일 GCS app이 여러 창을 소유 |
| 온보드가 관제용 계산까지 떠안음 | overlay 요구가 늘어남 | 온보드는 id/corners/mission 상태만 송신, 표시 계산은 GCS |

## 15. 완료 기준

이번 단계가 완료됐다고 보는 기준:

1. Pi에서 ArUco marker id와 corner가 검출된다.
2. 검출 결과가 telemetry JSON으로 GCS에 전달된다.
3. GCS는 marker 정보를 텍스트 로그로 주기 표시한다.
4. GCS 영상 창은 marker box/id/좌표/방향 overlay를 표시한다.
5. 온보드 영상 stream은 overlay 없는 원본 frame이다.
6. overlay drawing과 GUI 처리는 모두 GCS에서만 수행된다.
7. `frame_seq` 기반으로 video와 marker telemetry가 동기화된다.
8. 이후 라인트레이싱, 교차점 판단, marker map, grid position, command window로 확장 가능한 구조가 준비된다.

## 16. 이번 계획에서 고의로 미루는 것

- Dear ImGui 전체 도입
- command/control window 구현
- metric `Pos(mm)`, `Rot(deg)`의 정확한 pose estimation
- camera calibration tool
- line tracing
- intersection state machine
- marker map 영구 저장
- Pixhawk/MAVLink 연동

이 항목들은 ArUco detection과 GCS overlay가 안정화된 뒤 순서대로 붙인다.
