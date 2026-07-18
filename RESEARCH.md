# Research: uav-onboard grid mission state-machine work

작성일: 2026-05-20 / 결과 검토: 2026-07-18

> **상태: 조사 완료된 역사 기록.** 조사 당시 중심이던
> `tools/grid_mission_node.cpp`는 제거되었고 현재 composition root는
> `src/app/AstroquadOnboardApp.*`, 실행 파일은 `astroquad-onboard`다.
> 당시 제안한 marker-lock → blind entry → origin latch → hop-distance snake
> 흐름, marker window, 재방문과 복귀는 현재 `GridMission`에 반영되어 있다.
> 현재 구조는 `uav-onboard/ARCHITECTURE_OVERVIEW.md`, 실행·실비행 검증은
> [REAL_FLIGHT_ONBOARDING.md](REAL_FLIGHT_ONBOARDING.md)를 우선한다. 아래 파일명,
> 미연결 설정, 테스트 부족 평가는 2026-05-20 당시의 관찰로 보존한다.

## 읽은 문서와 코드

- `development-log/SYSTEM_SPEC.md`
- `uav-onboard/README.md`
- `uav-onboard/CMakeLists.txt`
- `uav-onboard/config/mission.toml`
- `uav-onboard/src/main.cpp`
- `uav-onboard/src/mission/GridMission.hpp`
- `uav-onboard/src/mission/GridMission.cpp`
- `uav-onboard/src/mission/GridCoordinateTracker.cpp`
- `uav-onboard/src/mission/IntersectionDecision.cpp`
- `uav-onboard/src/mission/SnakePlanner.cpp`
- `uav-onboard/src/control/GridControlMapper.hpp`
- `uav-onboard/src/control/GridControlMapper.cpp`
- `uav-onboard/tools/grid_mission_node.cpp`
- 관련 테스트 일부: `tests/CMakeLists.txt`, `tests/test_line_follow_mission.cpp`, `tests/test_autopilot_setpoints.cpp`

## 전체 프로젝트 이해

Astroquad는 GPS 없는 실내/운동장 격자 환경에서 UAV가 하향 카메라 기반 vision으로 임무를 수행하는 C++ 시스템이다.

최종 임무 흐름은 다음이다.

```text
이륙
  -> 라인 추종 및 격자 진입
  -> snake 방식 grid 탐색
  -> 교차점별 ArUco marker 기록
  -> marker 번호 역순 재방문
  -> 출발점 복귀
  -> 자동 착륙
```

Repo 경계는 다음처럼 나뉜다.

- `uav-onboard`: Raspberry Pi 카메라 처리, line/intersection/ArUco detection, mission state machine, guidance/control, MAVLink adapter, safety, telemetry/debug video 송신.
- `uav-gcs`: onboard telemetry/video 수신, vision overlay, grid-map log/display, 향후 command sender/dashboard.

현재 onboard 쪽은 최종 실행 파일 `uav_onboard`가 아직 기본 telemetry bring-up 수준이고, 실제 비행/vision 통합 검증은 staging executable들이 담당한다.

- `vision_debug_node`: camera + vision + telemetry + optional debug video. MAVLink control을 넣지 않는 디버그/튜닝용.
- `line_follow_node`: SITL/실기체 line-follow MVP staging.
- `grid_mission_node`: grid arena에서 full snake mission state machine을 검증하기 위한 현재 작업 중심 staging executable.
- `video_streamer`: raw MJPEG transport smoke tool.

중요한 구조 원칙은 detector 코드를 mission executable에 복사하지 않고 `onboard_core`, `onboard_vision`, `onboard_video`, `onboard_autopilot` 같은 library target으로 공유하는 것이다.

## onboard 주요 코드 기준선

`src/main.cpp`의 `uav_onboard`는 아직 최종 composition root가 아니다. 현재는 `network.toml`을 읽고 UDP bring-up telemetry를 주기적으로 보내는 단순 sender다.

실제 mission 구성은 staging tool 쪽에 있다. `tools/grid_mission_node.cpp`는 다음을 한 루프로 묶는다.

- frame source 선택 및 vision processing
- `IntersectionDecisionEngine` 업데이트
- `GridCoordinateTracker` peek/update 및 mission 승인 후 commit
- `GridMission` state machine 업데이트
- `GridControlMapper`로 mission intent를 body velocity setpoint로 변환
- `AutopilotMavlinkAdapter`를 통한 body velocity / LAND command 송신
- `VisionDebugPublisher`를 통한 GCS telemetry/video publish

mission/control 핵심 분리는 다음과 같다.

- `GridMission`: 임무 상태, grid node commit gate, marker 기록, snake boundary 판단, land/emergency land 전환을 담당한다.
- `GridCoordinateTracker`: intersection decision을 grid coordinate event로 peek하고, mission이 승인한 event만 commit한다.
- `IntersectionDecisionEngine`: raw intersection detection을 sliding evidence window로 안정화하고 `NodeRecord`, `TurnReady` 같은 event state를 만든다.
- `SnakePlanner`: boundary에서 다음 snake turn 방향과 완료 여부를 판단한다.
- `GridControlMapper`: `GridControlIntent`를 velocity/yaw/altitude setpoint로 변환한다.

## git diff 요약

`uav-onboard`의 현재 변경 파일은 5개다.

```text
config/mission.toml
src/control/GridControlMapper.cpp
src/mission/GridMission.cpp
src/mission/GridMission.hpp
tools/grid_mission_node.cpp
```

변경량은 대략 `478 insertions(+), 349 deletions(-)`이며, 전부 grid mission state machine과 그 설정/제어 연결부에 집중되어 있다.

## 최근 작업으로 추정되는 내용

최근 작업은 `Cycle 16`으로 표시된 grid arena 대응 리팩터링으로 보인다. 핵심 의도는 기존 "vertiport에서 grid까지 line을 찾아 들어간다"는 가정을 버리고, 새 arena layout에 맞게 "vertiport marker 위에서 yaw를 맞춘 뒤 blind forward로 첫 교차점까지 진입"하는 방식으로 바꾸는 것이다.

상태머신 변경:

- 제거된 상태: `VertiportYawAlign`, `OffPadForward`, `GridOriginLock`
- 이름/역할 변경: `VertiportVerify` -> `MarkerLockYaw`, `LineEnter` -> `EntryForward`
- 새 흐름:

```text
ArmTakeoff
  -> MarkerLockYaw
  -> EntryForward
  -> SnakeForward
  -> SnakeRecordNode / SnakeStopAtCenter / SnakeTurn90 / SnakeAdvanceOneCell / SnakeTurn90Again
  -> SnakeComplete
  -> Land
```

`MarkerLockYaw`의 목적:

- vertiport ArUco marker ID 23을 안정적으로 보고 marker center error가 tolerance 안에 들어오게 한다.
- 이와 동시에 이륙 시 yaw 기준에서 `marker_lock_yaw_delta_deg`만큼 회전한다.
- 현재 config는 `+90.0 deg`이며 주석상 ArduPilot/NED convention에서 positive yaw rate를 clockwise/right로 본다.
- `GridControlMapper`가 `MarkerHover` intent에서도 `target_yaw_rad`를 향한 yaw rate를 내도록 바뀌었다. 기존 marker hover controller가 yaw rate를 0으로 고정했기 때문에, marker 중심 유지와 yaw 회전을 동시에 하려는 변경으로 보인다.

`EntryForward`의 목적:

- 새 grid arena에는 vertiport에서 첫 grid intersection까지 이어지는 line이 없다고 가정한다.
- 따라서 post-rotation yaw를 고정하고 `ForwardBlind`로 전진한다.
- 첫 intersection을 감지하면 그 지점을 grid origin `(0,0)`으로 강제 설정하고, GCS에 synthetic origin event를 publish한다.
- vertiport texture나 ArUco 주변 무늬가 intersection으로 오인되는 것을 막기 위해 `hop_intersection_min_distance_m` 이상 이동해야 첫 intersection으로 인정한다.

hop-to-hop snake 변경:

- `SnakeForward`와 `SnakeAdvanceOneCell`은 `hop_start_local_x/y`를 기준으로 LOCAL_NED 이동거리를 계산한다.
- line following은 cell 전체에서 계속 켜지지 않고 `hop_align_start_m <= distance <= hop_align_end_m` 구간에서만 짧게 열린다.
- 그 외 구간은 `ForwardBlind`로 yaw를 고정해 라인/교차점 contour가 yaw를 흔드는 문제를 줄이려는 것으로 보인다.
- 다음 node commit은 `hop_intersection_min_distance_m` 이상 이동한 뒤의 `NodeRecord`만 받는다.
- `hop_max_distance_m`를 넘도록 다음 intersection을 못 보면 emergency land로 간다.

marker 안정화 변경:

- 기존 per-ID consecutive frame counter인 `marker_candidate_count_`가 제거됐다.
- `MarkerWindow`라는 sliding window가 추가되어 최근 `marker_window_frames` 프레임 안에서 같은 ID가 `marker_window_min_count`번 이상 나타나야 stable marker로 본다.
- window 안에 서로 다른 실제 marker ID가 섞이면 window를 flush한다.
- 의도는 intersection cross/T pattern에서 ArUco false positive가 몇 프레임 누적되어 잘못 commit되는 문제를 막는 것이다.

config 변경:

- `marker_lock_center_tol_norm`
- `marker_lock_yaw_delta_deg`
- `entry_forward_timeout_s`
- `entry_forward_speed_mps`
- `hop_align_start_m`
- `hop_align_end_m`
- `hop_max_distance_m`
- `hop_intersection_min_distance_m`
- `marker_window_frames`
- `marker_window_min_count`

값 조정:

- `vertiport_altitude_m`, `cruise_altitude_m`: 2.0m 기준.
- `snake_record_dwell_s`: 0.5s로 증가. node에서 잠깐 정지해 branch classification이 안정화되도록 한 것으로 보인다.
- `snake_post_turn_blind_s`: 1.5s.
- `snake_advance_timeout_s`: 15s.
- `forward_speed_advance_mps`: 0.30m/s.
- `altitude_ceiling_m`: 3.5m.

## 현재 작업이 향하던 목표

현재 변경은 "grid_arena_test_world에서 full snake mission staging을 실제로 굴리기 위한 Cycle 16 조정"으로 추정된다.

구체적으로는 다음 문제들을 해결하려던 흔적이 강하다.

- vertiport와 grid 사이에 line이 없는 새 arena layout 때문에 기존 `OffPadForward`/`LineEnter`/`GridOriginLock` 흐름이 맞지 않음.
- vertiport texture와 ArUco 주변 패턴이 false intersection event를 만들어 mission이 너무 일찍 grid origin 또는 boundary를 잡음.
- cell 주행 중 line/intersection detector의 순간적인 T/L/+ 또는 tilted line fit이 yaw controller를 흔듦.
- boundary turn 이후 이전 corridor의 잔여 line을 쫓아 column transition이 틀어짐.
- ArUco false positive가 marker commit으로 이어짐.

따라서 새 전략은 "marker lock + yaw rotate -> blind entry -> first intersection origin latch -> hop-distance gated snake traversal -> mid-cell only line correction -> sliding-window marker commit"이다.

## 확인된 주의점

빌드 확인:

```text
cmake --build build --target grid_mission_node
```

결과: 성공.

주의할 만한 잔여점:

- `entry_forward_speed_mps`는 config와 `GridMissionConfig`에 추가됐지만, 현재 `GridControlMapper`의 `ForwardBlind` 속도에는 연결되지 않은 것으로 보인다. 실제 blind forward 속도는 여전히 `GridControlMapperConfig::forward_speed_blind_mps` 기본값/설정 경로를 따른다.
- `tools/grid_mission_node.cpp`에서 `marker_lock_yaw_delta_deg`를 읽을 때 `value_or(-90.0)`을 사용한다. 현재 `config/mission.toml`에는 `+90.0`이 있어서 런타임 값은 의도대로 읽히지만, config key가 빠지면 `GridMissionConfig` 기본값 `+90 deg`와 다른 `-90 deg`가 적용된다.
- `GridMission.cpp`의 `handleMarkerLockYaw` 주석에는 "default -pi/2 = right 90"처럼 현재 config/default와 충돌하는 문구가 남아 있다.
- `GridMission.hpp`의 `stableMarkerCandidateCount()` 주석은 아직 예전 `marker_observation_min_frames` 기준 설명을 담고 있다.
- `IntersectionDecision.hpp`, `AltitudePolicy.hpp`, `grid_mission_node.cpp` 일부 주석에는 `LineEnter`, `OFF_PAD_FORWARD`, `GRID_ORIGIN_LOCK` 같은 old state 이름이 남아 있다. 동작보다 문서/가독성 문제에 가깝다.
- `snake_line_lost_warn_s`, `snake_line_lost_emergency_s`, `entry_forward_speed_mps` 등 일부 설정은 현재 코드 경로에서 실제 decision/control에 쓰이지 않는 것으로 보인다. 의도된 예비값인지 미연결인지 확인이 필요하다.
- current tests 목록에는 `GridMission`의 새 `MarkerLockYaw -> EntryForward -> SnakeForward` 흐름을 직접 검증하는 focused unit test가 보이지 않는다.

## 다음 작업 후보

1. `entry_forward_speed_mps`를 실제 `ForwardBlind` 속도에 연결할지 결정한다. 연결한다면 `GridControlIntent`별 속도를 분리하거나 `GridMissionOutput`에 forward speed override를 추가하는 방식이 필요하다.
2. yaw 방향 기본값을 한 곳으로 통일한다. `mission.toml`, `GridMissionConfig`, `grid_mission_node.cpp` fallback, 주석이 모두 `+90 deg = right`로 일치해야 한다.
3. old state 주석을 정리한다. 동작 변경은 아니지만 state machine을 디버깅할 때 혼동을 줄인다.
4. `GridMission` focused tests를 추가한다.
   - takeoff altitude reached 후 `MarkerLockYaw` 진입
   - marker centered + yaw stable 후 `EntryForward` 진입
   - min hop distance 전 false intersection 무시
   - min hop distance 후 first intersection을 `(0,0)`으로 latch
   - hop align window에서만 `LineFollow`, 나머지는 `ForwardBlind`
   - mixed marker IDs가 `MarkerWindow`를 flush
5. grid arena SITL에서 `grid_mission_node`를 실행해 telemetry log 기준으로 state transition과 hop distance gate가 기대대로 움직이는지 확인한다.
