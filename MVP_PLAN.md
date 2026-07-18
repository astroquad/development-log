# Astroquad MVP Plan

원 계획 작성: 2026-05-15 / 상태 검토: 2026-07-18

> **주의 (2026-07-04)**: 이 문서는 구 플랫폼(Pixhawk 1 / Raspberry Pi 4 /
> IMX519 / 3S 1.5kg) 기준으로 작성된 과거 계획이다. 1차 예선 통과 후
> 하드웨어가 전면 교체되었다(Pixhawk 6C Mini / Pi 5 / IMX296 mono GS /
> 4S 2.1kg). 현행 기준은 `SYSTEM_SPEC.md` §2와
> `uav-onboard/config/pixhawk6c_indoor_flow.params`를 따른다.

> **현재 상태:** 이 문서의 72시간 MVP 범위는 완료된 과거 milestone이다.
> native serial transport, RC/local-estimate gate, battery ingest,
> `line_follow_node` staging, 풀 grid snake/revisit/return 상태 머신과 실행별
> 비행 로그가 현재 구현되어 있다. 앞으로의 실행 계획은
> [REAL_FLIGHT_ONBOARDING.md](REAL_FLIGHT_ONBOARDING.md)의 짧은 실비행 반복
> 파이프라인을 따른다. 아래의 “남아 있다”는 표현은 당시 시점의 기록이다.

이 문서는 72시간 실기체 MVP와 1주일 기술개발계획서 작성 계획을 관리한다. 전체 시스템 기준은 `SYSTEM_SPEC.md`를 따른다.

## 1. 현재 MVP 범위

현재 72시간 목표는 전체 snake/marker mission 완성이 아니다.

목표:

```text
MTF-01 bring-up
  -> 자동 이륙
  -> 짧은 직선 line follow
  -> 종료 marker 또는 line end/lost/timeout/operator abort 중 하나에서 안전 착륙
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
- `line_follow_node --target sitl --vision gazebo --video`에서 heartbeat, GUIDED, arm, 2m takeoff, line-follow, ArUco approach, 3초 hover, land, complete까지 확인됨.
- MAVLink UDP transport는 autopilot heartbeat를 보낸 peer에 command 송신 대상을 고정한다. GCS/Mission Planner heartbeat가 같은 UDP port 주변에 섞여도 command peer를 가로채지 않아야 한다.
- `line_follow_node` control log는 `mode`, `local_xy`, `vel_ned`, `vx/vy/vz/yaw_rate`를 출력해 실제 이동과 명령 송신을 함께 확인할 수 있다.
- Raspberry Pi/Pixhawk 실기체용 native serial transport와 props-off bench 검증은 남아 있다.

## 6. P3 축소 Mission State Machine

3일 MVP 상태:

```text
IDLE
TAKEOFF
LINE_FOLLOW
MARKER_APPROACH
MARKER_HOVER
LAND
COMPLETE
ABORT
```

전환:

- `IDLE -> TAKEOFF`: CLI/config start 또는 임시 command.
- `TAKEOFF -> LINE_FOLLOW`: 목표 고도 도달.
- `LINE_FOLLOW -> MARKER_APPROACH`: ArUco marker detected.
- `LINE_FOLLOW -> LAND`: line end 감지, line lost timeout, runtime timeout, operator abort 중 하나.
- `MARKER_APPROACH -> MARKER_HOVER`: marker center가 tolerance 안으로 들어옴.
- `MARKER_HOVER -> LAND`: 3초 hover 완료.
- `LAND -> COMPLETE`: 착륙 완료.
- `any -> ABORT`: heartbeat loss, RC takeover, severe failsafe.

이 상태머신은 전체 mission state machine의 축소판이다. full snake, marker revisit, grid exploration 상태는 이 MVP에 넣지 않는다.

현재 상태:

- 축소 상태머신은 `LineFollowMission`으로 구현되어 SITL smoke에 사용된다.
- `line_ahead` 같은 화면 위쪽 line 존재 조건은 제거했다. 라인 추종 중에는 `line_detected`가 true인 한 계속 전진/추종하고, 라인이 더 이상 잡히지 않을 때 착륙한다.
- marker가 잠깐 안 보여도 line이 보이면 `MARKER_APPROACH`에서 line-follow fallback으로 계속 전진한다.
- 실기체 MVP gate는 여전히 짧은 직선 추종 + 안전 착륙이며, full snake/revisit은 제외한다.

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
- SITL에서 2m altitude hold, line-follow, marker approach, marker hover, land, complete가 반복 통과한다.
- props off bench에서 command inhibit/land path가 확인된다.

현재 상태:

- `GuidedVelocityController`는 normalized line center error와 axial line angle error를 받아 body-frame forward/lateral/yaw-rate setpoint를 만든다.
- 2m altitude hold는 local altitude, relative altitude, distance sensor 순으로 가능한 값을 사용한다.
- SITL에서는 `cmake --build build`, `ctest --test-dir build --output-on-failure`, Gazebo line-follow smoke가 통과했다.
- 안전 확장, RC takeover 감지, battery failsafe, real Pixhawk serial transport는 남아 있다.

## 8. P5 GCS Scope 축소

3일 MVP에서 GCS command channel은 필수가 아니다. GCS는 telemetry/video/log 관제에 집중한다.

허용 범위:

- `uav_gcs_vision_debug`로 line offset, video, telemetry, Pixhawk state 확인.
- mission start는 onboard CLI/config 또는 임시 local trigger로 처리.
- emergency는 RC takeover, Pixhawk mode switch, kill/land 절차를 우선.
- 비행 중 GCS 영상/telemetry는 `line_follow_node --video` 하나가 담당한다. `vision_debug_node`는 비행 없는 vision-only smoke 전용이며 동시에 실행하지 않는다.

미루는 것:

- full mission command UI
- marker count command
- backend 자유 전환 UI
- command ACK/retry UI

## 9. P6 첫 실기체 Gate

첫 실기체 자동 목표는 **짧은 직선 line follow + 안전 착륙**이다.

순서:

1. Props off: Pi-Pixhawk heartbeat, mode, arm inhibit, RC takeover 확인.
2. Native serial transport 또는 외부 MAVLink router 기반 UDP bridge 중 실제 연결 방식을 확정.
3. Manual/assisted: MTF-01 range/flow 기반 안정 hover 확인.
4. Auto takeoff only: 이륙 후 즉시 land.
5. Short line follow: 2-5m 직선 라인에서 저속 추종.
6. Line end/timeout/marker land: line 끝, marker, 또는 timeout에서 안전 착륙.

이 gate 전에는 교차점 회전, snake, marker revisit을 시도하지 않는다.

## 10. 개발 우선순위

1. MTF-01 bring-up gate 통과.
2. Raspberry Pi 4에서 `line_follow_node --target pixhawk1 --vision rpicam` 실기체 경로가 막히는 지점 확인.
3. Pixhawk native serial transport 구현 또는 MAVLink router/UDP bridge 운용 절차 확정.
4. props off bench -> auto takeoff/land -> short line follow -> marker/line end landing.
5. GCS command 축소, 관제/로그 집중.
6. full snake/marker revisit은 MVP 이후 진행.

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

# ASTRO QUAD 2차 예선 준비 회의

> **회의일:** 2026.06.24
>
>
> **목표:** 2차 예선까지 실기체 라인트레이싱 및 ArUco 호버링 완성
>

---

# 1. 주요 일정

| 날짜 | 내용 |
| --- | --- |
| 6/25 | 개발지원금 150만 원 입금 예정 |
| 6월 말 | 기자재 선정 및 주문 |
| 7월 초 | 기체 재조립 및 기본 비행 |
| 7월 중순 | 라인트레이싱·ArUco 호버링 |
| 7/20 | 기존 팀장 인턴 시작, 팀장 대행 체계 전환 |
| 7/30 13:00 | 기술개발보고서·비행데이터·영상 제출 |
| 8/4 | 2차 예선 대면 심사 |
| 9/5 | 본선 |

2차 예선은 발표 15분, 질의응답 5분이며, 기술개발보고서·비행데이터·비행영상이 필수 제출자료다. 발표자료는 16:9 형식, 약 20매 내외가 권장된다.

---

# 2. 2차 예선 전 목표

## 반드시 완성

- 실기체 정상 이륙 및 호버링
- Optical Flow 또는 대체 위치추정 방식 안정화
- 실제 환경 라인트레이싱
- ArUco 마커 탐지
- 마커 위 정지 또는 호버링
- GCS 영상·텔레메트리 출력
- 안전장치 및 비상착륙 시연

## 가능하면 완성

- 격자 자동 탐색
- 마커 위치 저장
- 마커 순서대로 재방문
- 시작점 복귀 및 자동착륙

대회 임무는 GPS 없이 약 2m 고도에서 10cm 폭의 격자선을 따라 비행하고, 교차점의 ArUco 마커를 탐지·저장한 뒤 번호 순서대로 재방문하는 구조다.

---

# 3. 현재 상태

## 완료

- Gazebo에서 룰 기반 전체 미션 구현
- 격자 탐색
- 라인 및 교차점 인식
- ArUco 탐지
- 마커 기록·재방문·복귀
- GCS 기본 구현

## 미완료

- 실기체 정상 이륙
- Optical Flow 기반 GUIDED 모드
- 실기체 라인트레이싱
- ArUco 호버링
- 전체 시스템 통합 시험

---

# 4. 최우선 기술 문제

## 문제 1. GUIDED 모드 진입 실패

Mission Planner에서 Optical Flow를 신뢰할 수 없다고 판단해 GUIDED 모드가 거절되고 있다.

### 확인 항목

- Optical Flow quality
- Rangefinder 거리값
- 센서 방향 및 장착 상태
- EKF source 설정
- Local Position 추정 여부
- 진동, 조도, 바닥 텍스처
- ArduPilot 로그 및 Pre-arm 메시지

### 방향

1. Optical Flow + Rangefinder + EKF 정상화
2. GUIDED 속도제어 재시험
3. 실패 시 GUIDED_NOGPS 또는 다른 위치추정 방식 검토

현재 온보드 구조는 비전 결과를 상태기계가 처리한 뒤 body-frame 속도 명령으로 변환해 ArduPilot에 전달하는 방식이므로, 자동비행 모드와 위치추정이 먼저 정상화되어야 한다.

---

## 문제 2. 모터 추력 부족

명령은 1600 수준으로 넣었지만 실제 모터 속도는 약 900RPM에 머물러 이륙하지 못했다(수동 조종은 정상적으로 가능함, 단 alt hold 모드로 라인 트레이싱하는 테스트 코드 실행시 모터가 돌긴 하나, 추력이 약해서 이륙하지 못 하는 문제가 있음).

### 확인 항목

- 배터리 전압 및 전압 강하
- ESC 캘리브레이션
- 출력 제한 파라미터
- 모터 KV
- 프로펠러 규격과 방향
- 모터·ESC 불량
- 기체 총중량
- 각 모터 RPM 편차
- FC 명령값이 실제 ESC까지 전달되는지
- Mission Planner Motor Test 결과

### 시험 순서

1. 프로펠러 제거 후 개별 모터 테스트
2. ESC 및 출력 범위 확인
3. RPM 비교
4. 프로펠러 장착 후 추력 시험
5. 기체 중량 대비 총추력 계산
6. 수동 호버링 성공 후 자동비행 시험

직접 모터 PWM을 제어하는 방식은 최종 제어 방식으로 사용하지 않고, 모터·ESC 문제 확인용으로만 사용한다.

---

# 5. 소프트웨어팀 할 일

- GUIDED 거절 원인 분석
- Optical Flow·Rangefinder·EKF 로그 확인
- 모터 RPM 부족 원인 분석 지원
- 실기체 MAVLink 연결 및 명령 시험
- 실제 카메라 라인·ArUco 검출
- 라인트레이싱 및 마커 호버링 통합
- 전체 코드 아키텍처 학습
- 모뎀 설치 후, 테일스케일 연결(인터넷을 통한 관제 영상 수신)

## 코드 학습 기준

함수와 문법을 모두 외우는 것이 아니라 다음 흐름을 설명할 수 있어야 한다.

```
Camera
→ Vision
→ Intersection / Grid Tracking
→ Mission FSM
→ Control Mapper
→ MAVLink
→ Flight Controller
```

각 팀원은 자신이 맡은 모듈의 입력, 출력, 수정 위치를 설명할 수 있어야 한다.

---

# 6. 하드웨어팀 할 일

개발지원금 입금 전까지 구매 후보와 견적을 준비한다.

## 검토 대상

- Flight Controller
- Raspberry Pi 5
- 카메라
- Optical Flow
- Rangefinder
- ESC
- 모터
- 프로펠러
- 배터리
- 전원 모듈
- 진동 방지 마운트
- 예비 부품

## 선정 기준

- ArduPilot 호환성
- 성능
- 무게
- 소비전력
- 납기
- 국내 구매 가능 여부
- 현재 문제 해결에 실제 도움이 되는지
- 150만 원 예산 내 우선순위

하드웨어팀은 다음 회의 전까지 **현재 부품 / 문제점 / 구매 후보 / 가격 / 구매 필요도**를 표로 정리한다.

---

# 7. 주차별 계획

## 6/24~6/30

- GUIDED 거절 원인 분석
- 모터 RPM 부족 원인 분석
- 기자재 견적 확정
- 코드 아키텍처 학습

## 7/1~7/7

- 기자재 수령 및 재조립
- 모터·ESC·센서 검증
- 수동 이륙·호버링

## 7/8~7/14

- GUIDED 또는 대체 자동제어 확정
- 실제 라인·ArUco 검출
- 온보드 명령 이동 시험

## 7/15~7/19

- 라인트레이싱
- ArUco 마커 호버링
- GCS 시연
- 안전장치 영상 확보

## 7/20~7/29

- 전체 미션 통합
- 비행영상·데이터 확보
- 기술개발보고서 및 발표자료 완성

---

# 8. 팀 운영 방식

앞으로 하드웨어팀과 소프트웨어팀은 같은 날 모여서 함께 작업한다.

- 정기 전체 모임 필참
- 중요한 일정이 있으면 사전 공유
- 불참 시 맡은 과업 결과물 제출
- 단순 참석이 아니라 실제 결과물 기준으로 기여도 판단
- 반복적으로 불참하거나 맡은 업무를 수행하지 않는 인원은 최종 보고서 명단에서 제외 가능

---

# 9. 팀장 역할 전환

기존 팀장은 7월 20일부터 인턴을 시작하므로 이후 현장 관리가 어렵다.

- 팀장 대행
- 소프트웨어 책임자
- 하드웨어 책임자
- 문서·영상 책임자
- 8월 4일 2차 예선 참석자

2차 예선은 팀당 최대 6명 참석 가능하며, 발표자·시스템통합·SW·HW·GCS·조종 담당 중심으로 선정한다.

---

# 10. 워크숍 영상 시청

전 팀원은 다음 영상 두 개를 다음 회의 전까지 시청한다.

1. 자율비행을 위한 상태기계와 행동트리

    https://youtu.be/LH6gZCrAeUg

2. 비전 기반 위치추정 기술

    https://youtu.be/GQ1yCByOXVg


각 영상에서 현재 프로젝트에 적용할 수 있는 내용을 2개씩 정리한다.

---

# ~~11. 오늘 결정사항~~

- 정기 모임 요일·시간
- 기술 문제별 담당자
- 기자재 견적 담당자
- 팀장 대행
- 2차 예선 참석 후보
- 이번 주 마감 업무

| 업무 | 담당자 | 마감 |
| --- | --- | --- |
| GUIDED·Optical Flow 분석 |  |  |
| 모터 RPM 부족 분석 |  |  |
| 기자재 견적 |  |  |
| 코드 아키텍처 학습 |  |  |
| 워크숍 영상 시청 | 전원 | 다음 회의 |
| 팀장 대행 선정 |  |  |
