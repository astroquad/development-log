# Astroquad Latency and Line Stability Improvement Plan

작성일: 2026-04-27  
상태: 계획 수립 전용. 아직 코드 수정 금지.
범위: `RESEARCH.md` 기준 최신 구조에서 latency 증가 원인 분석, line tracking point 튐 완화, 라인트레이싱 정확도 개선 계획.

## 1. 현재 진행 상황 이해

현재 구현 흐름은 다음 단계까지 와 있다.

```text
Pi camera
  -> rpicam MJPEG frame
  -> onboard JPEG decode to cv::Mat
  -> onboard ArUco detection
  -> onboard LineDetector
  -> onboard telemetry JSON
  -> onboard raw MJPEG debug video
  -> GCS video receive
  -> GCS marker/line overlay
  -> GCS vision log window
```

중요한 현재 설계:

- 온보드는 overlay drawing을 하지 않는다.
- 온보드는 rpicam에서 받은 원본 JPEG를 GCS로 그대로 보낸다.
- 온보드는 vision 처리를 위해 JPEG decode는 하지만, 현재 video stream을 위해 JPEG 재인코딩은 하지 않는다.
- GCS video는 best-effort debug channel이다.
- `LatestVideoSender`가 별도 worker thread로 video frame을 송신하므로, video UDP 송신 자체는 main vision loop를 직접 오래 block하지 않도록 이미 분리되어 있다.
- line overlay는 GCS에서 `vision.line.contour_px`와 `tracking_point_px`를 사용해 그린다.

최근 관찰된 latency:

| 모드 | 평균 latency |
|---|---:|
| 영상만 streaming | 약 150 ms |
| ArUco 추가 | 약 210 ms |
| ArUco + line tracing 추가 | 약 470 ms |

라인 추가 후 latency가 2배 이상 증가했으므로, 단순히 detector 하나가 추가된 수준으로 보기 어렵다. 먼저 계측을 강화해 병목을 분리한 뒤 최적화해야 한다.

## 2. Latency 증가 원인 후보 분석

### 2.1 가능성이 낮은 원인

#### JPEG 재인코딩

현재 코드 기준으로 onboard는 video stream용 JPEG를 재인코딩하지 않는다.

```text
rpicam-vid MJPEG stdout
  -> RpicamMjpegSource extracts JPEG
  -> VisionDebugPipeline decodes JPEG once for vision
  -> UdpMjpegStreamer sends original JPEG bytes
```

따라서 "라인 검출 때문에 JPEG 재인코딩 비용이 늘었다"는 원인은 현재 구조에서는 가능성이 낮다.

#### video UDP 송신이 main loop를 직접 block

`VisionDebugPipeline.cpp`에는 `LatestVideoSender`가 있고, 내부 worker thread가 `UdpMjpegStreamer::sendFrame()`을 호출한다. main loop는 `video_sender.submit(frame)`만 수행한다.

따라서 UDP video 송신이 느려져도 main loop는 오래된 debug frame을 덮어쓰고 최신 frame만 유지하는 구조다. 다만 `submit(frame)`이 `CameraFrame`을 값으로 받아 JPEG vector copy가 발생할 수 있으므로, 완전히 비용이 없는 것은 아니다.

### 2.2 가능성이 높은 원인

#### LineDetector가 거의 전체 frame을 처리함

현재 line 기본값:

```toml
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
mode = "auto"
```

`roi_top_ratio = 0.08`은 상단 절단 문제를 해결했지만, 결과적으로 frame의 약 92%를 line detector가 처리한다. 640x480 기준 대부분의 pixel이 threshold/morph/findContours 대상이다.

#### `auto` mode가 mask pipeline을 두 번 수행함

`mode = "auto"`에서는 밝은 라인 mask와 어두운 라인 mask를 모두 만든다.

```text
gray
  -> bright-line Otsu threshold
  -> morph open/close
  -> findContours

gray
  -> dark-line Otsu threshold
  -> morph open/close
  -> findContours
```

흰색 라인으로 확정된 테스트에서는 `light_on_dark`로 고정하면 line detector 비용을 크게 줄일 수 있다.

#### contour 후보마다 전체 크기 mask를 다시 그림

현재 `LineDetector::evaluateContour()`는 후보 contour마다 다음을 수행한다.

```cpp
cv::Mat contour_mask = cv::Mat::zeros(mask.size(), CV_8UC1);
cv::drawContours(contour_mask, draw_contours, 0, cv::Scalar(255), cv::FILLED);
```

즉, 후보 contour가 많으면 full ROI 크기의 mask allocation + draw가 반복된다. 바닥 무늬, 반사광, 노이즈가 많으면 후보가 많아지고 비용이 급증할 수 있다.

이 지점이 latency 470 ms의 가장 유력한 코드 병목이다.

#### contour가 많으면 telemetry payload와 GCS overlay도 커짐

현재 `max_contour_points = 80`으로 제한되어 있지만, line contour가 잡힐 때마다 최대 80개 point가 JSON으로 들어간다. JSON payload가 커지고 GCS는 이를 parse한 뒤 overlay primitive로 변환한다.

이 비용은 detector 비용보다 작을 가능성이 크지만, telemetry size와 GCS overlay primitive count를 계측해야 한다.

#### rpicam stdout pipe backlog 가능성

현재 vision loop가 camera fps보다 느려지면 `rpicam-vid` stdout pipe에 frame이 쌓일 수 있다. `RpicamMjpegSource`는 순차적으로 JPEG를 추출하므로, 처리 속도가 낮으면 실제 영상이 늦게 보일 수 있다.

주의할 점:

- `CameraFrame.timestamp_ms`는 JPEG를 추출한 시점에 찍힌다.
- 실제 camera exposure/capture 시점이 아니다.
- 따라서 GCS overlay에 표시되는 latency는 실제 camera-to-display latency보다 낮게 보일 수 있다.

즉, GCS 표시 latency가 470 ms라면 실제 체감 지연은 그보다 더 클 수도 있다.

## 3. Latency 개선을 위한 계측 계획

최적화 전에 먼저 병목을 숫자로 분리한다.

### 3.1 onboard 단계별 latency 계측 추가

`VisionDebugPipeline`에서 frame마다 다음 값을 측정한다.

| 항목 | 의미 |
|---|---|
| `read_frame_ms` | `camera.readFrame()` 소요 시간 |
| `jpeg_decode_ms` | `cv::imdecode()` 소요 시간 |
| `aruco_latency_ms` | 기존 ArUco detector 시간 |
| `line_latency_ms` | 기존 LineDetector 시간 |
| `telemetry_build_ms` | JSON 생성 시간 |
| `telemetry_bytes` | telemetry JSON byte 수 |
| `telemetry_send_ms` | UDP telemetry send 시간 |
| `video_submit_ms` | latest video slot submit 시간 |
| `video_jpeg_bytes` | frame JPEG byte 수 |
| `video_sent_frames` | video worker가 실제 송신한 frame 수 |
| `video_dropped_frames` | latest slot에서 덮어써진 debug frame 수 |

현재 protocol은 unknown field를 무시하는 구조이므로, 우선 `debug` object에 확장 field를 추가할 수 있다. 구현 시 `PROTOCOL.md` 양쪽을 갱신한다.

### 3.2 LineDetector 내부 계측 추가

line latency가 큰 경우를 더 쪼갠다.

| 항목 | 의미 |
|---|---|
| `line_mode_used` | `auto`, `light_on_dark`, `dark_on_light` |
| `line_mask_count` | 만든 mask 개수. `auto`면 2 |
| `line_roi_pixels` | 처리한 ROI pixel 수 |
| `line_contours_found` | `findContours`로 찾은 후보 수 |
| `line_candidates_evaluated` | scoring까지 들어간 후보 수 |
| `line_selected_contour_points` | telemetry로 보낸 contour point 수 |

이 값은 처음에는 console log 또는 vision log window에 표시하고, 필요하면 telemetry debug field로 올린다.

### 3.3 GCS latency 표시 개선

GCS vision log window에 다음을 같이 표시한다.

```text
[vision] frame=...
age=...
processing=...
decode=...
aruco=...
line=...
telemetry_bytes=...
video_bytes=...
video_sent=...
video_dropped=...
line_contours=...
line_candidates=...
```

목표는 "470 ms가 onboard line detector 때문인지, video 송신/수신 때문인지, GCS decode/display 때문인지"를 한 번에 분리하는 것이다.

## 4. Latency 최적화 계획

### 4.1 1순위: line mode 고정

현재 라인이 흰색이면 실전 테스트에서는 `auto` 대신 `light_on_dark`를 기본 실행으로 사용한다.

```bash
./build/vision_debug_node --config config --line-mode light_on_dark
```

효과:

- threshold/morph/findContours pass가 2회에서 1회로 줄어든다.
- 가장 쉽고 위험이 낮다.

계획:

- `config/vision.toml` 기본은 아직 `auto`로 둘지, 실전 preset만 `light_on_dark`로 둘지 계측 후 결정한다.
- README에는 실전 흰색 라인 테스트 명령을 `light_on_dark`로 명시한다.

### 4.2 2순위: multi ROI 분리

현재 하나의 넓은 ROI가 line tracking과 교차점 contour 표시를 모두 담당한다. 이 때문에 상단 절단은 줄었지만 처리량이 커졌다.

권장 분리:

| ROI | 목적 | 크기 |
|---|---|---|
| tracking ROI | green tracking point/offset 계산 | 중하단 중심, 예: y 0.45~0.85 |
| contour ROI | GCS magenta contour 표시 | 현재처럼 넓게, 예: y 0.08~1.0 |
| future intersection ROI | 십자 판단 | 중앙~상단 포함, 별도 scoring |

단기 구현:

- detector 계산은 tracking ROI 중심으로 줄인다.
- contour overlay는 selected connected contour가 있으면 넓은 ROI에서 얻되, 후보 탐색 수를 제한한다.

주의:

- 지금 사용자가 원하는 십자 contour 보존 때문에 overlay ROI를 너무 좁히면 안 된다.
- line 주행용 center와 교차점 표시용 contour를 같은 값으로 억지로 맞추지 않는다.

### 4.3 3순위: downscale line detection

라인 검출은 정밀한 ArUco corner 수준의 해상도가 필요하지 않다. line detector 입력만 축소해서 처리하고 결과 좌표를 원본 frame 좌표로 scale back한다.

권장 config:

```toml
[line]
process_width = 320
```

640x480을 320x240으로 줄이면 pixel 수가 1/4로 줄어 threshold/morph/findContours 비용이 크게 감소한다.

주의:

- contour point와 tracking point는 GCS overlay를 위해 원본 좌표로 복원해야 한다.
- 너무 낮추면 얇은 라인이 끊길 수 있으므로 320 width부터 시작한다.

### 4.4 4순위: per-candidate full mask 제거

현재 가장 의심되는 병목이다. 후보 contour마다 full-size `contour_mask`를 만드는 방식을 제거한다.

대체 방법:

1. 먼저 `cv::boundingRect(contour)`로 lookahead row 포함 여부를 확인한다.
2. bounding rect가 lookahead row를 포함하지 않으면 바로 skip한다.
3. row center 계산이 필요한 후보만 작은 local mask를 만든다.
4. local mask 크기는 `bounds.width * bounds.height`로 제한한다.
5. 후보 수가 너무 많으면 area 상위 N개만 평가한다.

권장 초기값:

```toml
[line]
max_candidates = 8
```

효과:

- 노이즈 contour가 많아도 full ROI mask allocation 반복을 피한다.
- line latency spike를 줄일 가능성이 크다.

### 4.5 5순위: contour/projection 기반 center 계산 개선

tracking point는 contour 전체 centroid보다 lookahead row projection이 더 좋다. 다만 지금 방식은 후보마다 mask를 만들어 row span을 찾는다.

대안:

- final binary mask에서 lookahead band 몇 줄을 projection한다.
- selected contour의 bounding box 안에서만 x projection을 계산한다.
- line 폭이 너무 넓은 경우 hard reject하지 말고 confidence 감점만 한다. 현재 십자 contour 보존을 위해 이 방향이 맞다.

### 4.6 6순위: telemetry contour point와 overlay 비용 제한

현재 `max_contour_points = 80`은 크게 나쁘지 않다. 다만 latency를 줄여야 하면 다음을 실험한다.

```toml
[line]
max_contour_points = 40
```

GCS overlay는 40개 point만 있어도 line border 확인에는 충분할 가능성이 크다.

주의:

- 교차점 판단 자체는 GCS overlay contour point가 아니라 onboard detector 내부 contour로 해야 한다.
- telemetry contour point는 관제 표시용이다.

## 5. 라인 tracking point 튐 문제 분석

현재 튐 현상은 다음 상황에서 발생할 수 있다.

1. 순간 조명 변화로 다른 contour가 선택됨
2. 바닥 무늬/그림자/발/반사광이 line 후보 점수에서 이김
3. 드론 진동으로 line이 ROI에서 끊김
4. 십자 교차점에서 line width가 갑자기 넓어져 tracking x가 이동함
5. frame마다 threshold/Otsu 결과가 달라짐
6. `auto` mode에서 bright/dark 후보가 frame마다 바뀜

현재 `LineDetector`는 stateless다. 즉, 이전 frame의 line 위치를 고려하지 않고 매 frame 새 후보를 고른다. 따라서 순간적인 오검출이 바로 green tracking point 튐으로 나타난다.

## 6. Line tracking 안정화 계획

### 6.1 새 상태 필터 추가: `LineTracker` 또는 `LineStabilizer`

`LineDetector`는 raw detection만 담당하고, 그 뒤에 temporal filter를 둔다.

권장 위치:

```text
uav-onboard/src/vision/LineStabilizer.hpp
uav-onboard/src/vision/LineStabilizer.cpp
```

또는 초기에는 `VisionDebugPipeline` 내부 helper로 시작하고, 안정화되면 별도 class로 분리한다.

입력:

```cpp
LineDetection raw;
std::uint32_t frame_seq;
std::int64_t timestamp_ms;
int image_width;
int image_height;
```

출력:

```cpp
LineDetection filtered;
LineFilterDebug debug;
```

### 6.2 필터 정책

#### Confidence gate

raw confidence가 너무 낮으면 바로 반영하지 않는다.

초기값:

```toml
[line_filter]
min_accept_confidence = 0.25
```

#### Jump gate

이전 accepted tracking point와 너무 멀리 떨어지면 reject 또는 hold한다.

초기값:

```toml
[line_filter]
max_offset_jump_px = 80
max_angle_jump_deg = 35
```

정규화 버전도 가능하다.

```text
max_offset_jump_px = image_width * 0.12
```

#### EMA filter

accepted raw detection을 바로 쓰지 않고 지수 이동 평균으로 부드럽게 만든다.

```text
filtered_x = alpha * raw_x + (1 - alpha) * prev_x
filtered_angle = alpha * raw_angle + (1 - alpha) * prev_angle
```

초기값:

```toml
[line_filter]
ema_alpha = 0.35
```

#### Short hold

진동으로 1~3 frame line이 끊긴 경우 이전 filtered result를 잠시 유지한다.

초기값:

```toml
[line_filter]
hold_frames = 3
```

hold 중에는 confidence를 점진적으로 낮춘다.

```text
held_confidence = prev_confidence * 0.85
```

#### Reject counting

갑자기 튄 raw detection은 버리되, 여러 frame 연속 같은 방향으로 나타나면 실제 line 변화일 수 있으므로 재획득한다.

초기 정책:

```text
if jump detected for 1~2 frames:
    hold previous
if jump persists for 3 frames:
    accept as reacquired line
```

### 6.3 GCS 표시

GCS vision log window에 raw/filtered 상태를 표시한다.

권장 표시:

```text
[line] raw=yes filtered=yes hold=no rejected=1
tracking=(x,y) raw=(x,y)
offset=...
angle=...
confidence=...
```

overlay 옵션:

- 기본 green point는 filtered tracking point
- raw point는 필요 시 작은 yellow point로 표시
- rejected frame은 log에만 표시하고 overlay는 이전 filtered point 유지

초기 구현은 telemetry schema를 크게 늘리지 않고 `debug.note` 또는 추가 debug fields로 시작할 수 있다. 안정화 후 `PROTOCOL.md`에 정식 field를 추가한다.

## 7. 사용자가 제시한 개선책 적용 판단

| 방법 | 현재 적용 판단 | 이유 |
|---|---|---|
| 1. ROI crop / 다중 ROI | 바로 적용 | latency와 안정성 모두에 가장 효과적. tracking ROI와 contour/intersection ROI를 분리해야 함 |
| 2. HSV/gray threshold | 조건부 적용 | 현재는 gray polarity가 가볍고 단순함. 라인 색이 확정되거나 명암 차가 부족하면 `color_hsv` 추가 |
| 3. 원근 변환 Bird's Eye View | 지금은 보류 | 계산 비용과 calibration 부담이 있음. 카메라 장착 각도 고정 후 control 단계에서 검토 |
| 4. morphology | 이미 적용, 튜닝 필요 | kernel size와 open/close 순서가 latency/정확도에 영향. ROI/downscale 후 재튜닝 |
| 5. largest contour or projection center | 적용 | 후보 전체 mask 대신 projection/상위 contour 평가로 변경. center 튐도 줄일 수 있음 |
| 6. fitLine angle | 이미 적용 | 유지. 단, low confidence contour의 angle은 filter에서 gate 처리 |
| 7. confidence 계산 | 개선 필요 | 현재 면적/폭/중앙성 중심. temporal consistency, ROI coverage, row projection 안정성을 추가 |
| 8. EMA 필터 | 바로 적용 | tracking point 튐 완화에 가장 직접적이고 계산 비용이 작음 |
| 9. GCS telemetry 표시 | 바로 적용 | latency 병목과 filter 동작을 현장에서 확인하려면 필수 |
| 10. 실제 영상 threshold 튜닝 | 바로 적용 | 경기장 환경 불확실성이 가장 큼. `line_detector_tuner` batch/preview 강화 필요 |

## 8. 구현 우선순위

### 1단계: 계측 먼저 추가

목표:

- 470 ms가 정확히 어디서 생기는지 확인한다.

작업:

1. `VisionDebugPipeline` 단계별 latency 측정 추가
2. `LineDetector` 내부 contour/mask/candidate count 측정 추가
3. telemetry debug fields 확장
4. GCS `VisionLogFormatter`에 latency breakdown 표시
5. README/PROTOCOL 갱신

완료 기준:

- GCS log window에서 `read/decode/aruco/line/telemetry/video` 비용을 볼 수 있다.
- `line_contours_found`, `line_candidates_evaluated`, `telemetry_bytes`, `video_dropped_frames`를 볼 수 있다.

### 2단계: low-risk latency 개선

목표:

- 동작을 크게 바꾸지 않고 latency를 먼저 낮춘다.

작업:

1. 흰색 라인 테스트 command를 `--line-mode light_on_dark`로 고정
2. `max_contour_points` 80 -> 40 실험
3. `line_detector_tuner`로 threshold 고정값 실험
4. `auto` mode는 현장 초기 탐색용으로만 사용

완료 기준:

- ArUco + line 평균 latency가 470 ms에서 유의미하게 감소한다.
- line overlay는 계속 표시된다.
- 십자 contour는 계속 connected contour로 보인다.

### 3단계: LineDetector hotspot 제거

목표:

- per-candidate full mask allocation/draw를 제거한다.

작업:

1. contour를 area/boundingRect로 1차 필터링
2. area 상위 N개 후보만 평가
3. lookahead row 계산은 boundingRect local mask 또는 projection 방식으로 변경
4. downscale `process_width=320` 옵션 추가

완료 기준:

- `line_latency_ms` spike가 줄어든다.
- 바닥 노이즈가 많아도 후보 평가 시간이 안정적이다.

### 4단계: LineStabilizer 추가

목표:

- 초록 tracking point가 갑자기 튀는 현상을 줄인다.

작업:

1. raw line detection 뒤 temporal filter 추가
2. confidence gate, jump gate, EMA, hold_frames 구현
3. filtered result를 telemetry/GCS overlay 기본값으로 사용
4. GCS log에 raw/filtered/rejected/hold 상태 표시

완료 기준:

- 1~3 frame의 순간 오검출이나 line 끊김에서 green point가 크게 튀지 않는다.
- 실제 큰 경로 변화가 여러 frame 지속되면 재획득한다.
- line이 완전히 사라지면 hold 후 detected=false로 내려간다.

### 5단계: 다중 ROI와 projection center

목표:

- latency와 안정성을 동시에 개선한다.

작업:

1. tracking ROI와 contour ROI 분리
2. tracking point는 lookahead band projection으로 계산
3. contour overlay는 connected contour를 유지
4. future intersection ROI 설계만 준비

완료 기준:

- tracking point 튐 감소
- 십자 contour 표시 유지
- line detector 처리 pixel 수 감소

### 6단계: 실제 영상 기반 튜닝 도구 강화

목표:

- 경기장 전까지 threshold/ROI/morphology 값을 감으로 맞추지 않는다.

작업:

1. `line_detector_tuner`가 mask/contour preview image 저장
2. 여러 image를 batch로 돌려 detection rate, average confidence, latency 출력
3. 캡처 이미지별 추천 threshold/mode를 비교

완료 기준:

- `line1~line4` 같은 캡처를 batch로 돌려 수치 비교 가능
- 현장 이미지가 생기면 config preset을 빠르게 선택 가능

## 9. 성능 목표

현재 관찰:

```text
video only:       ~150 ms
ArUco:            ~210 ms
ArUco + Line:     ~470 ms
```

1차 목표:

```text
ArUco + Line:     < 300 ms average
line_latency_ms:  < 40 ms average at 640x480 or downscaled line input
```

2차 목표:

```text
ArUco + Line:     210~260 ms range
line spike:       no repeated >100 ms spikes in normal lighting
```

판단 기준:

- GCS displayed latency만 보지 않는다.
- onboard `processing_latency_ms`, `line_latency_ms`, `video_dropped_frames`, GCS `age`를 함께 본다.

## 10. 구현 시 주의사항

- 이번 계획 단계에서는 코드 수정하지 않는다.
- 실제 구현 시 `PROTOCOL.md`와 `README.md` 양쪽 repo를 항상 갱신한다.
- `uav-gcs/docs/PROTOCOL.md`와 `uav-onboard/docs/PROTOCOL.md`는 동일하게 유지한다.
- line branch filtering을 단순 재도입하지 않는다. 이전 실험에서 십자 contour를 깨뜨렸다.
- 반사광 억제와 교차점 보존은 별도 output으로 풀어야 한다.
- Bird's Eye View는 카메라 장착 각도와 calibration이 고정된 뒤 검토한다.
- Pixhawk 제어보다 line/intersection vision 안정화가 우선이다.

## 11. 최종 결론

latency 470 ms의 가장 유력한 원인은 JPEG 재인코딩이나 video 송신 main-loop block이 아니라, `LineDetector`의 넓은 ROI, `auto` mode 2-pass 처리, contour 후보마다 full mask를 새로 만드는 구조, 그리고 line contour telemetry/GCS 표시 비용의 조합이다.

라인 tracking point 튐은 현재 detector가 stateless이고, raw detection을 곧바로 telemetry/overlay에 쓰기 때문에 발생한다. 이를 해결하려면 raw detector 자체를 과하게 보수적으로 만드는 것보다, detector 뒤에 temporal stabilizer를 두고 confidence gate, jump gate, EMA, short hold를 적용하는 것이 맞다.

가장 안전한 다음 구현 순서는 다음이다.

1. latency breakdown 계측
2. `light_on_dark` 고정과 contour point 축소 같은 low-risk 최적화
3. LineDetector 후보 평가 hotspot 제거
4. LineStabilizer 추가
5. multi ROI/projection center 개선
6. 실제 영상 기반 threshold tuning 강화
