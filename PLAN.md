# Astroquad Onboard 교차점 분류 및 디버그 영상 FPS 계획

최종 업데이트: 2026-04-30

범위: `development-log/RESEARCH.md`, `uav-onboard/docs/PROTOCOL.md`, `uav-onboard/PROJECT_SPEC.md`, `uav-onboard/README.md`와 `uav-onboard` 전체 소스 코드를 읽고 정리한 다음 작업 계획이다. 기존 `PLAN.md` 내용은 사용하지 않는다.

이미지 근거: 사용자가 2026-04-30 캡처 8장을 추가로 제공했다. 모두 GCS overlay가 켜진 화면이며, 밝은 흰색 계열 라인과 바닥/천 배경 위에서 magenta contour, green tracking point/text가 표시되어 있다. 아래 교차점 불안정 원인과 config 후보는 이 이미지 관찰을 반영해 수정한다.

## 1. 현재 시스템 이해

### 1.1 전체 시스템 아키텍처

현재 `uav-onboard`는 최종 미션 전체 중 camera capture, onboard vision, telemetry, optional debug video bring-up 단계까지 구현되어 있다.

현재 구현된 runtime 흐름:

```text
IMX519 camera
  -> rpicam-vid --codec mjpeg -o -
  -> RpicamMjpegSource
  -> CameraFrame(JPEG, frame_id, timestamp)
  -> cv::imdecode() once per frame
  -> ArucoDetector and/or LineDetector
  -> LineStabilizer
  -> protocol::BringupTelemetry
  -> UDP JSON telemetry
  -> optional latest-frame UDP MJPEG debug video
  -> GCS telemetry/video receive
  -> GCS-side overlay/log
```

목표 아키텍처는 아직 미구현인 Mission, Grid, Control, MAVLink, Safety가 뒤에 붙는 구조다.

```text
Camera -> Vision -> Mission State Machine -> Control -> Autopilot/MAVLink -> Pixhawk
Support: Telemetry -> GCS, Command <- GCS, Safety, Logging
```

중요 설계 원칙:

- Onboard는 marker/line/intersection 같은 미션 판단용 값을 계산한다.
- GCS는 수신, 표시, overlay drawing, log window를 담당한다.
- Onboard는 영상 위에 contour/text/label을 직접 그리지 않는다.
- Debug video는 best-effort이고 mission-critical loop를 막으면 안 된다.
- GCS 없이 onboard 단독 실행이 가능해야 하므로 교차점 판단 자체는 onboard에서 수행해야 한다.

### 1.2 Vision pipeline 구조

핵심 파일:

- `src/app/VisionDebugPipeline.cpp`: 현재 통합 vision runtime.
- `src/vision/ArucoDetector.*`: OpenCV ArUco marker detection.
- `src/vision/LineDetector.*`: line segmentation, contour scoring, tracking point 계산.
- `src/vision/LineStabilizer.*`: EMA smoothing, jump rejection, hold/reacquire.
- `src/vision/VisionTypes.hpp`: vision 내부 결과 구조.

현재 `LineDetector` 동작 요약:

1. 원본 이미지에서 `roi_top_ratio`만큼 상단을 제외한다.
2. ROI를 `process_width`로 resize한다. 기본은 960x720 입력에서 work image 약 480x331이다.
3. grayscale 변환 후 `mask_strategy`에 따라 threshold mask를 만든다.
4. `mode = auto`면 bright-line/dark-line mask를 모두 만들고, 고정 모드면 한쪽만 만든다.
5. morphology open/close/dilate를 적용한다.
6. `cv::findContours(..., RETR_EXTERNAL, ...)`로 외곽 contour를 찾는다.
7. area 기준 top `max_candidates`개만 평가한다.
8. `lookahead_y_ratio` 부근의 수평 band에서 foreground projection run을 찾는다.
9. contour score가 `confidence_min` 이상이면 tracking point, offset, angle, confidence, simplified contour를 반환한다.
10. `LineStabilizer`가 raw detection을 smoothing/hold/jump reject한다.

현재 line detector는 "라인 추종용 tracking point 하나"를 찾는 모듈이지, branch topology를 해석하는 모듈이 아니다.

### 1.3 Telemetry / video 전송 구조

Telemetry:

- `src/protocol/TelemetryMessage.*`가 `BringupTelemetry`를 JSON으로 직렬화한다.
- `src/network/UdpTelemetrySender.*`가 UDP datagram으로 GCS에 보낸다.
- `VisionDebugPipeline`에서는 processed camera frame마다 telemetry를 한 번 보낸다.
- `camera.frame_seq`는 raw MJPEG video packet의 `frame_id`와 맞춰 GCS에서 overlay를 동기화한다.

Debug video:

- `src/video/UdpMjpegStreamer.*`가 JPEG frame을 `AQV1` chunk packet으로 쪼개 UDP 송신한다.
- `VisionDebugPipeline` 내부 `LatestVideoSender`는 worker thread에서 최신 frame만 보낸다.
- worker가 밀리면 이전 frame은 drop되고 최신 frame으로 대체된다.
- 기본은 `debug_video.enabled = false`라 video off다.
- `--video`로 켜면 기본 `debug_video.send_fps = 5`로 decimation한다.
- camera capture/vision processing은 12 FPS 기본이고, GCS video 송신만 5 FPS로 제한된다.

### 1.4 Config 적용 방식

주요 config:

- `config/network.toml`
  - `gcs.ip`, `telemetry_port`, `command_port`, `video_port`
  - `telemetry.send_interval_ms`
- `config/vision.toml`
  - `[camera]`: IMX519/rpicam capture 설정
  - `[debug_video]`: optional GCS MJPEG debug stream 설정
  - `[line]`: line segmentation, contour scoring, stabilizer 설정
  - `[aruco]`: ArUco detector 설정

적용 방식:

- `loadNetworkConfig(config_dir)`와 `loadVisionConfig(config_dir)`가 TOML을 읽고, 실패 시 구조체 기본값을 사용한다.
- `tools/vision_debug_node.cpp`에서 CLI override를 config 구조체에 덮어쓴 뒤 `VisionDebugPipelineOptions`로 넘긴다.
- 현재 `vision_debug_node`에는 camera FPS용 `--camera-fps`가 있지만, GCS debug video 송신 FPS를 직접 바꾸는 CLI 옵션은 없다.

### 1.5 코드 스타일 및 모듈 분리 방식

현재 스타일:

- C++17, namespace는 `onboard::{common,camera,vision,protocol,network,video,app}`로 분리한다.
- 헤더는 public data structure/class 선언 중심, 구현 세부 helper는 `.cpp` anonymous namespace에 둔다.
- config는 `common`에 두고, detector는 config 구조체를 생성자에서 받아 내부에 보관한다.
- protocol 구조체는 vision 내부 타입과 분리되어 있고, `VisionDebugPipeline.cpp`에서 변환한다.
- 테스트는 assert 기반의 작은 executable을 `ctest`에 등록한다.
- OpenCV 의존 vision code는 `BUILD_TOOLS`와 `OpenCV_FOUND` 조건 아래 `onboard_vision` library에 묶인다.

이 계획에서도 같은 스타일을 유지한다.

## 2. 현재 LineDetector가 교차점에서 불안정한 이유

### 2.0 이미지 관찰 요약

추가 이미지 8장에서 공통으로 보이는 현상:

- 흰색/밝은 라인은 바닥과 충분히 대비된다. 현재 캡처 세트만 놓고 보면 `light_on_dark` 극성은 맞다.
- 교차점 전체가 하나의 안정적인 magenta contour로 잡히지 않는다.
- frame 2619, 5559에서는 아래쪽 세로 branch가 선택되고, frame 3195에서는 위쪽 세로 branch가 선택된다.
- frame 3939에서는 왼쪽 가로 branch, frame 4914/5256에서는 오른쪽 가로 branch가 선택된다.
- frame 12234/12540에서는 어두운 천/그림자 위의 흰 라인이 강하게 잡히지만, contour가 여러 branch를 잇는 듯하면서도 중심부/외곽이 비정상적으로 휘거나 겹친다.
- green tracking point가 교차점 중심 근처 또는 한쪽 branch 위에 머물고, text의 angle이 `-85`, `-80`, `6.9`, `-1`, `-73`, `8.6`처럼 branch 선택에 따라 크게 바뀐다.
- 바닥에는 작은 색상 speckle/짧은 edge fragment가 많지만, 선택된 contour 자체는 대부분 흰 라인 branch를 따라간다. 따라서 1차 원인은 노이즈가 아니라 branch 선택/분리 문제다.

### 2.1 contour 끊김 여부

이미지 기준으로는 교차점 중심에서 contour가 사실상 끊겨 있다. 흰 라인은 실제로는 `+` 또는 `T`에 가깝게 연결되어 보이지만, detector가 보내는 selected contour는 한 번에 한 branch만 잡는 경우가 많다.

구체적 판단:

- frame 2619: 하단 세로 라인만 길게 잡히고, 상단/좌우 branch는 같은 contour로 포함되지 않는다.
- frame 3195: 교차점 위쪽의 짧은 세로 branch만 잡힌다.
- frame 3939: 왼쪽 가로 branch가 잡히며, 교차점 중심을 지나 오른쪽 branch까지 이어지지 않는다.
- frame 4914/5256: 오른쪽 가로 branch만 잡힌다.
- frame 5559: 하단 세로 branch가 잡히고 가로 branch는 분리된다.

이 패턴은 `local_contrast_blur = 31`이 현재 라인 폭 대비 작아 흰 테이프의 내부 전체를 채우기보다 edge/부분 contrast 위주 mask를 만들고, `morph_close_kernel = 7`만으로 교차점 중심부와 branch 사이를 충분히 메우지 못하는 상황과 일치한다. 즉 교차점의 물리적 라인은 연결되어 있어도 mask/contour 단계에서 branch가 분리된다.

### 2.2 branch 분리 여부

branch 분리는 "알고리즘이 의도적으로 분리한 것"이 아니라 "mask/contour/scoring 결과가 매 프레임 한 branch를 선택하는 것"에 가깝다.

현재 `LineDetector`는 branch 개념이 없다.

- `cv::findContours(..., RETR_EXTERNAL, ...)`는 connected component 외곽만 본다.
- `contour_px`는 centerline/skeleton이 아니라 외곽선이다.
- `lineAngleDeg()`는 선택된 contour 하나에 `cv::fitLine()`을 적용한다.
- `evaluateContour()`는 `lookahead_y_ratio` 부근 band에서 가장 line-like한 projection run을 고른다.

따라서 교차점에서는 현재 바라보는 격자 타입이 아니라, lookahead band와 scoring에 가장 잘 맞는 branch 하나가 line으로 보고된다. 이미지에서 angle과 offset이 branch 방향에 따라 바뀌는 것이 이 증거다.

결론: 불안정의 핵심은 `LineDetector`가 "교차점 전체 topology"를 볼 수 없는 구조라는 점이다. line following contour와 intersection classification은 반드시 분리해야 한다.

### 2.3 노이즈 영향

노이즈는 존재하지만 1차 원인은 아니다.

이미지 전반에 바닥 목재 무늬, JPEG/조명 영향, 작은 색상 fragment가 보인다. 그러나 선택된 magenta contour는 대부분 작은 speckle이 아니라 흰 라인 branch 위에 있다. confidence도 대체로 `0.5~0.9` 수준이므로 detector가 완전히 엉뚱한 바닥 노이즈를 고르는 상황은 아니다.

노이즈가 실제로 문제가 되는 경우:

- frame 12234/12540처럼 어두운 천/그림자/장애물이 ROI 안에 들어와 흰 라인 대비가 과도하게 커지는 경우
- 바닥 무늬 fragment가 morphology close 후 라인 edge에 붙는 경우
- `line_contours_found`가 크게 늘고, 실제 branch가 아닌 큰 blob이 후보에 들어오는 경우

따라서 `local_contrast_threshold`를 무작정 올리기보다, 먼저 교차점 mask가 branch별로 끊기는 문제를 `local_contrast_blur`와 close 쪽에서 확인하는 것이 우선이다.

### 2.4 ROI 설정 적절성

`roi_top_ratio = 0.08` 자체는 현재 이미지에서 큰 문제로 보이지 않는다. 교차점과 라인은 대부분 ROI 안에 들어와 있고, 상단 8%만 제외해도 바닥 영역은 충분히 확보된다.

더 직접적인 문제는 `lookahead_y_ratio = 0.55`가 교차점 중심이나 가로 branch를 자주 관통한다는 점이다.

- green tracking point가 교차점 중심 또는 한쪽 branch 위에 잡히면서 선택 branch가 바뀐다.
- line following 용도에서는 교차점 중심보다 아래쪽, 즉 드론에 가까운 incoming trunk를 잡는 편이 더 안정적이다.
- 하지만 lookahead를 아래로 내리는 것은 추종 point 안정화에는 도움이 되어도 `+`, `T`, `L` 타입 분류를 해결하지 못한다.

따라서 ROI는 유지하고, line following lookahead와 intersection classification 영역을 분리한다.

## 3. 의미 있는 config 수정 후보만 제안

이미지 기반으로 실제 수정 의미가 있는 값만 제안한다. 아래 값 외에는 교차점 분류 문제를 우회적으로만 건드리므로 우선 변경하지 않는다.

| 항목 | 제안 | 적용 조건 | 이유 |
|---|---|---|---|
| `line.mode` | 현재 캡처 기준 `light_on_dark` 유지. 라인 극성이 미확정인 새 환경을 시험할 때만 `auto` 사용 | 이번 이미지처럼 밝은 흰 라인일 때 | 현재 문제는 극성 오판이 아니다. `auto`는 dark-line 후보까지 봐서 노이즈 후보가 늘 수 있으므로 실전 기본값으로 바로 바꾸지 않는다. |
| `line.local_contrast_blur` | `31`에서 `51` 또는 `61` 후보 비교 | 흰 테이프 내부가 채워지지 않고 edge/branch 단위로 끊길 때 | 현재 이미지의 branch 단절은 local contrast blur가 라인 폭 대비 작아 생긴 edge 중심 mask 가능성이 높다. 이 값이 가장 의미 있는 1차 후보다. |
| `line.morph_close_kernel` | `7`에서 `9` 또는 `11` 후보 비교 | `local_contrast_blur` 조정 후에도 교차점 중심 gap이 남을 때 | branch 사이 작은 gap을 메운다. 너무 키우면 바닥 speckle까지 붙을 수 있으므로 blur 후보 다음에 본다. |
| `line.lookahead_y_ratio` | 추종 안정화 목적으로 `0.62~0.72` 후보 비교 | green point가 교차점 중심/가로 branch에 계속 올라탈 때 | line following point를 교차점 중심에서 아래쪽 incoming trunk로 이동시킨다. 교차점 타입 분류 해결책은 아니다. |
| `line.lookahead_band_ratio` | 필요 시 `0.06`에서 `0.04` 후보 비교 | lookahead band가 가로 branch를 넓게 포함해 tracking x가 옆으로 튈 때 | band 높이를 줄여 branch 섞임을 줄인다. 너무 줄이면 한 프레임 노이즈에 약해진다. |
| `line.local_contrast_threshold` | 기본 `10` 유지, 노이즈가 실제 후보로 선택될 때만 `12~16` 비교 | `line_contours_found`가 과도하고, selected contour가 라인이 아닌 바닥 blob일 때 | 현재 이미지는 노이즈보다 branch 분리가 주원인이다. threshold 상향은 recall 손실이 있으므로 후순위다. |
| `line.roi_top_ratio` | `0.08` 유지 | 상단에 실제 바닥이 아닌 물체가 지속적으로 들어올 때만 상향 | 이번 이미지에서 ROI보다 lookahead/branch 문제가 더 크다. |

변경하지 말아야 할 후보:

- `min_area_px`: branch 분류 문제를 area threshold로 해결하면 짧은 실제 branch를 놓칠 수 있다.
- `max_candidates`: 후보 수를 늘려도 topology 분류가 생기지 않는다.
- `confidence_min`: 현재 선택 branch confidence가 이미 충분하므로 원인 해결이 아니다.
- `max_line_width_ratio`: 교차점 blob을 억지로 line으로 받거나 버리는 방향으로 치우치며, 타입 분류를 대신하지 못한다.
- `filter_*`: stabilizer는 branch 선택이 튄 뒤의 결과를 가릴 뿐, branch 존재 여부를 판단하지 못한다.
- `process_width`: 현재 이미지에서 라인 픽셀은 충분하다. 우선순위는 mask 연결과 교차점 분류다.

## 4. 교차점 알고리즘 선택

검토 대상:

1. Skeleton 기반 topology 방식
2. Circular Profiling / Ray Casting
3. Hough Transform

최종 추천: **Circular Profiling / Ray Casting 기반 분류**를 1차 구현으로 선택한다.

### 4.1 선택 이유

현재 프로젝트 조건에 잘 맞는 점:

- 드론은 항상 정면을 보고 이동할 예정이다.
- 경기장은 격자 구조이고, 1차 분류 타입은 `+`, `T`, `L`, `straight`로 제한된다.
- 현재 `LineDetector`가 이미 binary mask를 만드는 데 필요한 config와 morphology 전략을 갖고 있다.
- GCS debug overlay에 branch ray, score, 타입을 바로 표시할 수 있어 튜닝이 쉽다.
- 계산량이 작고 OpenCV 기본 기능만으로 구현 가능하다.
- line following contour와 intersection classification을 명확히 분리할 수 있다.

### 4.2 Skeleton 방식을 1차로 선택하지 않는 이유

Skeleton/topology 방식은 이론적으로 branch count 판단에 가장 직접적이다. 하지만 1차 구현에는 리스크가 크다.

- OpenCV `ximgproc::thinning` 사용 가능 여부가 빌드 환경마다 달라질 수 있다.
- 직접 Zhang-Suen thinning을 구현하면 새 알고리즘 코드가 커지고 테스트 범위가 늘어난다.
- 현재 mask가 교차점에서 조금만 끊기면 skeleton graph가 여러 component로 나뉜다.
- 바닥 노이즈나 morphology spur가 skeleton endpoint/branchpoint로 잘못 잡힐 수 있다.
- 안정적인 graph를 얻으려면 gap closing, spur pruning, branch length filtering, node merge가 추가로 필요하다.

따라서 skeleton은 2차 개선 후보로 남긴다. Ray casting 결과가 부족할 때, 같은 mask 위에서 skeleton graph를 추가 검증기로 붙이는 것이 더 안전하다.

### 4.3 Hough Transform을 선택하지 않는 이유

Hough는 line segment 검출에는 강하지만 현재 문제의 1차 해법으로는 과하다.

- line width, blur, threshold 상태에 따라 하나의 branch가 여러 segment로 쪼개질 수 있다.
- 교차점 중심과 branch 존재 여부를 다시 후처리로 합쳐야 한다.
- parameter가 많고 실제 바닥 노이즈에 민감하다.
- 현재 `LineDetector` mask/contour 기반 pipeline과 결과 공유가 약하다.
- telemetry overlay로 설명하기에는 ray score보다 해석성이 떨어진다.

Hough는 나중에 "길게 뻗은 branch 방향 검증" 보조 feature로는 사용할 수 있지만, 1차 classifier의 중심 알고리즘으로 두지 않는다.

## 5. 최종 알고리즘 흐름

### 5.1 입력

입력은 decoded BGR frame과 현재 line config다.

선택 입력:

- raw `LineDetection`: 현재 선택된 branch와 confidence를 참고하되, 교차점 중심으로 직접 채택하지 않는다.
- image size: source/work 좌표 변환.

라인 극성 미확정 대응:

- 이번 캡처처럼 밝은 흰 라인에서는 `line.mode = "light_on_dark"`를 유지한다.
- 라인 극성이 아직 정해지지 않은 새 재질/환경을 시험할 때만 `line.mode = "auto"`로 bright-line/dark-line mask를 모두 평가한다.
- 실제 라인 극성이 확정되면 고정 mode로 바꿔 처리량과 false positive를 줄인다.

### 5.2 공통 segmentation

`LineDetector.cpp`에 private로 들어 있는 ROI/resize/local contrast/morphology helper를 공용 모듈로 분리한다.

새 모듈 후보:

- `uav-onboard/src/vision/LineMaskBuilder.hpp`
- `uav-onboard/src/vision/LineMaskBuilder.cpp`

역할:

1. ROI geometry 계산.
2. source ROI를 work size로 resize.
3. grayscale 변환.
4. light/dark polarity별 mask 생성.
5. morphology 적용.
6. source/work 좌표 변환 helper 제공.

이렇게 하면 `LineDetector`와 `IntersectionDetector`가 같은 mask semantics를 공유한다.

추가 이미지 기준 보완점:

- 추종용 mask와 교차점 판정용 mask를 같은 원본에서 만들되, 교차점 판정에서는 branch 단절을 줄이기 위해 더 큰 close 또는 filled mask 변형을 내부적으로 사용할 수 있게 한다.
- 이 변형은 `LineDetector`의 tracking point behavior를 바로 바꾸지 않고, `IntersectionDetector` 내부 입력으로만 사용한다.
- config를 늘리기보다 1차 구현에서는 `local_contrast_blur`, `morph_close_kernel`, 라인 폭 추정치에서 파생한 내부 kernel을 사용한다.

### 5.3 중심 후보 선택

교차점 중심 후보는 selected line contour의 tracking point에만 의존하면 안 된다. 추가 이미지에서 green tracking point는 왼쪽/오른쪽/상단/하단 branch 위로 바뀌며, 실제 교차점 중심과 떨어지는 경우가 반복된다.

중심 후보 선택 원칙:

1. 기본 search window는 image center 주변 또는 frame 하단에서 들어오는 expected grid center 주변으로 둔다.
2. `raw_tracking_point_px`는 search window를 보조로 넓히는 힌트로만 사용하고, 중심 자체로 바로 채택하지 않는다.
3. 교차점 판정용 combined/fill mask에서 largest blob centroid를 1차 중심 후보로 추가한다.
4. work mask의 foreground density, 4방향 ray score 합, branch count, 중심 주변 채움 정도를 함께 점수화한다.
5. 후보 중심이 selected contour 바깥이어도, combined/fill mask에서 branch ray가 충분하면 교차점 중심으로 인정한다.

largest blob centroid 사용 방식:

- `cv::connectedComponentsWithStats()`로 교차점 판정용 mask의 connected component를 찾는다.
- 전체 frame의 largest blob을 무조건 쓰지 않고, search window와 겹치거나 image center에 가까운 blob 중 가장 큰 component를 우선한다.
- component centroid는 `center_seed_px`로 사용하고, 최종 center는 주변 grid search와 ray score로 다시 검증한다.
- largest blob이 긴 branch 하나인 경우 centroid가 교차점 중심에서 밀릴 수 있으므로, branch count가 2 미만이거나 ray score가 낮으면 최종 center로 확정하지 않는다.

search window:

- image center 기준 너비 35~45%, 높이 35~45% 정도를 1차 후보 영역으로 둔다.
- raw tracking point가 center window 밖으로 크게 벗어나면 center window와 raw point 주변 window를 모두 본다.
- largest blob centroid 주변도 후보 window로 추가한다.
- 후보마다 4방향 branch score를 계산한다.
- branch score 합, branch count, center density, image-center 거리 penalty로 최적 중심을 고른다.

center validation:

- 중심 주변 작은 disk/box에 foreground 밀도가 있어야 한다.
- 중심이 약간 비어 있어도 4방향 inner ray에 foreground가 충분하면 허용한다. 추가 이미지처럼 local contrast mask가 테이프 내부를 비울 수 있기 때문이다.
- score가 너무 낮으면 `Unknown` 또는 `None` 처리한다.

### 5.4 Ray casting / circular profiling

방향 정의:

- `Front`: 이미지 위쪽, 드론 진행 방향
- `Back`: 이미지 아래쪽
- `Left`: 이미지 왼쪽
- `Right`: 이미지 오른쪽

각 방향마다 중심에서 바깥쪽으로 ray strip을 샘플링한다.

샘플링 방식:

- inner radius는 중심 blob 자체를 피하기 위해 작게 둔다.
- outer radius는 frame 크기와 ROI 크기에서 계산한다.
- ray 주변 ±strip_width 픽셀을 같이 본다.
- foreground occupancy, 연속 run 길이, endpoint 도달 여부를 branch score로 합친다.

결과:

- `BranchObservation{direction, present, score, endpoint_px, angle_deg}`
- present threshold는 초기에는 코드 상수 또는 `line.intersection_threshold`에서 파생한다.
- 불필요한 config 증식을 막기 위해 1차 구현에서는 새 config를 많이 만들지 않는다.

### 5.5 타입 분류

branch presence bitmask로 분류한다.

```text
Front + Back + Left + Right -> "+"
3 branches                   -> "T"
2 adjacent branches           -> "L"
Front + Back                  -> "straight"
Left + Right                  -> "straight_sideways" 또는 "straight" with branch mask
그 외                         -> "unknown" 또는 "none"
```

`L`과 `straight`는 bitmask만으로 결정하지 않고 angle 기반 검증을 추가한다.

- present branch가 2개일 때 center에서 각 branch endpoint로 향하는 vector angle을 계산한다.
- 두 branch의 angle difference가 `150~180deg` 범위면 `straight`로 판정한다.
- angle difference가 `60~120deg` 범위면 `L`로 판정한다.
- 그 사이 애매한 값이면 `unknown`으로 낮추거나, temporal stabilizer가 이전 안정 타입을 hold하게 한다.
- 이때 사용하는 angle은 selected contour의 `fitLine` angle이 아니라 branch ray endpoint 기반 angle이다. 이미지에서 selected contour angle이 branch 선택에 따라 크게 바뀌므로 `fitLine` angle만으로 `L`/`straight`를 구분하면 안 된다.
- branch bitmask와 angle 판정이 충돌하면 angle 판정을 우선하되, score가 낮은 branch가 섞인 경우에는 `unknown`으로 처리한다.

Telemetry 표기는 사용자가 요청한 네 타입을 우선한다.

- `+`
- `T`
- `L`
- `straight`

내부 enum에는 `None`, `Unknown`을 추가해 invalid/low-confidence 상태를 표현한다.

기존 `vision.intersection_detected` 의미:

- `+`, `T`, `L`이면 true
- `straight`이면 false 또는 별도 `vision.intersection.type = "straight"`로 표현
- GCS log에는 straight도 표시한다.

### 5.6 출력

Vision 내부 출력:

```cpp
enum class IntersectionType {
    None,
    Unknown,
    Straight,
    L,
    T,
    Cross
};

enum class BranchDirection {
    Front,
    Right,
    Back,
    Left
};

struct BranchObservation {
    BranchDirection direction;
    bool present = false;
    float score = 0.0f;
    Point2f endpoint_px;
    float angle_deg = 0.0f;
};

struct IntersectionDetection {
    bool valid = false;
    bool intersection_detected = false;
    IntersectionType type = IntersectionType::None;
    IntersectionType raw_type = IntersectionType::None;
    bool stable = false;
    bool held = false;
    Point2f center_px;
    Point2f raw_center_px;
    float score = 0.0f;
    float raw_score = 0.0f;
    std::array<BranchObservation, 4> branches {};
    std::uint8_t branch_mask = 0;
    int branch_count = 0;
    int stable_frames = 0;
    float radius_px = 0.0f;
    int selected_mask_index = -1;
};
```

Protocol/GCS 출력은 enum 대신 string을 사용한다.

```json
"vision": {
  "intersection_detected": true,
  "intersection_score": 0.86,
  "intersection": {
    "valid": true,
    "type": "T",
    "raw_type": "T",
    "stable": true,
    "held": false,
    "center_px": {"x": 481.0, "y": 356.0},
    "raw_center_px": {"x": 479.0, "y": 358.0},
    "score": 0.86,
    "raw_score": 0.82,
    "branch_mask": 11,
    "branch_count": 3,
    "stable_frames": 4,
    "branches": [
      {"direction": "front", "present": true, "score": 0.92, "endpoint_px": {"x": 481.0, "y": 238.0}},
      {"direction": "right", "present": true, "score": 0.81, "endpoint_px": {"x": 604.0, "y": 356.0}},
      {"direction": "back", "present": false, "score": 0.12, "endpoint_px": {"x": 481.0, "y": 474.0}},
      {"direction": "left", "present": true, "score": 0.84, "endpoint_px": {"x": 358.0, "y": 356.0}}
    ]
  }
}
```

## 6. 시간복잡도 및 성능 고려

기본 frame은 960x720이고 line processing은 work image 약 480x331이다. work pixel 수 `N`은 약 159k다.

주요 비용:

- ROI resize: `O(N)`
- grayscale/local contrast/threshold: `O(N)` plus Gaussian blur 비용
- morphology: `O(N * kernel_area)`이지만 OpenCV 최적화 사용
- contour scan: `O(N)`
- ray casting: 방향 4개, 중심 후보 수 제한, 반경 제한이므로 실질적으로 `O(1)`에 가까운 작은 비용

성능 원칙:

- `LineDetector`와 `IntersectionDetector`가 mask 생성을 중복하지 않게 `LineMaskBuilder`를 공유한다.
- `line.mode = auto`는 mask가 2개라 비용이 증가한다. polarity 확정 후 고정 모드로 바꾼다.
- `debug.intersection_latency_ms`를 추가해 Pi 4에서 실제 비용을 본다.
- 목표는 기존 12 FPS vision processing을 유지하는 것이다.
- intersection latency가 2~3ms를 지속적으로 넘으면 후보 center search density를 줄이고, mask 재사용을 우선 최적화한다.

## 7. 새 모듈 설계

### 7.1 Onboard 파일 위치

추가/수정 후보:

- `uav-onboard/src/vision/LineMaskBuilder.hpp`
- `uav-onboard/src/vision/LineMaskBuilder.cpp`
- `uav-onboard/src/vision/IntersectionDetector.hpp`
- `uav-onboard/src/vision/IntersectionDetector.cpp`
- `uav-onboard/src/vision/IntersectionStabilizer.hpp`
- `uav-onboard/src/vision/IntersectionStabilizer.cpp`
- `uav-onboard/src/vision/VisionTypes.hpp`
- `uav-onboard/src/vision/LineDetector.hpp`
- `uav-onboard/src/vision/LineDetector.cpp`
- `uav-onboard/src/app/VisionDebugPipeline.cpp`
- `uav-onboard/src/protocol/TelemetryMessage.hpp`
- `uav-onboard/src/protocol/TelemetryMessage.cpp`
- `uav-onboard/src/common/VisionConfig.hpp`
- `uav-onboard/src/common/VisionConfig.cpp`
- `uav-onboard/tools/vision_debug_node.cpp`
- `uav-onboard/tools/line_detector_tuner.cpp`
- `uav-onboard/CMakeLists.txt`
- `uav-onboard/tests/test_intersection_detector.cpp`
- `uav-onboard/tests/test_telemetry_line_json.cpp`
- `uav-onboard/tests/CMakeLists.txt`

GCS overlay/log 표시를 위해 별도 계획으로 같이 수정할 파일:

- `uav-gcs/src/protocol/TelemetryMessage.hpp`
- `uav-gcs/src/protocol/TelemetryMessage.cpp`
- `uav-gcs/src/telemetry/TelemetryStore.hpp`
- `uav-gcs/src/telemetry/TelemetryStore.cpp`
- `uav-gcs/src/overlay/IntersectionOverlay.hpp`
- `uav-gcs/src/overlay/IntersectionOverlay.cpp`
- `uav-gcs/src/app/VisionDebugApp.cpp`
- `uav-gcs/src/telemetry/VisionLogFormatter.cpp`
- `uav-gcs/CMakeLists.txt`
- `uav-gcs/tests/test_telemetry_line_parse.cpp`
- `uav-gcs/tests/test_line_overlay.cpp` 또는 신규 `test_intersection_overlay.cpp`

### 7.2 `LineMaskBuilder`

주요 구조:

```cpp
enum class LinePolarity {
    LightOnDark,
    DarkOnLight
};

struct LineMaskGeometry {
    int source_width = 0;
    int source_height = 0;
    int roi_top = 0;
    int roi_height = 0;
    int work_width = 0;
    int work_height = 0;
    double scale_x = 1.0;
    double scale_y = 1.0;
};

struct LineMaskCandidate {
    cv::Mat mask;
    LinePolarity polarity = LinePolarity::LightOnDark;
};

struct LineMaskFrame {
    LineMaskGeometry geometry;
    std::vector<LineMaskCandidate> masks;
};

class LineMaskBuilder {
public:
    explicit LineMaskBuilder(const common::LineConfig& config);
    LineMaskFrame build(const cv::Mat& image) const;
};
```

설계 의도:

- 현재 `LineDetector.cpp` anonymous namespace에 있는 threshold/morphology/geometry helper를 재사용 가능한 형태로 옮긴다.
- `LineDetector`는 `LineMaskFrame`을 받아 contour scoring만 담당한다.
- `IntersectionDetector`는 같은 `LineMaskFrame`을 받아 branch scoring만 담당한다.

### 7.3 `IntersectionDetector`

주요 API:

```cpp
class IntersectionDetector {
public:
    explicit IntersectionDetector(const common::LineConfig& config);

    IntersectionDetection detect(
        const LineMaskFrame& masks,
        const LineDetection& raw_line) const;
};
```

입력:

- `LineMaskFrame`: segmentation 결과
- `LineDetection raw_line`: selected branch 위치와 line confidence 참고. 중심 후보는 image-center window와 branch ray score로 별도 선택

출력:

- `IntersectionDetection`: type, center, score, branch observations

Detector 내부 helper:

- `findCandidateCenter(...)`
- `largestBlobCentroidCandidate(...)`
- `scoreBranch(...)`
- `classifyBranches(...)`
- `classifyTwoBranchAngle(...)`
- `toSourcePoint(...)`
- `intersectionTypeToString(...)`

### 7.4 `IntersectionStabilizer`

교차점 raw 판정은 프레임 단위로 흔들릴 수 있으므로 `LineStabilizer`와 별도로 temporal smoothing을 둔다.

파일:

- `uav-onboard/src/vision/IntersectionStabilizer.hpp`
- `uav-onboard/src/vision/IntersectionStabilizer.cpp`

주요 API:

```cpp
class IntersectionStabilizer {
public:
    explicit IntersectionStabilizer(const common::LineConfig& config);

    IntersectionDetection update(const IntersectionDetection& raw);
    void reset();
};
```

동작 원칙:

- branch score는 방향별 EMA로 부드럽게 만든다.
- type은 최근 2~3 frame majority vote 또는 hysteresis로 안정화한다.
- `+`, `T`, `L`, `straight` 사이 전환은 raw score가 충분하고 같은 타입이 연속 관측될 때만 확정한다.
- raw result가 잠시 `unknown`이거나 score가 낮으면 `filter_hold_frames`와 비슷한 짧은 hold를 적용한다.
- center는 raw center와 이전 stable center를 EMA로 섞되, type이 바뀌는 순간에는 raw center 쪽 가중치를 높인다.
- stabilizer는 mission 판단용 stable type을 만들기 위한 것이며, 디버깅을 위해 raw type/score도 telemetry와 log에 같이 남긴다.

새 config를 즉시 늘리지는 않는다. 1차 구현에서는 `LineConfig`의 기존 filter 계열 값 또는 내부 상수를 사용하고, 실제 비행 로그에서 필요성이 확인될 때 `intersection_filter_*` 설정을 분리한다.

### 7.5 `VisionTypes.hpp`

추가할 구조:

```cpp
struct VisionResult {
    ...
    LineDetection line;
    IntersectionDetection intersection;

    bool line_detected = false;
    bool intersection_detected = false;
};
```

`intersection_detected`는 기존 protocol top-level summary와 맞춘다.

### 7.6 Protocol 구조

Onboard protocol:

```cpp
struct BranchTelemetry {
    std::string direction;
    bool present = false;
    double score = 0.0;
    Point2f endpoint_px;
};

struct IntersectionTelemetry {
    bool valid = false;
    bool detected = false;
    std::string type = "none";
    std::string raw_type = "none";
    bool stable = false;
    bool held = false;
    Point2f center_px;
    Point2f raw_center_px;
    double score = 0.0;
    double raw_score = 0.0;
    int branch_mask = 0;
    int branch_count = 0;
    int stable_frames = 0;
    std::vector<BranchTelemetry> branches;
};

struct VisionTelemetry {
    ...
    bool intersection_detected = false;
    double intersection_score = 0.0;
    IntersectionTelemetry intersection;
    ...
};
```

Backward compatibility:

- 기존 `vision.intersection_detected`와 `vision.intersection_score`는 유지한다.
- 새 `vision.intersection` object를 추가한다.
- `protocol_version` integer는 기존 규칙대로 `1`을 유지할 수 있다.
- 문서 버전은 `docs/PROTOCOL.md`에서 v1.6으로 올리는 것이 적절하다. GCS parser는 unknown field를 무시하므로 구버전 수신도 깨지지 않는다.

### 7.7 VisionDebugPipeline 호출 위치

현재 위치:

```cpp
if (options.enable_line && options.vision.line.enabled) {
    const auto raw_line = line_detector.detect(image);
    result.line = line_stabilizer.update(raw_line, frame.width);
    result.line_detected = result.line.detected;
}
```

변경 후 권장 흐름:

```cpp
if (options.enable_line && options.vision.line.enabled) {
    const auto line_masks = line_mask_builder.build(image);
    const auto raw_line = line_detector.detect(line_masks);
    const auto raw_intersection = intersection_detector.detect(line_masks, raw_line);
    const auto intersection = intersection_stabilizer.update(raw_intersection);

    result.line = line_stabilizer.update(raw_line, frame.width);
    result.intersection = intersection;
    result.line_detected = result.line.detected;
    result.intersection_detected = intersection.intersection_detected;
}
```

raw intersection은 frame-local observation으로 계산하고,
IntersectionStabilizer가 mission/debug용 stable result를 만든다.
raw와 stable 결과는 모두 telemetry/log에 남긴다.

측정 추가:

- `debug.intersection_latency_ms`
- console log `ix_ms=...`
- GCS log `[intersection] ...`

## 8. GCS overlay 및 log 출력 설계

### 8.1 화면 overlay

현재 GCS overlay:

- line contour: magenta
- tracking point: green
- line label: green text
- marker boxes/arrows: marker overlay

추가 overlay:

- intersection center: cyan filled circle + white outline
- present branch ray: yellow/cyan line from center to endpoint
- absent branch ray: 기본적으로 그리지 않음. 필요 시 debug mode에서 dim gray
- branch endpoint: 작은 circle
- type label: center 근처에 `IX T score 0.86` 형식
- branch label: endpoint 근처에 `F 0.92`, `R 0.81`처럼 짧게 표시

새 파일:

- `uav-gcs/src/overlay/IntersectionOverlay.hpp`
- `uav-gcs/src/overlay/IntersectionOverlay.cpp`

현재 `OverlayPrimitive`는 Line/Circle/Text만 있으므로 arrow primitive를 새로 만들 필요는 없다. arrowhead가 필요하면 짧은 line 두 개로 표현한다.

`VisionDebugApp.cpp` 통합 위치:

```cpp
auto line_overlays = overlay::buildLineOverlays(marker_frame->line);
auto intersection_overlays = overlay::buildIntersectionOverlays(marker_frame->intersection);
auto marker_overlays = overlay::buildMarkerOverlays(marker_frame->markers);
```

권장 순서:

1. line contour
2. intersection branch rays/type
3. marker overlay

이렇게 하면 교차점 정보가 line contour 위에서 보이고 marker overlay와도 충돌이 적다.

### 8.2 Console log 포맷

Onboard `VisionDebugPipeline` 기존 frame log 뒤에 추가한다.

예시:

```text
frame=128 markers=0 line=yes raw_line=yes held=no rejected_jump=no line_conf=0.74 ix_type=T ix_raw=T ix_valid=yes ix_stable=yes ix_held=no ix_detected=yes ix_score=0.86 ix_center=(481.0,356.0) branches=F:0.92,R:0.81,B:0.12,L:0.84 ix_ms=1.3 read_ms=...
```

규칙:

- `ix_type`: `none`, `unknown`, `straight`, `L`, `T`, `+`
- `ix_raw`: temporal smoothing 전 raw type
- `ix_valid`: classifier가 의미 있는 결과를 냈는지
- `ix_stable`: stabilizer가 안정 판정으로 확정했는지
- `ix_held`: raw 판정이 약해서 직전 stable result를 hold했는지
- `ix_detected`: 실제 교차점인지. `straight`면 `no`
- `branches`: `F/R/B/L` 고정 순서
- `ix_ms`: intersection detector latency

GCS vision log 추가:

```text
[intersection] type=T raw=T stable=yes held=no valid=yes detected=yes score=0.86 center=(481.0,356.0) branches=F:0.92 R:0.81 B:0.12 L:0.84 mask=11
```

### 8.3 Telemetry 확장 여부

확장한다.

이유:

- 최종 mission/grid/control은 onboard 판단 결과를 사용해야 한다.
- GCS overlay/log만으로는 mission-critical 정보가 되지 않는다.
- 기존 protocol에 `intersection_detected`, `intersection_score` placeholder가 이미 있다.
- GCS는 unknown field를 무시하므로 structured object 추가가 안전하다.

Telemetry 확장 원칙:

- `vision.intersection_detected`, `vision.intersection_score`는 유지한다.
- `vision.intersection` structured object를 추가한다.
- branch별 score를 telemetry에 보내 overlay/debug를 가능하게 한다.
- `type`, `score`, `center_px`는 stabilized 결과를 기본으로 보낸다.
- `raw_type`, `raw_score`, `raw_center_px`, `stable`, `held`, `stable_frames`를 함께 보내 temporal smoothing이 판정을 숨기지 않게 한다.
- 너무 많은 debug point array는 보내지 않는다. branch endpoint 4개 정도면 충분하다.

## 9. GCS 송신 FPS CLI 설계

요구: 아무 옵션을 주지 않으면 현재처럼 GCS debug video는 5 FPS로 유지하고, 실행 시 `--fps 12`처럼 주면 송신 frame rate를 늘린다.

현재 상태:

- camera FPS override: `--camera-fps`
- debug video send FPS: config `debug_video.send_fps`
- runtime CLI override 없음

권장 구현:

`tools/vision_debug_node.cpp`에 `--fps <n>`을 추가한다.

의미:

- `--fps`는 GCS debug video 송신 FPS다.
- camera capture FPS가 아니다.
- camera capture FPS는 기존 `--camera-fps`를 계속 사용한다.

Usage 문구:

```text
  --fps <n>            Override GCS debug video send FPS, default 5 when video is enabled
  --camera-fps <n>     Override rpicam capture FPS
```

동작:

- 옵션이 없으면 `debug_video.send_fps` config 기본값 5 유지.
- `--video --fps 12`면 12 FPS로 GCS에 보낸다.
- 사용자가 `--fps 12`만 줬을 때는 디버깅 의도가 명확하므로 `--video`를 암시적으로 켜는 방향을 권장한다.
- `--fps`와 `--no-video`를 같이 주면 충돌로 보고 usage error를 내는 편이 명확하다.
- 값은 `1..camera.fps` 범위로 clamp한다. 예를 들어 camera가 12 FPS인데 `--fps 30`이면 12로 제한하고 startup log에 표시한다.

수정 위치:

- `Options`에 `int debug_video_fps_override = 0;`
- `parseOptions()`에 `--fps`
- config override 이후 `vision_config.debug_video.send_fps`에 반영
- `send_video` 결정 로직에서 `--fps`가 있으면 video on
- `VisionDebugPipeline`은 이미 `options.vision.debug_video.send_fps`를 사용하므로 큰 변경이 필요 없다.

검증:

```bash
./build/vision_debug_node --config config --line-only --video
# startup: video_send_fps: 5

./build/vision_debug_node --config config --line-only --video --fps 12
# startup: video_send_fps: 12

./build/vision_debug_node --config config --line-only --fps 12
# 권장 동작: video on, video_send_fps: 12

./build/vision_debug_node --config config --line-only --no-video --fps 12
# 권장 동작: 옵션 충돌 error
```

## 10. 구현 단계 계획

### Phase 1: 기반 구조 분리

1. `LineMaskBuilder` 추가.
2. `LineDetector` private helper 중 mask/geometry 관련 코드를 `LineMaskBuilder`로 이동.
3. 기존 `LineDetector::detect(cv::Mat)` 동작은 유지한다.
4. 내부적으로 `LineMaskBuilder`를 사용하거나, 새 overload `detect(LineMaskFrame)`를 추가한다.
5. 기존 `test_line_stabilizer`, `test_telemetry_line_json` 통과 확인.

목표: behavior를 바꾸지 않고 mask 생성 경계를 분리한다.

### Phase 2: IntersectionDetector 구현

1. `IntersectionDetector.hpp/.cpp` 추가.
2. `IntersectionDetection`, `IntersectionType`, `BranchObservation`을 `VisionTypes.hpp`에 추가.
3. largest blob centroid 후보 생성과 ray score 기반 center refinement를 구현한다.
4. `L` vs `straight`는 branch bitmask와 branch endpoint angle을 함께 사용한다.
5. synthetic image 기반 `test_intersection_detector.cpp` 작성.
6. 테스트 케이스:
   - bright `+`
   - bright `T`
   - bright `L`
   - bright `straight`
   - dark `+`
   - 중심부 작은 gap
   - sparse noise speckles
7. 제공된 2026-04-30 screenshot 세트에서 branch score가 실제 보이는 방향과 맞는지 offline smoke test로 확인한다.
8. `line.mode = auto`와 고정 polarity를 모두 검증한다.

목표: 실제 camera 없이 분류 로직을 회귀 테스트 가능하게 만든다.

### Phase 3: IntersectionStabilizer 구현

1. `IntersectionStabilizer.hpp/.cpp` 추가.
2. type majority vote, branch score EMA, short hold를 구현한다.
3. raw result와 stabilized result를 모두 보존하는 data flow를 만든다.
4. synthetic frame sequence 테스트를 추가한다.
5. 테스트 케이스:
   - `straight -> T` 전환이 1프레임 노이즈로 튀지 않는지
   - `L`과 `straight`가 angle threshold 주변에서 흔들릴 때 hold되는지
   - raw `unknown` 1~2프레임에서 직전 stable type이 유지되는지

목표: 실제 이동 중 frame-to-frame 타입 튐을 줄이되 raw debug 정보는 잃지 않는다.

### Phase 4: VisionDebugPipeline 통합

1. `VisionDebugPipeline.cpp`에서 line mask, raw line, intersection을 같은 frame에서 계산한다.
2. `debug.intersection_latency_ms` 추가.
3. onboard console frame log에 `ix_*` 필드를 추가한다.
4. `line_latency_ms`와 `intersection_latency_ms`를 분리해서 측정한다.
5. startup log에 intersection classifier on/off와 threshold를 표시한다.

목표: GCS 없이도 console에서 type 판정이 보이게 한다.

### Phase 5: Telemetry protocol 확장

1. onboard `TelemetryMessage`에 `IntersectionTelemetry`와 `BranchTelemetry` 추가.
2. `buildTelemetryJson()`에 `vision.intersection` object 추가.
3. 기존 `vision.intersection_detected`, `vision.intersection_score` 계속 채운다.
4. raw/stabilized intersection field를 모두 직렬화한다.
5. `tests/test_telemetry_line_json.cpp`에 intersection field assert 추가.
6. `docs/PROTOCOL.md`를 v1.6으로 업데이트한다.
7. 같은 변경을 `uav-gcs/docs/PROTOCOL.md`에도 반영해야 한다.

목표: structured telemetry로 GCS overlay/log와 나중 mission logic을 연결한다.

### Phase 6: GCS overlay/log 통합

1. GCS parser에 `IntersectionTelemetry` 추가.
2. `TelemetryStore::VisionFrame`에 intersection 추가.
3. `IntersectionOverlay` 추가.
4. `VisionDebugApp.cpp`에서 line overlay 뒤에 intersection overlay 삽입.
5. `VisionLogFormatter.cpp`에 `[intersection]` line 추가.
6. GCS tests 업데이트.

목표: video 화면에서 branch 방향과 타입을 즉시 확인한다.

### Phase 7: GCS 송신 FPS CLI 구현

1. `vision_debug_node`에 `--fps` 추가.
2. `--fps`와 `--camera-fps` 의미를 usage에서 명확히 분리.
3. `--fps`가 있으면 debug video send FPS override.
4. `--fps` 단독 사용 시 video enable 여부는 "enable"로 구현한다.
5. `--no-video --fps` 조합은 error로 처리한다.
6. README 실행 예시에 `--video --fps 12` 추가.

목표: 기본 5 FPS 유지, 디버그 시 12 FPS 선택 가능.

## 11. 검증 계획

### 11.1 Local build/test

Onboard:

```bash
cmake -S . -B build-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

GCS:

```powershell
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
ctest --test-dir build --output-on-failure
```

### 11.2 Offline image validation

`line_detector_tuner`를 확장하거나 신규 `intersection_detector_tuner`를 만든다.

권장 출력:

```text
line_detected=true
line_tracking=(...)
intersection_valid=true
intersection_type=T
intersection_score=0.86
branches=F:0.92 R:0.81 B:0.12 L:0.84
```

테스트 이미지는 `uav-onboard/test_data/images/`에 실제 경기장 유사 샘플로 저장한다. 이번에 제공된 원본 경로는 `c:/Users/mseoky/Documents/공모전/로봇항공기경연대회(중급)_2026/비전_영상_캡처/` 아래 screenshot 8장이다.

### 11.3 Runtime validation

기본 5 FPS:

```bash
./build/vision_debug_node --config config --line-only --line-mode auto --video
```

고 FPS debug:

```bash
./build/vision_debug_node --config config --line-only --line-mode auto --video --fps 12
```

확인할 것:

- GCS log의 `video_target_fps`가 5 또는 12로 맞는지
- `video_skipped_frames`가 기대대로 줄어드는지
- `video_send_failures`가 증가하지 않는지
- `processing_fps`가 12 근처를 유지하는지
- `intersection_latency_ms`가 안정적인지
- `ix_type`이 `straight`, `L`, `T`, `+`에서 frame-to-frame으로 과도하게 튀지 않는지
- `ix_raw`가 튀더라도 `ix_type`이 stabilizer 때문에 과도하게 흔들리지 않는지
- `L`과 `straight` 경계에서 branch angle difference가 log로 납득 가능한지

### 11.4 실제 이미지/현장 재검증

이번에 제공된 8장 기준 1차 판단은 다음과 같다.

1. selected contour는 교차점 전체 blob이 아니라 한쪽 branch로 자주 분리된다.
2. 노이즈는 보이지만 selected contour가 노이즈를 고르는 상황은 주된 문제가 아니다.
3. largest blob centroid는 유용한 중심 후보지만, 긴 branch 하나가 largest blob일 수 있어 ray score 검증이 필요하다.
4. `L`과 `straight`는 branch count뿐 아니라 branch angle difference로 검증해야 한다.
5. `local_contrast_blur`, `morph_close_kernel`, `lookahead_y_ratio`가 실제 튜닝 가치가 있는 값이다.
6. `LineDetector` config만으로 `+`, `T`, `L` 분류를 해결할 수 없으므로 별도 `IntersectionDetector`와 `IntersectionStabilizer`가 필요하다.

추가 현장 capture가 확보되면 다음 순서로 판단한다.

1. selected contour가 교차점 전체 blob인지, branch 하나만인지 확인한다.
2. `line_contours_found`와 `candidates_evaluated`로 contour fragmentation 여부를 확인한다.
3. `raw_line`, `held`, `rejected_jump`로 stabilizer가 교차점 frame을 jump로 보고 있는지 확인한다.
4. `branches=F/R/B/L` score가 실제 영상과 맞는지 확인한다.
5. config 후보는 동일 조건 3장 이상의 representative frame에서만 조정한다.

## 12. 리스크 및 보류 항목

- Ray casting은 카메라가 격자와 크게 회전되어 있으면 branch 방향 판단이 약해진다. 드론이 항상 정면을 보는 운용 조건을 전제로 한다.
- line이 매우 넓거나 교차점 중심이 화면 밖에 가까우면 `L`과 `straight`가 혼동될 수 있다.
- `straight`는 실제 교차점이 아니므로 `intersection_detected=false`와 `type="straight"`를 동시에 표현해야 한다.
- largest blob centroid는 라인 전체가 하나로 붙은 경우에는 좋지만, 긴 branch 하나만 붙은 경우에는 중심이 branch 중앙으로 밀릴 수 있다. 그래서 최종 center가 아니라 후보로만 사용한다.
- angle 기반 `L`/`straight` 판정은 카메라 yaw/원근/렌즈 왜곡에 따라 threshold가 필요하다. 초기 threshold는 넓게 잡고, temporal stabilizer로 경계 흔들림을 줄인다.
- `IntersectionStabilizer`는 1~2프레임 지연을 만들 수 있다. mission control에 쓰기 전 raw/stable 차이를 log로 확인해야 한다.
- 라인 극성이 미확정인 동안 `auto` 모드는 비용과 false positive 가능성이 증가한다.
- Skeleton topology는 1차 구현에서 제외하지만, ray casting으로 해결되지 않는 branch ambiguity가 남으면 2차 검증기로 추가한다.
- Mission/Grid/Control/MAVLink는 이번 작업 범위가 아니다. 이번 작업은 "현재 바라보는 라인 모양을 안정적으로 분류하고 디버깅 가능하게 만드는 것"까지다.

## 13. 이번 작업의 완료 기준

기능 완료:

- `vision_debug_node --line-only --line-mode auto --video --fps 12` 실행 시 GCS video가 12 FPS target으로 송신된다.
- 아무 FPS 옵션이 없으면 기존처럼 debug video target은 5 FPS다.
- Onboard console에 `ix_type`, `branches`, `ix_score`, `ix_ms`가 표시된다.
- Onboard/GCS log에 raw type과 stabilized type이 같이 표시된다.
- GCS overlay에 교차점 중심, branch 방향, 타입 label이 표시된다.
- GCS log에 `[intersection]` line이 표시된다.
- telemetry에는 기존 summary field와 새 structured `vision.intersection`이 같이 들어간다.

품질 완료:

- 기존 telemetry/line stabilizer tests가 계속 통과한다.
- synthetic intersection tests가 `+`, `T`, `L`, `straight`, light/dark polarity를 검증한다.
- protocol 문서가 onboard/GCS 양쪽에서 동일하게 업데이트된다.
- debug video가 best-effort 최신 frame 송신 원칙을 유지한다.
- onboard overlay drawing은 추가하지 않는다.
