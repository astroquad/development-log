# Astroquad MVP Plan

최종 업데이트: 2026-05-15

이 문서는 72시간 실기체 MVP와 1주일 기술개발계획서 작성 계획을 관리한다. 전체 시스템 기준은 `SYSTEM_SPEC.md`를 따른다.

## 1. 현재 MVP 범위

현재 72시간 목표는 전체 snake/marker mission 완성이 아니다.

목표:

```text
MTF-01 bring-up
  -> 자동 이륙
  -> 짧은 직선 line follow
  -> line end, line lost, timeout, operator abort 중 하나에서 안전 착륙
```

MVP 밖으로 뺀 것:

- full snake grid exploration
- ArUco marker revisit
- official coordinate conversion
- 교차점 회전/분기 의사결정
- GCS full command UI
- marker count command
- backend switching UI
- command ACK/retry UI

## 2. P-1 MTF-01 Bring-Up Gate

MTF-01 optical flow/range가 Pixhawk/ArduPilot에서 안정적으로 인식되는지 먼저 확인한다. 이 gate가 실패하면 GUIDED velocity 기반 자동 line follow는 진행하지 않는다.

확인 항목:

- Pixhawk 포트/baud/프로토콜 설정 확인.
- ArduPilot에서 optical flow와 rangefinder 수신 확인.
- 바닥 높이 변화에 range가 일관되게 반응하는지 확인.
- props off 상태에서 EKF/local estimate 관련 오류 확인.
- 실기체 저고도 hover에서 altitude/range가 튀지 않는지 확인.

완료 기준:

- Mission Planner 또는 MAVLink telemetry에서 flow/range 상태가 확인된다.
- GUIDED 또는 optical-flow 기반 position/velocity hold가 가능한 조건이라고 판단된다.
- 실패 시 MVP를 RC/ALT_HOLD 보조 실험 또는 vision-only 시연으로 낮춘다.

## 3. P0 Vision Baseline Freeze

- `vision_debug_node` 동작을 freeze한다.
- line follow에 필요한 `vision.line.*`만 1차 제어 입력으로 사용한다.
- intersection, grid-node, ArUco는 telemetry에 남기되 MVP control에는 사용하지 않는다.

완료 기준:

- onboard tests 통과
- GCS tests 통과
- Pi metadata-only line-only 5분 이상 실행에서 frame/telemetry loop 유지

## 4. P1 VisionPipeline 분리

구현 목표:

- `VisionDebugPipeline`에서 detector 실행 부분을 `VisionPipeline` 또는 유사한 library class로 분리한다.
- `vision_debug_node`는 기존처럼 debug/video/telemetry에 집중한다.
- 임시 `mission_node` 또는 `line_follow_node` 실행 파일을 허용한다.
- 임시 실행 파일은 3일 MVP staging target이며, 최종 안정화 후 `uav_onboard`로 흡수한다.

완료 기준:

- `vision_debug_node` 동작이 유지된다.
- staging executable이 같은 vision library를 링크해 line offset을 읽을 수 있다.
- detector 코드를 복사한 파일이 생기지 않는다.

현재 상태:

- `FrameSource` 계열은 `fake`, `gazebo`, `rpicam`으로 분리되어 있다.
- `VisionProcessor`가 ArUco/line/intersection vision result를 생성한다.
- `VisionDebugPublisher`가 `vision_debug_node`와 `line_follow_node`에서 GCS telemetry/video 송출에 재사용된다.
- Gazebo vision-only smoke와 SITL line-follow smoke는 통과했으며, line-follow 제어 성능 튜닝은 별도 작업이다.

## 5. P2 MAVLink Adapter 최소 구현

최소 기능:

- heartbeat receive/send
- arm/disarm
- set GUIDED mode
- takeoff
- land
- body-frame velocity setpoint send
- battery/mode/armed/altitude/range/heartbeat 상태 저장
- UDP SITL endpoint와 serial/USB Pixhawk endpoint를 config로 선택

완료 기준:

- SITL에서 arm -> GUIDED -> takeoff -> 짧은 전진 velocity -> land 가능
- props off Pixhawk bench에서 heartbeat/mode/arm command path 확인

현재 상태:

- UDP SITL transport와 MAVLink adapter는 staging 구현됨.
- `line_follow_node --target sitl --vision gazebo --video`에서 heartbeat, GUIDED, arm, takeoff, line-follow, land, complete까지 확인됨.
- Raspberry Pi/Pixhawk 실기체용 serial transport와 props-off bench 검증은 남아 있다.

## 6. P3 축소 Mission State Machine

3일 MVP 상태:

```text
IDLE
TAKEOFF
LINE_FOLLOW
LAND
COMPLETE
ABORT
```

전환:

- `IDLE -> TAKEOFF`: CLI/config start 또는 임시 command.
- `TAKEOFF -> LINE_FOLLOW`: 목표 고도 도달.
- `LINE_FOLLOW -> LAND`: line end 감지, line lost timeout, runtime timeout, operator abort 중 하나.
- `LAND -> COMPLETE`: 착륙 완료.
- `any -> ABORT`: heartbeat loss, RC takeover, severe failsafe.

이 상태머신은 전체 mission state machine의 축소판이다. full snake, marker revisit, grid exploration 상태는 이 MVP에 넣지 않는다.

현재 상태:

- 축소 상태머신은 `LineFollowMission`으로 구현되어 SITL smoke에 사용된다.
- marker hover transition은 존재하지만, line-follow 제어 튜닝 전이므로 실기체 MVP gate는 여전히 짧은 직선 추종 + 안전 착륙이다.

## 7. P4 Line-Follow Controller와 Safety

Control 입력:

- `line.detected`
- `line.center_offset_px`
- `line.angle_deg`
- `line.confidence`

초기 제어:

- forward velocity는 낮게 고정한다.
- lateral velocity 또는 yaw rate는 offset/angle 기반 P 제어로 제한한다.
- line lost가 짧으면 hover/stop, 길면 land.
- velocity/yaw/altitude는 hard clamp한다.

완료 기준:

- SITL 또는 fake vision replay에서 line offset sign에 맞는 제어 명령이 나온다.
- props off bench에서 command inhibit/land path가 확인된다.

## 8. P5 GCS Scope 축소

3일 MVP에서 GCS command channel은 필수가 아니다. GCS는 telemetry/video/log 관제에 집중한다.

허용 범위:

- `uav_gcs_vision_debug`로 line offset, video, telemetry, Pixhawk state 확인.
- mission start는 onboard CLI/config 또는 임시 local trigger로 처리.
- emergency는 RC takeover, Pixhawk mode switch, kill/land 절차를 우선.

미루는 것:

- full mission command UI
- marker count command
- backend 자유 전환 UI
- command ACK/retry UI

## 9. P6 첫 실기체 Gate

첫 실기체 자동 목표는 **짧은 직선 line follow + 안전 착륙**이다.

순서:

1. Props off: Pi-Pixhawk heartbeat, mode, arm inhibit, RC takeover 확인.
2. Manual/assisted: MTF-01 range/flow 기반 안정 hover 확인.
3. Auto takeoff only: 이륙 후 즉시 land.
4. Short line follow: 2-5m 직선 라인에서 저속 추종.
5. Line end/timeout land: line 끝 또는 timeout에서 안전 착륙.

이 gate 전에는 교차점 회전, snake, marker revisit을 시도하지 않는다.

## 10. 개발 우선순위

1. MTF-01 bring-up gate 통과.
2. `VisionPipeline` 분리.
3. MAVLink adapter 최소 구현.
4. GUIDED body velocity line-follow controller 구현.
5. 축소 상태머신 구현.
6. GCS command 축소, 관제/로그 집중.
7. props off bench -> auto takeoff/land -> short line follow -> line end landing.
8. full snake/marker revisit은 MVP 이후 진행.

## 11. 1주일 기술개발계획서 작성 계획

참고 PDF 구조를 따른다.

- 전체 시스템의 개요
- 자동비행 시스템 및 구현 기술
- 임무 장치 및 수행 방법
- 구성품의 적정성
- 지상 및 비행 시험 계획
- 시스템 설계 및 제작상의 특장점
- 안전장치에 대한 설명
- 개발팀 현황

보고서 핵심 메시지:

- 최종 목표는 grid snake 탐색과 marker revisit이다.
- 1차 실기체 MVP는 안전한 line-follow 자동비행으로 범위를 낮춘다.
- Pixhawk1, MTF-01, Raspberry Pi 4, IMX519, GCS의 역할이 분리되어 있다.
- ROS-like 모듈 구조로 vision/mission/control/MAVLink/safety가 분리된다.
- 안전 gate를 통과한 범위만 실기체에서 확장한다.
