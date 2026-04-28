# 고고도 라인 트레이싱 개선 계획 및 수행 현황

작성일: 2026-04-28
상태: 1차 구현 완료. Raspberry Pi 실기 테스트 대기.

## 1. 문제 요약

최근 테스트에서 약 40~100cm 고도에서는 흰색 라인이 안정적으로 잡혔지만, 약 150cm 이상에서는 라인 폭이 영상에서 너무 얇아지고 바닥 밝기/무늬/천 경계가 라인 후보로 섞였다. 이 때문에 magenta contour가 넓은 바닥 blob을 따라가거나 green tracking point가 좌우로 튀었다.

대회 조건은 라인 폭 `10cm (TBR)`, 비행 고도 `2m (TBR)`이므로 최소 180~190cm에서도 라인 중심 추적이 가능해야 한다. 단, Raspberry Pi Zero 2 W에서 mission control이 우선이므로 영상 송출과 디버그 overlay가 mission loop를 막으면 안 된다.

## 2. 설계 원칙

- GCS overlay는 관찰용 debug channel이다. 실제 판단에 필요한 라인 중심 계산은 onboard에서 끝낸다.
- 새 detector 파일을 만들지 않고 기존 `LineDetector`와 `LineStabilizer` 안에서 개선한다.
- line tracking과 future intersection 판단은 목적이 다르다. 이번 작업은 tracking point 안정화에 집중하고, `+`, `T`, `L` 교차점 판단은 다음 단계로 남긴다.
- 화면을 단순히 3:4:3으로 hard crop하지 않는다. 교차점 branch를 잃을 수 있기 때문이다. 대신 lookahead band와 stabilizer를 사용해 tracking point만 안정화한다.
- latency를 줄이기 위해 전체 해상도를 무작정 올리지 않는다. `process_width`만 320에서 480으로 올려 고고도 라인의 pixel 수를 보강한다.

## 3. 구현한 변경사항

### 3.1 Local Contrast Mask

기존 global grayscale threshold는 밝은 바닥 반사와 흰색 라인을 잘 분리하지 못했다. 기본 mask 전략을 `local_contrast`로 바꿔, 각 pixel이 주변 local background보다 얼마나 밝은지를 기준으로 라인 후보를 만든다.

관련 설정:

```toml
[line]
mask_strategy = "local_contrast"
local_contrast_blur = 31
local_contrast_threshold = 12
```

효과:

- 고도 증가로 라인이 얇아져도 주변 바닥 대비 밝은 선 구조를 더 잘 잡는다.
- 창문빛/그늘/흙바닥처럼 전체 밝기가 변하는 환경에서 global threshold보다 안정적이다.
- `mask_strategy = "global"`로 되돌리면 기존 threshold 방식도 사용할 수 있다.

### 3.2 Process Width 480

기존 `process_width = 320`은 Pi에는 가볍지만, 150~200cm 고도에서 10cm 라인이 너무 적은 pixel로 줄어드는 문제가 있었다. 기본값을 480으로 올렸다.

관련 설정:

```toml
process_width = 480
```

기대 효과:

- 같은 카메라 해상도에서 라인 pixel 폭을 약 1.5배 확보한다.
- local test 기준 `line_detector_tuner`와 `vision_debug_node`는 정상 빌드된다.
- 실제 Pi latency는 현장 테스트에서 `debug.line_latency_ms`, `debug.processing_latency_ms`, GCS latency display로 확인해야 한다.

### 3.3 Lookahead Band Projection

기존 방식은 lookahead y 한 줄 또는 contour 전체 형상에 영향을 많이 받았다. 이제 lookahead y 주변의 짧은 horizontal band에서 mask pixel을 x 방향으로 projection하고, plausible width와 center score가 좋은 run을 tracking point로 선택한다.

관련 설정:

```toml
lookahead_y_ratio = 0.55
lookahead_band_ratio = 0.06
min_line_width_px = 8
max_line_width_ratio = 0.22
```

효과:

- magenta contour가 약간 넓거나 교차형이어도 green point는 lookahead band의 라인 중심에 더 가깝게 유지된다.
- 한 줄 noise 때문에 tracking point가 튀는 현상이 줄어든다.
- intersection branch 분석은 아직 구현하지 않았지만, 전체 contour overlay는 유지된다.

### 3.4 Stabilizer 개선

기존 EMA/hold/jump rejection에 다음 두 가지를 추가했다.

```toml
filter_confidence_alpha_min = 0.35
filter_max_offset_velocity_ratio = 0.08
```

동작:

- confidence가 낮은 measurement는 EMA alpha를 낮춰 더 천천히 반영한다.
- jump rejection을 통과한 measurement라도 frame당 x 이동량을 제한해 green point가 갑자기 크게 이동하지 않게 한다.
- 라인이 1~3 frame 순간적으로 끊기면 hold로 이전 tracking point를 유지한다.

## 4. 로컬 검증 결과

OpenCV C++ build:

```powershell
cmake -S . -B build-opencv-local -DCMAKE_BUILD_TYPE=Release -DOpenCV_DIR=C:/msys64/ucrt64/lib/cmake/opencv4 -DBUILD_TESTS=OFF
cmake --build build-opencv-local --target line_detector_tuner vision_debug_node
```

Unit tests:

```powershell
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

결과:

- `telemetry_line_json`: passed
- `line_stabilizer`: passed

첨부 이미지 기반 `line_detector_tuner` 결과:

| 이미지 | 결과 | tracking point / offset | confidence |
|---|---|---|---:|
| `v2_line1_1.png` | detected | `(479.45, 404.05)`, `-5.05px` | `0.89` |
| `v2_line1_2.png` | detected | `(487.02, 402.53)`, `5.02px` | `0.76` |
| `v2_line2_1.png` | detected | `(349.17, 402.05)`, `-133.83px` | `0.84` |
| `v2_line2_2.png` | detected | `(448.59, 400.47)`, `-29.91px` | `0.87` |

연습용 격자 이미지도 기본/그늘/햇빛 조건에서 모두 detected 상태로 확인했다.

## 5. Raspberry Pi 테스트 계획

GCS 노트북:

```powershell
cd uav-gcs
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

라인만 볼 때:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

확인 항목:

- GCS overlay에서 magenta contour가 라인 주변에 잡히는지
- green tracking point가 라인 중심 근처에 유지되는지
- 150cm, 180cm, 190cm, 200cm에서 `confidence`, `raw/filtered`, `held`, `rejected_jump` 상태가 타당한지
- GCS 좌상단 latency가 이전 230~280ms 수준을 크게 넘지 않는지
- GCS vision log의 `line_latency_ms`, `processing_latency_ms`, `video_dropped_frames`가 비정상적으로 증가하지 않는지

## 6. 남은 작업

1. Pi 실기에서 480px local contrast의 실제 CPU 비용을 측정한다.
2. 야외 흙바닥, 직사광, 그늘 조건에서 `local_contrast_threshold`와 `local_contrast_blur`를 튜닝한다.
3. 라인 추적 결과와 교차점 후보를 분리한 `IntersectionCandidate` 경로를 추가한다.
4. `+`, `T`, `L` 교차점 판단을 위해 wide ROI의 branch/span 분석을 구현한다.
5. 실제 비행 진동이 들어간 영상으로 stabilizer parameter를 재조정한다.
