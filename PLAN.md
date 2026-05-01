# Astroquad Vision/Grid 다음 개발 계획

최종 업데이트: 2026-05-01

이 문서는 `development-log/RESEARCH.md`, `uav-gcs/README.md`, `uav-onboard/README.md`와 2026-05-01 사용자 요청을 기준으로 작성한 다음 작업 계획이다. 추후 에이전트는 이 파일을 먼저 읽고, 아래 범위 안에서 바로 구현을 진행한다.

## 1. 현재 프로젝트 이해

Astroquad는 Raspberry Pi 4 + IMX519-78 onboard와 Windows GCS를 나눈 C++ 프로젝트다.

- `uav-onboard`: rpicam MJPEG capture, ArUco/line/intersection detection, intersection decision, local grid node telemetry 생성.
- `uav-gcs`: UDP telemetry/video 수신, raw camera 영상 위 GCS-side overlay, 별도 vision log window 표시.
- 현재 목표 단계는 실제 Pixhawk/MAVLink 제어가 아니라 비전/디버그/좌표화 완성이다.
- onboard는 overlay를 그리지 않는다. 모든 marker/line/intersection/debug overlay는 GCS에서 그린다.
- protocol 문서는 v1.7이며 JSON `protocol_version`은 호환성 때문에 integer `1`이다.

이미 구현된 핵심 요소:

- `uav-onboard/src/vision/LineMaskBuilder.*`: line/intersection 공통 mask 생성.
- `uav-onboard/src/vision/LineDetector.*`: line following용 contour와 tracking point 산출.
- `uav-onboard/src/vision/IntersectionDetector.*`: front/right/back/left ray score 기반 `+`, `T`, `L`, `straight` 분류.
- `uav-onboard/src/mission/IntersectionDecision.*`: 최근 branch evidence window로 node event, turn candidate, front availability, overshoot risk 산출.
- `uav-onboard/src/mission/GridCoordinateTracker.*`: 첫 안정 node를 local `(0,0)`으로 기록하고 heading 기준 다음 node 좌표를 증가.
- `uav-gcs/src/telemetry/VisionLogFormatter.*`: vision log window 텍스트 생성.
- `uav-gcs/src/overlay/LineOverlay.*`, `IntersectionOverlay.*`: 현재 camera overlay primitive 생성.

아직 부족한 핵심 요소:

- 전체 grid를 동적으로 누적하는 GCS-side map/ASCII renderer.
- 실제 snake traversal policy. 현재 `VisionDebugPipeline.cpp`에서 `IntersectionDecisionEngine::update(..., false)`로 `turn_expected`가 항상 `false`다.
- `GridCoordinateTracker` heading 갱신. 현재 live pipeline에서 `notifyTurnCompleted()` 또는 `setHeading()`을 실제 policy와 연결하지 않는다.
- 라인 중심점 overlay 단순화와 카메라 중앙 기준 horizontal offset 표시.
- 굵은 흰색 line을 하나의 채워진 blob으로 더 안정적으로 잡는 mask 개선.

## 2. 사용자 요구사항 정리

### 2.1 GCS vision log에 동적 grid 표시

GCS 로그 화면에 실시간 ASCII grid를 표시한다.

- `s`에서 출발해 처음 grid에 도착한 교점을 local `(0,0)`으로 본다.
- 처음에는 `s -> (0,0)` 진입 라인만 그릴 수 있다.
- 새로운 교차점 node event가 들어올 때마다 grid map을 갱신한다.
- grid 전체 크기는 처음부터 알 수 없으므로 visited/discovered node의 bounding box 기준으로 화면을 확장한다.
- 현재 드론 위치와 heading을 `>`, `<`, `^`, `v` 중 하나로 표시한다.
- 방문한 좌표는 누적 저장하고, 이미 방문한 좌표는 snake traversal이 다시 선택하지 않게 한다.
- 최종적으로 5x5처럼 전체 node를 방문하면 아래와 같은 형태의 ASCII grid가 로그에 보인다.

```text
<---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|
s
```

### 2.2 Snake 순회 정책

현재 비전 개발 단계에서는 사람이 top-down camera를 들고 드론처럼 움직인다고 가정한다. 즉, 카메라는 항상 현재 진행 방향을 정면으로 보며 이동한다.

요구되는 snake 동작:

- 첫 안정 교점 `(0,0)`에서 snake를 시작한다.
- 한 row를 직진하다가 row 끝 또는 policy상 turn 지점에 도달하면 같은 방향 90도 회전을 두 번 수행한다.
- 예: 처음 row 끝의 `L` 교차로에서는 우회전, 한 칸 이동, 다시 우회전 후 다음 row를 반대 방향으로 직진한다.
- 다음 row 끝에서는 좌회전, 한 칸 이동, 다시 좌회전 후 다시 반대 방향으로 직진한다.
- 이처럼 row가 바뀔 때마다 turn side가 `right -> left -> right -> ...`로 번갈아야 한다.
- 중요한 점: 전진 branch가 있어도 snake policy상 row 전환이 필요하면 회전해야 한다. 따라서 `front_available == true`만으로 `continue straight`를 결정하면 안 된다.
- 아직 실제 제어 코드는 작성하지 않는다. 이번 단계에서는 vision/debug telemetry와 grid/snake 상태가 맞게 나오는지 확인하는 것이 목표다.

### 2.3 GCS camera overlay 단순화

현재 GCS overlay는 정보가 너무 많다. 다음처럼 줄인다.

- 현재 tracing 중인 line 중심점만 빨간 원으로 표시한다.
- 빨간 원은 반드시 카메라 화면의 수직 중앙선 높이에 놓는다. 즉 `y = frame_height / 2`로 고정한다.
- 초록색 라인은 카메라 중심점 `(frame_width / 2, frame_height / 2)`에서 빨간 원까지 수평으로 표시한다.
- 이 초록색 라인은 나중에 line tracing 제어에서 사용할 lateral offset을 직관적으로 보여주기 위한 것이다.
- line label, confidence text, branch score text, cyan/yellow debug label처럼 화면을 복잡하게 만드는 정보는 기본 overlay에서 제거하거나 debug option 뒤로 숨긴다.
- 교차점 overlay는 카메라 중심 기준 위쪽 범위 안에서 포착되는 "앞으로 도착할 교차점" 또는 현재 도착한 교차점의 branch 방향만 간단히 표시한다.
- 교차점 표시는 모양 또는 방향 요약만 보여준다. 예: `+`, `T`, `L`, `F/R/L` 같은 compact 정보.

### 2.4 굵은 흰색 line 인식 개선

현재는 굵은 흰색 line 전체가 아니라 일부 edge 또는 얇은 strip만 magenta contour로 잡히는 경향이 있다.

목표:

- 첫 번째/두 번째 참고 이미지처럼 분홍색 overlay가 굵은 line 전체 외곽을 하나의 연결된 blob으로 감싸야 한다.
- 밝은 배경, 넓은 흰색 테이프/종이, 조명 gradient에서도 line 내부가 비지 않게 mask를 보강한다.
- line tracking point는 contour centroid가 아니라 카메라 중앙 높이의 horizontal scan band에서 line 폭 중심을 사용한다.

## 3. 권장 구현 순서

이번 작업은 아래 순서로 진행한다. 실제 MAVLink 제어는 이 계획의 범위 밖이다.

1. GCS grid map/ASCII renderer를 먼저 만든다.
2. onboard snake policy/debug state를 만들고 `turn_expected`와 heading 갱신을 연결한다.
3. GCS overlay를 단순화한다.
4. line mask를 굵은 흰색 line에 맞게 개선한다.
5. 이미지 smoke/replay와 unit test를 추가한다.

## 4. GCS 동적 grid map 설계

### 4.1 파일 위치

새 파일 후보:

- `uav-gcs/src/telemetry/GridMapTracker.hpp`
- `uav-gcs/src/telemetry/GridMapTracker.cpp`
- `uav-gcs/tests/test_grid_map_tracker.cpp`

수정 후보:

- `uav-gcs/src/telemetry/VisionLogFormatter.hpp`
- `uav-gcs/src/telemetry/VisionLogFormatter.cpp`
- `uav-gcs/src/telemetry/TelemetryStore.hpp`
- `uav-gcs/src/telemetry/TelemetryStore.cpp`
- `uav-gcs/src/app/VisionDebugApp.cpp`
- `uav-gcs/CMakeLists.txt`

### 4.2 상태 모델

`GridMapTracker`는 GCS process 안에서 telemetry frame을 순서대로 관찰한다.

필수 상태:

- `std::map<GridCoord, Node>` visited/discovered nodes.
- `std::set<Edge>` discovered edges.
- `std::optional<GridCoord> current_coord`.
- `Heading current_heading`.
- `bool saw_start`.
- `GridCoord origin = {0,0}`.
- `uint32_t last_node_id` 또는 마지막 `frame_seq`로 중복 event 방지.

Node 필드:

- `x`, `y`
- `topology`
- `grid_branch_mask`
- `first_node`
- `visited_order`

Edge 규칙:

- node event가 들어오면 이전 node와 현재 node 사이 edge를 추가한다.
- `grid_branch_mask`에 north/east/south/west branch가 있으면 known edge 후보로 저장한다.
- 단, 반대편 node가 아직 없으면 endpoint를 `unknown`으로 두거나 drawing에서 생략한다. 처음 구현은 known node끼리 연결된 edge만 그려도 된다.

중복 처리:

- `vision.grid_node.valid == true`인 frame만 node event로 처리한다.
- 같은 `grid_node.id`를 이미 처리했다면 무시한다.
- `vision.intersection_decision.node.valid`와 `vision.grid_node.valid`가 둘 다 있으면 `vision.grid_node`를 canonical event로 사용한다.

### 4.3 ASCII rendering 규칙

좌표계:

- local `x`는 오른쪽으로 증가한다.
- local `y`는 아래쪽으로 증가한다.
- `(0,0)`은 첫 grid node다.
- official competition origin 변환은 아직 하지 않는다.

Canvas:

- node는 `+` 위치에 둔다.
- 수평 edge는 `---`, 수직 edge는 `|`로 그린다.
- 현재 드론 위치는 마지막 node 위치의 `+`를 heading arrow로 대체한다.
- heading이 unknown이면 `@` 또는 `D`를 사용한다.
- start `s`는 first node의 arrival heading 반대 방향 바깥에 둔다. arrival heading이 unknown이면 첫 구현은 `(0,0)` 아래에 `s`를 둔다.
- bounding box는 visited node 기준으로 계산하고 start marker가 들어갈 여백을 추가한다.

렌더링 예:

```text
[grid-map] local origin=(0,0) nodes=7 current=(2,1) heading=east
+---+--->
|   |
+---+
|
s
```

로그 연결:

- `VisionLogFormatter::formatVisionLog()` 마지막 또는 `[grid-node]` 다음에 `[grid-map]` block을 붙인다.
- `formatVisionLog()`가 순수 formatter라면 `GridMapTracker`를 `VisionDebugApp`에서 유지하고 `formatVisionLog(frame, stats, grid_map_text)`처럼 넘기는 편이 낫다.
- 대안으로 `TelemetryStore`가 latest frame뿐 아니라 `GridMapTracker`를 갖게 할 수 있지만, GCS state와 log formatting이 섞일 수 있으므로 처음 구현은 `VisionDebugApp` local state를 권장한다.

### 4.4 테스트

`test_grid_map_tracker.cpp`에서 다음을 검증한다.

- 첫 node `(0,0)`만 들어오면 `s`와 `(0,0)`이 표시된다.
- `(0,0) -> (1,0) -> (2,0)` event 후 수평 edge와 `>` arrow가 표시된다.
- row 전환 `(2,0) -> (2,1) -> (1,1)` 후 수직 edge와 `<` arrow가 표시된다.
- 중복 `grid_node.id`는 node count를 증가시키지 않는다.
- negative coordinate가 들어와도 bounding box가 깨지지 않는다.

## 5. Onboard snake policy 설계

### 5.1 파일 위치

새 파일 후보:

- `uav-onboard/src/mission/SnakeTraversalPlanner.hpp`
- `uav-onboard/src/mission/SnakeTraversalPlanner.cpp`
- `uav-onboard/tests/test_snake_traversal.cpp`

수정 후보:

- `uav-onboard/src/app/VisionDebugPipeline.cpp`
- `uav-onboard/src/mission/GridCoordinateTracker.*`
- `uav-onboard/src/mission/IntersectionDecision.*`
- `uav-onboard/src/protocol/TelemetryMessage.*`
- `uav-gcs/src/protocol/TelemetryMessage.*`
- `uav-gcs/tests/test_telemetry_line_parse.cpp`
- `uav-onboard/tests/test_telemetry_line_json.cpp`
- `uav-onboard/CMakeLists.txt`

### 5.2 Planner 책임

`SnakeTraversalPlanner`는 실제 제어 명령을 내리지 않고 debug/mission state만 계산한다.

입력:

- 현재 local coord.
- 현재 heading.
- accepted topology.
- `grid_branch_mask`.
- visited set.

출력:

- `turn_expected`: 다음 intersection decision에서 side turn을 pre-arm할지.
- `planned_turn`: `left`, `right`, `none`.
- `planned_heading_after_turn`.
- `next_coord_candidate`.
- `visited` update.
- `sweep_row_index` 또는 `turn_side`.

필수 정책:

- row 내부에서는 현재 heading 방향의 unvisited neighbor가 있으면 직진한다.
- row 끝에서는 같은 방향 90도 turn을 두 번 수행한다.
- 첫 row 끝 turn side는 초기 진행 방향 기준으로 오른쪽을 기본값으로 둔다.
- 다음 row 끝에서는 왼쪽, 그 다음은 오른쪽으로 번갈아 수행한다.
- 전진 branch가 존재해도 이미 visited이거나 snake policy상 row 전환 지점이면 `turn_expected = true`가 되어야 한다.
- branch가 존재하지 않는 방향으로는 이동 후보를 만들지 않는다.
- 이미 visited인 node는 기본적으로 next target으로 선택하지 않는다. 단, 시작 진입 segment `s -> (0,0)`은 grid visited set에 포함하지 않는다.

### 5.3 Heading 연결

현재 live pipeline은 `GridCoordinateTracker` heading이 `Unknown`으로 남기 쉽다.

구현 원칙:

- 첫 node 기록 전에는 initial heading을 config 또는 default로 둔다. vision-only 기본값은 `north` 또는 `east` 중 하나로 명확히 문서화한다.
- `SnakeTraversalPlanner`가 turn을 계획하면, 실제 제어가 없는 vision-only 모드에서는 다음 node를 기록하기 전에 planner의 expected heading을 `GridCoordinateTracker::setHeading()`에 반영한다.
- 실제 MAVLink 제어가 붙는 미래 단계에서는 `notifyTurnCompleted()`를 실제 yaw 완료 event와 연결한다.

주의:

- hand-held top-down 테스트에서는 사람이 카메라 yaw를 돌리는 것이 실제 turn completion이다. 지금은 IMU/yaw feedback이 없으므로 planner의 discrete expected heading이 truth source다.

### 5.4 교차점 중복 저장 방지

교차점 node를 한 번 저장한 뒤 같은 물리 교차점이 여러 frame 동안 계속 보이면서 중복 저장되는 것을 막아야 한다.

권장 방식:

- `record_node_once_frames`만으로 막지 말고, 이동 상태에 따른 explicit cooldown state를 둔다.
- 설정은 초 단위와 frame 단위를 모두 지원한다. 예: `node_ignore_after_straight_ms = 2000`, `node_ignore_after_turn_departure_ms = 2000`.
- 12FPS 기준 2초는 약 24 frame이다. frame rate가 바뀌어도 같은 시간 의미를 유지하려면 `fps_assumption`으로 frame 수를 계산한다.

직진 계속 시:

- node 저장 후 planner action이 `continue`이면 `IgnoreAfterNodeStraight` 상태로 들어간다.
- 이 상태에서는 2초 동안 `L/T/+`가 다시 안정적으로 인식되어도 새 `grid_node` event를 만들지 않는다.
- line tracking, branch score, overlay/log용 raw intersection은 계속 계산하되 mission/grid 저장만 막는다.
- 2초가 지나고, 현재 frame의 교차점 중심이 turn/record zone을 벗어났거나 line-only cruise 상태로 돌아온 뒤 다음 node를 받을 수 있게 한다.

회전 시:

- node 저장 후 planner action이 `turn_left` 또는 `turn_right`이면 `IgnoreDuringTurn` 상태로 들어간다.
- 90도 회전 중에는 모든 교차점 저장을 무시한다.
- 회전 완료를 확인한 뒤 `IgnoreAfterTurnDeparture` 상태로 들어가고, 출발 후 2초 동안 추가 교차점 저장을 무시한다.
- 즉 회전 케이스의 무시 구간은 `회전 시작 -> 회전 완료 -> 출발 후 2초` 전체다.
- 회전이 두 번 연속 필요한 snake row 전환에서는 첫 90도 회전, 한 칸 전진, 두 번째 90도 회전을 각각 별도 state로 다루되, 각 turn segment마다 동일한 ignore 규칙을 적용한다.

상태 이름 후보:

```text
Cruise
NodeRecorded
IgnoreAfterNodeStraight
TurnPending
IgnoreDuringTurn
TurnCompleted
IgnoreAfterTurnDeparture
```

구현 위치:

- `IntersectionDecisionEngine`은 raw decision과 `event_ready` 후보를 계속 계산한다.
- 실제 node 저장 허용 여부는 `SnakeTraversalPlanner` 또는 별도 `NodeEventGate`가 결정한다.
- `GridCoordinateTracker::update()`는 gate가 허용한 event만 받는다.
- GCS log에는 `node_gate=ignore_straight`, `node_gate=ignore_turn`, `ignore_remaining_ms=...` 같은 compact 상태를 표시한다.

주의:

- 2초는 현재 hand-held/저속 테스트에는 합리적인 보수값이다. 실제 비행에서는 속도, cell 간격, FPS에 맞춰 줄이거나 늘려야 한다.
- 너무 긴 ignore는 인접 교차점 간격이 짧거나 이동 속도가 빠를 때 다음 node를 놓칠 수 있다.
- 따라서 완료 기준에는 “2초 고정”이 아니라 “기본 2초, config로 조정 가능”을 넣는다.

### 5.5 수동 회전 완료 인식

현재 드론/IMU/flight-controller 상태가 없으므로 onboard가 실제 yaw 완료를 직접 알 수 없다. 이번 vision-only 단계에서는 아래 중 하나를 사용한다.

1. **Debug-only 고정 시간 방식**
   - planner가 turn을 결정하면 `expected_heading`을 즉시 또는 `manual_turn_duration_ms` 후 갱신한다.
   - 그동안 `IgnoreDuringTurn`으로 node 저장을 막는다.
   - 사람이 그 시간 안에 카메라를 90도 돌린다고 가정한다.
   - 구현이 가장 쉽지만, 사람이 늦게 돌리면 heading과 실제 화면이 어긋난다.

2. **Vision-only 회전 완료 추정**
   - turn 시작 후 line detector가 다시 안정적으로 line을 잡고, line center offset이 일정 범위 안이며, line angle이 camera forward axis에 가까운 frame이 `N`개 연속 나오면 turn completed로 본다.
   - 추가로 planned direction의 branch가 front branch로 바뀌었는지 branch mask transition을 확인한다.
   - 예: 우회전 계획이면 turn 전 right branch가 turn 후 front branch로 안정적으로 보여야 한다.
   - 이 방식은 IMU 없이도 hand-held 테스트에 쓸 수 있지만, 영상만으로 정확한 90도 yaw를 보장하지는 않는다.

3. **수동 확인 입력**
   - GCS command channel이 아직 없으므로 당장 구현한다면 onboard process stdin 또는 임시 keyboard hotkey로 `turn done`을 입력받는다.
   - 나중에 command channel이 생기면 GCS에서 `TURN_DONE` debug command를 보내도록 바꾼다.
   - 사람이 직접 회전 완료를 알려주므로 state mismatch가 가장 적다.

권장 순서:

- 1차 구현은 `Debug-only 고정 시간 방식 + Vision-only 안정 조건` 조합으로 한다.
- 즉 최소 `manual_turn_min_ms` 동안은 무조건 ignore하고, 그 이후 line center/angle이 안정되면 turn completed로 처리한다.
- 안정 조건이 일정 timeout 안에 충족되지 않으면 GCS log에 `turn_wait_timeout`을 표시하고 수동 확인 입력을 허용한다.
- 실제 드론이 생기면 이 부분은 Pixhawk yaw target reached 또는 onboard IMU yaw delta 기반 `notifyTurnCompleted()`로 대체한다.

### 5.6 Telemetry 확장 여부

최소 구현은 기존 `vision.grid_node.arrival_heading`, `grid_branch_mask`만 정확히 채워도 GCS grid renderer가 동작한다.

더 나은 debug를 위해 protocol v1.8 후보로 아래 field를 추가할 수 있다.

```json
"vision": {
  "snake": {
    "enabled": true,
    "state": "row_forward",
    "current_coord": {"x": 2, "y": 1},
    "current_heading": "west",
    "planned_turn": "left",
    "turn_expected": true,
    "node_gate": "ignore_turn",
    "ignore_remaining_ms": 1200,
    "turn_completion": "vision_stable",
    "next_coord": {"x": 2, "y": 2},
    "visited_count": 8
  }
}
```

protocol을 확장하면 반드시 양쪽 문서를 같이 갱신한다.

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`
- `uav-onboard/src/protocol/TelemetryMessage.*`
- `uav-gcs/src/protocol/TelemetryMessage.*`

처음 작업에서는 protocol 확장을 최소화하고, 필요한 경우에만 `vision.snake`를 추가한다.

## 6. GCS overlay 단순화 설계

### 6.1 Line overlay

수정 위치:

- `uav-gcs/src/overlay/LineOverlay.*`
- `uav-gcs/tests/test_line_overlay.cpp`

변경 목표:

- magenta contour는 유지하되 line 전체 blob contour만 보여준다.
- green tracking point와 label은 제거한다.
- 빨간 원을 `(line.tracking_point_px.x, frame_height / 2)`에 그린다.
- 초록 line을 `(frame_width / 2, frame_height / 2)`에서 빨간 원까지 그린다.
- 이때 빨간 원의 y는 telemetry의 `tracking_point_px.y`를 그대로 쓰지 말고 frame center y를 사용한다.

필요 API 변경:

- 현재 `buildLineOverlays(const LineTelemetry&)`는 frame width/height를 모른다.
- `buildLineOverlays(const LineTelemetry&, int frame_width, int frame_height)` overload를 추가한다.
- 기존 test/호출부를 새 signature로 옮긴다.

`VisionDebugApp.cpp` 호출부:

```cpp
auto line_overlays = overlay::buildLineOverlays(
    marker_frame->line,
    marker_frame->width,
    marker_frame->height);
```

### 6.2 Intersection overlay

수정 위치:

- `uav-gcs/src/overlay/IntersectionOverlay.*`
- `uav-gcs/tests/test_intersection_overlay.cpp`

변경 목표:

- 기본 overlay에서 branch score text, decision full label, yellow endpoint circles를 제거한다.
- top/ahead ROI 안에 있는 intersection만 간단히 표시한다.
- ROI 기준은 첫 구현에서 `intersection.center_px.y <= frame_height * 0.60`으로 둔다. 현재 도착한 교점도 보려면 `turn_zone_y_max`와 맞춰 `0.68`까지 허용할 수 있다.
- 표시 내용은 center 작은 원과 compact type/direction label만 둔다.
- branch ray는 짧은 방향 표시만 남기고 점수 텍스트는 제거한다.

필요 API 변경:

- `buildIntersectionOverlays(intersection, decision, frame_width, frame_height)` overload 추가.
- detailed/debug overlay는 나중에 `ui.toml` option으로 되살릴 수 있게 helper를 분리한다.

### 6.3 UI config 후보

당장 필수는 아니지만 `uav-gcs/config/ui.toml`에 다음 option을 추가할 수 있다.

```toml
[overlay]
mode = "clean" # clean or debug
intersection_ahead_y_max = 0.68
show_line_label = false
show_branch_scores = false
```

현재 config loader가 `vision_debug_main.cpp` 안에 있으므로, option을 늘릴 때는 `VisionDebugOptions`에 반영한다.

## 7. 굵은 흰색 line mask 개선 설계

### 7.1 문제 원인

현재 `local_contrast_blur = 31`과 local contrast mask는 굵은 흰색 line 내부보다 edge contrast를 더 잘 살릴 수 있다. 넓은 테이프/종이는 중앙부가 local background와 비슷하게 계산되어 line 전체가 채워진 blob이 아니라 얇은 strip으로 남는다.

### 7.2 개선 방향

수정 위치:

- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/LineDetector.*`
- `uav-onboard/src/common/VisionConfig.*`
- `uav-onboard/config/vision.toml`
- `uav-onboard/tools/line_detector_tuner.cpp`
- `uav-onboard/tests/test_intersection_detector.cpp` 또는 신규 line mask test

권장 구현:

- `LineMaskBuilder`에 wide white line용 mask strategy를 추가한다. 후보 이름: `white_fill` 또는 `local_contrast_fill`.
- 기존 local contrast mask와 absolute white mask를 union한다.
- absolute white mask는 HSV 또는 Lab 기반으로 만든다.
  - HSV 후보: low saturation + high value.
  - 예: `S <= 90`, `V >= 145`를 시작점으로 두되 config화한다.
- union mask 이후 close/dilate/fill을 적용해 내부 빈 공간을 메운다.
- `LineDetector`가 받는 contour는 fill된 mask의 largest connected component가 되도록 한다.
- `IntersectionDetector::bridgeMask()`는 이미 close/dilate를 추가로 하므로, line detector 쪽 mask도 동일한 `LineMaskBuilder` 결과를 공유하게 유지한다.

config 후보:

```toml
[line]
mask_strategy = "white_fill"
white_v_min = 145
white_s_max = 90
fill_close_kernel = 11
fill_dilate_kernel = 3
```

`LineConfig`에 field를 추가할 때는 parser 기본값도 함께 추가한다.

### 7.3 Tracking point y 고정

제어용 tracking point는 카메라 중앙 높이에서 산출되어야 한다.

- `config/vision.toml`의 `line.lookahead_y_ratio`를 `0.50`으로 바꾸는 것을 1차 후보로 둔다.
- 더 확실하게 하려면 `LineDetector`에서 source lookahead y를 `height * lookahead_y_ratio`로 계산하되 기본값을 `0.50`으로 변경한다.
- GCS overlay는 frame center y를 사용하므로, telemetry y가 잠깐 다르더라도 화면 표시는 요구사항을 지킨다.

### 7.4 Tool 개선

`line_detector_tuner`는 현재 숫자 결과만 출력한다. mask 개선 검증을 위해 option을 추가한다.

후보:

```powershell
.\build\line_detector_tuner.exe --config config --image sample.png --mask white_fill --output out
```

출력:

- `mask.png`
- `overlay.png`
- detected contour point count, bounding box, area, tracking x/y, offset

## 8. Smoke/replay 검증 계획

### 8.1 입력 이미지

사용자 첨부 이미지와 기존 smoke 산출물을 기준으로 한다.

- 굵은 흰색 line close-up 캡처 2장.
- 현재 얇게 인식되는 GCS 캡처 1장.
- `development-log/grid-smoke-20260501/mid_entry_rotated_centered/`
- `development-log/grid-smoke-20260501/edge_entry_rotated_centered/`

원본 이미지 파일 경로는 사용자 환경의 한글 경로에 있으므로, 테스트 데이터로 쓸 경우 repo 내부 `uav-onboard/test_data/images/` 또는 `development-log/` 아래에 복사해서 상대 경로로 관리한다.

### 8.2 Onboard test

실행 후보:

```powershell
cd uav-onboard
cmake -S . -B build-tests -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

OpenCV tools smoke:

```powershell
.\build\line_detector_tuner.exe --config config --image test_data/images/line_wide_white.png --mask white_fill --output test_data/logs/line_wide_white
.\build\grid_image_smoke.exe --config config --image test_data/images/grid_sample.png --output test_data/logs/grid_smoke --scenario sample
```

검증 기준:

- wide white line의 contour가 line 양쪽 edge만이 아니라 line 전체 blob 외곽을 감싼다.
- `tracking_point.y`가 frame center 또는 config `lookahead_y_ratio = 0.50`에 맞는다.
- `intersection` branch mask가 `+`/`T`/`L`에서 이전보다 덜 누락된다.
- `GridCoordinateTracker` heading이 unknown으로 남지 않는다.
- snake path coordinate가 `snake_full_field.csv`, `snake_from_entry.csv`에서 expected와 일치한다.

### 8.3 GCS test

실행 후보:

```powershell
cd uav-gcs
cmake -S . -B build-tests -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

검증 기준:

- `test_line_overlay`는 red center circle과 green horizontal offset line을 확인한다.
- `test_intersection_overlay`는 score text 없이 compact branch/type 표시를 확인한다.
- `test_grid_map_tracker`는 node event sequence를 ASCII grid로 정확히 렌더링한다.
- `VisionLogFormatter`가 grid map block을 포함해도 기존 packet/line/intersection 로그가 깨지지 않는다.

## 9. 구현 시 주의사항

- 이번 단계에서 Pixhawk/MAVLink control loop를 구현하지 않는다.
- onboard critical path에 GCS video drawing이나 heavy logging을 넣지 않는다.
- debug video는 best-effort다. mission 판단에 필요한 grid/snake state는 telemetry 이전에 onboard에서 계산한다.
- GCS grid renderer는 표시용이다. 최종 mission policy source of truth는 onboard snake planner가 되어야 한다.
- official competition origin/axis 변환은 이번 범위 밖이다. 모든 좌표는 local exploration coordinate다.
- branch bit 순서는 기존 코드를 따른다.
  - camera branch: front=bit0, right=bit1, back=bit2, left=bit3.
  - grid branch: north=bit0, east=bit1, south=bit2, west=bit3.
- `+` false upgrade guard는 유지한다. 약한 네 번째 branch 때문에 `T`를 `+`로 올리는 회귀를 만들지 않는다.
- GCS overlay를 줄이더라도 vision log에는 tuning에 필요한 raw 값이 남아야 한다.
- 한글 경로 이미지를 자동 테스트에 직접 의존하지 않는다. 필요한 sample은 repo 내부 test data로 복사해 상대 경로를 사용한다.
- node 저장 후 ignore/cooldown 중에도 detector 자체는 계속 돌린다. 무시하는 것은 grid node 저장과 mission event 생성뿐이다.
- 수동 hand-held 회전 단계에서는 fixed-time, vision-stable, manual-confirm 중 어떤 turn completion source를 썼는지 telemetry/log에 남긴다.

## 10. 완료 기준

작업 완료로 보려면 다음이 충족되어야 한다.

- GCS vision log에 node event가 들어올 때마다 ASCII grid가 갱신된다.
- 첫 node는 local `(0,0)`으로 표시되고 `s` 진입 segment가 보인다.
- 현재 좌표와 heading arrow가 보인다.
- snake traversal은 visited node를 재방문 대상으로 고르지 않는다.
- 전진 branch가 있어도 policy상 row 전환 지점이면 `turn_expected` 또는 planned turn이 표시된다.
- node 저장 후 직진이면 기본 2초 동안 같은 교차점이 재저장되지 않는다.
- 회전이면 90도 회전 중과 회전 후 출발 기본 2초 동안 교차점이 재저장되지 않는다.
- 직진/회전 ignore 시간은 config로 조정 가능하다.
- hand-held 테스트에서 turn completion source가 fixed-time, vision-stable, manual-confirm 중 무엇인지 로그로 확인된다.
- GCS camera overlay는 red line-center circle + green horizontal offset line + compact intersection 방향 표시만 기본으로 보여준다.
- 빨간 원의 y 좌표는 항상 frame center y다.
- 굵은 흰색 line sample에서 magenta contour가 line 전체 blob을 감싼다.
- `uav-gcs`와 `uav-onboard` 관련 tests가 통과한다.
- README 또는 protocol을 바꾼 경우 양쪽 문서가 동기화된다.
