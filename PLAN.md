# ArUco marker at intersection 대응 계획

최종 업데이트: 2026-05-13

## 목표

3x3 cell 격자 경기장에서 ArUco marker가 교차점 위에 있어도 onboard가 기존 line tracing, L/T/+ 교차점 판단, grid node 기록을 유지하고, GCS가 marker/intersection/grid overlay와 log를 올바르게 표시하도록 만든다.

이번 작업은 `uav-onboard/PROJECT_SPEC.md`의 최종 미션 요구사항인 "교차점마다 ArUco marker ID를 인식하고 격자 좌표와 묶어 저장"을 위한 vision 안정화 작업이다. GCS는 `uav-gcs/PROJECT_SPEC.md`에 맞게 onboard telemetry를 표시만 하며, 판단 로직을 다시 수행하지 않는다.

## 절대 원칙

- 기존 line tracing 및 교차점 판단 성능에 영향을 주면 안 된다.
- marker가 없거나 marker-aware 기능이 비활성화된 frame에서는 현재 `LineMaskBuilder`, `LineDetector`, `IntersectionDetector`, `IntersectionDecisionEngine` 결과가 유지되어야 한다.
- `IntersectionDecisionEngine`의 최근 branch evidence window 기반 판단은 수정하지 않는다.
- marker 자체를 교차점으로 간주하지 않는다. 최종 node 채택은 기존 branch evidence로만 결정한다.
- marker 처리는 line/intersection detector 앞단의 optional occlusion 보정으로 제한한다.
- GCS overlay/log는 telemetry 시각화와 검증 보조용이다. mission-critical 판단은 onboard state에 남긴다.

## 현재 판단

현재 white/black 모두 marker가 없거나 marker가 판단을 방해하지 않는 상황에서는 line 및 교차점 판단 로직과 성능이 만족스럽다. 따라서 이번 작업은 detector 전반을 다시 튜닝하는 일이 아니라, "marker가 교차점 위에 있을 때만" line mask 오염과 ArUco 검출 실패를 격리하는 작업이어야 한다.

PLAN의 기존 아이디어 중 camera sweep, full replay tool, GCS overlay 세부 표현은 유용하지만 1차 구현 범위로는 과하다. 먼저 mode별 최소 보정과 회귀 테스트를 통과시키고, 그 다음 필요할 때 확장한다.

## 확인된 문제

테스트 입력:

- `grid_images/black_grid_with_aruco_marker.png`
- `grid_images/white_grid_with_aruco_marker.png`
- `c:/Users/mseoky/Pictures/Screenshots/black_line_only.png`
- `c:/Users/mseoky/Pictures/Screenshots/black_not_line_only.png`
- `c:/Users/mseoky/Pictures/Screenshots/white_line_only.png`
- `c:/Users/mseoky/Pictures/Screenshots/white_not_line_only.png`

공통 pipeline:

```text
ArucoDetector
  -> LineMaskBuilder
  -> LineDetector + IntersectionDetector
  -> IntersectionStabilizer
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> TelemetryMessage
  -> GCS overlay/log
```

관찰:

- `light_on_dark` white T 교차점에서는 ArUco marker 내부의 흰색 bit가 line foreground로 잡힌다.
- white not-line-only에서는 marker가 검출되는 frame이 있어도 marker polygon을 line mask에서 제거하지 않기 때문에 `IX unknown`으로 떨어진다.
- white marker 검출은 blur/JPEG/focus 영향으로 frame마다 잡혔다가 사라지는 flicker가 있다.
- `dark_on_light` black + 교차점에서는 교차점 판단은 비교적 잘 된다.
- black not-line-only에서는 marker 검정 외곽과 검정 line이 붙어 표준 ArUco detector가 marker contour/quiet zone을 찾지 못한다.

## Mode별 실행 계획

### Track A: `light_on_dark` white line

우선순위: 1순위.

문제:

- white line과 ArUco 내부 흰색 bit가 같은 polarity다.
- marker 내부 bit가 line contour와 intersection center candidate를 오염시킨다.
- ArUco ID가 flicker되면 marker polygon 기반 masking도 같이 흔들릴 수 있다.

개선책:

1. marker-aware 기능을 config-gated optional path로 추가한다.
   - 기본 marker 없는 line tracing path는 그대로 둔다.
   - `marker_mask.enabled = true`일 때만 marker 영역을 line mask에서 제외한다.
2. direct ArUco 검출 결과가 있으면 marker polygon을 line mask ignore 영역으로 쓴다.
   - polygon은 marker 외곽보다 line width 1~2배 정도 확장한다.
   - `light_on_dark`에서는 polygon 내부 흰색 bit를 foreground에서 제거한다.
3. direct ArUco가 잠깐 실패하면 최근 polygon을 짧게 hold한다.
   - 3~6 frame 정도만 유지한다.
   - held marker는 telemetry/log에 `source=held`로 남긴다.
4. ArUco ID가 전혀 없어도 high-contrast black square 후보를 optional occluder로 쓸 수 있게 한다.
   - 이 fallback은 marker-size, aspect ratio, intersection 근접 조건을 만족할 때만 활성화한다.
   - 일반 흰 line tracing에 영향을 주지 않도록 넓은 square 후보에만 제한한다.
5. `IntersectionDetector`는 marker 영역을 branch evidence가 아니라 occlusion hole로 본다.
   - marker polygon 바깥쪽 ray continuity가 충분하면 해당 branch를 present로 인정한다.
   - marker 내부 foreground는 branch score에 더하지 않는다.

성공 기준:

- `white_line_only.png`, `white_not_line_only.png`에서 T 교차점이 `decision.accepted_type = T`로 유지된다.
- marker 내부 흰색 bit가 line contour 또는 intersection center로 선택되지 않는다.
- marker 검출이 flicker되어도 held/fallback mask 때문에 T 판단이 흔들리지 않는다.
- marker 없는 기존 white line 테스트와 smoke 결과는 성능 저하가 없다.

### Track B: `dark_on_light` black line

우선순위: 2순위.

문제:

- black line과 ArUco black border/body가 같은 polarity다.
- marker 외곽이 black grid line과 붙으면 ArUco detector가 독립 marker contour와 quiet zone을 찾지 못한다.
- 다만 현재 black + 교차점 branch 판단은 비교적 안정적이므로 line/intersection 경로를 크게 건드리면 안 된다.

개선책:

1. black line/intersection 판단은 최대한 그대로 둔다.
   - marker body가 실제 branch처럼 false-upgrade를 만들 때만 occlusion 보정을 적용한다.
   - black + 판단이 이미 잘 되는 경우에는 `IntersectionDecisionEngine`과 threshold를 바꾸지 않는다.
2. ArUco fallback은 line mask가 아니라 ArUco ROI 검출 경로에 둔다.
   - 교차점 주변에서 line width보다 넓은 검정 square plateau를 marker candidate로 찾는다.
   - candidate ROI에서 grid arm을 suppress하고, 흰 quiet-zone padding을 붙인 뒤 threshold/upscale해서 `detectMarkers`를 재시도한다.
3. fallback 실패 시 물리 설계 보정도 검토한다.
   - black line 위 marker 주변에 흰색 quiet zone 또는 mounting patch를 둔다.
   - marker black border가 black grid line과 직접 닿지 않게 최소 여백을 둔다.
   - 가능하면 marker를 line 위가 아니라 교차점 중심의 별도 흰색 plate 위에 붙인다.
4. fresh detection이 가끔 성공하지만 frame마다 깜빡이는 경우에는 ROI를 키우지 않고 marker temporal hold로 안정화한다.
   - `MarkerStabilizer`가 최근 marker를 `aruco.hold_frames` 동안 유지한다.
   - 기본값은 `hold_frames = 9`이며, 12FPS 기준 약 0.75초다.
   - stale marker가 line mask를 지우지 않도록 occlusion mask에는 fresh detection만 사용한다.
   - line tracking X 보정에는 기존 detection interval 안의 marker만 사용하고, 더 오래 hold된 marker는 ID telemetry 안정화용으로만 쓴다.

성공 기준:

- `black_line_only.png`, `black_not_line_only.png`에서 + 교차점이 `decision.accepted_type = +`로 유지된다.
- black marker 검출률이 개선되되, 기존 black + branch 판단이 나빠지지 않는다.
- fallback이 실패하는 물리 조건은 log와 산출물로 명확히 남기고, 경기장 제작 조건으로 되돌려 판단한다.

## 공통 설계 경계

대상 onboard 파일 후보:

- `uav-onboard/src/vision/ArucoDetector.*`
- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/IntersectionDetector.*`
- `uav-onboard/src/vision/VisionTypes.hpp`
- `uav-onboard/src/app/VisionDebugPipeline.cpp`
- `uav-onboard/src/protocol/TelemetryMessage.*`

대상 GCS 파일 후보:

- `uav-gcs/src/protocol/TelemetryMessage.*`
- `uav-gcs/src/telemetry/TelemetryStore.*`
- `uav-gcs/src/telemetry/VisionLogFormatter.*`
- `uav-gcs/src/overlay/MarkerOverlay.*`
- `uav-gcs/src/overlay/IntersectionOverlay.*`

구현 경계:

- `LineMaskBuilder`에는 optional ignore/occlusion mask만 전달한다. mask가 없으면 기존 결과가 유지되어야 한다.
- `IntersectionDetector`에는 optional marker occlusion polygon만 전달한다. polygon이 없으면 기존 branch scoring이 유지되어야 한다.
- `IntersectionDecisionEngine`은 수정하지 않는다.
- `GridCoordinateTracker`는 marker-node association 저장만 확장하고, 좌표 이동 판단은 기존 branch mask 기준을 유지한다.
- GCS parser는 optional telemetry field를 무시할 수 있어야 하며, 기존 packet 호환성을 깨지 않는다.

## 최소 구현 순서

1. 현재 성능 baseline 고정
   - marker 없는 기존 white/black line, L/T/+ 교차점 테스트 결과를 저장한다.
   - 기존 `grid_image_smoke` 또는 현재 smoke 산출물과 비교 기준을 정한다.
2. 실기 캡처 4장 단일-frame 회귀 테스트 추가
   - black line-only: expected `+`
   - black not-line-only: expected `+`, marker 검출 여부 기록
   - white line-only: expected `T`
   - white not-line-only: expected `T`, marker 검출/held 여부 기록
3. Track A `light_on_dark` 먼저 구현
   - marker polygon/held/fallback occluder를 line mask에서 제거한다.
   - T 판단이 회복되는지 확인한다.
4. Track B `dark_on_light` 구현
   - black + 교차점 판단 non-regression을 먼저 확인한다.
   - ArUco ROI fallback을 추가하고, 실패 시 물리 quiet-zone 필요성을 기록한다.
5. marker-node association telemetry 추가
   - node event와 marker ID/local coord를 묶어 저장한다.
   - marker가 검출되지 않고 occluder만 있는 경우는 association하지 않는다.
6. GCS log/overlay 최소 확장
   - marker source: `direct|held|roi_fallback|none`
   - marker mask used: `yes/no`
   - associated node: `(x,y)`와 marker ID
7. 2 m top-down crop/replay 검증
   - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/white/`
   - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/black/`

## Non-regression 기준

이번 작업은 기존 성능을 보존하는 것이 핵심이다. 다음 중 하나라도 깨지면 실패로 본다.

- marker가 없는 frame에서 line offset, line angle, contour, intersection type이 기존 대비 의미 있게 달라짐.
- marker-aware 기능이 disabled일 때 현재 테스트 결과가 달라짐.
- 기존 white/black line tracing smoke에서 line detected 비율이 떨어짐.
- 기존 L/T/+ 교차점 판단이 `unknown`, 잘못된 T/+/L, 또는 늦은 node event로 바뀜.
- `IntersectionDecisionEngine` threshold 조정으로 marker 없는 일반 교차점 판단 성능이 바뀜.
- GCS overlay/log 변경이 telemetry 수신, video 표시, metadata-only 실행을 깨뜨림.

검증 항목:

- onboard unit/smoke:
  - marker 없는 기존 `test_intersection_detector`
  - 기존 `test_intersection_decision`
  - marker occlusion synthetic test: `light_on_dark`와 `dark_on_light` 분리
  - 실기 캡처 4장 회귀 테스트
- GCS:
  - telemetry parser backward compatibility
  - overlay primitive test
  - marker-node log formatting test
- 산출물:
  - `uav-gcs/logs/<timestamp>-marker-live-capture-baseline/`
  - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/white/`
  - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/black/`

## 권장 테스트 명령

초기 baseline:

```powershell
cd uav-onboard
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
ctest --test-dir build-opencv-tests --output-on-failure
.\build-opencv-tests\aruco_detector_tester.exe --config config --image ..\grid_images\white_grid_with_aruco_marker.png
.\build-opencv-tests\aruco_detector_tester.exe --config config --image ..\grid_images\black_grid_with_aruco_marker.png
```

실기 캡처 회귀 테스트 도구가 추가된 뒤:

```powershell
cd uav-onboard
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\white_line_only.png" --line-mode light_on_dark --expected T
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\white_not_line_only.png" --line-mode light_on_dark --expected T --enable-aruco
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\black_line_only.png" --line-mode dark_on_light --expected +
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\black_not_line_only.png" --line-mode dark_on_light --expected + --enable-aruco
```

GCS:

```powershell
cd uav-gcs
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
ctest --test-dir build-tests --output-on-failure
```

## 보류할 항목

다음은 필요하지만 1차 구현 범위에서는 뒤로 미룬다.

- full dashboard UI
- 과도한 GCS overlay 색상/스타일 구분
- 모든 ArUco detector parameter 튜닝 UI
- 장시간 focus/JPEG/exposure sweep 자동화
- full mission snake/revisit policy 구현
- MAVLink/Pixhawk control loop

## 리스크와 판단 기준

- white line에서는 marker 내부 흰색 bit를 line에서 제거하지 않으면 ArUco 검출률을 올려도 T 판단이 계속 깨질 수 있다.
- black line에서는 software fallback만으로 ArUco 검출이 어려울 수 있다. marker black border와 black grid line이 붙으면 물리적으로 quiet zone이 사라지기 때문이다.
- 모니터 촬영은 실제 인쇄 경기장보다 blur, refresh artifact, moire가 크다. 실기 캡처는 worst-case 회귀 입력으로 쓰되, 최종 튜닝은 실제 경기장 재질과 조명에서 다시 해야 한다.
- marker-aware 보정은 detector layer에 국한한다. decision layer special-case는 현재 안정적인 branch evidence 정책을 흔들 수 있으므로 사용하지 않는다.

## 아직 해결하지 못한 것
- 검정 격자 기준 ArUco marker ID의 짧은 깜빡임은 `MarkerStabilizer` hold로 1차 완화했다.
- marker가 거의 한 번도 fresh detection되지 않는 조건은 아직 남을 수 있다. 이 경우에는 software hold가 아니라 marker 주변 흰색 quiet zone/plate 같은 물리 조건 또는 중앙부 template fallback 주기 튜닝이 필요하다.
