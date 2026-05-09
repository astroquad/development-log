# ArUco marker at intersection 대응 계획

최종 업데이트: 2026-05-10

## 목표

3x3 cell 격자 경기장 이미지(`grid_images/black_grid_with_aruco_marker.png`, `grid_images/white_grid_with_aruco_marker.png`)를 2 m 고도 IMX519 top-down 카메라 입력처럼 crop/replay하면서, ArUco marker가 교차점 위에 있어도 onboard가 기존 L/T/+ 교차점 판단 방식을 유지하고 GCS가 marker/intersection/grid overlay와 log를 올바르게 남기도록 만든다.

중요 원칙:

- `IntersectionDecisionEngine`의 최근 branch evidence window 기반 판단은 유지한다.
- marker 자체를 "교차점으로 간주"하는 shortcut은 넣지 않는다.
- marker는 line/intersection detector의 occlusion/false-positive 요인으로만 보정하고, 최종 node 채택은 기존 branch evidence가 결정한다.
- GCS overlay는 telemetry를 시각화만 한다. mission-critical 판단은 onboard telemetry/state에 남긴다.

## 현재 확인한 사실

- 현재 pipeline은 `ArucoDetector -> LineMaskBuilder -> LineDetector + IntersectionDetector -> IntersectionStabilizer -> IntersectionDecisionEngine -> GridCoordinateTracker -> TelemetryMessage -> GCS overlay/log` 흐름이다.
- 현재 `ArucoDetector`는 OpenCV `cv::aruco::ArucoDetector`를 원본 gray frame에 바로 적용한다.
- 현재 `IntersectionDetector`는 line mask에서 center candidate를 찾고, front/right/back/left ray score로 branch mask를 만든다.
- 현재 `IntersectionDecisionEngine`은 detector의 branch score를 window로 모아 `+`, `T`, `L`, `straight`를 결정한다.
- 현재 GCS는 marker polygon, line, intersection branch overlay를 이미 갖고 있고, telemetry parser도 `vision.markers`, `vision.intersection`, `vision.intersection_decision`, `vision.grid_node`를 읽는다.
- `grid_image_smoke`는 기존 smoke 산출물 생성기가 있지만, 현재 geometry 검출이 white line 중심이고 crop 크기도 실제 2 m IMX519 footprint보다 넓다. 이번 검증 목적에는 확장이 필요하다.
- baseline `aruco_detector_tester` 결과:
  - `white_grid_with_aruco_marker.png`: `DICT_4X4_50` marker 4개 검출.
  - `black_grid_with_aruco_marker.png`: marker 0개 검출. 검정 marker 외곽이 검정 grid line과 붙어 OpenCV가 독립 marker contour를 못 잡는 케이스로 보인다.

## 2026-05-10 실기 캡처에서 추가 확인한 현상

테스트 조건:

- 모니터에 격자 경기장을 띄우고 IMX519 top-down으로 촬영했다.
- 화면상의 크기는 2 m 고도에서 경기장을 보는 정도의 비율에 맞췄다.
- 캡처 파일:
  - `c:/Users/mseoky/Pictures/Screenshots/black_line_only.png`
  - `c:/Users/mseoky/Pictures/Screenshots/black_not_line_only.png`
  - `c:/Users/mseoky/Pictures/Screenshots/white_line_only.png`
  - `c:/Users/mseoky/Pictures/Screenshots/white_not_line_only.png`

관찰:

- black `+` 교차점:
  - line-only에서는 ArUco 검출이 꺼져 있으므로 marker overlay가 없는 것이 정상이다.
  - not-line-only에서도 marker는 전혀 검출되지 않았다.
  - 교차점 branch 판단은 white보다 안정적이며 `IX +`까지 나오는 편이다.
  - 원인은 marker의 검정 외곽/몸체가 검정 grid line과 연결되어 표준 ArUco detector가 marker의 독립 사각 contour와 outer quiet zone을 찾지 못하는 것으로 본다.
- white `T` 교차점:
  - line-only에서 ArUco marker 내부의 흰색 cell이 line mask foreground로 잡힌다.
  - 실제 T branch보다 marker 내부 흰색 패턴이 더 강한 후보가 되어 `IX unknown` 또는 잘못된 center/ray가 나온다.
  - not-line-only에서는 marker가 잡히는 frame도 있지만, intersection detector가 marker polygon을 line mask에서 제거하지 않으므로 여전히 `IX unknown`이 나온다.
  - marker 검출은 frame마다 잡혔다가 사라지는 flicker가 있다. 현재 캡처 품질에서는 camera blur, display blur, JPEG 압축, focus 오차가 같이 영향을 주는 것으로 본다.

추론:

- 문제는 두 층으로 분리해야 한다.
  - `marker identification`: marker ID/corners를 안정적으로 읽는 문제.
  - `marker occlusion masking`: marker ID를 못 읽은 frame에서도 marker-like 영역이 line/intersection 판단을 망치지 않게 하는 문제.
- white 실패는 "marker ID를 읽느냐"보다 "marker 내부 흰색을 line으로 쓰지 않게 하는 것"이 먼저다.
- black 실패는 소프트웨어 fallback으로 완화할 수 있지만, 검정 line과 marker black border가 물리적으로 붙으면 ArUco가 요구하는 외곽 contrast/quiet zone이 사라진다. 실제 경기장에서는 marker 주변에 흰색 여백 또는 line과 구분되는 mounting patch가 있는지 확인해야 한다.

## 전제와 좌표 해석

- 사용자가 말한 3x3은 3x3 cell 경기장으로 해석한다. 따라서 line 교차점은 4x4 lattice가 된다.
- 실제 cell 길이 4.0 m, line 폭 0.1 m, marker 크기 0.5 m를 검증 tool의 물리 scale로 사용한다.
- full-field 이미지에서 grid line center 간격을 검출해 `px_per_meter = median_cell_spacing_px / 4.0`으로 환산한다.
- 2 m 고도 crop은 IMX519 FOV를 config 값으로 둔다. 초기값은 IMX519-78 기준으로 두되, 실제 렌즈/카메라 calibration 후 수치만 바꿀 수 있게 한다.
- replay 출력 frame 크기는 현재 onboard camera config와 맞춰 `960x720`으로 resize한다.

## 구현 방안

### 0. 카메라/입력 품질 baseline을 먼저 고정하기

대상 파일:

- `uav-onboard/config/vision.toml`
- `uav-onboard/tools/vision_debug_node.cpp`
- `uav-onboard/tools/aruco_detector_tester.cpp`
- 검증 산출물 `uav-gcs/logs/.../capture_baseline/`

방안:

1. 2 m top-down 조건에서 focus sweep을 먼저 수행한다.
   - `lens_position` 후보를 여러 값으로 돌려 marker corner sharpness, ArUco 검출률, line mask 안정성을 기록한다.
   - 기존 기본값 `0.67`은 고정값으로 가정하지 않는다.
2. onboard detection 입력의 JPEG 품질을 올려 비교한다.
   - 현재 camera `jpeg_quality = 45`는 telemetry/debug에는 가볍지만 ArUco corner와 내부 bit pattern에는 불리할 수 있다.
   - 2 m marker test에서는 `80~90` 후보를 baseline으로 비교한다.
   - GCS debug video 품질이 아니라 onboard가 decode하는 camera MJPEG 품질을 기준으로 본다.
3. shutter/exposure를 blur가 적은 쪽으로 고정한다.
   - 모니터 촬영 테스트는 flicker/refresh 영향이 있으므로 실제 인쇄 경기장과 분리해서 기록한다.
   - 움직이는 드론에서는 motion blur가 더 커질 수 있으므로 `shutter_us`, gain, exposure mode 후보를 별도 표로 남긴다.
4. ArUco detector tester에 blur/contrast 지표를 출력하는 옵션을 추가한다.
   - Laplacian variance 또는 corner gradient score를 `README.md`/summary에 남긴다.
   - "검출 실패"와 "영상이 흐림"을 분리해서 판단한다.

성공 기준:

- 같은 marker를 300 frame 이상 볼 때 direct ArUco 검출률, held marker 검출률, line decision 성공률을 함께 기록한다.
- white/black 각각 camera quality/focus 변경 전후 결과를 `uav-gcs/logs`에 남긴다.

### 1. ArUco detector를 marker-contour 병합과 blur에 강하게 만들기

대상 파일:

- `uav-onboard/src/vision/ArucoDetector.*`
- `uav-onboard/src/common/VisionConfig.*`
- `uav-onboard/tools/aruco_detector_tester.cpp`
- `uav-onboard/tests/` 신규 또는 기존 테스트

방안:

1. 기존 원본 frame `detectMarkers`를 1차 경로로 유지한다.
2. OpenCV ArUco parameter를 config로 확장한다.
   - `corner_refinement_method`, `corner_refinement_win_size`, `min_corner_distance_rate`, `polygonal_approx_accuracy_rate`, `min_otsu_std_dev`, `error_correction_rate`, `min_distance_to_border` 후보를 노출한다.
   - blur가 있는 live capture에서는 ROI upscale 후 adaptive threshold를 적용하는 경로를 비교한다.
3. direct 검출 결과는 temporal stabilizer로 hold한다.
   - 같은 ID/corner가 최근 N frame 안에서 안정적으로 보였으면 3~6 frame 정도 marker polygon을 유지한다.
   - held marker는 `source=held`로 표시하고, ID association에는 쓰되 새 ID 발견처럼 취급하지 않는다.
4. 1차 검출 실패 또는 line polarity가 `dark_on_light`일 때 fallback을 추가한다.
5. fallback은 dark/bright polarity별 marker candidate ROI를 찾고, grid line과 marker 외곽이 붙은 경우를 분리한다.
   - dark mask에서 distance transform을 사용해 0.1 m line보다 훨씬 두꺼운 0.5 m marker core 후보를 분리한다.
   - 후보의 aspect ratio, 면적, side length 범위를 config로 제한한다.
   - 후보 주변 ROI를 추출하고, marker square 밖으로 이어지는 긴 수평/수직 grid line 성분을 배경색으로 suppress한 뒤 OpenCV ArUco 검출을 다시 수행한다.
   - 검정 line 위 검정 marker는 독립 contour가 없을 수 있으므로, intersection 주변 "line 폭보다 넓은 검정 사각 plateau"를 후보로 삼아 ROI를 만든다.
   - ROI에는 인위적인 흰색 quiet zone padding을 붙이고, outside-grid-arm suppression 후 threshold/upscale해서 `detectMarkers`를 다시 시도한다.
   - 필요하면 inverted ROI도 시도하되, 검출 결과는 중복 제거한다.
6. marker observation에 품질 정보를 추가할지 검토한다.
   - 최소 후보: `id`, `center_px`, `corners_px`, `orientation_deg`는 유지.
   - 추가 후보: `source = direct|roi_fallback|held`, `side_px`, `confidence_like`, `blur_score`, `rejected_reason`.
   - protocol은 top-level `protocol_version = 1`을 유지하고 optional field만 추가한다.

성공 기준:

- white 이미지에서 기존처럼 4개 검출.
- black 이미지에서 marker 4개 검출.
- synthetic 또는 real crop에서 marker가 line과 붙어도 중복 검출이 생기지 않음.
- 실기 white not-line-only 캡처처럼 direct 검출이 flicker되어도 held polygon은 line masking에 계속 공급된다.

### 2. marker-like occluder mask를 line mask보다 먼저 만들기

대상 파일:

- `uav-onboard/src/vision/ArucoDetector.*`
- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/VisionTypes.hpp`
- `uav-onboard/src/app/VisionDebugPipeline.cpp`
- 신규 `MarkerMaskBuilder` 또는 `MarkerStabilizer`

방안:

1. marker ID가 읽힌 경우:
   - marker polygon을 line mask의 ignore/occlusion 영역으로 변환한다.
   - polygon을 marker 외곽보다 line width 1~2배만큼 확장한다.
2. marker ID가 안 읽힌 경우:
   - black square candidate, high-contrast square candidate, 이전 held marker polygon으로 `marker_like_occluder`를 만든다.
   - white line mode에서는 검정 square 내부의 흰색 ArUco bit pattern을 line foreground에서 제거한다.
   - black line mode에서는 검정 marker body가 line foreground로 합쳐지는 것을 occlusion으로 취급한다.
3. `LineMaskBuilder`는 marker 영역 내부를 foreground로 쓰지 않는다.
   - `light_on_dark`: marker polygon 내부의 흰색 bit cell을 제거한다.
   - `dark_on_light`: marker polygon 내부의 검정 body를 제거한다.
   - 제거 후 끊긴 line은 `IntersectionDetector`의 branch ray scoring에서 occlusion bridge로 보정한다.
4. debug telemetry에 marker mask 상태를 추가한다.
   - `marker_mask_source=direct|fallback|held|none`
   - `marker_mask_count`
   - `marker_mask_area_px`
   - `marker_mask_used_for_line=yes/no`

성공 기준:

- `white_line_only.png`와 같은 조건에서 marker 내부 흰색 pattern이 line contour로 선택되지 않는다.
- `white_not_line_only.png`처럼 marker가 검출된 frame에서도 intersection detector가 marker 내부가 아니라 실제 T branch를 기준으로 판단한다.
- marker ID가 순간적으로 사라져도 held/fallback mask 때문에 line decision이 흔들리지 않는다.

### 3. line/intersection 판단은 marker-aware occlusion 보정만 추가하기

대상 파일:

- `uav-onboard/src/vision/IntersectionDetector.*`
- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/VisionTypes.hpp`
- `uav-onboard/src/app/VisionDebugPipeline.cpp`
- `uav-onboard/tests/test_intersection_detector.cpp`

방안:

1. `IntersectionDetector::detect`에 marker observations 또는 marker occlusion polygons를 전달할 수 있는 overload를 추가한다.
2. marker polygon을 `LineMaskGeometry`의 work 좌표로 변환하고, line 폭만큼 expand한 occlusion mask를 만든다.
3. branch ray scoring에서 marker 영역 안의 sample은 denominator에서 제외하거나 marker 뒤쪽부터 scoring을 시작한다.
   - 목적은 marker 때문에 line이 끊겨도 branch score가 과도하게 낮아지지 않게 하는 것이다.
   - 반대로 검정 marker body가 큰 foreground blob으로 잡혀 `+`로 false-upgrade되는 것도 막아야 한다.
4. center candidate 선택에서는 marker blob 중심만으로 교차점 candidate가 되지 않도록 한다.
   - line mask component centroid 후보를 쓰되, marker occlusion 영역 내부 foreground density는 center validity score에서 감점 또는 제외한다.
5. white T 교차점 전용 보강:
   - marker polygon과 실제 line이 겹친 방향은 "occluded branch candidate"로 표시하고, polygon 바깥쪽 ray가 충분히 이어지면 branch present로 인정한다.
   - marker 내부 foreground는 branch 증거가 아니라 occlusion hole로만 취급한다.
6. black + 교차점 전용 보강:
   - marker body가 line과 붙은 큰 검정 component를 `+` branch 자체로 쓰지 않는다.
   - branch present는 marker polygon 바깥의 4방향 line continuity로 판단한다.
7. marker-aware detector가 내보내는 것은 여전히 `IntersectionDetection.branches[].score/present`, `branch_mask`, `type`이다. `IntersectionDecisionEngine`은 수정하지 않는 것을 기본 방침으로 한다.

성공 기준:

- marker 없는 기존 L/T/+/straight 테스트가 그대로 통과.
- marker가 중심을 가리는 L/T/+ synthetic 테스트가 light/dark polarity 모두 통과.
- marker가 검정 line과 붙은 black-grid crop에서도 branch mask가 marker square 때문에 `+`로 잘못 올라가지 않음.
- 실제 캡처 기준:
  - black `+`: line-only와 not-line-only 모두 `decision.accepted_type = +`.
  - white `T`: line-only와 not-line-only 모두 `decision.accepted_type = T`.
  - white not-line-only에서 marker ID가 검출되든 held되든 intersection 판단은 `unknown`으로 떨어지지 않음.

### 4. marker와 grid node association 저장하기

대상 파일:

- `uav-onboard/src/mission/GridCoordinateTracker.*`
- `uav-onboard/src/protocol/TelemetryMessage.*`
- `uav-gcs/src/protocol/TelemetryMessage.*`
- `uav-gcs/src/telemetry/TelemetryStore.*`
- `uav-gcs/src/telemetry/VisionLogFormatter.*`

방안:

1. `GridNodeEvent` 또는 별도 `MarkerNodeAssociation`에 node와 marker 연결 정보를 optional로 저장한다.
2. association 조건:
   - `intersection_decision.event_ready == true`인 frame 또는 최근 N frame 안의 marker를 대상으로 한다.
   - marker center와 accepted intersection center의 거리가 marker side 또는 node radius 기반 threshold 이내여야 한다.
   - 같은 node lockout 기간에는 같은 marker를 중복 기록하지 않는다.
3. telemetry optional field 예시:
   - `vision.grid_node.marker_id`
   - `vision.grid_node.marker_center_px`
   - `vision.grid_node.marker_distance_px`
   - `vision.grid_node.marker_associated`
   - 또는 `vision.marker_node` 별도 object
4. GCS parser는 field가 없어도 기존 telemetry를 정상 처리하게 default 값을 둔다.
5. marker association은 mission 판단을 바꾸지 않고 log/overlay/추후 mission policy 입력으로만 저장한다.

성공 기준:

- marker가 있는 교차점에서 node event가 발생하면 marker id와 local coord가 같은 telemetry packet 또는 같은 replay step 산출물에 기록된다.
- marker가 frame 안에 있어도 아직 node event가 아니면 grid node에 연결하지 않는다.

### 5. GCS overlay와 vision log 개선

대상 파일:

- `uav-gcs/src/overlay/MarkerOverlay.*`
- `uav-gcs/src/overlay/IntersectionOverlay.*`
- `uav-gcs/src/app/VisionDebugApp.cpp`
- `uav-gcs/src/telemetry/VisionLogFormatter.*`
- `uav-gcs/tests/test_intersection_overlay.cpp`
- 신규 marker/grid overlay 테스트

방안:

1. 기존 marker polygon overlay는 유지한다.
2. marker mask/held/fallback 상태를 overlay에 구분해 표시한다.
   - direct ArUco: 기존 녹색 marker polygon.
   - held marker: 점선 또는 다른 색상.
   - fallback occluder only: ID 없는 사각 mask를 얇은 회색/주황색 box로 표시.
3. associated marker가 있으면 교차점 label 또는 marker label에 node coord를 같이 표시한다.
   - 예: `ID 2 node(1,2)` 또는 `M2 @(1,2)`.
4. `IntersectionOverlay`가 현재 decision 인자를 거의 쓰지 않으므로, accepted decision label을 실제 overlay에 반영한다.
   - detector raw type과 decision accepted type이 다르면 둘 다 보이게 한다.
5. `VisionLogFormatter`에 marker-node association과 marker mask 상태를 추가한다.
   - `[grid-node] ... marker_id=2 marker_dist=...`
   - `[marker-mask] source=direct|fallback|held|none count=... area=... used_for_line=yes`
   - `[markers] ... associated_node=(x,y)` 또는 별도 `[marker-node]` line
6. grid map pane은 기본 ASCII 구조를 유지하되, marker id 표시가 필요하면 node line에만 남기고 map 문자는 과도하게 복잡하게 만들지 않는다.

성공 기준:

- GCS video overlay에서 marker outline, branch rays, accepted node label이 서로 모순 없이 보인다.
- log만 봐도 어떤 marker가 어떤 local grid node에 저장됐는지 확인 가능하다.

## 검증/replay 계획

### 검증 tool 확장

기존 `grid_image_smoke`를 확장하거나 새 tool `marker_grid_replay`를 만든다. 기존 tool을 과하게 복잡하게 만들면 새 tool이 낫다.

필수 기능:

1. 입력:
   - `--image ../../grid_images/black_grid_with_aruco_marker.png`
   - `--image ../../grid_images/white_grid_with_aruco_marker.png`
   - `--line-mode dark_on_light|light_on_dark`
   - `--cell-m 4.0 --line-width-m 0.1 --marker-m 0.5 --altitude-m 2.0`
   - `--camera-width 960 --camera-height 720`
   - `--fov-mode diagonal|horizontal --fov-deg <value>`
2. geometry:
   - white grid는 bright line mask로 line centers 검출.
   - black grid는 dark line mask로 line centers 검출.
   - 3x3 cell이면 4x4 intersection lattice로 expected topology를 만든다.
3. replay:
   - full-field image를 고정하고 drone crop center를 snake path를 따라 이동시킨다.
   - crop은 camera heading 기준으로 회전해 현재 onboard의 "진행 방향 앞쪽" 판단과 맞춘다.
   - 각 node마다 approach frames를 여러 장 만든다. centered crop 한 장만 쓰지 않는다.
   - `frame_seq`, `timestamp_ms = frame_seq * 83`으로 12 FPS window 판단을 재현한다.
4. detector 실행:
   - ArUco, line, intersection, stabilizer, decision, grid tracker를 실제 pipeline과 최대한 같은 순서로 호출한다.
   - GCS overlay builder도 호출해 overlay PNG를 만든다.
5. 산출물:
   - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/black/`
   - `uav-gcs/logs/<timestamp>-aruco-marker-grid-smoke/white/`
   - 각 폴더에 `README.md`, `config.json`, `geometry.csv`, `crop_manifest.csv`, `telemetry.jsonl`, `summary.csv`, `failures.csv`, `grid_map.txt`, `frames/`, `overlays/` 저장.

### Acceptance criteria

white/black 각각:

- marker 검출률: full-field 및 replay crop에서 expected marker가 검출되어야 한다.
- marker mask 안정성: direct marker 검출이 실패한 frame에서도 held/fallback occluder mask가 line 판단에 공급되어야 한다.
- marker-node association: marker가 놓인 교차점에서 `marker_id -> local_coord`가 기록되어야 한다.
- node 판단: marker가 있는 교차점에서도 expected L/T/+와 `decision.accepted_type`이 일치해야 한다.
- branch mask: marker 때문에 `L -> T/+`, `T -> +`로 false-upgrade되지 않아야 한다.
- snake 좌표: replay path의 expected local coord와 `GridCoordinateTracker` node coord가 일치해야 한다.
- overlay: 각 marker event 주변 frame의 overlay PNG에서 marker polygon, branch rays, accepted decision, node label이 같은 위치를 가리켜야 한다.
- 성능: 추가 ArUco fallback과 marker-aware scoring 후에도 12 FPS 기준 frame budget을 넘는 frame이 summary에 표시되어야 하며, 평균 processing latency 목표는 83 ms 미만으로 둔다.

### 권장 실행 명령

초기 baseline:

```powershell
cd uav-onboard
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
.\build-opencv-tests\aruco_detector_tester.exe --config config --image ..\grid_images\white_grid_with_aruco_marker.png
.\build-opencv-tests\aruco_detector_tester.exe --config config --image ..\grid_images\black_grid_with_aruco_marker.png
```

구현 후 검증 예시:

```powershell
cd uav-onboard
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
cmake --build build-opencv-tests
ctest --test-dir build-opencv-tests --output-on-failure
.\build-opencv-tests\marker_grid_replay.exe --config config --image ..\grid_images\white_grid_with_aruco_marker.png --line-mode light_on_dark --output ..\uav-gcs\logs\<timestamp>-aruco-marker-grid-smoke\white
.\build-opencv-tests\marker_grid_replay.exe --config config --image ..\grid_images\black_grid_with_aruco_marker.png --line-mode dark_on_light --output ..\uav-gcs\logs\<timestamp>-aruco-marker-grid-smoke\black
```

GCS 테스트:

```powershell
cd uav-gcs
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

실기 캡처 회귀 테스트:

```powershell
cd uav-onboard
$env:PATH="C:\msys64\ucrt64\bin;$env:PATH"
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\black_line_only.png" --line-mode dark_on_light --expected +
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\black_not_line_only.png" --line-mode dark_on_light --expected + --enable-aruco
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\white_line_only.png" --line-mode light_on_dark --expected T
.\build-opencv-tests\marker_frame_replay.exe --config config --image "c:\Users\mseoky\Pictures\Screenshots\white_not_line_only.png" --line-mode light_on_dark --expected T --enable-aruco
```

## 작업 순서

1. 실기 캡처 4장을 baseline 회귀 입력으로 등록한다. 현재 실패/성공 상태를 `uav-gcs/logs/<timestamp>-marker-live-capture-baseline/`에 남긴다.
2. 카메라 focus/JPEG/exposure baseline을 잡고, blur score와 ArUco 검출률을 함께 기록한다.
3. marker stabilizer와 marker-like occluder mask를 먼저 구현한다. 이 단계의 목표는 white T에서 marker 내부 흰색 pattern을 line으로 쓰지 않게 하는 것이다.
4. marker-aware line/intersection scoring을 구현하고 synthetic L/T/+ marker occlusion unit test 및 실기 캡처 4장 회귀 테스트를 통과시킨다.
5. ArUco detector fallback을 구현하고 `aruco_detector_tester`로 black/white full-field 검출을 통과시킨다.
6. black marker가 계속 실패하면 소프트웨어 fallback 결과와 함께 물리 설계 변경안을 판단한다.
   - black line 위 marker에는 흰색 quiet zone/mounting patch를 추가한다.
   - marker black border가 black grid line과 직접 닿지 않게 최소 여백을 둔다.
   - 가능하면 marker를 line 위가 아니라 교차점 중심의 별도 흰색 plate 위에 붙인다.
7. marker-node association telemetry를 추가하고 onboard/GCS protocol parser 테스트를 갱신한다.
8. GCS overlay/log에 association 및 marker-mask 상태 표시를 추가하고 overlay 테스트를 갱신한다.
9. marker grid replay를 2 m IMX519 crop 방식으로 실행해 `uav-gcs/logs`에 black/white 결과를 남긴다.
10. 결과를 `development-log/RESEARCH.md` 또는 별도 검증 메모에 요약한다.

## 리스크와 판단 기준

- black grid 이미지처럼 marker 외곽이 검정 line과 완전히 붙으면 표준 ArUco 검출만으로는 부족하다. fallback이 실패하면 실제 경기장 marker 주변에 밝은 quiet zone을 확보하는 물리 설계 변경도 후보로 남긴다.
- white grid에서는 marker 내부 흰색 bit가 line mask foreground와 같은 polarity라서, ArUco ID 검출률이 좋아져도 marker polygon을 line mask에서 제거하지 않으면 T 판단은 계속 깨질 수 있다.
- ArUco 검출 flicker는 marker ID 저장 문제뿐 아니라 line masking 문제도 만든다. direct 검출 결과만 쓰지 말고 held/fallback occluder mask를 별도 상태로 유지해야 한다.
- 모니터 촬영은 실제 인쇄 경기장보다 blur, refresh artifact, moire가 클 수 있다. 다만 이번 캡처는 detector가 견뎌야 할 worst-case 회귀 입력으로 보관한다.
- IMX519 FOV와 실제 focus/exposure가 다르면 2 m crop footprint가 달라진다. replay tool은 FOV를 config화하고, 실제 카메라 calibration 값으로 교체 가능해야 한다.
- debug video는 best-effort라 검증의 기준이 아니다. 최종 pass/fail은 replay telemetry/log/summary와 onboard state 기준으로 판단한다.
- marker-aware 보정은 detector layer에 국한한다. decision layer까지 marker special-case가 들어가면 현재 안정화된 branch evidence 정책을 흔들 수 있으므로 마지막 수단으로 둔다.
