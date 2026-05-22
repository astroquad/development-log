# Plan: Marker Manhattan Revisit

Date: 2026-05-22 KST

## Goal

Extend existing `uav-onboard/tools/grid_mission_node.cpp`. Do not create a new runtime program.

Flow:

```text
Grid snake search
-> synthesize/drain remaining grid nodes
-> freeze grid map
-> revisit found ArUco markers by Manhattan route
-> land
```

Hard requirements:

- Reuse existing snake/hover/turn/pass-through code as much as possible.
- Do not delete existing snake/dwell behavior; keep fallback switches.
- Revisit order is selected at startup: `asc`, `desc`, or `none`.
- During revisit, do not add new grid nodes/edges to GCS map.
- During revisit, keep showing live drone coord/heading on the already-built map.
- At each revisited marker, hover centered for the existing marker-hover duration/logic.
- GCS marker panel shows revisited markers with strikethrough/fallback visited mark.
- U-turn policy for first implementation: two verified 90-degree turns, not a new direct 180-degree turn.

## Files

Onboard add:

- `uav-onboard/src/mission/MarkerRevisitPlanner.hpp`
- `uav-onboard/src/mission/MarkerRevisitPlanner.cpp`
- `uav-onboard/tests/test_marker_revisit_planner.cpp`

Onboard modify:

- `uav-onboard/CMakeLists.txt`
- `uav-onboard/config/mission.toml`
- `uav-onboard/src/mission/GridMission.hpp`
- `uav-onboard/src/mission/GridMission.cpp`
- `uav-onboard/src/mission/GridCoordinateTracker.hpp`
- `uav-onboard/src/mission/GridCoordinateTracker.cpp`
- `uav-onboard/src/mission/MarkerRegistry.hpp`
- `uav-onboard/src/mission/MarkerRegistry.cpp`
- `uav-onboard/src/protocol/TelemetryMessage.hpp`
- `uav-onboard/src/protocol/TelemetryMessage.cpp`
- `uav-onboard/tools/grid_mission_node.cpp`
- `uav-onboard/tests/test_grid_mission_entry_origin.cpp` or new `test_grid_mission_revisit.cpp`

GCS modify:

- `uav-gcs/src/protocol/TelemetryMessage.hpp`
- `uav-gcs/src/protocol/TelemetryMessage.cpp`
- `uav-gcs/src/telemetry/GridMapTracker.hpp`
- `uav-gcs/src/telemetry/GridMapTracker.cpp`
- `uav-gcs/src/telemetry/MarkerTracker.hpp`
- `uav-gcs/src/telemetry/MarkerTracker.cpp`
- `uav-gcs/src/app/VisionDebugApp.cpp`
- `uav-gcs/tests/test_telemetry_line_parse.cpp`
- `uav-gcs/tests/test_grid_map_tracker.cpp`
- optionally add `uav-gcs/tests/test_marker_tracker.cpp`

## Reuse

Reuse from `GridMission`:

- `ForwardBlind` for straight motion: yaw lock + lateral line-centering
- `MarkerHover` path: `populateMarkerInputs`, `beginMarkerHover`, `markerHoverComplete`
- pass-through logic idea: commit/pose update without dwell on middle nodes
- `StopAndCenter` before turn nodes
- 90-degree yaw turn logic from `SnakeTurn90` / `SnakeTurn90Again`
- `synthesizeRemainingColumnNodes()` before revisit starts

Do not reuse:

- `SnakePlanner` for revisit route choice. Revisit target order/path is deterministic from marker coords, not boundary discovery.

## Onboard Data Model

Add revisit order:

```cpp
enum class RevisitOrder { None, Asc, Desc };
```

Add planner types:

```cpp
struct MarkerRevisitTarget {
    int id = -1;
    GridCoord coord;
};

struct RevisitSegment {
    GridHeading heading = GridHeading::Unknown;
    int cells = 0;
};

struct RevisitLeg {
    int marker_id = -1;
    GridCoord target;
    std::vector<RevisitSegment> segments;
};
```

Add `MarkerRecord` fields:

```cpp
bool revisited = false;
std::int64_t revisited_ms = 0;
```

Add `MarkerRegistry` API:

```cpp
bool markRevisited(int aruco_id, std::int64_t timestamp_ms);
std::size_t revisitedGridMarkerCount() const;
```

Add `GridCoordinateTracker` API:

```cpp
void setCurrentPose(GridCoord coord, GridHeading heading);
```

`setCurrentPose` must update only `current_coord_` and `current_heading_`; it must not mutate `nodes_`, node IDs, or map edges.

## Planner Rules

Input:

- current coord
- current heading
- marker records with valid grid coords
- order: `asc` or `desc`

Order:

- `asc`: marker id increasing
- `desc`: marker id decreasing
- `none`: no revisit; keep existing landing behavior

For each next target:

1. Build candidate path `X axis first, then Y axis`.
2. Build candidate path `Y axis first, then X axis`.
3. Drop zero-length segments.
4. Score each candidate:
   - same heading: `0`
   - 90-degree turn: `1`
   - U-turn: `2`
5. Choose lower score.
6. Tie-break by candidate whose first segment matches current heading.
7. Tie-break by longer first segment.

U-turn representation:

- Do not add direct 180 turn state initially.
- If heading needs opposite direction, emit two 90-degree turns at the same node before moving.

Planner unit tests:

- asc order
- desc order
- same-row route
- same-column route
- X-first vs Y-first lower turn cost
- current-heading tie-break
- U-turn costs as two 90-degree turns
- zero-distance target produces immediate marker hover/no movement segment

## GridMission States

Add states:

```cpp
RevisitInit,
RevisitForward,
RevisitStopAtTurn,
RevisitTurn90,
RevisitMarkerHover,
RevisitComplete,
```

Transition outline:

```text
SnakeComplete
  if pending_synth_events_ not empty:
    drain as today
  else if revisit_order == none or no grid markers:
    Land as today
  else:
    RevisitInit

RevisitInit
  build RevisitLeg list from MarkerRegistry
  set grid_map_finalized=true
  choose first segment/target
  if already at target: RevisitMarkerHover
  else if turn needed: RevisitStopAtTurn
  else: RevisitForward

RevisitForward
  intent=ForwardBlind
  target_yaw_rad = heading yaw for active segment
  on nodeJustRecorded + distance gate:
    update current pose only
    if target marker coord reached: RevisitMarkerHover
    else if segment cells complete and next heading differs: RevisitStopAtTurn
    else continue ForwardBlind

RevisitStopAtTurn
  intent=StopAndCenter
  reuse velocity settle / center gate
  then RevisitTurn90

RevisitTurn90
  intent=YawTurn
  rotate one 90-degree step
  if U-turn still needs second 90: stay/loop RevisitTurn90
  else enter RevisitForward or RevisitMarkerHover

RevisitMarkerHover
  intent=MarkerHover
  focus target marker id
  reuse marker hover helper and timeout
  on complete: MarkerRegistry.markRevisited(id)
  choose next leg or RevisitComplete

RevisitComplete
  Land
```

Important:

- During all revisit states, `commit_tracker_advance=false`.
- Revisit node arrivals call `GridCoordinateTracker::setCurrentPose`, not `commitAdvance`.
- GCS map freeze is a telemetry flag and also protected by no new onboard grid commits.

## Config And CLI

Add to `config/mission.toml`:

```toml
revisit_order = "asc"                 # asc | desc | none
revisit_passthrough_regular_nodes = true
revisit_marker_hover_s = 3.0          # optional; default can reuse snake_marker_hover_s
revisit_turn_center_required = true
```

Add to `grid_mission_node` CLI:

```text
--revisit-order <asc|desc|none>
```

CLI overrides config.

## Telemetry

Onboard `MissionMarkerEntry` add:

```cpp
bool revisited = false;
double revisited_s = 0.0;
```

Onboard `MissionTelemetry` add:

```cpp
bool revisit_active = false;
bool grid_map_finalized = false;
std::string revisit_order = "none";
int revisit_target_id = -1;
int revisit_remaining = 0;
```

JSON keys:

```json
{
  "mission": {
    "revisit_active": true,
    "grid_map_finalized": true,
    "revisit_order": "asc",
    "revisit_target_id": 3,
    "revisit_remaining": 2,
    "markers_found": [
      {
        "id": 1,
        "grid": [-1, -2],
        "grid_valid": true,
        "first_seen_s": 204.7,
        "revisited": true,
        "revisited_s": 398.2
      }
    ]
  }
}
```

`grid_mission_node` publish policy:

- Before revisit: publish grid nodes as today.
- During revisit: do not create new committed grid nodes.
- During revisit: mission grid coord/heading must continue updating.
- Prefer also suppressing repeated `last_committed_event` during revisit if practical.

## GCS Behavior

`GridMapTracker`:

- Add `grid_map_finalized_`.
- `observeMission(mission)` sets `grid_map_finalized_ |= mission.grid_map_finalized`.
- If `grid_map_finalized_`, `observe(grid_node)` ignores new nodes/edges.
- `observeMission(mission)` still updates live drone coord/heading.
- Existing map remains visible.

`VisionDebugApp` observe order:

```cpp
grid_map.observeMission(parsed->mission);
grid_map.observe(parsed->vision.grid_node);
grid_map.observeDronePosition(...);
marker_tracker.observe(parsed->mission);
```

Reason: freeze flag must be applied before same-packet `grid_node`.

`MarkerTracker`:

- Parse and store `MissionMarkerEntry::revisited`.
- Render revisited marker with strikethrough if usable.
- Plain-text fallback: prefix `[x]`.

Suggested rendering:

```text
id=1  grid=(-1,-2)  t=204.7s
[x] id=2  grid=(-3,-4)  t=315.2s
```

If Win32 edit control renders combining strikethrough reliably, apply it to the whole visited line; otherwise keep `[x]`.

## Implementation Order

1. Add `MarkerRevisitPlanner` and planner tests.
2. Add `MarkerRegistry.revisited` fields/API.
3. Add `GridCoordinateTracker::setCurrentPose`.
4. Add telemetry structs/JSON fields on onboard and GCS.
5. Add CLI/config parsing for `--revisit-order`.
6. Add `GridMission` revisit states and internal route state.
7. Wire `SnakeComplete` to `RevisitInit` after synthetic drain.
8. Implement `RevisitForward` using existing `ForwardBlind` and node distance gate.
9. Implement `RevisitStopAtTurn` and `RevisitTurn90` by extracting/reusing current turn helpers where sensible.
10. Implement `RevisitMarkerHover` using current marker hover helpers.
11. Freeze GCS map on `grid_map_finalized`.
12. Add marker visited rendering.
13. Run tests and SITL checks.

## Tests

Onboard:

```bash
cmake --build build --target test_marker_revisit_planner test_grid_mission_entry_origin grid_mission_node
ctest --test-dir build -R 'marker_revisit|grid_mission|guided_velocity_controller' --output-on-failure
```

GCS:

```bash
cmake --build build --target test_telemetry_line_parse test_grid_map_tracker uav_gcs_vision_debug
ctest --test-dir build --output-on-failure
```

Add assertions:

- `--revisit-order none` keeps existing land-after-snake behavior.
- `asc` and `desc` produce expected marker order.
- Revisit middle nodes do not emit grid commits.
- Revisit updates mission current coord/heading.
- GCS ignores new grid nodes after `grid_map_finalized`.
- GCS still moves arrow after `grid_map_finalized`.
- Revisited markers render as visited.

## Open Decisions

- On target marker lost for full hover timeout:
  - Recommended first behavior: log `revisit_marker_lost`, mark not revisited, continue to next target.
  - Conservative option: retry once or emergency land.
- After all markers revisited:
  - Current plan: land at last marker unless return-home is requested later.
  - Return-home can be a later planner target using the same route machinery.
