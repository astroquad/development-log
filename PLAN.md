# Plan: Full Mission Return-To-Vertiport

Date: 2026-05-22 KST

## Goal

Extend the existing `grid_mission_node` flow. Do not create a new runtime
program.

Target mission:

```text
snake grid search
-> synthetic grid completion/drain
-> marker revisit, always enabled
-> return to grid origin (0,0)
-> face south on the grid
-> leave grid toward vertiport with yaw frozen and line tracing disabled
-> find active vertiport ArUco marker
-> marker-center hover/align
-> land
-> publish mission_complete
-> exit
```

Keep existing movement primitives. This is mostly route composition, state
sequencing, and telemetry.

## Requirements

- `--revisit-order` default is `desc` when omitted.
- `--revisit-order asc` is the only alternate order.
- Remove public `none` mode from CLI/config/docs. Revisit is now part of the
  mission, not an optional feature.
- After all marker revisits, always return to `(0,0)`.
- Use the same Manhattan route planner/forward/turn/hover mechanics already
  used for marker revisit.
- At `(0,0)`, rotate so the drone faces grid south (`v` on GCS map).
- Align on the `(0,0)` intersection center before exiting the grid.
- From `(0,0)` to the vertiport, fly yaw-frozen forward with line inputs off.
- Search for the runtime active vertiport marker id latched during takeoff.
- If the vertiport marker is not found, use the same timeout budget as
  `entry_forward_timeout_s`.
- Once the vertiport marker is visible, use existing marker-centering hover
  control, then land.
- Once leaving the grid toward the vertiport, GCS grid map must hide the drone
  arrow because the drone is no longer on a grid node.
- After successful landing/disarm, onboard sends a final success telemetry
  packet and exits.
- GCS shows `mission complete!` when that final success telemetry is received.

## Files

Onboard modify:

- `uav-onboard/src/mission/MarkerRevisitPlanner.hpp`
- `uav-onboard/src/mission/MarkerRevisitPlanner.cpp`
- `uav-onboard/src/mission/GridMission.hpp`
- `uav-onboard/src/mission/GridMission.cpp`
- `uav-onboard/src/protocol/TelemetryMessage.hpp`
- `uav-onboard/src/protocol/TelemetryMessage.cpp`
- `uav-onboard/tools/grid_mission_node.cpp`
- `uav-onboard/config/mission.toml`
- `uav-onboard/README.md`

GCS modify:

- `uav-gcs/src/protocol/TelemetryMessage.hpp`
- `uav-gcs/src/protocol/TelemetryMessage.cpp`
- `uav-gcs/src/telemetry/GridMapTracker.hpp`
- `uav-gcs/src/telemetry/GridMapTracker.cpp`
- `uav-gcs/src/telemetry/VisionLogFormatter.cpp`
- optionally `uav-gcs/src/telemetry/MarkerTracker.cpp` if the side panel should
  also show mission completion.

Tests optional but useful:

- `uav-onboard/tests/test_marker_revisit_planner.cpp`
- `uav-onboard/tests/test_grid_mission_revisit.cpp`
- `uav-gcs/tests/test_telemetry_line_parse.cpp`
- `uav-gcs/tests/test_grid_map_tracker.cpp`

## CLI And Config

Change public order model:

```cpp
enum class RevisitOrder { Asc, Desc };
```

Preferred implementation:

- `GridMissionConfig::revisit_order = RevisitOrder::Desc`
- `parseRevisitOrder()` accepts only `asc` and `desc`
- `--revisit-order` omitted -> `desc`
- `--revisit-order none` -> CLI error
- remove `revisit_order = "none"` from `config/mission.toml`; set `"desc"` or
  omit the key
- update README commands and examples

Compatibility note:

- If keeping `RevisitOrder::None` internally reduces code churn, do not expose
  it in CLI/config/docs and never choose it by default. Remove it later when
  convenient.

## Planner Reuse

Generalize `MarkerRevisitPlanner` into a reusable Manhattan waypoint planner
without changing its current behavior.

Minimal path:

- Keep `MarkerRevisitPlanner` name for now.
- Add a helper that builds a single `RevisitLeg` to an arbitrary `GridCoord`.
- Use it for:
  - marker-to-marker revisit legs
  - final return leg from current coord to `(0,0)`

Rules stay the same:

- choose X-first or Y-first by lower turn cost
- tie-break by first segment matching current heading
- tie-break by longer first segment
- U-turn remains two verified 90-degree turns

## New GridMission States

Add states after `RevisitComplete`:

```cpp
ReturnHomeInit,
ReturnHomeForward,
ReturnHomeStopAtTurn,
ReturnHomeTurn90,
ReturnHomeAlignOrigin,
ReturnHomeFaceSouth,
ReturnVertiportForward,
ReturnVertiportMarkerHover,
MissionComplete,
```

State reuse:

- `ReturnHomeForward` mirrors `RevisitForward`.
- `ReturnHomeStopAtTurn` mirrors `RevisitStopAtTurn`.
- `ReturnHomeTurn90` mirrors `RevisitTurn90`.
- `ReturnHomeAlignOrigin` reuses `IntersectionCenter`/center gate logic from
  entry-origin centering.
- `ReturnHomeFaceSouth` reuses one-or-two-step 90-degree yaw turn logic.
- `ReturnVertiportForward` reuses `ForwardBlind` intent but explicitly disables
  line inputs.
- `ReturnVertiportMarkerHover` reuses `MarkerHover`, `populateMarkerInputs`,
  `beginMarkerHover`, and full marker hover dwell/centering behavior.

## State Flow

```text
RevisitComplete
  -> ReturnHomeInit

ReturnHomeInit
  build single Manhattan leg to (0,0)
  if already at (0,0): ReturnHomeAlignOrigin
  else if heading mismatch: ReturnHomeStopAtTurn
  else: ReturnHomeForward

ReturnHomeForward
  intent=ForwardBlind
  target_yaw_rad = active route segment yaw
  commit_tracker_advance=false
  on nodeJustRecorded + hop distance gate:
    setCurrentPose(next coord, segment heading)
    if reached (0,0): ReturnHomeAlignOrigin
    else continue/turn like revisit

ReturnHomeStopAtTurn
  intent=StopAndCenter
  reuse velocity settle + branch settle
  then ReturnHomeTurn90

ReturnHomeTurn90
  intent=YawTurn
  one verified 90-degree step
  if U-turn needs second 90: stay ReturnHomeTurn90
  else ReturnHomeForward

ReturnHomeAlignOrigin
  intent=IntersectionCenter
  center the (0,0) intersection using existing center gates
  when centered/stable: ReturnHomeFaceSouth

ReturnHomeFaceSouth
  intent=YawTurn
  rotate to GridHeading::South
  when yaw stable:
    setCurrentPose((0,0), South)
    armHopStart()
    set grid_pose_visible=false for GCS
    ReturnVertiportForward

ReturnVertiportForward
  intent=ForwardBlind
  target_yaw_rad = yaw for GridHeading::South
  line_detected=false, center_error=0, angle_error=0
  wait until active vertiport marker id is visible
  timeout = entry_forward_timeout_s
  on marker visible: ReturnVertiportMarkerHover

ReturnVertiportMarkerHover
  intent=MarkerHover
  focus active_vertiport_marker_id_
  center over marker using existing marker controller
  after marker hover dwell / centered gate: Land

Land
  existing LAND mode
  when disarmed: MissionComplete

MissionComplete
  mission_finished=true
  mission_complete=true
  landing_success=true
  next tick/after final telemetry: Done
```

Important:

- Return-home grid movement must not mutate `nodes_` or GCS map nodes.
- Use `GridCoordinateTracker::setCurrentPose()` only.
- GCS map remains frozen after snake completion.
- During return-home `(0,0)` movement, live arrow is still shown on the frozen
  grid.
- During off-grid vertiport forward/marker hover/landing, live arrow is hidden.

## Vertiport Return Details

Marker id:

- Use `active_vertiport_marker_id_`.
- If it is still `-1`, treat as mission fault and emergency land or use config
  fallback only as a last-resort failsafe. Do not leak fallback ID to normal
  GCS status as an observed marker.

Forward search:

- Start at centered `(0,0)`, heading south.
- Disable line centering and intersection/node handling.
- Use yaw lock only.
- Move with the same speed family as `EntryForward` or `ForwardBlind`; prefer
  existing `ForwardBlind` output if it is already stable.
- Timeout after `config_.entry_forward_timeout_s`.

Marker hover:

- Use existing marker control; this should be the final precision alignment
  over the vertiport.
- After successful alignment/dwell, request LAND mode.
- Keep `mission_complete=false` until disarmed.

## Telemetry Additions

Onboard protocol `MissionTelemetry` add:

```cpp
bool return_active = false;
std::string return_phase = "none";       // none|to_origin|face_south|to_vertiport|landing
bool grid_pose_visible = true;
bool vertiport_return_active = false;
bool vertiport_acquired = false;
bool landing_success = false;
bool mission_complete = false;
```

Populate from `GridMissionOutput` in `grid_mission_node`.

GCS parse the same fields.

GCS behavior:

- `GridMapTracker::observeMission()`:
  - if `grid_pose_visible=false`, clear/hide mission arrow and ignore drone
    fractional position
  - keep frozen nodes and marker glyphs
- `VisionLogFormatter`:
  - if `mission_complete=true`, show `mission complete!`
  - while return is active, show `return=<phase>`
  - keep `marker_revisit=yes/no` when revisit order is active

Example final log:

```text
=== Mission ===
state=MISSION_COMPLETE  intent=land
markers=4/4  snake_complete=yes  marker_revisit=yes  return=landing
mission complete!
------------------------------------------------------------
```

## Implementation Notes

- Keep all code in existing modules; do not add a new executable.
- Prefer adding small helpers inside `GridMission` over duplicating large
  handler bodies:
  - `buildReturnHomePlan()`
  - `startCurrentReturnLeg()`
  - `handleRouteForward(...)`
  - `handleRouteStopAtTurn(...)`
  - `handleRouteTurn90(...)`
  - `setGridPoseVisible(bool)`
- If generic route helpers become too invasive, copy the revisit handlers first
  and refactor only after build passes.
- Preserve existing snake search behavior.
- Preserve existing marker 3-second hover semantics:
  - grid marker during snake search: full dwell
  - marker on turn node: full dwell
  - marker revisit: full dwell
  - vertiport final alignment: full dwell unless field testing shows this is
    too slow

## Acceptance Checklist

- `grid_mission_node --revisit-order desc` works.
- `grid_mission_node` with omitted `--revisit-order` behaves as `desc`.
- `--revisit-order asc` works.
- `--revisit-order none` is rejected.
- Snake search still completes and freezes the GCS grid map.
- Marker revisit still marks GCS markers `[>]` then `[x]`.
- After all markers are revisited, drone returns to `(0,0)`.
- Drone faces south at `(0,0)`.
- Drone exits grid toward vertiport with line inputs disabled.
- GCS grid arrow disappears once drone leaves grid.
- Vertiport marker acquisition triggers marker centering.
- Successful landing publishes `mission_complete=true`.
- GCS prints `mission complete!`.
- Onboard exits after final success telemetry.
- Build passes:
  - `cmake --build astroquad/uav-onboard/build`
  - `cmake --build astroquad/uav-gcs/build`
