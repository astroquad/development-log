# Astroquad Research Snapshot

최종 업데이트: 2026-05-15

이 문서는 다음 에이전트가 **이 파일만 읽고도** 현재 Astroquad 프로젝트의 구조, 목표, 진행상황, 금지사항, 바로 다음 작업 지점을 파악하도록 만든 작업용 요약이다. 더 자세한 기준 문서는 `SYSTEM_SPEC.md`, `MVP_PLAN.md`, `uav-onboard/PROJECT_SPEC.md`, `uav-gcs/PROJECT_SPEC.md`를 따른다.

## 1. 전체 목표

Astroquad는 GPS 없는 실내/운동장 격자 환경에서 UAV가 하향 카메라 기반 라인 인식, 교차점 판단, ArUco marker 인식을 사용해 탐색 임무를 수행하는 C++ 시스템이다.

최종 목표:

```text
이륙
  -> 라인 추종 및 격자 진입
  -> snake 방식 grid 탐색
  -> 교차점별 ArUco marker 기록
  -> marker 번호 역순 재방문
  -> 출발점 복귀
  -> 자동 착륙
```

현재 72시간 실기체 MVP는 최종 목표의 축소판이다.

```text
MTF-01 bring-up
  -> 자동 이륙
  -> 짧은 직선 line follow
  -> 종료 marker 또는 line end/lost/timeout/operator abort 중 하나에서 안전 착륙
```

MVP에서 제외된 것:

- full snake grid exploration
- ArUco marker revisit
- official coordinate conversion
- 교차점 회전/분기 의사결정
- GCS full command UI
- marker count command
- backend switching UI
- command ACK/retry UI

## 2. 하드웨어와 제어 기준

Target hardware:

| 장치 | 역할 |
|---|---|
| Pixhawk1 | ArduPilot 비행 제어기. 자세 안정화, 모터 출력, 모드 관리 |
| MicoAir MTF-01 | Optical flow + ToF range. GPS 없는 수평 이동 추정과 바닥 거리 측정 |
| Raspberry Pi 4 | OpenCV vision 처리, mission/control 판단, MAVLink 명령 송신 |
| IMX519 CSI Camera | 하향 영상 입력. line/intersection/ArUco 인식 |
| RC Receiver | 수동 takeover와 비상 개입 |

제어 primary는 ArduPilot `GUIDED` mode + MAVLink `SET_POSITION_TARGET_LOCAL_NED` body-frame velocity setpoint다. fallback은 `ALT_HOLD` + `RC_CHANNELS_OVERRIDE`로 계획되어 있지만 degraded/manual-like test path일 뿐이다.

MTF-01 bring-up은 실기체 자동 line-follow의 선행 gate다. ArduPilot에서 optical flow/range가 안정적으로 들어오지 않으면 GUIDED velocity 기반 실비행 line-follow를 진행하지 않는다.

## 3. Repo 역할

### uav-onboard

책임:

- Raspberry Pi camera capture
- ArUco/line/intersection detection
- marker/line/intersection stabilization
- local grid-node event telemetry
- mission state machine
- guidance/control backend
- MAVLink adapter
- safety/failsafe
- telemetry/debug video sender

주요 실행 파일:

- `uav_onboard`: 최종 onboard composition root. 현재는 basic telemetry sender에 가까우며, 최종적으로 vision/mission/control/safety/telemetry/MAVLink를 조립해야 한다.
- `vision_debug_node`: vision bring-up/debug program. camera capture, detector execution, telemetry, optional debug video만 담당한다.
- `line_follow_node`: 임시 SITL/MVP staging executable. auto takeoff, 짧은 line-follow, safe landing, GCS telemetry/video publish, SITL MAVLink control path를 함께 검증한다.
- `video_streamer`: raw MJPEG transport smoke tool.
- `line_detector_tuner`, `aruco_detector_tester`, `grid_image_smoke`, `marker_grid_replay`: offline vision tuning/regression tools.

### uav-gcs

책임:

- onboard telemetry receive/parse/display
- raw debug video receive/display
- GCS-side marker/line/intersection overlay
- vision log and grid-map log display
- future command sender and mission dashboard
- event/command logging and replay

주요 실행 파일:

- `uav_gcs`: 최종 GCS composition root. 현재는 console telemetry receiver에 가깝다.
- `uav_gcs_vision_debug`: 현재 핵심 관제 도구. camera window + vision log window.
- `uav_gcs_video`: raw MJPEG viewer.
- `mock_onboard`, `log_replayer`: development tools.

## 4. 현재 구현 기준선

이미 잘 되는 것으로 확인된 흐름:

- `uav-onboard/README.md`의 **Gazebo SITL Target Commands**를 순서대로 실행하면 Gazebo/SITL 환경이 정상 실행된다.
- Windows GCS가 WSL onboard에서 보내는 Gazebo 하향 camera MJPEG stream을 수신한다.
- onboard에서 처리한 vision telemetry를 GCS가 받아 marker/line/intersection overlay를 그린다.
- `vision_debug_node --target sitl --vision gazebo --video` path는 비행 없는 Gazebo camera + onboard vision + GCS overlay smoke에 사용 가능하다.
- `line_follow_node --target sitl --vision gazebo --video`는 camera capture, vision processing, GCS telemetry/video 송신, MAVLink SITL control을 함께 담당한다. 비행 중 `vision_debug_node`를 동시에 실행하지 않는다.
- `line_follow_node`는 실행 직후 startup video를 GCS로 보내고, heartbeat/GUIDED/arm/2m takeoff 후 line-follow control에 들어간다.
- SITL에서 line-follow, ArUco marker approach, marker 중심 3초 hover, land, disarm/complete가 검증됐다.
- `LAND` 진입 후에도 disarm 또는 timeout까지 landing video를 계속 송신한다.

구현됨:

- Pi 4 + IMX519 `rpicam-vid` MJPEG frame capture
- Gazebo `FrameSource`와 SITL runtime profile
- Astroquad Gazebo vision world/fixtures
- UDP JSON telemetry send/receive
- opt-in UDP MJPEG debug video send/receive
- GCS discovery beacon and video unicast switch
- onboard ArUco detection
- onboard line tracing and line stabilizer
- onboard intersection classifier/stabilizer
- marker-aware line/intersection mask 처리
- `MarkerStabilizer`
- `IntersectionDecisionEngine`
- `GridCoordinateTracker`
- GCS marker/line/intersection overlay
- GCS `[intersection-decision]`, `[grid-node]`, `[grid-map]` log display
- SITL MAVLink UDP heartbeat/mode/arm/takeoff/body-velocity/land staging path
- MAVLink UDP command peer pinning. Autopilot heartbeat를 보낸 peer에 command 송신 대상을 고정해 GCS heartbeat가 command peer를 가로채지 못하게 한다.
- `line_follow_node` startup video streaming과 landing video streaming
- Gazebo Iris camera FOV tuning. 현재 하향 camera FOV는 `uav-onboard/sim/gazebo/models/iris_with_downward_camera/model.sdf`의 `<horizontal_fov>1.15</horizontal_fov>`다.

미구현 또는 staging:

- full mission state machine
- Pixhawk native serial MAVLink transport
- 실기체 Pixhawk/MTF-01/GUIDED local estimate bench 검증
- safety monitor expansion
- GCS command channel
- full snake mission policy
- marker revisit policy
- official coordinate conversion
- file logging/persistent replay system

## 5. 현재 반복 테스트 절차

Windows GCS:

```powershell
cd astroquad\uav-gcs
.\build\uav_gcs_vision_debug.exe --config config
```

WSL Gazebo/SITL:

```bash
bash ~/fly_test.sh
```

WSL vision-only GCS smoke:

```bash
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"

cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --line-mode light_on_dark \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

WSL marker fixture smoke:

```bash
cd ~/astroquad/uav-onboard
./build/vision_debug_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --gazebo-topic /world/astroquad_marker_center_fixture/model/astroquad_static_downward_camera/link/downward_camera_link/sensor/downward_camera/image \
  --aruco-only \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

WSL vision-driven flight smoke:

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node \
  --config config \
  --target sitl \
  --vision gazebo \
  --video \
  --gcs-ip "$WINDOWS_GCS_IP"
```

현재 정상 관찰:

- `line_follow_node` 하나만 실행하면 GCS video/telemetry까지 같이 송신된다.
- `vision_debug_node`를 동시에 실행하면 GCS telemetry/video source가 꼬일 수 있으므로 금지한다.
- `startup_video_frames=N` 로그가 이륙 전 영상 송신을 의미한다.
- `mode=GUIDED`, `local_xy`, `vel_ned`, `vx/vy/vz/yaw_rate` 로그로 command와 실제 이동을 함께 확인한다.
- marker가 보이면 `MARKER_APPROACH`, marker 중심 tolerance 안에 들어오면 `MARKER_HOVER`, 3초 후 `LAND`, 이후 `landing_video_frames=N`, `COMPLETE`가 나와야 한다.

## 6. 현재 line_follow_node 구조와 의심 지점

`tools/line_follow_node.cpp` 현재 흐름:

```text
FrameSource(fake/gazebo/rpicam)
  -> VisionProcessor
  -> VisionDebugPublisher(GCS telemetry/video, optional)
  -> toLineControlInput()
  -> GuidedVelocityController
  -> AutopilotMavlinkAdapter::sendBodyVelocity()
  -> MAVLink SET_POSITION_TARGET_LOCAL_NED / MAV_FRAME_BODY_NED
```

현재 mission state:

```text
IDLE -> TAKEOFF -> LINE_FOLLOW -> MARKER_APPROACH -> MARKER_HOVER -> LAND -> COMPLETE
any state -> ABORT
```

중요 코드 위치:

- `uav-onboard/tools/line_follow_node.cpp`
- `uav-onboard/src/mission/LineFollowMission.*`
- `uav-onboard/src/control/GuidedVelocityController.*`
- `uav-onboard/src/autopilot/AutopilotMavlinkAdapter.*`
- `uav-onboard/src/safety/SafetyMonitor.*`
- `uav-onboard/config/runtime.sitl.toml`
- `uav-onboard/config/mission.toml`
- `uav-onboard/config/autopilot.toml`
- `uav-onboard/config/safety.toml`

현재 제어 기준:

- `toLineControlInput()`은 line center offset을 image half-width 기준 normalized error로 넘긴다.
- `GuidedVelocityController`는 normalized lateral error, line angle error, altitude error를 body-frame `vx_forward_mps`, `vy_right_mps`, `vz_down_mps`, `yaw_rate_rad_s`로 변환한다.
- 목표 고도는 2m이고, local altitude를 우선 사용하며 relative altitude와 distance sensor를 fallback으로 본다.
- line-follow 중에는 `line_detected`가 true인 동안 계속 전진한다. 화면 위쪽 line 존재 여부를 별도로 gate하지 않는다.
- line이 완전히 사라지고 marker도 없으면 안전 착륙한다.
- marker가 detected되면 marker 중심을 기준으로 approach하고, centered 상태가 유지되면 3초 hover 후 land한다.

실기체 전 남은 의심 지점:

- Pixhawk native serial transport는 아직 intentionally not implemented 상태다.
- 외부 MAVLink router/bridge 없이 `--target pixhawk1 --vision rpicam`만으로는 실제 Pixhawk 제어가 시작되지 않는다.
- MTF-01 optical flow/range 기반 EKF local estimate가 GUIDED velocity를 받을 만큼 안정적인지 아직 props-off/props-on gate를 통과하지 않았다.
- SITL controller는 검증됐지만 실기체 IMX519 lens/focus/FOV, line 폭, 바닥 texture, 조명에 맞춘 gain/threshold 재튜닝은 필요하다.

Raspberry Pi 4 전환 판단:

- vision/GCS path는 Raspberry Pi 4 + IMX519에서 이어서 사용할 수 있다.
- native serial transport 구현 전에는 코드 변경 없이 바로 자동비행하는 것이 아니라, 별도 MAVLink UDP bridge가 필요하다.
- bridge를 쓴다면 `--target pixhawk1 --vision rpicam --autopilot udp://... --gcs-ip <gcs-ip>` 형태로 실행할 수 있다.
- 그래도 props-off heartbeat/mode/arm-inhibit/land/RC takeover와 `LOCAL_POSITION_NED` 갱신 확인 전에는 prop 장착 자동비행을 금지한다.

## 7. 통신 구조

Protocol 문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

현재 protocol document version은 v1.7이고, JSON top-level `protocol_version`은 호환성을 위해 integer `1`이다.

| Channel | Direction | Transport | Default port | Status |
|---|---|---|---:|---|
| Telemetry | onboard -> GCS | UDP JSON | 14550 | implemented |
| Command | GCS -> onboard | TCP JSON | 14551 | planned |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 | implemented |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 | implemented |

GCS는 onboard metadata로만 overlay를 그린다. GCS가 line/marker/intersection detection을 다시 수행하지 않는다.

## 8. 아키텍처 경계

목표 구조:

```text
CameraSource
  -> VisionPipeline
  -> MissionStateMachine
  -> GuidanceController
  -> ControlBackend
  -> AutopilotMavlinkAdapter

Support:
  SafetyMonitor observes Vision/Mission/Autopilot/GCS state
  TelemetryPublisher observes state snapshots
  CommandReceiver injects high-level mission commands
  DebugVideoPublisher observes camera frames only
```

경계 규칙:

- `VisionPipeline`/`VisionProcessor`: image/frame metadata를 받아 `VisionOutput`/`VisionResult`를 만든다. flight mode, MAVLink, RC override를 모른다.
- `MissionStateMachine`: vision event와 high-level command를 받아 mission state/path intent를 만든다. JPEG/image와 MAVLink packet을 모른다.
- `GuidanceController`: mission intent와 line error를 `ControlSetpoint`로 변환한다.
- `ControlBackend`: `ControlSetpoint`를 GUIDED velocity 또는 RC override 출력으로 변환한다.
- `AutopilotMavlinkAdapter`: MAVLink 송수신만 담당한다. mission 판단을 하지 않는다.
- `SafetyMonitor`: heartbeat, line lost, RC takeover, battery, timeout을 감시해 command inhibit/land/abort intent를 낸다.
- `DebugVideoPublisher`와 GCS camera window는 관제용이다. mission-critical 경로가 아니다.

## 9. 절대 금지사항

- `vision_debug_node` detector 코드를 mission executable에 복사하지 않는다.
- `VisionDebugPipeline` 내부에 MAVLink/control 코드를 넣지 않는다.
- GCS가 mission 판단을 하게 만들지 않는다.
- GCS가 onboard vision detection을 다시 수행하게 만들지 않는다.
- debug video를 mission-critical 경로로 넣지 않는다.
- GCS video latency/age를 표시하지 않는다. Pi/Windows clock sync가 보장되지 않아 오해를 만든다.
- full snake/revisit/command UI를 첫 line-follow MVP의 선행 조건으로 만들지 않는다.
- MTF-01 bring-up gate 없이 실기체 GUIDED velocity line-follow를 강행하지 않는다.
- serial Pixhawk transport가 없는데 실기체 자동비행이 준비된 것처럼 취급하지 않는다.
- unrelated refactor나 대규모 UI 전환을 현재 line-follow 제어 튜닝 작업에 섞지 않는다.

## 10. 다음 작업 제안

현재 가장 중요한 다음 작업은 **실기체 전환 gate**다.

작업 후보:

- MTF-01 optical flow/range가 ArduPilot EKF local estimate에 안정적으로 반영되는지 확인한다.
- Pixhawk native serial MAVLink transport를 구현하거나, 외부 MAVLink router/UDP bridge 운용 절차를 확정한다.
- Raspberry Pi 4에서 `line_follow_node --target pixhawk1 --vision rpicam` 경로를 props-off로 실행해 heartbeat, mode, arm inhibit, land, RC takeover를 검증한다.
- `LOCAL_POSITION_NED`, altitude source, mode, arming state가 control log와 telemetry에 기대대로 들어오는지 확인한다.
- 실기체 IMX519 focus/FOV, line 폭, threshold, controller gain을 현장 조건에 맞게 낮은 속도부터 다시 튜닝한다.

완료 기준은 “Pi에서 binary가 실행된다”가 아니라, props-off bench에서 Pixhawk 통신/모드/안전 경로가 확인되고, MTF-01 기반 local estimate가 GUIDED velocity command를 받을 만큼 안정적이라는 점이 확인되는 것이다.
