# Astroquad System Spec

최종 업데이트: 2026-05-21

이 문서는 `uav-onboard`와 `uav-gcs`를 모두 포함하는 전체 시스템 기준
문서다. Repo별 세부 책임과 실행 방법은 각 repo의 `PROJECT_SPEC.md`와
`README.md`를 따른다.

## 1. 시스템 목표

Astroquad는 GPS 없는 실내/운동장 격자 환경에서 UAV가 하향 카메라 기반
라인 인식, 교차점 판단, ArUco marker 인식을 사용해 탐색 임무를 수행하는
C++ 기반 시스템이다.

최종 목표:

```text
이륙
  -> grid 진입
  -> snake 방식 grid 탐색
  -> 교차점별 ArUco marker 기록
  -> marker 번호 역순 재방문
  -> 출발점 복귀
  -> 자동 착륙
```

현재 구현 기준:

- Vision/GCS path는 재사용 가능한 라이브러리로 분리되어 있다.
- `line_follow_node`는 SITL 및 guarded Pixhawk1 line-follow staging을 담당한다.
- `grid_mission_node`는 현재 grid arena snake mission SITL staging의 중심이다.
- 최종 실행 파일 목표는 여전히 `uav_onboard`와 `uav_gcs`지만, 두 target은
  아직 final mission/dashboard composition root가 아니다.

## 2. 하드웨어 기준

| 모델/장치 | 연결 위치 | 역할 |
|---|---|---|
| Pixhawk1 | 기체 중앙 | 비행 제어기. 자세 안정화, 모터 출력, 센서 융합, 모드 관리 |
| MicoAir MTF-01 Optical Flow & Range Sensor | Pixhawk TELEM2 또는 SERIAL4/5 | GPS 없이 수평 이동 추정 + 바닥 거리 측정 |
| Raspberry Pi 4 | Pixhawk USB 또는 TELEM1 | OpenCV 비전, mission state, MAVLink command sender |
| IMX519 Camera | Raspberry Pi CSI | 하향 영상 입력 |
| Power Module | Pixhawk POWER | 배터리 전압/전류 측정 |
| RC Receiver / RC path | Pixhawk RC IN 또는 운용 중인 RC 경로 | 수동 takeover, 비상 개입 |
| ESC/Motor/PDB | Pixhawk MAIN OUT + 전원분배 | 실제 모터 구동 |
| TFmini Plus | 기본 구성 제외 | MTF-01 range가 불안정할 때 예비 rangefinder |
| External Compass | 제외 | 초기 기준은 Pixhawk 내부 compass/IMU |

## 3. Repo 역할

### uav-onboard

담당:

- Raspberry Pi camera capture
- Fake/Gazebo/rpicam frame source abstraction
- ArUco/line/intersection detection and stabilization
- Intersection decision and local grid-node telemetry
- Grid mission state machine and snake planner
- Marker registry/stability gate
- Guidance/control mapping to body velocity setpoints
- MAVLink UDP and serial transport
- Pixhawk bench tools and line-follow staging
- Safety/failsafe and telemetry/debug video sender

Current executables:

| Executable | Current role |
|---|---|
| `uav_onboard` | Basic telemetry sender; final onboard composition root target. |
| `vision_debug_node` | Vision/GCS bring-up and tuning. |
| `line_follow_node` | Short line-follow SITL/guarded real Pixhawk1 staging. |
| `grid_mission_node` | Current grid arena snake mission SITL staging. |
| `mavlink_probe` | No-arm Pixhawk/MAVLink/local-estimate probe. |
| `mavlink_motor_test` | Props-removed low-throttle motor command check. |
| `video_streamer` | Raw MJPEG transport smoke tool. |

Current grid mission command:

```bash
./build/grid_mission_node --config config --target sitl --vision gazebo \
  --world grid --line-mode dark_on_light --marker-count 4 \
  --video --gcs-ip <windows-gcs-ip>
```

### uav-gcs

담당:

- UDP telemetry receive/parse/display
- UDP MJPEG debug video receive/display
- GCS discovery beacon
- GCS-side marker/line/intersection overlay from onboard metadata
- Vision/system/camera/video logs
- Local committed-node grid map display
- Future mission dashboard and command sender

Current executables:

| Executable | Current role |
|---|---|
| `uav_gcs` | Basic telemetry receiver; final GCS composition root target. |
| `uav_gcs_vision_debug` | Current primary monitoring UI. |
| `uav_gcs_video` | Raw MJPEG viewer. |
| `mock_onboard`, `log_replayer` | Development tools. |

Windows Ninja example:

```powershell
.\build\uav_gcs_vision_debug.exe --config config
```

## 4. 모듈 경계

ROS를 직접 쓰지는 않지만, typed input/output만 공유하는 node graph식 구조를
유지한다.

```text
FrameSource
  -> VisionProcessor
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> GridMission / LineFollowMission
  -> GridControlMapper / GuidedVelocityController
  -> AutopilotMavlinkAdapter

Support:
  SafetyMonitor observes Vision/Mission/Autopilot/GCS state
  VisionDebugPublisher sends telemetry/debug video
  GCS observes telemetry/video only
```

경계 규칙:

- Vision code does not know flight mode or MAVLink.
- Mission code produces state/control intent; it does not encode MAVLink packets.
- Control mapping converts intent to `ControlSetpoint`.
- Autopilot adapter owns MAVLink send/receive only.
- GCS does not promote vision candidates into mission decisions.
- Debug video is not mission-critical.

## 5. 현재 알고리즘 기준

### Vision

- `LineMaskBuilder` creates filled bright/dark/local-contrast masks on a resized ROI.
- `LineDetector` scores candidate contours and measures tracking X from an
  anchor/lookahead projection band.
- `LineStabilizer` applies EMA/hold/reacquire/jump rejection.
- `IntersectionDetector` scores branch rays and classifies `straight`, `L`, `T`, `+`.
- `IntersectionDecisionEngine` aggregates branch evidence over a short window
  and emits `node_record`, `turn_confirm`, `turn_ready`, cooldown, and
  overshoot-risk telemetry.
- ArUco detector/stabilizer reports current-frame markers; grid mission applies
  a separate sliding marker window before committing marker IDs.

### Grid mission

Current grid mission is designed for the Gazebo grid arena:

- 3m x 3m vertiport at origin with ArUco ID 23.
- No line from vertiport to grid.
- 5 x 8 grid cells, 3m cell size, dark lines on light ground.
- Four grid ArUco markers expected by default.

State flow:

```text
ARM_TAKEOFF
  -> MARKER_LOCK_YAW
  -> ENTRY_FORWARD
  -> ENTRY_CENTER_ORIGIN
  -> SNAKE_LAUNCH_ALIGN
  -> SNAKE_FORWARD
  -> SNAKE_RECORD_NODE
  -> SNAKE_STOP_AT_CENTER
  -> SNAKE_TURN_90
  -> SNAKE_ADVANCE_ONE_CELL
  -> SNAKE_TURN_90_AGAIN
  -> SNAKE_COMPLETE
  -> LAND
```

Important rules:

- Mission does not use Gazebo ground-truth pose.
- `ENTRY_FORWARD` flies yaw-locked blind from vertiport toward the first grid node.
- The first grid node becomes local `(0,0)` only after `ENTRY_CENTER_ORIGIN`
  centers the intersection under the camera and passes velocity/stability gates.
- `GridCoordinateTracker::update()` is peek-only; only mission-approved
  `commitAdvance()` changes the grid coordinate.
- Between nodes, line following opens only in a mid-cell align window; otherwise
  yaw remains locked and forward velocity is blind.
- Boundary turns use strict snake alternation. Missing expected branch completes
  the snake rather than silently backtracking.
- Current mission lands after expected markers are found or snake completes.
  Marker reverse revisit and return-home remain future work.

## 6. 제어 전략

Primary:

- ArduPilot `GUIDED`
- MAVLink `SET_POSITION_TARGET_LOCAL_NED`
- Body-frame velocity setpoint

Implemented transports:

- SITL/Gazebo: UDP MAVLink.
- Real Pixhawk1 bench/line-follow: serial/USB serial MAVLink.

Current safety boundary:

- `line_follow_node` has guarded Pixhawk1 serial support with no-arm smoke,
  RC/local-estimate gates, and explicit `--allow-arm-takeoff`.
- `mavlink_probe` and `mavlink_motor_test` support bench verification.
- `grid_mission_node --target pixhawk1` is not enabled for real arm/takeoff;
  use `--no-arm` smoke only.

Fallback `ALT_HOLD + RC_CHANNELS_OVERRIDE` remains a design option, but current
implemented mission paths use GUIDED velocity/local setpoints.

## 7. 통신 구조

공통 protocol 문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

현재 protocol document version은 v1.8이고 JSON top-level
`protocol_version`은 integer `1`이다.

| Channel | Direction | Transport | Default port | Status |
|---|---|---|---:|---|
| Telemetry | onboard -> GCS | UDP JSON | 14550 | implemented |
| Command | GCS -> onboard | TCP JSON | 14551 | planned |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 | implemented |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 | implemented |

Important current telemetry:

- `vision.line`
- `vision.intersection`
- `vision.intersection_decision`
- `vision.grid_node`
- `vision.drone_position`
- `vision.markers`
- `debug.note`

`vision.grid_node` means the latest committed onboard grid node. In
`grid_mission_node`, it is resent every frame for UDP loss tolerance.

## 8. Gazebo/SITL 기준

Launchers:

```bash
bash ~/astroquad/uav-onboard/scripts/line_tracing_test.sh
bash ~/astroquad/uav-onboard/scripts/grid_arena_test.sh
```

Grid mission loop:

```bash
# Windows
.\build\uav_gcs_vision_debug.exe --config config

# WSL
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"
cd ~/astroquad/uav-onboard
./build/grid_mission_node --config config --target sitl --vision gazebo \
  --world grid --line-mode dark_on_light --marker-count 4 \
  --video --gcs-ip "$WINDOWS_GCS_IP"
```

`grid_arena_test_world` uses `astroquad_grid_course` and
`iris_with_downward_camera`. The grid runtime profile is
`config/runtime.sitl.grid.toml`.

## 9. 현재 구현 기준선

Implemented:

- Pi 4 + IMX519 `rpicam-vid` MJPEG capture.
- Fake/Gazebo/rpicam frame sources.
- ArUco, line, intersection, marker stabilization.
- Shared `VisionProcessor`.
- UDP JSON telemetry and UDP MJPEG debug video.
- GCS discovery and video unicast.
- GCS marker/line/intersection overlay.
- GCS vision/grid logs and committed-node ASCII map.
- MAVLink UDP and serial transports.
- Pixhawk no-arm probe and props-removed motor test tools.
- `line_follow_node` startup/flight/landing video path.
- Grid arena Gazebo world and `grid_mission_node` SITL staging.
- Grid mission entry centering, hop-distance gating, strict snake alternation,
  and sliding marker commit window.

Staging / not final:

- `uav_onboard` final mission composition root.
- `uav_gcs` final dashboard composition root.
- Grid mission real Pixhawk arm/takeoff.
- Structured GCS mission-state telemetry from `grid_mission_node`.
- GCS command channel.
- Marker reverse revisit.
- Return-home and official coordinate conversion.
- Persistent file logging/replay.

## 10. 실기체 전환 gate

Before real autonomous flight:

- MTF-01 optical flow/range must feed ArduPilot EKF local estimate stably.
- `LOCAL_POSITION_NED`, rangefinder, optical-flow quality, heartbeat, battery,
  and RC/takeover path must be verified.
- Props-off motor order and low-throttle checks must pass.
- Manual hover and RC takeover must be confirmed.
- `line_follow_node --mavlink-smoke` and `mavlink_probe --strict-local-estimate`
  must pass.
- `grid_mission_node` real arm/takeoff remains closed until SITL mission and
  no-arm serial smoke are deliberately promoted.

## 11. 문서 역할

- `SYSTEM_SPEC.md`: 전체 시스템 목적, module boundary, current baseline.
- `uav-onboard/PROJECT_SPEC.md`: onboard repo responsibilities, algorithms,
  targets, build/test.
- `uav-onboard/README.md`: current onboard runbook.
- `uav-onboard/sim/gazebo/README.md`: Gazebo worlds/models/run commands.
- `uav-gcs/PROJECT_SPEC.md`: GCS responsibilities, modules, status.
- `uav-gcs/README.md`: current GCS build/run/test guide.
- `docs/PROTOCOL.md`: telemetry/video/discovery wire-format spec.
- `TROUBLESHOOTING.md`: historical issue/decision log.

`RESEARCH.md` and `PLAN.md` are scratchpads for active work and are not the
long-term source of truth.
