# Astroquad System Spec

최종 업데이트: 2026-05-15

이 문서는 `uav-onboard`와 `uav-gcs`를 모두 포함하는 전체 시스템 기준 문서다. Repo별 세부 책임은 각 repo의 `PROJECT_SPEC.md`를 따른다.

문서 역할:

- `SYSTEM_SPEC.md`: 전체 시스템 목적, 하드웨어, 모듈 경계, 최종 실행 파일, 통신 구조.
- `MVP_PLAN.md`: 72시간/1주일 단위 큰 개발 계획과 시험 gate.
- `RESEARCH.md`: 현재 한 스텝 작업을 시작할 때 조사한 사실을 임시 기록.
- `PLAN.md`: 현재 한 스텝 작업의 실행 계획과 체크리스트를 임시 기록.
- `TROUBLESHOOTING.md`: 반복되는 문제와 해결 이력.

## 1. 시스템 목표

Astroquad는 GPS 없는 실내/운동장 격자 환경에서 UAV가 하향 카메라 기반 라인 인식, 교차점 판단, ArUco marker 인식을 사용해 탐색 임무를 수행하는 C++ 기반 시스템이다.

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

단기 MVP는 최종 목표의 부분 집합이며, 세부 범위는 `MVP_PLAN.md`에서 관리한다.

## 2. 하드웨어 기준

| 모델/장치 | 연결 위치 | 역할 |
|---|---|---|
| Pixhawk1 | 기체 중앙 | 비행 제어기. 자세 안정화, 모터 출력, 센서 융합, 모드 관리 |
| MicoAir MTF-01 Optical Flow & Range Sensor | Pixhawk TELEM2 또는 SERIAL4/5 | GPS 없이 수평 이동 추정 + 바닥 거리 측정. Optical Flow + ToF range 통합 |
| Raspberry Pi 4 | Pixhawk USB 또는 TELEM1 | OpenCV 비전 처리, 라인/교차점/ArUco 판단, MAVLink 제어 명령 전송 |
| IMX519 Camera | Raspberry Pi CSI | 하향 영상 입력. 라인 중심, 라인 각도, ArUco 마커 인식 |
| Power Module | Pixhawk POWER | Pixhawk 전원 공급, 배터리 전압/전류 측정 |
| RC Receiver | Pixhawk RC IN | 수동 조종 takeover, 비상 개입 |
| ESC/Motor/PDB | Pixhawk MAIN OUT + 전원분배 | 실제 모터 구동 |
| TFmini Plus | 기본 구성에서는 제외 | MTF-01 range가 불안정할 때 예비 rangefinder |
| External Compass | 제외 | 일단 Pixhawk 내부 compass/IMU 기준으로 진행 |

## 3. Repo 역할

### uav-onboard

담당:

- Raspberry Pi camera capture
- ArUco/line/intersection detection
- marker and line stabilization
- local grid-node event telemetry
- mission state machine
- guidance/control backend
- MAVLink adapter
- safety/failsafe
- telemetry/debug video sender

최종 실행 파일:

```bash
./build/uav_onboard --config config
```

Debug/staging 실행 파일:

- `vision_debug_node`: camera/vision/telemetry/debug video bring-up.
- `video_streamer`: raw MJPEG transport smoke tool.
- `mission_node` 또는 `line_follow_node`: 임시 staging target 허용. 안정화 후 `uav_onboard`로 흡수한다.

Gazebo/SITL staging:

- WSL에서 `bash ~/fly_test.sh`로 Astroquad Gazebo vision world와 ArduCopter SITL을 실행한다.
- main world는 Iris 하향 camera, 어두운 ground, 폭 10cm 흰색 직선 line, 출발점 기준 3m 전방 50cm x 50cm ArUco ID 1 marker를 포함한다.
- `vision_debug_node --target sitl --vision gazebo --video`는 Gazebo 하향 camera frame을 GCS로 보내고 기존 line/marker/intersection overlay telemetry를 유지한다.
- `line_follow_node --target sitl --vision gazebo --video`는 같은 vision/GCS path와 MAVLink UDP SITL control path를 함께 검증하는 staging executable이다. 비행 중에는 camera, vision, telemetry, video 송신을 이 프로세스 하나가 담당한다.
- `line_follow_node`는 이륙 전 startup video, 비행 중 overlay video, 착륙 중 landing video를 모두 GCS로 보낸다.
- `vision_debug_node`와 `line_follow_node`를 동시에 GCS video/telemetry sender로 실행하지 않는다.
- Gazebo Iris camera zoom은 `uav-onboard/sim/gazebo/models/iris_with_downward_camera/model.sdf`의 `<horizontal_fov>`에서 조정한다. 작은 값은 zoom-in, 큰 값은 zoom-out이다.
- 실기체 기본값은 Raspberry Pi 4 + IMX519 + Pixhawk1이며, Gazebo 값은 runtime profile 또는 CLI option으로만 선택한다.

### uav-gcs

담당:

- onboard telemetry receive/parse/display
- raw debug video receive/display
- GCS-side marker/line/intersection overlay
- vision log and grid-map log display
- future command sender and mission dashboard
- event/command logging and replay

최종 실행 파일:

```powershell
.\build\Release\uav_gcs.exe --config config
```

Ninja 빌드:

```powershell
.\build\uav_gcs.exe --config config
```

Debug 실행 파일:

- `uav_gcs_vision_debug`: vision 관제/튜닝용.
- `uav_gcs_video`: raw MJPEG viewer.
- `mock_onboard`, `log_replayer`: development tools.

## 4. 모듈 경계

ROS를 직접 쓰지는 않지만, 설계 원칙은 ROS node graph처럼 각 모듈이 typed input/output만 공유하고 내부 상태를 침범하지 않는 형태다.

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

- `VisionPipeline`: image/frame metadata를 받아 `VisionOutput`을 만든다. flight mode, MAVLink, RC override를 모른다.
- `MissionStateMachine`: vision event와 high-level command를 받아 mission state/path intent를 만든다. JPEG/image와 MAVLink packet을 모른다.
- `GuidanceController`: mission intent와 line error를 `ControlSetpoint`로 변환한다.
- `ControlBackend`: `ControlSetpoint`를 GUIDED velocity 또는 RC override 출력으로 변환한다.
- `AutopilotMavlinkAdapter`: MAVLink 송수신만 담당한다. mission 판단을 하지 않는다.
- `SafetyMonitor`: heartbeat, line lost, RC takeover, battery, timeout을 감시해 command inhibit/land/abort intent를 낸다.

## 5. 코드 재사용 원칙

CMake 링크는 코드 공유의 수단이지만, CMake만으로 구조가 해결되지는 않는다. 현재 이미 존재하는 재사용 기반:

- onboard: `onboard_core`, `onboard_video`, `onboard_vision`
- GCS: `gcs_core`, `gcs_video`

필요한 리팩터링:

- detector 실행 부분을 `VisionPipeline` 또는 동등한 library class로 분리한다.
- `vision_debug_node`는 `VisionPipeline + telemetry + optional debug video`만 묶는다.
- line-follow staging executable은 `VisionPipeline + mission/control/safety`를 묶는다.
- 최종 안정화된 runtime은 `uav_onboard`로 흡수한다.

금지:

- `vision_debug_node` detector 코드를 mission executable에 복사하지 않는다.
- `VisionDebugPipeline` 내부에 MAVLink/control 코드를 넣지 않는다.
- GCS가 mission 판단을 하게 만들지 않는다.
- debug video를 mission-critical 경로로 넣지 않는다.

## 6. 제어 전략

Primary:

- ArduPilot `GUIDED`
- MAVLink `SET_POSITION_TARGET_LOCAL_NED`
- body-frame velocity setpoint

Fallback:

- ArduPilot `ALT_HOLD`
- MAVLink `RC_CHANNELS_OVERRIDE`
- degraded/manual-like test path only

판단:

- A안 GUIDED velocity가 primary다. line offset/angle을 물리 단위 velocity/yaw setpoint로 바꾸기 쉽고 SITL 재사용성이 높다.
- A안은 MTF-01 optical flow/range 기반 local estimate 품질에 의존한다.
- B안 RC override는 local estimate 의존이 낮지만 튜닝 의존과 RC takeover 충돌 위험이 크다.
- mission logic은 A/B로 나누지 않는다. 공통 `ControlSetpoint`를 만들고 출력 backend만 교체한다.

## 7. 통신 구조

공통 protocol 문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

현재 protocol document version은 v1.7이고, JSON top-level `protocol_version`은 호환성을 위해 integer `1`이다.

| Channel | Direction | Transport | Default port | Status |
|---|---|---|---:|---|
| Telemetry | onboard -> GCS | UDP JSON | 14550 | implemented |
| Command | GCS -> onboard | TCP JSON | 14551 | planned |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 | implemented |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 | implemented |

## 8. 현재 구현 기준선

구현됨:

- Pi 4 + IMX519 `rpicam-vid` MJPEG frame capture
- Gazebo `FrameSource`와 SITL runtime profile
- Astroquad Gazebo vision course/fixtures
- UDP JSON telemetry send/receive
- opt-in UDP MJPEG debug video send/receive
- GCS discovery beacon and video unicast switch
- onboard ArUco detection
- onboard line tracing and line stabilizer
- onboard intersection classifier and temporal stabilizer
- marker-aware line/intersection mask 처리
- `MarkerStabilizer`
- `IntersectionDecisionEngine`
- `GridCoordinateTracker`
- GCS marker/line/intersection overlay
- GCS `[intersection-decision]`, `[grid-node]`, `[grid-map]` log display
- SITL MAVLink UDP heartbeat/mode/arm/takeoff/body-velocity/land staging path
- SITL line-follow mission staging: 2m takeoff/altitude hold, line-follow, ArUco marker approach, 3초 marker hover, land, complete
- MAVLink UDP command peer pinning. Autopilot heartbeat peer에 command 송신 대상을 고정해 GCS heartbeat가 command peer를 가로채지 못하게 한다.
- `line_follow_node` startup video와 landing video streaming

미구현 또는 staging:

- full mission state machine
- Pixhawk native serial MAVLink transport
- 실기체 Pixhawk/MTF-01/GUIDED local estimate bench 검증
- safety monitor expansion
- GCS command channel
- full snake mission policy
- marker revisit policy
- official coordinate conversion

## 9. Raspberry Pi 4 실기체 전환 판단

현재 온보드 코드는 Raspberry Pi 4에서 vision/GCS path를 재사용할 수 있도록 구성되어 있다.

바로 사용할 수 있는 것:

- IMX519 `rpicam-vid` MJPEG capture
- onboard line/ArUco/intersection vision 처리
- GCS telemetry/video 송신
- `line_follow_node`의 mission/control state 구조

아직 옵션 변경만으로 바로 실기체 자동비행까지 이어지지 않는 것:

- `--target pixhawk1 --vision rpicam`은 native serial Pixhawk MAVLink transport가 필요하다.
- 현재 serial transport는 intentionally not implemented 상태이므로, 직접 serial 제어 경로는 추가 구현이 필요하다.
- 코드 수정 없이 시도하려면 별도 MAVLink router/bridge가 Pixhawk serial/USB를 UDP endpoint로 노출해야 한다. 이때 `line_follow_node`는 `--autopilot udp://...`로 그 endpoint를 사용한다.

실기체 gate:

- MTF-01 optical flow/range가 ArduPilot EKF local estimate에 안정적으로 반영되어야 한다.
- GUIDED mode에서 `LOCAL_POSITION_NED`와 altitude source가 정상 갱신되어야 한다.
- props-off 상태에서 heartbeat, mode change, arm inhibit, land, RC takeover를 검증한다.
- prop 장착 전 GCS unicast video, camera focus/FOV, line width, detector threshold, controller gain을 현장 조건에 맞춘다.

결론: Raspberry Pi 4에서 vision과 GCS 송신은 이어서 쓸 수 있지만, native serial transport 구현 또는 외부 UDP bridge 절차 확정 없이는 “실행 옵션만 바꿔 바로 자동비행” 단계가 아니다.

## 10. 보고서 작성 기준

1차 기술개발계획서는 다음 흐름을 따른다.

- 전체 시스템 개요
- 자동비행 시스템 및 구현 기술
- 임무 장치 및 수행 방법
- 구성품 적정성
- 지상 및 비행 시험 계획
- 시스템 설계 및 제작 특장점
- 안전장치 설명
- 개발팀 현황

보고서에서 최종 목표와 단기 MVP를 분리해서 설명한다. 단기 MVP는 `MVP_PLAN.md`를 근거로 한다.
