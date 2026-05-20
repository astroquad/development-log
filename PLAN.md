# Plan: grid mission `(0,0)` origin centering fix

작성일: 2026-05-20 KST

## 0. 현재 문제 요약

실행 조건:

```bash
./build/grid_mission_node --config config --target sitl --vision gazebo \
  --world grid --line-mode dark_on_light --marker-count 4 \
  --video --gcs-ip 172.22.128.1
```

관찰:

- `ARM_TAKEOFF -> MARKER_LOCK_YAW -> ENTRY_FORWARD`까지는 의도대로 진행된다.
- vertiport ArUco ID 23은 인식되고, yaw는 약 `1.55 rad`까지 돌아간다.
- `ENTRY_FORWARD` 중 첫 grid L 교차점을 보지만, 카메라 중심이 교차점 중심에 정렬되지 않은 상태에서 `(0,0)`으로 확정된다.
- 이후 `SNAKE_FORWARD`에서 다음 노드를 찾지 못하고 `EMERGENCY_LAND`로 들어간다.

핵심 로그:

```text
t=7.004   st=ENTRY_FORWARD yaw=1.54 idec=cruise
t=8.088   st=ENTRY_FORWARD idec=turn_ready type=L cy=0.54
t=8.595   st=ENTRY_FORWARD idec=turn_ready type=L cy=0.56
t=9.769   st=ENTRY_FORWARD idec=turn_confirm type=L cy=0.73
t=10.341  st=SNAKE_FORWARD coord=(0,0) hd=north nodes=1 idec=turn_confirm type=L cy=0.72
...
t=13.771  st=SNAKE_FORWARD line=0
...
t=20.139  st=EMERGENCY_LAND
```

`cy=0.72`인 상태에서 이미 `(0,0)`으로 넘어갔다는 점이 중요하다. 현재 `stop_center_target_cy = 0.55` 기준으로 보면 첫 origin latch가 교차점 중심 정렬 뒤가 아니라, 교차점이 화면 하단으로 지나간 뒤에 발생하고 있다.

## 1. 원인 분석

직접 원인:

- `GridMission::handleEntryForward()`가 첫 교차점을 `intersection_seen && min_progress_ok`만으로 origin으로 확정한다.
- 이 조건에는 교차점 중심이 카메라 중심 또는 목표 Y 위치에 왔는지 확인하는 gate가 없다.
- origin을 확정하는 순간 `last_node_local_x/y`와 `hop_start_local_x/y`도 현재 LOCAL_NED 위치로 잡힌다.
- 따라서 실제 `(0,0)` 교차점 중심이 아니라, 교차점이 카메라에 보이기 시작했거나 화면 아래로 지난 위치가 다음 hop의 기준점이 된다.

코드상 직접 위치:

```text
uav-onboard/src/mission/GridMission.cpp
  handleEntryForward()
    if (intersection_seen && min_progress_ok) {
      tracker_->forceOrigin(GridCoord{0, 0}, GridHeading::North);
      last_node_local_x_ = *in.local_x_m;
      last_node_local_y_ = *in.local_y_m;
      armHopStart(in);
      transition(GridState::SnakeForward, ...);
    }
```

보조 원인:

- `ENTRY_FORWARD`에는 감속/정렬 상태가 없다. 첫 L이 화면에 들어와도 계속 `ForwardBlind`로 전진한다.
- `StopAndCenter`는 `SnakeStopAtCenter`에서만 사용되고, 첫 origin lock에는 쓰이지 않는다.
- 현재 `StopAndCenter`도 Y 방향 decel만 있고 X 방향 lateral centering은 없다.
- `entry_forward_speed_mps = 0.30`이 config에 있지만 실제 `ForwardBlind` 속도에는 연결되어 있지 않다. 현재 `ForwardBlind`는 `GridControlMapperConfig::forward_speed_blind_mps` 기본값 경로를 탄다.
- `IntersectionDecisionEngine`은 `ARM_TAKEOFF`/`MARKER_LOCK_YAW` 동안에도 계속 업데이트되어 vertiport texture에서 `+`, `T` 같은 candidate를 만든다. tracker commit은 막혀 있지만 decision window/record lockout에는 잔상이 남을 수 있다.
- `SNAKE_FORWARD`는 mid-cell align window에서만 line follow를 열기 때문에, origin 시작점이 틀어지면 line을 잃고 다음 node를 못 잡기 쉽다.

직접적인 failsafe는 `hop_max_distance_m` 초과로 추정된다.

- `entry_forward_timeout`은 이미 지나 `SNAKE_FORWARD`에 들어갔으므로 아님.
- altitude ceiling도 아님. 로그상 `agl ~= 2.0m`이고 ceiling check는 rangefinder를 우선 사용한다.
- heartbeat/mission timeout/max intersections도 정황상 아님.
- `SNAKE_FORWARD`에서 다음 교차점을 못 잡은 채 잘못된 hop 기준점으로 계속 진행하다 `hop_distance_exceeded`에 해당하는 경로로 `EmergencyLand`에 들어간 것으로 보는 것이 가장 자연스럽다.

## 2. 수정 목표

목표:

```text
MarkerLockYaw
  -> EntryForward
  -> EntryCenterOrigin
  -> SnakeForward
```

핵심 원칙:

- 첫 grid intersection을 "보았다"와 "그 위에 정렬했다"를 분리한다.
- `(0,0)` origin publish, tracker forceOrigin, `last_node_local_x/y`, `hop_start_local_x/y`는 정렬 완료 후에만 수행한다.
- 첫 origin 정렬 중에는 yaw를 `yaw_align_target_rad_`로 계속 고정한다.
- 교차점 center X/Y가 tolerance 안에 들어오고, 속도가 충분히 낮은 상태가 몇 frame 유지된 뒤에만 `SnakeForward`로 넘긴다.
- failsafe 값을 늘려 문제를 숨기지 않는다.

## 3. 구현 계획

### 3.1 GridMission state 추가

`GridState`에 새 상태를 추가한다.

```cpp
EntryCenterOrigin
```

상태 의미:

- `EntryForward`에서 첫 grid intersection candidate를 발견하면 바로 origin latch하지 않고 이 상태로 전환한다.
- 이 상태는 교차점 중심을 카메라 중심/목표점에 맞추는 전용 상태다.
- 정렬 완료 후에만 `(0,0)`을 publish하고 `SnakeForward`로 진입한다.

권장 흐름:

```text
EntryForward
  - yaw-frozen ForwardBlind
  - vertiport false positive guard
  - first L/T/+ candidate 발견
  -> EntryCenterOrigin

EntryCenterOrigin
  - yaw-frozen intersection centering
  - center_x_norm, center_y_norm tolerance 확인
  - velocity low/stable frames 확인
  - origin latch
  - decision engine reset/cooldown
  -> SnakeForward
```

### 3.2 EntryForward acceptance 조건 변경

현재 조건:

```cpp
intersection_seen && min_progress_ok
```

수정 후:

- `EntryForward`는 origin latch를 하지 않는다.
- `EntryForward`는 충분히 그럴듯한 첫 교차점을 보면 `EntryCenterOrigin`으로만 전환한다.
- false positive 방지를 위해 다음 gate를 같이 둔다.

권장 gate:

- `accepted_type`이 `L`, `T`, `Cross` 중 하나.
- `center_y_norm`이 너무 아래로 지나가지 않은 상태. 예: `center_y_norm < entry_center_late_y`.
- 최소 전진거리 gate는 기존 `hop_intersection_min_distance_m = 1.0`을 그대로 쓰기보다 entry 전용으로 분리한다.
  - 예: `entry_intersection_min_distance_m = 0.4~0.7`
  - 이유: 현재 로그에서 `cy=0.54~0.56`로 가장 좋은 순간이 t=8.1~8.6인데, 기존 1.0m gate 때문에 이 시점을 놓친 것으로 보인다.
- vertiport marker ID 23이 여전히 보여도 첫 grid line이 보일 수 있으므로 `mks=23 present` 자체를 hard reject로 쓰지는 않는다.
- 대신 `IntersectionDecisionEngine` reset과 distance/center gate로 vertiport texture 잔상을 제거한다.

### 3.3 EntryCenterOrigin control 구현

현재 `StopAndCenter`는 Y 방향만 처리한다.

```cpp
if (center_y_norm < stop_center_target_cy) {
    vx_forward = small positive taper;
}
```

첫 origin에는 X/Y 2D centering이 필요하다.

추가할 데이터:

- `GridMissionOutput::intersection_center_x_norm`
- `GridControlMapperInput::intersection_center_x_norm`
- 가능하면 log에도 `cx`, `cy`, `hop` 출력

정규화:

```text
center_x_norm = (center_px.x - width * 0.5) / (width * 0.5)
center_y_norm = center_px.y / height
```

권장 제어:

- `vx_forward_mps = kp_y * (entry_center_target_y - center_y_norm)`
- `vy_right_mps = kp_x * center_x_norm` 또는 marker hover와 같은 sign convention 재사용
- `yaw_rate_rad_s = computeYawRate(current_yaw, yaw_align_target)`
- `vz_down_mps = altitude hold`
- 속도는 작은 값으로 clamp한다. 예: `entry_center_max_v_mps = 0.10~0.15`
- `center_y_norm`이 target보다 커졌을 때는 아주 작은 reverse도 허용한다. 예: max reverse `0.05~0.08m/s`

주의:

- sign은 Gazebo GCS overlay로 확인한다. marker hover가 이미 vertiport 중심 잡기에 성공하므로 가능하면 그 sign convention과 맞춘다.
- 첫 구현에서는 `EntryCenterOrigin` 전용 intent를 새로 두는 것이 가장 안전하다. 기존 `StopAndCenter`를 snake boundary에도 쓰고 있기 때문에 동작을 무리하게 바꾸면 boundary turn이 같이 흔들릴 수 있다.

권장 새 intent:

```cpp
GridControlIntent::IntersectionCenter
```

### 3.4 Origin latch 조건

`EntryCenterOrigin`에서만 아래 작업을 수행한다.

- `tracker_->forceOrigin(GridCoord{0, 0}, GridHeading::North)`
- `intersections_recorded_ = 1`
- `last_node_local_x/y = current local position`
- synthetic `origin_publish_event`
- `armHopStart(in)`
- `transition(GridState::SnakeForward, ...)`

정렬 완료 조건:

- `abs(center_x_norm) <= entry_center_x_tol_norm`
- `abs(center_y_norm - entry_center_target_y) <= entry_center_y_tol_norm`
- `local_velocity_xy_mps <= snake_stop_velocity_threshold_mps` 또는 entry 전용 threshold
- 위 조건이 `entry_center_stable_frames` 이상 연속

초기 권장값:

```toml
entry_intersection_min_distance_m = 0.5
entry_center_target_y = 0.55
entry_center_x_tol_norm = 0.08
entry_center_y_tol_norm = 0.06
entry_center_stable_frames = 3
entry_center_timeout_s = 5.0
entry_center_max_v_mps = 0.12
entry_center_max_reverse_mps = 0.06
```

### 3.5 Decision window reset

다음 전환점에서 `IntersectionDecisionEngine`을 reset 또는 cooldown한다.

- `MarkerLockYaw -> EntryForward`
- `EntryForward -> EntryCenterOrigin`
- `EntryCenterOrigin -> SnakeForward`

목적:

- vertiport texture에서 생긴 `+`, `T`, `L` 잔상을 첫 grid origin 판단에 섞지 않는다.
- `(0,0)`으로 확정한 직후 같은 교차점 잔상이 곧바로 boundary watchdog이나 다음 node로 재사용되지 않게 한다.

구현 위치:

- `GridMission`은 이미 `IntersectionDecisionEngine* decision_engine_`을 optional로 들고 있다.
- state transition 직전에 `decision_engine_->reset()` 또는 `startCooldown()`을 호출한다.
- `EntryCenterOrigin -> SnakeForward`에서는 `reset()` 후 `armHopStart(in)`를 수행하는 쪽이 더 명확하다.

### 3.6 Entry speed config 연결

현재 `entry_forward_speed_mps`가 실제 setpoint에 연결되어 있지 않다.

수정 옵션:

1. `GridMissionOutput`에 `forward_speed_override_mps`를 추가한다.
2. `GridControlMapperInput`에도 같은 optional 값을 전달한다.
3. `ForwardBlind`에서 override가 있으면 그 값을 사용한다.

최소 수정:

- `grid_mission_node.cpp`에서 `cfg.mapper.forward_speed_blind_mps = cfg.mission.entry_forward_speed_mps`로 연결할 수 있다.
- 단, 이 방법은 `SnakeForward`의 blind 속도까지 같이 바꾸므로 entry와 snake 속도를 분리하려면 override 방식이 낫다.

권장:

- entry 전용 속도와 snake blind 속도를 분리한다.
- 이번 문제는 첫 origin 진입에서 발생하므로 `EntryForward`만 느리게 시작할 수 있어야 한다.

### 3.7 로그 보강

현재 로그만으로는 정확히 어떤 failsafe가 발생했는지와 hop distance가 얼마인지 바로 보이지 않는다.

추가 권장 필드:

```text
intent=<...>
safety=<...>
hop=<m>
cx=<normalized>
cy=<normalized>
phase=<approach_phase>
vx/vy/yawrate=<command>
```

특히 `SNAKE_FORWARD`에서 `hop_distance_m`가 `hop_max_distance_m=3.5`에 도달하는 순간을 확인해야 한다.

`out.last_safety_event`는 이미 출력 경로가 있으나, 이번 로그에는 safety reason이 보이지 않았다. `transition(EmergencyLand)` 직후 같은 frame에서 reason이 남는지 확인하고, 필요하면 `last_safety_event_`에도 `hop_distance_exceeded`를 저장한다.

## 4. 테스트 계획

### 4.1 Unit test

새 focused test를 추가한다.

권장 파일:

```text
uav-onboard/tests/test_grid_mission_entry_origin.cpp
```

테스트 케이스:

1. `MarkerLockYaw`에서 marker centered + yaw stable이면 `EntryForward`로 간다.
2. `EntryForward`에서 L intersection을 봐도 바로 `SnakeForward`로 가지 않고 `EntryCenterOrigin`으로 간다.
3. `EntryCenterOrigin`에서 center error가 tolerance 밖이면 origin을 publish하지 않는다.
4. center X/Y가 tolerance 안이고 velocity가 낮은 frame이 누적되면 origin event를 publish하고 `SnakeForward`로 간다.
5. origin latch 후 `hop_start_local_x/y`가 정렬 완료 위치 기준으로 잡힌다.
6. mixed/stale decision window가 origin latch에 영향을 주지 않도록 reset/cooldown 호출 효과를 검증한다.

### 4.2 Build

```bash
cmake --build build --target grid_mission_node
cmake --build build --target test_grid_mission_entry_origin
ctest --test-dir build --output-on-failure
```

### 4.3 SITL 재현 검증

기존과 같은 명령으로 재실행한다.

```bash
./build/grid_mission_node --config config --target sitl --vision gazebo \
  --world grid --line-mode dark_on_light --marker-count 4 \
  --video --gcs-ip 172.22.128.1
```

성공 로그 기준:

```text
MARKER_LOCK_YAW
ENTRY_FORWARD
ENTRY_CENTER_ORIGIN
SNAKE_FORWARD coord=(0,0) nodes=1
SNAKE_RECORD_NODE coord=(0,-1) or next expected node
```

검증 포인트:

- `ENTRY_FORWARD`에서 L을 봐도 바로 `nodes=1`이 되지 않는다.
- `ENTRY_CENTER_ORIGIN` 동안 `cx`, `cy`가 target으로 수렴한다.
- `(0,0)` publish 시점의 `cy`가 `0.55 ± tolerance` 근처다.
- `(0,0)` publish 시점의 `cx`가 0 근처다.
- `SNAKE_FORWARD` 진입 직후 line이 유지된다.
- `hop_distance_exceeded` 없이 다음 교차점이 기록된다.

## 5. 피해야 할 임시 처방

하지 말 것:

- `hop_max_distance_m`만 크게 늘려서 emergency를 늦추기.
- `hop_intersection_min_distance_m`만 낮춰서 더 빨리 origin을 찍기.
- `EntryForward`에서 raw `intersection.valid`를 더 쉽게 받아들이기.
- GCS overlay 기준으로 사람이 보기엔 맞아 보인다는 이유로 tracker origin만 보정하기.
- `SnakeForward` line-follow window를 무작정 전체 cell로 넓히기.

이 문제는 failsafe threshold 문제가 아니라 origin latch 시점 문제다. `(0,0)` 기준점이 틀리면 이후 snake traversal 전체가 누적 오차를 안고 시작한다.

## 6. 우선순위

1. `EntryCenterOrigin` 상태 추가.
2. origin latch를 `EntryCenterOrigin` 완료 시점으로 이동.
3. intersection center X/Y를 mission output과 mapper input에 연결.
4. `IntersectionCenter` intent 또는 entry 전용 centering control 추가.
5. decision window reset/cooldown 추가.
6. `entry_forward_speed_mps` 실제 연결.
7. 로그 보강.
8. unit test와 SITL 재검증.

## 7. 예상 결과

수정 후에는 첫 L 교차점이 화면에 들어온 순간이 아니라, 카메라 중심축이 교차점 중심에 맞은 뒤 `(0,0)`이 확정되어야 한다.

그러면 `last_node_local_x/y`와 `hop_start_local_x/y`가 실제 첫 grid node 중심 기준으로 잡히고, `SnakeForward`의 3m hop 거리와 `hop_align_start_m`/`hop_align_end_m` window가 실제 grid cell geometry와 다시 맞게 된다. 그 결과 현재처럼 첫 node 직후 line을 잃고 `hop_distance_exceeded`로 착륙하는 현상이 사라지는 것이 기대된다.
