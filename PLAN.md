# Astroquad 다음 단계 개발 계획: 교차점 의사결정과 상대 좌표 기록

최종 업데이트: 2026-05-01

범위: `development-log/RESEARCH.md`의 최신 구현 상태와 2026-04-30 GCS 캡처 관찰, 연습용 격자 이미지 2장, 그리고 “최근 짧은 window에서 우선순위가 높은 교차점 타입을 채택하되 전체 교차점마다 오래 정지하지 않는다”는 운용 아이디어를 반영한 다음 개발 계획이다.

기존 `PLAN.md` 내용은 더 이상 기준으로 사용하지 않는다.

## 1. 결론

다음 단계는 **OpenCV 파라미터를 먼저 크게 바꾸는 것**도 아니고, **Pixhawk까지 포함한 전체 mission/control 상태머신을 바로 시작하는 것**도 아니다.

가장 적절한 다음 작업은 다음이다.

1. 현재 `IntersectionDetector`와 `IntersectionStabilizer` 위에 얇은 **IntersectionDecisionEngine**을 만든다.
2. 이 엔진은 최근 0.25~0.75초 정도의 관측 queue/ring buffer를 사용해 branch evidence를 누적한다.
3. `+ > T > L > straight > unknown` 우선순위는 topology 확정에만 사용하되, 단일 frame의 상위 타입 튐은 바로 채택하지 않는다.
4. 모든 `L/T/+` 교차점은 상대 좌표를 저장하는 grid node event로 취급한다.
5. 첫 번째로 도착한 grid node를 local `(0,0)`으로 둔다. 이 좌표는 official origin이 아니라 탐색 시작 기준 local origin이다.
6. `L/T/+` 타입만으로 감속/정지를 결정하지 않는다.
7. 감속/정지/회전 후보는 “현재 진행 방향의 front branch가 더 이상 안정적으로 보이는지”와 나중에 붙을 snake/grid policy가 결정한다.
8. 실제 Pixhawk 제어는 아직 붙이지 않고, 먼저 telemetry/log/overlay로 node event, 상대 좌표, branch topology, overshoot risk가 맞는지 검증한다.

이 방향이 맞는 이유:

- 현재 오분류는 대부분 branch 누락에 의한 downgrade다.
- `+`를 `T`/`L`로, `T`를 `L`/`unknown`으로 보는 문제는 짧은 시간 축에서 branch evidence를 합치면 완화된다.
- 반대로 드물게 더 많은 branch로 false upgrade가 뜰 가능성은 mission에 위험하므로, “한 번이라도 `+`가 뜨면 `+`”가 아니라 “4방향 branch evidence가 각각 최소 기준을 넘으면 `+`”로 처리해야 한다.
- 대회 시간상 교차점마다 2~3초 정지는 과하다. 정지는 front branch가 사라지거나 snake policy가 방향 전환을 요구하는 지점에서만 제한적으로 써야 한다.
- 외곽 라인의 중간 지점은 `T`여도 계속 직진해야 할 수 있고, 내곽 라인의 `T`도 단순 통과 지점일 수 있다. 따라서 `T/L`을 곧바로 turn action으로 해석하면 안 된다.
- 아직 full grid snake/control은 없지만, 상대 좌표 기록 없이는 다음 단계 검증이 어려우므로 최소한의 `GridCoordinateTracker`는 이번 단계에 포함한다.

## 2. 현재 구현 기준점

이미 구현된 것:

- `uav-onboard/src/vision/LineMaskBuilder.*`
- `uav-onboard/src/vision/LineDetector.*`
- `uav-onboard/src/vision/LineStabilizer.*`
- `uav-onboard/src/vision/IntersectionDetector.*`
- `uav-onboard/src/vision/IntersectionStabilizer.*`
- `vision.intersection` telemetry v1.6
- GCS `IntersectionOverlay`
- GCS `[intersection]` log
- `vision_debug_node --fps`

현재 runtime 흐름:

```text
camera JPEG
  -> decode
  -> LineMaskBuilder
  -> LineDetector
  -> IntersectionDetector
  -> LineStabilizer
  -> IntersectionStabilizer
  -> telemetry/log/GCS overlay
```

다음 단계 후 목표 흐름:

```text
camera JPEG
  -> decode
  -> LineMaskBuilder
  -> LineDetector
  -> IntersectionDetector
  -> LineStabilizer
  -> IntersectionStabilizer
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> telemetry/log/GCS overlay
  -> future Grid/Mission/Control
```

## 3. 해결해야 하는 실제 문제

### 3.1 밝은 배경에서 `unknown`이 자주 뜸

원인:

- 밝은 바닥에서는 흰 선과 배경의 local contrast가 낮다.
- mask가 선 내부 전체를 채우지 못하고 edge/얇은 strip 위주로 남는다.
- branch ray가 충분히 긴 foreground run을 못 밟으면 branch count가 줄어든다.

대응 방향:

- 당장 OpenCV 값을 크게 바꾸기보다, branch score와 branch evidence를 시간 축으로 누적한다.
- 이후 실제 log를 보고 `intersection_threshold = 0.70~0.75`, `process_width = 320`, `local_contrast_blur = 21~25` 후보를 비교한다.

### 3.2 `+`가 `T`/`L`, `T`가 `L`로 downgrade됨

원인:

- 현재 classifier는 branch count 기반이다.
- 한 branch가 threshold 아래로 떨어지면 타입이 바로 낮아진다.
- branch score가 `0.75~0.84` 근처에서 threshold `0.8` 주변을 오가면 frame마다 타입이 바뀐다.

대응 방향:

- 단일 frame type이 아니라 최근 N frame의 branch별 evidence를 본다.
- `+`, `T`, `L` 우선순위는 branch별 반복 증거가 있을 때만 적용한다.
- 최종 type stabilizer와 별도로 mission/action decision용 short-window aggregator를 둔다.

### 3.3 `straight`와 `unknown`이 섞임

원인:

- straight는 양방향 branch가 모두 present여야 한다.
- 한쪽 branch가 frame 밖에 있거나 mask가 약하면 branch count 1이 되어 `unknown`이 된다.

대응 방향:

- cruise 중에는 `unknown`과 `straight`를 모두 “계속 전진”으로 처리한다.
- straight 여부 자체를 오래 확정하려고 멈추지 않는다.
- 교차점 후보가 아닌 상황에서 unknown은 mission action을 만들지 않는다.

### 3.4 2~3초 정지는 시간 오버헤드가 큼

운용 판단:

- `+` 교차점은 좌표 기록만 하면 되고, 회전할 필요가 없다.
- `T`도 위치에 따라 단순 통과 지점일 수 있다.
- `L`도 일반적으로는 행/열 끝 회전 지점일 가능성이 높지만, type만 보고 즉시 회전 명령을 내리면 안 된다.
- 회전 필요 여부는 현재 진행 방향의 `front` branch가 계속 존재하는지와 grid/snake policy가 결정해야 한다.
- 따라서 모든 교차점에서 긴 정지는 맞지 않는다.

대응 방향:

- node 기록 confirmation과 turn confirmation을 분리한다.
- `L/T/+`는 모두 0.25~0.5초 정도의 짧은 window로 grid node event를 기록한다.
- 감속/정지는 `front` branch가 사라지는 후보 또는 future grid policy가 방향 전환을 요구하는 후보에서만 0.3~0.8초 confirm한다.

### 3.5 12FPS에서 전진 중 판단/정지가 가능한가

이 문제는 실제 Pixhawk 제어를 붙일 때만 생각하면 늦다. 이번 단계에서 실제 stop/turn 명령을 구현하지는 않더라도, decision layer는 나중 control이 쓸 수 있는 timing/position 정보를 반드시 남겨야 한다.

핵심 판단:

- 12FPS는 frame 간격이 약 83ms다.
- branch evidence를 6 frame 모으면 이미 약 0.5초가 지난다.
- 여기에 camera capture, decode, vision, MAVLink command, Pixhawk 반응, 감속 시간이 더해진다.
- 따라서 “직진 불가/회전 필요”를 교차점 중심에 도달한 뒤 확정하면 늦을 수 있다.
- 회전 가능성은 중심에 닿은 뒤 판단하는 것이 아니라, 접근 중에 `front` branch 유지 여부와 side branch evidence로 pre-arm 하고 turn zone 안에서 commit해야 한다.

필요한 계산:

```text
decision_time = confirm_frames / fps
reaction_time = camera_pipeline_latency + command_latency + controller_response_latency
stopping_distance = speed * (decision_time + reaction_time) + speed^2 / (2 * decel)
required_lead_distance = stopping_distance + safety_margin
```

초기에는 실제 거리 scale이 없으므로 meter 단위로 바로 판단하지 않는다. 대신 image-space 기준을 먼저 기록한다.

```text
center_y_norm = intersection_center_y / image_height
approach_phase = far | approaching | turn_zone | late | passed
overshoot_risk = center_y_norm이 late zone에 들어갔는데 required_turn이 아직 확정되지 않은 상태
```

대응 방향:

- `IntersectionDecisionEngine`은 type만 내지 말고 `approach_phase`, `center_y_norm`, `overshoot_risk`를 함께 낸다.
- `TurnReady`는 topology type만으로 내지 않고, `front` branch 부재 또는 grid policy의 turn 요구와 turn zone 조건이 모두 맞을 때만 낸다.
- turn zone 이전에는 `PrepareTurn` 또는 `turn_candidate`만 표시한다.
- turn zone을 지나친 뒤에는 무리하게 turn하지 않고 `overshoot_risk=true` 또는 `too_late=true`를 표시한다.
- 실제 control 구현 단계에서는 `turn_candidate`가 보이면 cruise speed를 낮추고, `TurnReady`에서 hover/turn을 명령한다.

운용 결론:

- 12FPS에서도 저속 접근이면 충분히 가능하다.
- 하지만 최대 전진 속도는 vision FPS와 confirm frame 수로 제한해야 한다.
- 교차점 중심에 정확히 멈추는 능력은 detector 문제가 아니라 “접근 속도 + 감속 프로파일 + front branch/turn zone gating” 문제다.
- 이번 계획에는 이 정보를 telemetry/log로 남기는 것까지 포함한다.
- 실제 Pixhawk stop/turn 제어는 다음 단계에서 구현한다.

### 3.6 연습용 격자 topology와 상대 좌표 기준

연습용 격자 이미지를 기준으로 보면 경기장은 대략 rectangular grid 형태일 가능성이 높다.

예상 topology:

```text
외곽 라인: L T T T ... T L
내곽 라인: T + + + ... + T
```

중요한 해석:

- `L`, `T`, `+`는 모두 위치 기록 대상이다.
- `+`만 좌표로 저장하는 방식은 부족하다.
- `T`는 항상 회전 지점이 아니다. 외곽 라인을 따라 이동할 때 중간 `T`는 계속 직진해야 하는 node일 수 있다.
- `L`은 행/열 끝일 가능성이 높지만, 그래도 action은 topology type만이 아니라 현재 heading과 탐색 policy로 결정해야 한다.
- 실제 회전 조건은 “현재 카메라가 바라보는 진행 방향에 front branch가 더 이상 없다” 또는 “snake policy가 이 node에서 방향 전환을 요구한다”에 가깝다.

좌표 기준:

- 첫 번째로 안정적으로 도착한 grid node를 local `(0,0)`으로 둔다.
- 이 local `(0,0)`은 official origin이 아닐 수 있다.
- 이륙 지점에서 grid로 진입하는 연결선이 official `(0,0)`으로 이어질지, 중간 node로 이어질지는 아직 확정 정보가 없다.
- 따라서 초기 구현에서는 `origin_status = local_only`로 두고, 나중에 시작 마커/경기장 규칙/탐색된 bounding box로 official 좌표 변환을 한다.

좌표 업데이트 원칙:

```text
첫 node:
  local_coord = (0, 0)
  current_heading = initial_grid_heading

다음 node:
  local_coord += heading_vector(current_heading)

회전 완료:
  current_heading = rotated_heading
```

branch 저장 원칙:

- camera-relative branch mask(`front/right/back/left`)를 current heading 기준의 grid-relative branch mask(`north/east/south/west`)로 변환해 저장한다.
- 이렇게 해야 나중에 같은 node를 다른 방향에서 보더라도 동일한 topology map으로 합칠 수 있다.
- 첫 진입 segment가 diagonal이면 grid axis와 맞지 않을 수 있으므로, 첫 node에서는 `initial_grid_heading`을 강제로 확정하지 않고 `entry_bootstrap` 상태를 둔다.

## 4. 구현 우선순위

### 4.1 하지 않을 것

이번 단계에서 하지 않을 것:

- Pixhawk/MAVLink 실제 제어 명령 송신
- 전체 grid snake mission 완성
- 모든 교차점에서 2~3초 정지하는 dwell 로직
- OpenCV 파라미터를 감으로 바로 기본값 변경
- Skeleton/Hough 기반 detector 재작성
- 실제 거리 기반 정지 제어와 yaw turn 제어
- official competition coordinate 확정

### 4.2 먼저 할 것

이번 단계에서 먼저 할 것:

1. `IntersectionDecisionEngine` 신규 모듈 추가
2. 짧은 ring buffer 기반 branch evidence 누적
3. `L/T/+` 전체를 grid node event로 기록
4. `GridCoordinateTracker` 신규 모듈 추가
5. 첫 안정 grid node를 local `(0,0)`으로 저장
6. camera-relative branch mask를 heading 기준 grid-relative branch mask로 변환
7. type-based turn 판단을 제거하고 `front_available`, `required_turn`, `turn_candidate`를 분리
8. decision state/queue/branch evidence/node coordinate telemetry 추가
9. GCS log/overlay에 decision state와 local coordinate 표시
10. unit test로 downgrade/false-upgrade/cooldown/node 중복/좌표 증가 시나리오 검증
11. `center_y_norm`, `approach_phase`, `overshoot_risk`를 telemetry/log에 추가
12. 12FPS에서 confirm window와 전진 속도 사이의 timing budget을 문서화
13. 이후 OpenCV 후보값은 실제 branch evidence log를 보고 비교

## 5. 새 모듈 설계

### 5.1 파일 위치

추천 위치:

```text
uav-onboard/src/mission/IntersectionDecision.hpp
uav-onboard/src/mission/IntersectionDecision.cpp
uav-onboard/src/mission/GridCoordinateTracker.hpp
uav-onboard/src/mission/GridCoordinateTracker.cpp
```

이유:

- `IntersectionDetector`는 현재 프레임의 vision classification이다.
- `IntersectionStabilizer`는 vision 결과의 temporal smoothing이다.
- “지금 계속 전진할지, 교차점을 기록할지, 회전 후보로 볼지, cooldown인지”는 mission/action decision이다.
- “현재 node가 local grid 좌표로 어디인지, 어떤 branch topology를 갖는지”도 vision 자체가 아니라 mission/localization 경계다.
- 따라서 `src/vision`보다 `src/mission` 경계가 맞다.
- 아직 실제 control output은 내지 않고, debug decision만 telemetry에 싣는다.

대안:

- `src/vision/IntersectionDecision.*`에 둘 수도 있지만, 나중에 grid/snake search와 결합될 때 mission layer로 옮겨야 한다.
- 처음부터 `mission`에 두는 편이 장기 구조와 맞다.

### 5.2 주요 타입

새 enum:

```cpp
enum class IntersectionDecisionState {
    Cruise,
    Candidate,
    NodeRecord,
    TurnConfirm,
    TurnReady,
    Cooldown,
};

enum class IntersectionAction {
    None,
    ContinueStraight,
    RecordNode,
    PrepareTurn,
    TurnLeft,
    TurnRight,
    HoldPosition,
};

enum class GridHeading {
    North,
    East,
    South,
    West,
    Unknown,
};
```

주의:

- 이번 단계에서는 `TurnLeft`/`TurnRight`를 실제 제어로 보내지 않는다.
- action은 telemetry/log/debug용이다.
- 실제 방향 결정은 나중에 grid snake policy가 붙을 때 확정한다.

새 구조체:

```cpp
struct BranchEvidence {
    int present_frames = 0;
    float max_score = 0.0f;
    float sum_score = 0.0f;
    float average_score = 0.0f;
};

struct IntersectionDecisionSample {
    std::uint32_t frame_seq = 0;
    std::int64_t timestamp_ms = 0;
    vision::IntersectionType type = vision::IntersectionType::None;
    vision::IntersectionType raw_type = vision::IntersectionType::None;
    bool valid = false;
    bool stable = false;
    bool held = false;
    float score = 0.0f;
    std::uint8_t branch_mask = 0;
    std::array<float, 4> branch_scores {};
    std::array<bool, 4> branch_present {};
    vision::Point2f center_px;
};

struct IntersectionDecision {
    IntersectionDecisionState state = IntersectionDecisionState::Cruise;
    IntersectionAction action = IntersectionAction::ContinueStraight;
    vision::IntersectionType accepted_type = vision::IntersectionType::None;
    vision::IntersectionType best_observed_type = vision::IntersectionType::None;
    bool event_ready = false;
    bool turn_candidate = false;
    bool required_turn = false;
    bool front_available = false;
    bool node_recorded = false;
    bool cooldown_active = false;
    std::uint8_t accepted_branch_mask = 0;
    int window_frames = 0;
    int age_ms = 0;
    float confidence = 0.0f;
    vision::Point2f center_px;
    float center_y_norm = 0.0f;
    std::string approach_phase;
    bool overshoot_risk = false;
    bool too_late_to_turn = false;
    std::array<BranchEvidence, 4> branch_evidence {};
};

struct GridCoord {
    int x = 0;
    int y = 0;
};

struct GridNodeEvent {
    bool valid = false;
    std::uint32_t node_id = 0;
    GridCoord local_coord;
    vision::IntersectionType topology = vision::IntersectionType::Unknown;
    GridHeading arrival_heading = GridHeading::Unknown;
    std::uint8_t camera_branch_mask = 0;
    std::uint8_t grid_branch_mask = 0;
    bool first_node = false;
    bool origin_local_only = true;
};
```

### 5.3 새 config

추천 config 위치:

```toml
[intersection_decision]
enabled = true
fps_assumption = 12
cruise_window_frames = 6
turn_confirm_frames = 8
cooldown_frames = 8
min_cross_branch_frames = 2
min_t_branch_frames = 2
min_l_branch_frames = 3
min_branch_score = 0.72
high_confidence_score = 0.85
candidate_min_frames = 2
turn_confirm_required = 3
record_node_once_frames = 18
turn_zone_y_min = 0.42
turn_zone_y_max = 0.68
late_zone_y = 0.78
min_prearm_frames = 2
front_missing_frames = 2
node_advance_min_frames = 4
```

초기값 해석:

- `cruise_window_frames = 6`: 12FPS 기준 약 0.5초
- `turn_confirm_frames = 8`: 12FPS 기준 약 0.67초
- `cooldown_frames = 8`: 회전 후 약 0.67초 판단 무시
- `min_cross_branch_frames = 2`: `+` 채택 시 4방향 각각 최소 2 frame evidence 필요
- `min_t_branch_frames = 2`: `T` 채택 시 3방향 각각 최소 2 frame evidence 필요
- `min_l_branch_frames = 3`: `L`은 false upgrade보다 안정성을 위해 2방향 각각 최소 3 frame evidence 필요
- `min_branch_score = 0.72`: 현재 캡처에서 0.75~0.84 주변이 많으므로 0.8보다 낮은 decision threshold 후보
- `record_node_once_frames = 18`: 같은 grid node를 12FPS 기준 1.5초 안에 중복 기록하지 않기 위한 lockout
- `turn_zone_y_min/max`: 교차점 중심이 이 image-space band 안에 있을 때만 `TurnReady`를 허용
- `late_zone_y`: 이 값을 지나도 required turn이 확정되지 않으면 `overshoot_risk`로 표시
- `min_prearm_frames`: turn zone 진입 전 turn 후보를 최소 몇 frame 관찰해야 하는지
- `front_missing_frames`: 현재 heading 기준 front branch가 사라졌다고 보기 위한 최소 frame 수
- `node_advance_min_frames`: 같은 node 중복 기록을 피하고 다음 node로 넘어갔다고 보기 위한 최소 frame 수

구현 위치:

```text
uav-onboard/src/common/VisionConfig.hpp
uav-onboard/src/common/VisionConfig.cpp
uav-onboard/config/vision.toml
```

이 config는 detector threshold와 다르다.

- `line.intersection_threshold`: single-frame ray present threshold
- `intersection_decision.min_branch_score`: decision evidence threshold

나중에 중복이 커지면 통합할 수 있지만, 지금은 역할이 다르므로 분리한다.

## 6. Decision 알고리즘

### 6.1 입력

매 frame마다 다음을 입력받는다.

```cpp
IntersectionDecision update(
    const vision::IntersectionDetection& intersection,
    int frame_width,
    int frame_height,
    std::uint32_t frame_seq,
    std::int64_t timestamp_ms,
    bool turn_expected);
```

`turn_expected`는 이번 단계에서는 기본 `false`로 둘 수 있다. 이 값은 “type이 `T/L`라서 돈다”는 뜻이 아니라, future grid/snake policy가 “현재 node에서 방향 전환을 요구한다”는 뜻이다.

추후 grid/snake policy가 들어오면:

- 지금 행 끝인지
- 현재 방향 전환이 필요한 column/row인지
- 현재 marker 탐색 패턴상 회전해야 하는지

를 반영한다.

`GridCoordinateTracker`는 decision에서 확정된 node event를 입력받는다.

```cpp
GridNodeEvent update(
    const IntersectionDecision& decision,
    std::uint32_t frame_seq,
    std::int64_t timestamp_ms);

void notifyTurnCompleted(GridHeading new_heading);
void resetLocalOrigin();
```

초기 동작:

- 첫 `L/T/+` node event를 local `(0,0)`으로 둔다.
- 첫 진입선이 grid axis와 다를 수 있으므로 `arrival_heading`은 `Unknown` 또는 bootstrap heading으로 시작한다.
- 이후 같은 heading으로 다음 node를 확정할 때마다 `local_coord += heading_vector(current_heading)`을 적용한다.
- official origin은 아직 확정하지 않고 `origin_local_only=true`로 telemetry에 싣는다.

디버그 단계에서는 CLI 옵션으로 강제할 수 있다.

```text
--turn-expected
--decision-debug
```

단, 처음 구현에서는 CLI를 늘리기보다 telemetry/log만으로도 충분하다.

### 6.2 Branch evidence 누적

최근 window에 대해 branch별 증거를 계산한다.

```text
for each sample in window:
  for each branch F/R/B/L:
    if branch.present || branch.score >= min_branch_score:
      evidence[branch].present_frames += 1
      evidence[branch].max_score = max(...)
      evidence[branch].sum_score += branch.score
```

중요:

- single-frame `type`보다 branch score를 우선한다.
- `+`가 한 번 떴다는 사실만으로 `+`를 채택하지 않는다.
- `+` 채택은 4방향 branch evidence가 각각 기준을 넘는지 본다.
- 이렇게 하면 “더 많은 branch로 0.1% 튀는 경우”의 위험을 줄일 수 있다.

### 6.3 타입 우선순위 적용

이 단계의 타입은 action이 아니라 topology label이다.

우선순위:

```text
+ > T > L > straight > unknown
```

단, 적용 조건은 branch evidence 기반이다.

`+` 채택 조건:

```text
F/R/B/L 모두 present_frames >= min_cross_branch_frames
그리고 average/max score가 충분함
```

`T` 채택 조건:

```text
4방향 중 3방향 present_frames >= min_t_branch_frames
그리고 나머지 1방향은 약함
```

`L` 채택 조건:

```text
2방향 present_frames >= min_l_branch_frames
그리고 두 방향 angle relation이 직각 계열
```

`straight` 채택 조건:

```text
서로 반대 방향 2개 present
그리고 L/T/+ 조건은 미충족
```

`unknown`:

```text
위 조건 모두 미충족
```

주의:

- `T` 또는 `L`로 확정되어도 곧바로 회전하지 않는다.
- `+`, `T`, `L` 모두 `GridNodeEvent` 기록 대상이다.
- 감속/회전 후보는 type보다 `front_available=false`, `required_turn=true`, `turn_expected=true`, `turn_zone` 조건으로 결정한다.

### 6.4 State machine

초기 상태:

```text
Cruise
```

상태 전이:

```text
Cruise
  - accepted type none/unknown/straight -> ContinueStraight
  - accepted type L/T/+ -> NodeRecord
  - front_available=false 또는 turn_expected=true -> Candidate or TurnConfirm

NodeRecord
  - RecordNode event emit
  - same node duplicate lockout
  - GridCoordinateTracker에 local coordinate/topology/branch mask 전달
  - immediately return to Cruise

Candidate
  - front_missing 또는 policy turn 후보가 candidate_min_frames 이상 유지되면 TurnConfirm
  - L/T/+ node가 충분히 확인되면 NodeRecord
  - unknown/straight로 사라지면 Cruise

TurnConfirm
  - short window에서 front branch 부재 또는 policy turn 요구 재확인
  - L/T/+ node가 아직 기록되지 않았으면 RecordNode 후 계속 판단
  - required_turn이 충분하고 center가 turn_zone 안이면 TurnReady
  - required_turn은 충분하지만 center가 아직 turn_zone 전이면 Candidate 유지
  - required_turn은 충분하지만 center가 late_zone을 지났으면 overshoot_risk 표시 후 Cruise 또는 recovery 후보
  - 부족하면 Cruise 또는 Candidate

TurnReady
  - debug action emit
  - future control에서는 여기서 stop/turn command
  - 현재 구현에서는 telemetry/log만 emit
  - external reset 또는 manual simulated turn 후 Cooldown

Cooldown
  - queue clear
  - cooldown_frames 동안 판단 무시
  - 종료 후 Cruise
```

이번 단계에서는 실제 제어가 없으므로 `TurnReady`에 들어가도 드론이 돌지 않는다.

따라서 `vision_debug_node`에서는 다음 중 하나를 선택해야 한다.

1. `TurnReady`를 log에 남긴 뒤 즉시 `Cooldown`으로 보내는 debug mode
2. `TurnReady`를 유지해서 사람이 GCS log에서 확인하는 mode

권장:

- 초기 구현은 `TurnReady` 유지.
- 수동 테스트에서 확정 타입이 맞는지 보기 쉽다.
- 실제 control이 붙기 전에는 자동으로 cooldown을 시작하지 않는다.

### 6.5 `L/T/+`는 모두 좌표로 기록한다

node 기록 원칙:

- `L/T/+`는 모두 `RecordNode` event를 생성할 수 있다.
- 같은 node를 여러 번 기록하지 않도록 lockout을 둔다.
- node가 확인되는 동안에도 front branch가 있으면 line following은 계속 진행한다.
- topology type은 local map 기록용이고, action은 별도 판단한다.

이유:

- 대회 시간상 node마다 정지할 필요가 없다.
- 외곽 중간 `T`, 내곽 `+`, 일부 `L/T`는 좌표 기록만 하고 통과해야 할 수 있다.
- 좌표와 branch topology를 남겨야 나중에 grid size, 외곽/내곽, snake path를 안정적으로 추론할 수 있다.

### 6.6 회전 후보는 type이 아니라 front branch로 판단한다

회전 후보 처리 원칙:

- 지금은 실제 turn policy가 없으므로 debug decision만 만든다.
- 나중에 snake search policy가 붙으면 “현재 회전해야 하는 node”에서만 short confirm을 활성화한다.
- 회전이 필요 없는 구간에서는 `L/T/+`가 떠도 node만 기록하고 계속 전진할 수 있다.
- 현재 heading 기준 `front` branch가 안정적으로 사라지고 side branch가 있으면 required turn 후보가 된다.

초기 debug 기준:

```text
L/T/+가 최근 window에서 충분하면 node 기록 후보
front branch가 front_missing_frames 이상 약하면 required_turn 후보
left/right branch 중 하나가 충분하면 turn direction 후보
front branch가 충분하면 L/T/+ type과 무관하게 continue 후보
```

### 6.7 접근 phase와 오버슈트 방지

`L/T/+` type evidence만으로 바로 `TurnReady`가 되면 안 된다. 12FPS에서는 확정이 늦으면 이미 교차점 중심을 지나친 뒤일 수 있고, 특히 `T`는 통과 node일 수도 있기 때문이다.

image-space phase:

```text
far
  - intersection center가 아직 turn zone보다 멀다
  - required_turn evidence가 있으면 pre-arm만 한다

approaching
  - center_y_norm이 turn zone에 가까워진다
  - required_turn candidate를 유지하고 speed-down 요청 후보로 표시한다

turn_zone
  - center_y_norm이 turn_zone_y_min/max 안에 있다
  - front_missing 또는 turn_expected가 충분하면 TurnReady 가능

late
  - center_y_norm이 late_zone_y를 넘었다
  - 아직 TurnReady가 아니면 overshoot_risk=true

passed
  - center가 사라지거나 line shape가 교차점 이후 형태로 바뀐다
  - queue clear 또는 cooldown 후보
```

초기 구현의 한계:

- 실제 world distance는 아직 알 수 없다.
- 따라서 `turn_zone_y_min/max`는 현장 capture로 보정해야 한다.
- Raspberry Pi에서 실제 고도/카메라 각도/속도 조건으로 log를 찍고, 교차점 중심에 가장 가깝게 멈춰야 하는 image y band를 찾는다.

미래 control 연결 원칙:

- `turn_candidate=true`가 처음 뜨면 속도를 낮춘다.
- `TurnReady`가 뜨면 hover 또는 brake command를 낸다.
- `overshoot_risk=true`이면 급회전하지 말고 지나친 교차점으로 표시하거나 recovery state로 넘긴다.
- 회전 완료 후 `notifyTurnCompleted()`를 호출해서 cooldown을 시작한다.

## 7. Telemetry / Log / Overlay 확장

### 7.1 Onboard protocol 확장

새 telemetry object 후보:

```json
"intersection_decision": {
  "state": "cruise",
  "action": "continue",
  "accepted_type": "straight",
  "best_observed_type": "T",
  "event_ready": false,
  "turn_candidate": false,
  "required_turn": false,
  "front_available": true,
  "node_recorded": false,
  "cooldown_active": false,
  "accepted_branch_mask": 3,
  "window_frames": 6,
  "age_ms": 500,
  "confidence": 0.78,
  "center_px": { "x": 480.0, "y": 360.0 },
  "center_y_norm": 0.55,
  "approach_phase": "turn_zone",
  "overshoot_risk": false,
  "too_late_to_turn": false,
  "node": {
    "valid": true,
    "id": 12,
    "local_coord": { "x": 3, "y": 1 },
    "topology": "T",
    "arrival_heading": "east",
    "camera_branch_mask": 11,
    "grid_branch_mask": 14,
    "origin_local_only": true
  },
  "branches": [
    { "direction": "front", "present_frames": 2, "max_score": 0.84, "average_score": 0.75 },
    { "direction": "right", "present_frames": 0, "max_score": 0.20, "average_score": 0.12 }
  ]
}
```

기존 field는 유지한다.

- `vision.intersection`
- `vision.intersection_detected`
- `vision.intersection_score`

새 object 위치:

```text
vision.intersection_decision
```

또는:

```text
mission.intersection_decision
```

권장:

- 이번 단계는 아직 mission command가 없으므로 `vision.intersection_decision`으로 둔다.
- 나중에 mission telemetry가 본격화되면 `mission.grid` 또는 `mission.intersection`으로 이동/복제한다.

### 7.2 Debug latency

새 debug field:

```json
"intersection_decision_latency_ms": 0.05
```

예상 비용:

- ring buffer 크기는 6~12 samples 수준이다.
- 계산량은 branch 4개에 대한 단순 누적이므로 사실상 무시 가능하다.

### 7.3 Onboard console log

현재:

```text
ix_type=T ix_raw=T ix_valid=yes ix_score=0.86 branches=...
```

추가:

```text
dec_state=cruise
dec_action=continue
dec_type=+
dec_best=T
dec_window=6
dec_conf=0.82
dec_mask=15
dec_event=record_node
dec_front=yes
dec_req_turn=no
grid=(3,1)
grid_origin=local
dec_y=0.55
dec_phase=turn_zone
dec_overshoot=no
dec_ms=0.04
```

예시:

```text
frame=3412 line=yes ix_type=T ix_raw=L ix_score=0.77 branches=F:0.91,R:0.0,B:0.72,L:0.88 dec_state=node_record dec_action=record_node dec_type=T dec_front=yes dec_req_turn=no grid=(3,1) dec_window=6 dec_conf=0.80 dec_y=0.55 dec_phase=turn_zone dec_overshoot=no
```

### 7.4 GCS log

`VisionLogFormatter.cpp`에 추가:

```text
[intersection-decision] state=cruise action=continue accepted=T best=T event=node turn=no front=yes coord=(3,1) origin=local conf=0.78 y=0.55 phase=turn_zone overshoot=no window=6 branches=F:2/0.84 R:0/0.20 B:1/0.72 L:2/0.90
```

### 7.5 GCS overlay

초기 overlay는 과하지 않게 추가한다.

추가 표시:

- cyan intersection label 옆에 decision state/action 한 줄
- `DEC T node (3,1)` 또는 `DEC turn? front=no`
- local coordinate `(x,y)` 표시
- turn zone이면 `ZONE`, late/overshoot이면 `LATE` 또는 `OVR` 표시
- cooldown 중이면 `CD 5`처럼 남은 frame 표시

주의:

- 영상 위 텍스트가 이미 많다.
- branch score label과 겹치지 않게 center 아래쪽 또는 우상단 작은 text로 둔다.
- line/intersection overlay보다 mission decision overlay를 더 크게 만들지 않는다.

## 8. Config / OpenCV 값 조정 계획

### 8.1 당장 기본값으로 바꾸지 않을 것

다음 값은 당장 기본값 변경하지 않는다.

- `line.local_contrast_blur`
- `line.morph_close_kernel`
- `line.process_width`
- `line.intersection_threshold`
- `camera.width`
- `camera.height`

이유:

- 현재 문제는 detector 자체와 mission decision이 섞여 있다.
- 먼저 decision layer로 downgrade에 강해지는지 확인해야 한다.
- 이후에도 밝은 배경 `unknown`이 많으면 OpenCV 값을 근거 있게 바꾼다.

### 8.2 실험 후보

Decision layer 구현 후 비교할 후보:

| Profile | Camera | process_width | blur | close | intersection_threshold | 목적 |
|---|---:|---:|---:|---:|---:|---|
| baseline | 960x720 | 480 | 31 | 7 | 0.80 | 현재 기준 |
| fast-line | 640x480 | 320 | 21 | 5 | 0.75 | line/intersection 성능 우선 |
| balanced | 960x720 | 320 | 25 | 7 | 0.75 | ArUco 해상도 유지 + vision CPU 절감 |
| recall | 960x720 | 480 | 25 | 9 | 0.70 | 밝은 배경 branch recall 우선 |

평가 기준:

- `+`가 `T/L`로 downgrade되는 비율
- `T`가 `L/unknown`으로 downgrade되는 비율
- `L` 정확도 유지
- `straight`가 `unknown`으로 떨어지는 비율
- false upgrade 여부: `L -> T/+`, `T -> +`
- `intersection_latency_ms`
- `processing_fps`
- `video_send_failures`

### 8.3 왜 지금 바로 blur를 낮추지 않는가

사용자 판단처럼 640x480 + 낮은 blur는 성능 측면에서 타당한 후보이다.

하지만:

- 현재 line/intersection 처리는 `process_width=480` work image에서 돌아간다.
- camera만 640x480으로 낮추고 `process_width=480`을 유지하면 line/intersection CPU는 크게 줄지 않는다.
- blur를 낮추면 밝은 배경에서 edge strip 문제가 완화될 수도 있지만, 반대로 noise fragment가 늘 수 있다.
- therefore `camera`, `process_width`, `blur`, `close`, `threshold`는 묶어서 profile로 검증해야 한다.

## 9. 구현 단계

### Phase 0: 기준 확인

목표:

- 현재 repo 상태에서 plan 기준이 맞는지 확인한다.

작업:

1. `uav-onboard`와 `uav-gcs`가 clean인지 확인한다.
2. `uav-onboard` 기본 tests 실행.
3. OpenCV 사용 가능 환경에서는 `test_intersection_detector`도 실행.
4. `uav-gcs` tests 실행.
5. protocol docs onboard/GCS 동일성 확인.

완료 기준:

- 현재 구현을 깨지 않고 다음 작업 시작 가능.

### Phase 1: Config 구조 추가

파일:

```text
uav-onboard/src/common/VisionConfig.hpp
uav-onboard/src/common/VisionConfig.cpp
uav-onboard/config/vision.toml
```

작업:

1. `IntersectionDecisionConfig` 구조체 추가.
2. `VisionConfig`에 `intersection_decision` field 추가.
3. TOML `[intersection_decision]` parsing 추가.
4. default 값은 code와 toml에 동일하게 반영.
5. README에 새 config 의미 추가.

테스트:

- config parser는 현재 별도 test가 부족하다. 필요하면 `test_vision_config.cpp`를 추가한다.
- 최소한 build로 compile regression 확인.

### Phase 2: IntersectionDecisionEngine 구현

파일:

```text
uav-onboard/src/mission/IntersectionDecision.hpp
uav-onboard/src/mission/IntersectionDecision.cpp
```

작업:

1. `IntersectionDecisionSample`, `BranchEvidence`, `IntersectionDecision` 구조 정의.
2. `IntersectionDecisionEngine` class 구현.
3. 내부 ring buffer는 `std::deque` 또는 고정 크기 vector index로 구현.
4. `reset()` 구현.
5. `startCooldown()` 또는 `notifyTurnCompleted()` API 추가.
6. branch evidence 누적 helper 구현.
7. `classifyWindow()` 구현.
8. state transition 구현.
9. `center_y_norm` 계산과 `approach_phase` 판정 구현.
10. `front_available`과 `required_turn` 계산 구현.
11. `TurnReady`는 `front_available=false` 또는 `turn_expected=true`, side branch evidence, turn zone 조건이 모두 맞을 때만 발생하도록 제한.
12. late zone 이후에는 `overshoot_risk`와 `too_late_to_turn`을 표시하고 무리한 turn action은 만들지 않음.

권장 API:

```cpp
class IntersectionDecisionEngine {
public:
    explicit IntersectionDecisionEngine(const common::IntersectionDecisionConfig& config);

    IntersectionDecision update(
        const vision::IntersectionDetection& intersection,
        int frame_width,
        int frame_height,
        std::uint32_t frame_seq,
        std::int64_t timestamp_ms,
        bool turn_expected);

    void reset();
    void startCooldown();
};
```

초기 `turn_expected` 처리:

- 지금은 항상 `true`로 두면 turn 후보 확인 동작을 보기 쉽다.
- 하지만 실전 로직 전에는 CLI/config로 선택 가능하게 할 수도 있다.
- 추천 초기값은 `turn_expected = true`가 아니라 `false`다. 이유는 실제 control 없는 상태에서 `TurnReady`가 계속 떠도 행동이 없기 때문이다.
- 대신 debug log에서 `turn_candidate=true`를 표시한다.

절충:

- `turn_expected`가 false여도 front-missing 기반 `turn_candidate`는 표시한다.
- front branch가 사라진 경우는 policy 없이도 required turn 후보가 될 수 있다.
- `turn_expected=true`는 front branch가 있어도 snake/grid policy상 의도적으로 회전해야 하는 planned turn에 사용한다.

### Phase 2b: GridCoordinateTracker 구현

파일:

```text
uav-onboard/src/mission/GridCoordinateTracker.hpp
uav-onboard/src/mission/GridCoordinateTracker.cpp
```

작업:

1. `GridCoord`, `GridHeading`, `GridNodeEvent` 구조 정의.
2. 첫 안정 node를 local `(0,0)`으로 초기화.
3. `origin_local_only=true`를 기본값으로 유지.
4. `camera_branch_mask`를 current heading 기준 `grid_branch_mask`로 회전 변환.
5. 같은 node 중복 기록 lockout 구현.
6. heading이 확정된 뒤 다음 node마다 `local_coord += heading_vector(current_heading)` 적용.
7. diagonal entry 또는 heading 미확정 상태를 위한 `entry_bootstrap` 처리.
8. `notifyTurnCompleted(new_heading)`로 heading 갱신.
9. 발견 node map을 `std::map<GridCoord, GridNodeEvent>` 또는 작은 vector로 보관.

주의:

- 이번 단계에서는 official `(0,0)` 확정을 하지 않는다.
- 탐색 시작 기준 local `(0,0)`만 기록한다.
- 나중에 경기장 규칙상 official origin이 확인되면 local map을 translation/rotation해서 변환한다.

### Phase 3: Unit tests

파일:

```text
uav-onboard/tests/test_intersection_decision.cpp
uav-onboard/tests/CMakeLists.txt
```

테스트 케이스:

1. Cruise에서 `unknown`/`straight`가 섞여도 action은 `ContinueStraight`.
2. 최근 6 frame 중 4방향 branch가 각각 2회 이상 나오면 `RecordNode`.
3. `+`가 단 1회만 튀면 `RecordNode`가 나오지 않음.
4. `T`가 `L`/`unknown`으로 downgrade되어도 3방향 evidence가 충분하면 accepted type은 `T`.
5. `L`에서 한 frame `+` false upgrade가 튀어도 4방향 evidence 부족이면 `+`가 되지 않음.
6. `T`가 떠도 front branch가 있으면 `required_turn=false`, action은 `ContinueStraight`.
7. front branch가 사라지고 left/right branch가 충분하면 `turn_candidate=true`.
8. `turn_expected=true` 또는 front-missing이 충분하고 center가 turn zone 안이면 `TurnReady`.
9. turn evidence가 충분해도 center가 turn zone 밖이면 `TurnReady`가 나오지 않음.
10. late zone 이후 required turn이 확정되면 `overshoot_risk=true`, `too_late_to_turn=true`.
11. 첫 node는 local `(0,0)`으로 기록됨.
12. heading이 확정된 뒤 다음 node는 heading 방향으로 좌표가 1칸 증가함.
13. 같은 node lockout 동안 `RecordNode`가 중복 발생하지 않음.
14. camera-relative branch mask가 heading에 따라 grid-relative branch mask로 변환됨.
15. `startCooldown()` 후 window가 비워지고 cooldown_frames 동안 판단 무시.

완료 기준:

- 기존 tests와 새 test 모두 통과.

### Phase 4: VisionDebugPipeline 통합

파일:

```text
uav-onboard/src/app/VisionDebugPipeline.cpp
uav-onboard/CMakeLists.txt
```

작업:

1. `IntersectionDecisionEngine` instance 생성.
2. `GridCoordinateTracker` instance 생성.
3. 매 frame `result.intersection` 이후 frame width/height와 함께 decision update 호출.
4. `decision.event_ready`인 경우 coordinate tracker update 호출.
5. decision/grid latency 측정.
6. `VisionResult` 또는 local telemetry conversion에 decision과 grid node 추가.
7. console log에 `dec_*`, `grid_*` fields 추가.
8. startup log에 `[intersection_decision]` config 출력.
9. `center_y_norm`, `approach_phase`, `overshoot_risk`, `local_coord`, `front_available`을 GCS 없이도 console에서 확인 가능하게 출력.

주의:

- decision update는 line이 disabled이면 실행하지 않는다.
- image decode 실패 시 decision도 빈 값 처리.
- `IntersectionDecisionEngine`은 GCS video 여부와 무관해야 한다.
- `GridCoordinateTracker`도 GCS video 여부와 무관해야 한다.

### Phase 5: Telemetry protocol 확장

파일:

```text
uav-onboard/src/protocol/TelemetryMessage.hpp
uav-onboard/src/protocol/TelemetryMessage.cpp
uav-onboard/docs/PROTOCOL.md
uav-gcs/src/protocol/TelemetryMessage.hpp
uav-gcs/src/protocol/TelemetryMessage.cpp
uav-gcs/docs/PROTOCOL.md
```

작업:

1. `IntersectionDecisionTelemetry` 구조체 추가.
2. `BranchEvidenceTelemetry` 구조체 추가.
3. `GridNodeTelemetry` 구조체 추가.
4. onboard JSON serialization 추가.
5. GCS parser 추가.
6. protocol 문서 v1.7로 업데이트.
7. 기존 `protocol_version` integer는 `1` 유지.
8. onboard/GCS protocol 문서 동일성 확인.

새 field:

```text
vision.intersection_decision
vision.grid_node
debug.intersection_decision_latency_ms
```

`vision.intersection_decision`에는 최소한 다음 field를 포함한다.

```text
state
action
accepted_type
best_observed_type
turn_candidate
event_ready
required_turn
front_available
center_y_norm
approach_phase
overshoot_risk
too_late_to_turn
branch evidence
node/local coordinate
```

테스트:

- `uav-onboard/tests/test_telemetry_line_json.cpp` 확장
- `uav-gcs/tests/test_telemetry_line_parse.cpp` 확장

### Phase 6: GCS log/overlay 통합

파일:

```text
uav-gcs/src/telemetry/TelemetryStore.hpp
uav-gcs/src/telemetry/TelemetryStore.cpp
uav-gcs/src/telemetry/VisionLogFormatter.cpp
uav-gcs/src/overlay/IntersectionOverlay.cpp
```

작업:

1. `VisionFrame`에 intersection decision 추가.
2. `VisionFrame`에 latest grid node/local coordinate 추가.
3. `[intersection-decision]` log line 추가.
4. `[grid-node]` log line 추가.
5. overlay label에 decision state/action/local coordinate 추가.
6. `test_intersection_overlay.cpp` 또는 새 `test_intersection_decision_overlay.cpp` 추가/확장.

주의:

- overlay text는 너무 많이 늘리지 않는다.
- 우선 log 중심으로 검증하고 overlay는 최소 표시만 한다.

### Phase 7: Runtime smoke

로컬:

```powershell
cd uav-onboard
cmake -S . -B build-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

OpenCV local:

```powershell
cmake -S . -B build-opencv-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -DOpenCV_DIR=C:/msys64/ucrt64/lib/cmake/opencv4
cmake --build build-opencv-tests
$env:PATH='C:\msys64\ucrt64\bin;' + $env:PATH
ctest --test-dir build-opencv-tests --output-on-failure
```

GCS:

```powershell
cd uav-gcs
cmake -S . -B build-tests -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build-tests
ctest --test-dir build-tests --output-on-failure
```

Raspberry Pi:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --video --fps 12
```

확인:

- console에 `dec_state`, `dec_action`, `dec_type`, `dec_conf` 표시
- console/GCS log에 `dec_y`, `dec_phase`, `dec_overshoot`, `grid=(x,y)` 표시
- GCS log에 `[intersection-decision]` 표시
- GCS log에 `[grid-node]` 표시
- `L/T/+` 모두 멈춤 없이 `record_node` event가 나오는지
- `T`에서 front branch가 있으면 `required_turn=false`로 유지되는지
- front branch가 사라지는 node에서만 turn candidate가 생기는지
- turn zone 전에는 candidate로만 보이고, turn zone 안에서만 `TurnReady`가 가능한지
- late zone 이후에는 `overshoot_risk`가 뜨는지
- `unknown/straight`가 cruise에서 action을 만들지 않는지

## 10. Acceptance Criteria

기능 완료:

- `IntersectionDecisionEngine`이 최근 frame queue 기반으로 branch evidence를 누적한다.
- 우선순위는 `+ > T > L > straight > unknown`이지만, 단일 frame false upgrade를 채택하지 않는다.
- `L/T/+`는 `RecordNode` event로 표시되고 front branch가 있으면 계속 전진 action이 유지된다.
- 첫 안정 node는 local `(0,0)`으로 기록된다.
- subsequent node는 current heading 기준으로 local coordinate가 증가한다.
- `T`라도 front branch가 있으면 `required_turn=false`로 유지된다.
- `TurnReady`는 front branch 부재 또는 future grid policy의 turn 요구와 side branch evidence, `center_y_norm` turn zone 조건이 모두 맞을 때만 발생한다.
- late zone 이후 required turn이 확정되면 즉시 turn action을 만들지 않고 `overshoot_risk`/`too_late_to_turn`을 표시한다.
- cooldown 시작 시 queue가 비워지고 일정 frame 동안 판단이 무시된다.
- telemetry/log/GCS에서 detector type, decision type, local grid coordinate를 구분해서 볼 수 있다.

품질 완료:

- 기존 onboard/GCS tests가 계속 통과한다.
- 새 decision unit tests가 downgrade, false upgrade, cooldown, node lockout을 검증한다.
- 새 decision unit tests가 front branch 기반 turn 판단, turn zone gating, overshoot risk를 검증한다.
- 새 grid coordinate tests가 first-node origin, heading-based coordinate advance, branch mask rotation을 검증한다.
- protocol 문서가 onboard/GCS 양쪽에서 동일하다.
- debug video best-effort 원칙을 건드리지 않는다.
- onboard 영상 overlay drawing은 추가하지 않는다.

성능 완료:

- `intersection_decision_latency_ms`는 0.1ms 수준 또는 그 이하가 기대된다.
- `processing_fps`가 decision layer 추가 후에도 12FPS 근처를 유지한다.
- ring buffer 크기는 작고 heap churn이 크지 않아야 한다.

## 11. 이후 단계

Decision layer가 안정적으로 동작한 뒤 다음을 진행한다.

1. 실제 캡처 기반 profile 비교: baseline / fast-line / balanced / recall
2. 밝은 배경에서 `intersection_threshold = 0.70~0.75` 후보 비교
3. `process_width = 320`, `local_contrast_blur = 21~25` 성능 비교
4. local grid map을 탐색된 bounding box 기준으로 official coordinate 후보에 변환
5. start entry가 official `(0,0)`인지 중간 node인지 경기장 규칙/마커/탐색 결과로 판정
6. snake search policy 추가
7. 실제 고도/속도에서 `turn_zone_y_min/max`, `late_zone_y` 보정
8. 12FPS 기준 최대 cruise speed와 approach speed 산정
9. `turn_expected`를 실제 grid policy에서 공급
10. Pixhawk/MAVLink control loop 연결
11. `turn_candidate -> speed down`, `TurnReady -> brake/hover/turn` 제어 연결

## 12. 핵심 리스크

- `+` false upgrade가 드물게라도 있으면 위험하므로 단일 최고 타입 채택은 금지한다.
- branch evidence threshold를 너무 낮추면 밝은 바닥 무늬가 false branch가 될 수 있다.
- threshold를 너무 높이면 현재처럼 downgrade가 계속된다.
- 실제 드론 이동 속도에서는 0.5초 window도 이동 거리가 생긴다. 속도와 frame rate 기준으로 window frame 수를 조정해야 한다.
- 12FPS에서 빠르게 전진하면 required turn 확정 전에 교차점 중심을 지나칠 수 있다. 따라서 speed-down pre-arm과 turn zone gating이 필요하다.
- image-space turn zone은 고도/카메라 각도/속도에 의존한다. 실제 RasPi/기체 조건에서 반드시 보정해야 한다.
- 첫 grid node를 local `(0,0)`으로 두면 official origin과 다를 수 있다. 이 차이를 telemetry에 `origin_local_only=true`로 명시해야 한다.
- entry segment가 diagonal이면 첫 heading이 grid axis와 바로 일치하지 않을 수 있다. `entry_bootstrap` 없이 좌표를 확정하면 map이 회전/전단될 수 있다.
- `T`를 무조건 회전 지점으로 해석하면 외곽 중간 node에서 불필요하게 멈추거나 회전할 수 있다. action은 반드시 front branch와 grid policy 기반이어야 한다.
- 회전 후 cooldown은 반드시 필요하다. 그렇지 않으면 방금 지나온 교차점이나 회전 중 화면 흔들림을 다음 교차점으로 오인할 수 있다.
- 아직 control이 없으므로 `TurnReady`는 실제 행동이 아니다. telemetry/log상 decision event로만 취급해야 한다.
