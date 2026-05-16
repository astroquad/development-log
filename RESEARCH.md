# Astroquad Research Snapshot

Last updated: 2026-05-16 KST

Scope: `line_follow_node`를 RC 채널 값 검증 없이 진행한다고 가정했을 때의
구현 현황과 실제 비행 안정성 판단.

이번 판단에서 제외하는 항목:

- full snake/grid exploration
- marker revisit
- GCS command UI
- RC bring-up 자체
- Pixhawk parameter 자동 write
- battery/failsafe parameter 자동 설정

## 결론

`line_follow_node`의 자동 이륙, 짧은 직선 라인 추종, ArUco marker 접근,
3초 hover, 착륙까지의 MVP 실행 경로는 코드상 구현되어 있다.

RC 값을 무시하고 진행하는 우회 경로도 이미 있다. 실기체 Pixhawk1 실행 시
`--unsafe-assume-rc-present`를 붙이면 `runtime.pixhawk1.toml`의
`rc_required=true`가 런타임에서 `false`로 바뀌고, `waitRcReady()`가
MAVLink `RC_CHANNELS`를 기다리지 않는다.

다만 이 상태는 "비행 가능 코드 경로가 있다"는 의미이지, "실기체 비행
안정성이 확보됐다"는 의미는 아니다. RC 우회는 조종기 기반 kill/mode
takeover 확인을 소프트웨어 안전 조건에서 제거하므로 실제 첫 비행 안정성은
낮게 봐야 한다.

## RC 무시 실행 조건

실기체 자동 arm/takeoff 경로는 기본적으로 막혀 있다. Pixhawk1 serial target에서
명령을 실제로 보내려면 여전히 `--allow-arm-takeoff`가 필요하다.

RC 값을 무시하고 실제 mission 경로로 들어가는 명령 형태:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --vision rpicam --line-mode light_on_dark --video \
  --allow-arm-takeoff --unsafe-assume-rc-present
```

어두운 라인을 따라가야 하면 `--line-mode dark_on_light`를 사용한다.

이 옵션을 붙였을 때 바뀌는 것:

- RC preflight gate를 통과 처리한다.
- mission 중 RC missing/stale에 의한 LAND fallback을 비활성화한다.
- MAVLink adapter의 RC decode 코드는 남아 있지만 safety 판단에는 쓰지 않는다.

바뀌지 않는 것:

- Pixhawk heartbeat는 필요하다.
- real serial target에서는 optical-flow/range/EKF/local-position 기반 local
  hold estimate gate가 필요하다.
- mode가 `GUIDED`에서 벗어나면 operator takeover로 보고 companion velocity
  command를 중단한다.
- 자동 mission에는 `--allow-arm-takeoff`가 필요하다.
- line lost, mission timeout, marker timeout, heartbeat lost safety는 남아 있다.

## 구현된 범위

`line_follow_node`

- `--target sitl|pixhawk1`, `--vision fake|gazebo|rpicam`,
  `--line-mode auto|light_on_dark|dark_on_light`를 지원한다.
- Pixhawk1 serial target은 `runtime.pixhawk1.toml`의 serial device와 baudrate를
  사용한다.
- real serial target은 `--allow-arm-takeoff` 없이는 자동 arm/takeoff mission을
  거부한다.
- `--mavlink-smoke`, `--no-arm`, `--dry-run`은 mode/arm/takeoff/velocity 명령을
  보내지 않는 확인 경로다.
- `--unsafe-assume-rc-present`는 RC gate만 우회한다.

Vision

- `VisionProcessor`를 공유해 `vision_debug_node`와 `line_follow_node`가 같은
  detector 경로를 쓴다.
- frame source는 `fake`, `gazebo`, `rpicam`으로 분리되어 있다.
- line center offset, line angle, confidence가 controller 입력으로 들어간다.
- GCS telemetry/video 송신은 `line_follow_node --video`에서 가능하다.
- 비행 중에는 `vision_debug_node`와 `line_follow_node`를 동시에 GCS video sender로
  실행하지 않는 전제가 유지된다.

Mission state

구현된 상태 흐름:

```text
IDLE -> TAKEOFF -> LINE_FOLLOW -> MARKER_APPROACH
  -> MARKER_HOVER -> LAND -> COMPLETE
any state -> ABORT
```

주요 전환 조건:

- 목표 고도 도달 후 `LINE_FOLLOW`.
- line detected와 confidence가 충분하면 전진 추종.
- marker detected면 `MARKER_APPROACH`.
- marker center가 tolerance 안에 들어오면 `MARKER_HOVER`.
- marker hover 3초 후 LAND.
- line lost timeout, marker timeout, mission timeout, safety land면 LAND.
- heartbeat lost 또는 GUIDED 이탈이면 ABORT.

Control

- 기본 전진 속도는 `0.25 m/s`.
- line offset은 body-frame lateral velocity로, line angle과 offset은 yaw-rate로
  P 제어된다.
- lateral velocity clamp는 `0.35 m/s`, yaw-rate clamp는 `0.6 rad/s`.
- confidence가 낮거나 line이 없으면 전진 명령을 0으로 두고 altitude/local hold
  쪽으로 전환한다.
- altitude hold는 사용 가능한 altitude/range 값을 기준으로 `vz_down`을 제한한다.
- real serial target은 takeoff 전 local XY hold anchor를 잡고, takeoff settle과
  landing descent에서도 local hold를 사용하려 한다.

Safety

- heartbeat 미수신/timeout은 ABORT.
- `GUIDED` mode 이탈은 operator takeover로 보고 LAND도 velocity도 추가로 보내지
  않고 종료한다.
- line lost timeout과 mission timeout은 LAND.
- RC 우회 옵션이 없으면 Pixhawk1 profile은 fresh `RC_CHANNELS`가 필요하다.
- RC 우회 옵션이 있으면 RC missing/stale safety는 작동하지 않는다.
- battery low, GCS link lost, Pixhawk parameter sanity는 현재 mission loop에서
  강제 safety gate로 구현되어 있지 않다.

## 안정성 판단

SITL/코드 수준 안정성:

- mission state machine, guided velocity controller, safety monitor, MAVLink RC
  decode에 unit coverage가 있다.
- Gazebo/SITL line-follow smoke 통과 기록이 문서에 남아 있다.
- 이번 문서 정리 과정에서는 build/test/SITL을 새로 실행하지 않았다.

실기체 안정성:

- RC를 무시하고 진행할 경우 첫 실비행 안정성은 아직 검증되지 않았다.
- `line_follow_node`는 속도와 yaw-rate를 보수적으로 clamp하지만, 실제 기체 tune,
  motor order/direction, prop direction, optical-flow drift, camera exposure,
  line false-positive에는 별도 실기체 검증이 필요하다.
- local hold estimate gate가 남아 있으므로 optical flow/range/EKF가 불안정하면
  이륙 전 중단된다. 이 gate는 유지해야 한다.
- 착륙은 local XY anchor가 있으면 guided descent를 시도하고, 불가능하면 LAND mode
  fallback을 사용한다. 실제 지면 근접/안정 판단은 range와 local velocity에 의존한다.
- RC 우회 시 조종기 입력이 보이지 않아도 mission은 진행하므로, 조종기 기반 즉시
  개입을 안전 근거로 삼을 수 없다. mode 변경을 다른 경로로 넣으면 ABORT는 되지만,
  이것은 RC takeover 검증을 대체하지 못한다.

현재 판단:

```text
SITL/벤치 코드 경로: 구현됨
실기체 no-arm 확인: 필요
props-off motor/command 확인: 필요
manual hover/tune 확인: 필요
RC 우회 실비행 안정성: 미검증, 낮음
```

## RC 우회로 진행하기 전 최소 gate

RC gate는 생략하더라도 아래 gate는 생략하지 않는다.

```bash
./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 12000 --strict-local-estimate

./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

Vision-only line 확인:

```bash
./build/vision_debug_node --config config \
  --line-only --line-mode light_on_dark --video
```

Props-off motor 확인:

```bash
./build/mavlink_motor_test --config config --target pixhawk1 \
  --motor 1 --percent 5 --seconds 1 --props-removed
```

Motor 2, 3, 4도 같은 방식으로 확인한다.

실제 자동 mission은 위 확인, battery/failsafe 판단, manual hover, GCS video/telemetry
확인이 끝난 뒤에만 시도한다.

## 남은 리스크

- RC 우회는 의도적으로 RC safety layer를 제거한다.
- battery monitor/failsafe와 arming check 상태는 실제 Pixhawk에서 다시 확인해야 한다.
- line detector는 조명, 바닥 재질, 라인 색에 민감하므로 `--line-mode`와 threshold
  tuning이 필요할 수 있다.
- marker approach는 marker가 보일 때는 marker centering을 하고, marker가 사라져도
  line이 보이면 line-follow fallback으로 계속 움직일 수 있다.
- local estimate가 흔들리면 takeoff hold, low-confidence hold, guided landing이
  모두 흔들릴 수 있다.
- GCS video는 관찰 보조일 뿐 mission-critical safety가 아니다.

## 다음 작업

1. Raspberry Pi/Pixhawk에서 `--strict-local-estimate`와 `--mavlink-smoke` 재확인.
2. `vision_debug_node --line-only --video`로 실제 라인 색상과 overlay 안정성 확인.
3. props-off motor order/direction과 LAND/disarm command path 확인.
4. manual hover로 optical-flow/range 기반 hold 안정성 확인.
5. RC 우회를 계속 쓸지, 최소한 mode/kill을 보장할 별도 operator interrupt를 둘지
   결정.
