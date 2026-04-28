# 라인 트레이싱 정확도 및 관제 영상 프레임 개선 계획

작성일: 2026-04-28
상태: v3 Pi camera/GCS 캡처 결과 반영. 아직 코드 수정 전 계획.

## 1. 현재 진행상황 요약

현재 구조는 다음과 같다.

```text
Pi camera
  -> rpicam-vid MJPEG capture
  -> onboard JPEG decode
  -> onboard ArUcoDetector
  -> onboard LineDetector
  -> onboard LineStabilizer
  -> UDP telemetry JSON
  -> UDP MJPEG debug video
  -> GCS frame sync
  -> GCS marker/line overlay
  -> GCS vision log
```

중요 파일:

| 저장소 | 파일 | 역할 |
|---|---|---|
| `uav-onboard` | `src/app/VisionDebugPipeline.cpp` | camera read, JPEG decode, ArUco/line 실행, telemetry/video 송신 |
| `uav-onboard` | `src/vision/LineDetector.cpp` | local contrast mask, contour 후보, lookahead band projection |
| `uav-onboard` | `src/vision/LineStabilizer.cpp` | EMA, hold, jump rejection, velocity limit |
| `uav-onboard` | `src/video/UdpMjpegStreamer.cpp` | JPEG frame을 UDP chunk로 송신 |
| `uav-onboard` | `src/camera/RpicamMjpegSource.cpp` | `rpicam-vid --codec mjpeg` stdout frame source |
| `uav-gcs` | `src/app/VisionDebugApp.cpp` | UDP video 수신, telemetry matching, overlay, log window |
| `uav-gcs` | `src/video/JpegFrameReassembler.cpp` | UDP chunk를 JPEG frame으로 재조립 |
| `uav-gcs` | `src/ui/VideoWindow.cpp` | JPEG decode, overlay drawing, OpenCV window 표시 |

현재 line 기본값:

```toml
[line]
mode = "light_on_dark"
mask_strategy = "local_contrast"
lookahead_band_ratio = 0.06
local_contrast_blur = 31
local_contrast_threshold = 12
confidence_min = 0.25
process_width = 480
filter_max_offset_velocity_ratio = 0.08
```

v3 테스트 결론:

- 이전보다 라인 탐지 정확도는 확실히 좋아졌다.
- 약 2m 고도에서도 검은 천 위의 흰 라인은 잡힌다.
- iPad 격자선의 십자 모양도 잡힌다.
- 흰색 테두리 진열장의 L자 형태도 잡힌다.
- 그러나 방 바닥 + 휴지 조합은 여전히 실패한다.
- 검은 천 위 흰 라인은 잡히지만 contour가 두 갈래 edge처럼 갈라지거나 라인 절반만 얇게 잡히는 문제가 있다.
- GCS latency 표시는 낮아졌지만 video FPS/drop이 심해 관제용으로 부족하다.
- Pi Zero 2 W 발열이 심해졌고, 추후 mission logic + Pixhawk 제어까지 올릴 때 여유가 걱정된다.

## 2. v3 이미지별 해석

| 이미지 | 조건 | 관찰 | 판단 |
|---|---|---|---|
| `v3_line1.png` | 방 바닥 + 휴지, 약 2m | 라인 미검출 | 휴지/마루 재질의 반사와 low contrast가 local contrast 기준에서도 부족한 것으로 보임 |
| `v3_line2_1.png` | 검은 천 + 휴지, 약 2m | 검출 성공, contour가 양쪽 edge처럼 갈라짐 | mask가 라인 내부를 채우지 못하고 밝기 경계만 잡는 상태 |
| `v3_line2_2.png` | 검은 천 + 휴지, 약 2m | 검출 성공, 라인의 한쪽/절반만 얇게 잡힘 | threshold 또는 morphology가 line interior를 충분히 연결하지 못함 |
| `v3_line3.png` | iPad 가상 격자, 약 50cm | 십자 격자 검출 성공 | 교차점 형태를 지나치게 reject하지 않는 현재 방향은 맞음 |
| `v3_line4.png` | 흰 테두리 진열장, 약 1.4m | L자 테두리 검출 성공 | line-like 고대비 구조는 잘 잡지만, 실제 임무에서는 카메라가 바닥만 보도록 ROI/자세 제한 필요 |

중요한 해석:

- `v3_line2`의 갈라짐은 tracking 실패가 아니라 segmentation 후처리 부족에 가깝다.
- `v3_line1`은 알고리즘만의 문제가 아니라 테스트 material과 바닥 반사 조건이 실제 경기장과 다를 가능성이 크다.
- `v3_line3`, `v3_line4`가 잘 잡히는 것은 교차점/L자 확장 가능성에는 좋은 신호지만, 동시에 아무 흰 테두리나 라인으로 볼 수 있다는 의미이기도 하다.

## 3. 현재 최우선 목표

이번 다음 작업의 우선순위는 두 가지다.

1. 라인 트레이싱 정확도
   - 2m 고도에서 10cm 라인을 안정적으로 잡는다.
   - 라인이 두 갈래 edge로 갈라지지 않고 하나의 두께 있는 라인으로 연결되게 한다.
   - 흰 라인과 검정 라인 모두 대응 가능한 구조를 준비한다.
   - 십자, T자, L자 교차점 가능성은 보존한다.

2. 관제 영상 프레임
   - 단순히 latency 숫자가 낮은 것보다 안정적인 complete frame 수신이 중요하다.
   - GCS 영상이 한 자릿수 FPS로 떨어지면 관제용 debug channel로도 부족하다.
   - 영상 송출은 mission-critical이 아니므로, 필요하면 안정적인 10~15FPS로 제한하고 오래된 frame은 버린다.

## 4. 라인 트레이싱 개선 방향

### 4.1 두 갈래 edge 문제 해결

현재 `local_contrast`는 라인 내부 전체보다 라인과 배경의 경계를 강하게 잡는 경향이 있다. 그래서 검은 천 위 흰 휴지가 두 개의 parallel edge처럼 보일 수 있다.

해결 방향:

1. morphology를 open/close로 분리한다.
   - 현재 `morph_kernel` 하나로 open 후 close를 수행한다.
   - local contrast에서는 open이 얇은 라인 내부를 더 깎을 수 있다.
   - `morph_open_kernel`은 작게 또는 1로 두고, `morph_close_kernel`을 더 크게 둬 갈라진 edge를 채운다.

2. local contrast mask 후 close/dilate를 조금 더 적극적으로 적용한다.
   - 목표는 라인 edge 두 개를 하나의 connected component로 묶는 것이다.
   - 단, 너무 키우면 iPad 격자나 교차점 주변이 과하게 두꺼워지므로 config로 조절한다.

3. lookahead projection에서는 가까운 parallel runs를 하나의 line run으로 merge한다.
   - contour overlay가 완전히 예쁘지 않아도 tracking point는 line center에 있어야 한다.
   - 현재도 projection 기반이라 중심점은 어느 정도 맞지만, 갈라진 contour가 confidence/angle을 흔들 수 있다.

추천 초기 설정 후보:

```toml
[line]
morph_open_kernel = 1
morph_close_kernel = 7
local_contrast_threshold = 10
line_run_merge_gap_px = 12
```

주의:

- `local_contrast_threshold`를 낮추면 `v3_line1` 같은 low contrast 조건에는 유리하지만 바닥 반사 false positive가 늘 수 있다.
- 따라서 threshold 하향은 morphology close와 함께 단계적으로 비교해야 한다.

### 4.2 방 바닥 + 휴지 테스트 판단

`v3_line1`은 현재 포기해도 되는지 고민할 만하다. 판단은 다음과 같다.

- 휴지는 표면 질감이 불규칙하고, 카메라 고도 2m에서는 라인 edge가 매우 약해진다.
- 밝은 마루는 반사와 무늬가 있고, 흰 휴지와 local brightness 차이가 작다.
- 실제 경기장은 흙바닥 + 흰 테이프 또는 송진가루일 가능성이 높고, 마루 + 휴지보다 line/background contrast가 다를 수 있다.

결론:

- `v3_line1`을 성공 기준의 핵심 benchmark로 두면 과튜닝 위험이 있다.
- 그러나 완전히 버리지는 말고 hard case로 유지한다.
- 다음부터는 실제 경기장에 가까운 테스트 material을 우선한다.

권장 테스트 material:

- 흙색 종이/보드/매트 위 흰색 무광 테이프
- 실제 운동장 흙 위 흰 테이프 또는 흰 가루
- 검은 테이프/검은 라인도 같은 배경에서 별도 테스트
- 햇빛, 그늘, 흐림 조건별 샘플

### 4.3 검정 라인 가능성 대응

현재 구조는 이미 `mode = "dark_on_light"`를 지원한다. local contrast도 밝은 라인일 때는 `gray - local_background`, 어두운 라인일 때는 `local_background - gray` 방식으로 확장 가능하다.

추천 방향:

1. 경기 전 line polarity를 명시적으로 선택할 수 있게 한다.
   - 흰 라인: `--line-mode light_on_dark`
   - 검정 라인: `--line-mode dark_on_light`
   - 미확정/탐색: `--line-mode auto`

2. `auto`는 항상 최종 모드로 쓰기보다 탐색/초기화용으로 쓴다.
   - light/dark mask를 모두 계산하면 CPU가 증가하고 false positive도 늘 수 있다.
   - tracking이 안정되면 선택된 polarity를 몇 초간 lock하는 방식이 좋다.

3. 색상 이름으로 `green`, `blue`를 직접 입력하는 방식은 아직 우선순위가 낮다.
   - 경기 라인이 흰색/검정색이면 brightness polarity가 더 안정적이다.
   - 실제 라인이 컬러로 확정되면 HSV color gate를 추가한다.

추천 실행 정책:

```text
대회 전 라인 색 확인 가능:
  light_on_dark 또는 dark_on_light 고정

대회 전 라인 색 미확정:
  auto로 탐색
  confidence가 일정 시간 유지되면 해당 polarity lock
  lock 이후에는 반대 polarity 계산 생략
```

### 4.4 교차점 가능성 보존

`v3_line3`의 십자와 `v3_line4`의 L자 검출은 장점이다. 따라서 다음 튜닝에서 너무 보수적으로 “일직선만 인정”하면 안 된다.

정책:

- line tracking point는 lookahead band에서 안정적으로 계산한다.
- contour overlay는 교차점 branch를 보존한다.
- intersection 판단은 별도 `IntersectionCandidate` 경로로 구현한다.
- tracking confidence와 intersection confidence를 섞지 않는다.

다음 단계 구조:

```text
LineDetector
  -> tracking mask/result: line center, offset, confidence
  -> debug contour: GCS overlay
  -> future intersection candidate: +/T/L branch analysis
```

## 5. 관제 영상 프레임 드랍 분석

현재 screenshots에는 `latency -60ms ~ -92ms`처럼 음수도 보인다. 이는 실제 latency가 음수라는 뜻이 아니라 Pi와 GCS 노트북의 clock offset이 맞지 않는다는 신호다. 따라서 지금은 latency 숫자만으로 개선 여부를 판단하면 안 된다.

다음 지표가 더 중요하다.

- GCS complete video FPS
- UDP packet 수신률
- incomplete JPEG frame drop 수
- onboard `video_sent_frames`
- onboard `video_dropped_frames`
- JPEG byte size
- chunk count per frame
- GCS JPEG decode/display 시간
- Pi `read_frame_ms`, `jpeg_decode_ms`, `aruco_latency_ms`, `line_latency_ms`, `processing_latency_ms`
- Pi CPU 온도와 throttling 상태

### 5.1 가능한 원인

현재 video path:

```text
rpicam-vid MJPEG frame
  -> onboard keeps original JPEG bytes
  -> UdpMjpegStreamer splits JPEG into 1200-byte chunks
  -> sends all chunks in a tight loop
  -> GCS receives chunks
  -> JpegFrameReassembler completes only if all chunks arrive
  -> VideoWindow decodes JPEG and draws overlay
```

프레임 드랍의 가능성이 큰 원인:

1. UDP chunk 손실
   - JPEG 한 장이 20~60KB이면 1200-byte UDP chunk가 수십 개 필요하다.
   - chunk 하나만 빠져도 GCS는 그 frame을 완성하지 못한다.

2. burst 송신
   - onboard가 한 frame의 chunk를 아주 빠르게 몰아서 보내면 Wi-Fi/OS buffer에서 손실이 생길 수 있다.

3. GCS 수신과 display가 같은 loop에 있음
   - `VisionDebugApp`이 frame 수신, JPEG decode, overlay, window draw, log update를 한 thread loop에서 처리한다.
   - 화면 draw가 늦으면 UDP socket buffer를 빨리 비우지 못해 packet loss가 늘 수 있다.

4. Pi CPU/발열
   - Pi는 camera MJPEG read, JPEG decode, ArUco, line, JSON, UDP chunk send를 동시에 한다.
   - 발열로 throttling이 생기면 처리 주기가 흔들리고 video worker가 밀릴 수 있다.

5. debug video FPS가 실제 네트워크가 감당하는 FPS보다 높음
   - 현재 `video.fps = 15`, `640x480`, `jpeg_quality = 50`.
   - complete frame 기준으로 안정적인 10FPS가 15FPS보다 관제에는 더 나을 수 있다.

## 6. 관제 영상 개선 방향

### 6.1 먼저 계측 추가

코드 수정 첫 단계는 최적화가 아니라 정확한 계측이어야 한다.

추가할 지표:

Onboard:

- `video_chunk_count`
- `video_send_ms`
- `video_frame_skip_count`
- `camera_jpeg_bytes`
- `effective_video_fps`
- 가능하면 Pi temperature/throttled 상태

GCS:

- UDP packets received
- malformed packets
- completed JPEG frames
- incomplete/dropped frame count
- average chunk count
- JPEG decode ms
- overlay draw/display ms
- actual displayed FPS
- frame inter-arrival ms

이 지표가 있어야 frame drop이 네트워크 손실인지, GCS display 병목인지, Pi 처리 병목인지 구분할 수 있다.

### 6.2 GCS video receive thread 분리

현재 GCS는 수신과 표시가 같은 loop에 가깝다. 다음 구조가 더 안전하다.

```text
VideoReceiveThread
  -> UDP socket을 계속 drain
  -> complete JPEG frame만 latest buffer에 저장

UiThread
  -> latest complete frame만 가져와 decode/display
  -> overlay/log 표시
```

효과:

- OpenCV `imshow`, `waitKey`, log window update가 UDP 수신을 막지 않는다.
- socket buffer overflow 가능성이 줄어든다.
- 오래된 frame은 표시하지 않고 최신 complete frame만 표시할 수 있다.

### 6.3 video 송신 rate 제한

모든 vision frame을 video로 보낼 필요는 없다. 관제 영상은 안정적인 FPS가 중요하다.

추천 옵션:

```toml
[video]
width = 640
height = 480
fps = 12
jpeg_quality = 45

[debug_video]
send_fps = 10
chunk_pacing_us = 150
```

방향:

- detector는 필요한 주기로 유지한다.
- video는 10FPS 또는 12FPS로 decimate한다.
- chunk 사이에 짧은 pacing을 넣어 Wi-Fi burst loss를 줄인다.
- complete FPS가 좋아지는지 확인한 뒤 pacing 값을 줄이거나 제거한다.

### 6.4 JPEG 크기 줄이기

우선순위는 다음과 같다.

1. `jpeg_quality 50 -> 45 -> 40` 비교
2. `video.fps 15 -> 12 -> 10` 비교
3. 필요 시 `640x480 -> 480x360` 비교

주의:

- source resolution을 낮추면 line detection도 같은 낮은 frame에서 수행되므로 고도 2m 인식이 나빠질 수 있다.
- Pi에서 별도 저해상도 video를 재인코딩해서 보내는 방식은 CPU를 더 쓸 수 있으므로 당장 우선순위가 낮다.
- 먼저 JPEG quality/FPS/송신 pacing/GCS receive thread로 해결을 시도한다.

### 6.5 latency 표시 보정

현재 음수 latency가 보이므로 clock sync 또는 latency 표시 방식 개선이 필요하다.

선택지:

1. Pi와 노트북 시간을 NTP로 맞춘다.
2. GCS에서 첫 frame 기준 clock offset을 추정해 보정 latency를 표시한다.
3. 절대 latency와 별도로 local inter-arrival FPS를 표시한다.

관제 품질 판단에는 `displayed FPS`, `complete FPS`, `dropped/incomplete frames`를 반드시 같이 봐야 한다.

## 7. Pi Zero 2 W와 Pi Camera 계속 사용 가능성

결론부터 말하면, “라인 트레이싱 + 미션 판단 + Pixhawk 명령” 자체는 Pi Zero 2 W에서도 가능성이 있다. 그러나 “ArUco + line + 고품질 관제 영상 + 지속적인 overlay/log + 발열 없는 장시간 운용”까지 모두 동시에 요구하면 여유가 매우 작다.

### 7.1 가능성이 있는 이유

- MAVLink/Pixhawk 명령 송신 자체는 CPU 비용이 작다.
- mission logic도 영상 처리에 비하면 가볍다.
- 실제 경기 중에는 GCS 영상이 mission-critical이 아니므로 영상 FPS/품질을 낮추거나 끌 수 있다.
- line detector는 ROI/process width 기반으로 제한할 수 있다.

### 7.2 위험한 이유

- Pi Zero 2 W는 CPU와 열 여유가 작다.
- 현재 이미 발열이 심하다면 thermal throttling으로 FPS와 latency가 흔들릴 수 있다.
- ArUco와 line을 매 frame 모두 수행하면 CPU 사용량이 커진다.
- UDP MJPEG 송신은 CPU보다 네트워크/packet loss 문제가 크지만, 전체 시스템 부하를 키운다.
- 야외 햇빛에서는 camera exposure/gain 변화와 motion blur가 추가된다.

### 7.3 권장 운영 정책

개발/테스트:

- fan 또는 heatsink를 반드시 장착한다.
- `vcgencmd measure_temp`
- `vcgencmd get_throttled`
- `top` 또는 `htop`
- GCS FPS/drop log를 함께 기록한다.

실제 미션:

- line tracking은 우선 유지한다.
- ArUco는 marker 탐색 구간 또는 낮은 주기로만 돌린다.
- GCS video는 5~10FPS debug용으로 제한하거나 필요 시 끈다.
- Pixhawk 제어 loop는 video worker와 분리하고, video가 밀려도 제어가 막히지 않게 한다.

하드웨어 판단:

- Pi Zero 2 W로 계속 진행은 가능하지만 냉각과 영상 decimation이 전제다.
- 20FPS 이상 관제 영상과 ArUco+line 동시 처리, 장시간 야외 운용까지 안정적으로 원하면 Raspberry Pi 4/5 또는 CM4급 보드가 더 안전하다.
- 대회 규정, 무게, 전력, 장착 공간이 허용하면 상위 Pi로 전환하는 것이 리스크를 크게 줄인다.

## 8. 다음 코드 작업 우선순위

### 1단계: 계측 강화

목표:

- frame drop의 원인이 onboard, network, GCS 중 어디인지 분리한다.

작업:

- onboard telemetry debug에 video chunk/send 지표 추가
- GCS video receiver에 completed/incomplete/drop/FPS 지표 추가
- GCS log에 actual displayed FPS 표시
- latency 표시에는 clock offset 주의 문구 또는 보정 추가

완료 기준:

- `video_sent_frames`, `video_dropped_frames`, `completed_frames`, `incomplete_frames`, `display_fps`를 동시에 볼 수 있다.

### 2단계: 라인 mask 후처리 개선

목표:

- `v3_line2_1`, `v3_line2_2`에서 라인이 두 갈래로 갈라지지 않고 하나의 두께 있는 contour로 잡히게 한다.

작업:

- `morph_open_kernel`, `morph_close_kernel` 분리
- local contrast 후 close/dilate 튜닝
- projection run merge gap 추가
- `v3_line2_*`, `v3_line3`, `v3_line4` 회귀 확인

완료 기준:

- 검은 천 위 흰 라인이 하나의 connected contour에 가깝게 보인다.
- green tracking point가 라인 중심에 유지된다.
- iPad 십자와 L자 contour가 과하게 사라지지 않는다.

### 3단계: 영상 FPS 개선

목표:

- GCS 관제 영상이 안정적으로 10~15 complete FPS를 유지하게 한다.

작업:

- GCS video receive thread 분리
- video send FPS decimation 추가
- JPEG quality/FPS matrix 테스트
- optional chunk pacing 추가

초기 테스트 matrix:

| config | 해상도 | quality | send FPS | 예상 |
|---|---:|---:|---:|---|
| A | 640x480 | 50 | 15 | 현재 기준 |
| B | 640x480 | 45 | 12 | 1차 권장 |
| C | 640x480 | 40 | 10 | packet loss 감소 우선 |
| D | 480x360 | 45 | 12 | line 정확도 영향 확인 필요 |

완료 기준:

- complete/display FPS가 10FPS 이상으로 안정적이다.
- latency 표시가 clock offset에 속지 않는다.
- line detection confidence가 해상도/quality 변경 후에도 유지된다.

### 4단계: 검정 라인 대응

목표:

- 흙바닥 + 흰 라인뿐 아니라 흙바닥 + 검정 라인도 준비한다.

작업:

- `dark_on_light` local contrast 테스트
- `auto` mode의 CPU 비용 측정
- light/dark polarity lock 설계
- 가능하면 GCS 또는 CLI에서 현재 polarity를 명확히 표시

완료 기준:

- 흰 라인 모드와 검정 라인 모드 모두 같은 테스트 matrix로 검증된다.
- auto mode를 쓸 때 false positive와 CPU 증가가 허용 범위인지 판단된다.

### 5단계: Pi 열/성능 한계 판단

목표:

- Pi Zero 2 W를 계속 쓸지, 상위 보드가 필요한지 근거를 만든다.

테스트 조합:

| 모드 | video | ArUco | line | 목적 |
|---|---|---|---|---|
| line-only no-video | off | off | on | 순수 line 비용 |
| line-only video | on | off | on | video가 line에 주는 영향 |
| aruco+line no-video | off | on | on | 순수 vision 비용 |
| aruco+line video | on | on | on | 현재 최악 조건 |
| mission-like | low FPS/on or off | low rate | on | 실제 경기 예상 조건 |

기록:

- 평균/최대 `line_latency_ms`
- 평균/최대 `aruco_latency_ms`
- 평균/최대 `processing_latency_ms`
- complete/display FPS
- Pi 온도
- throttling 여부
- CPU 사용률
- 5분 이상 연속 실행 안정성

판단 기준:

- throttling이 반복되면 냉각 또는 보드 업그레이드 필요
- mission-like 모드에서 line loop가 안정적이면 Pi Zero 2 W 유지 가능
- 관제 영상까지 15FPS 이상 요구하면 Pi Zero 2 W는 리스크가 큼

## 9. 이번 계획의 결론

현재 알고리즘 방향은 맞다. v3 결과에서 십자, L자, 원거리 흰 라인 검출이 개선된 것이 확인됐다. 다음 병목은 라인 내부를 채우는 mask 후처리와 관제 영상의 complete FPS다.

가장 먼저 해야 할 일은 계측 강화다. 지금은 latency 숫자가 음수로 나올 정도로 clock 기준이 불안정하고, frame drop 원인도 onboard/GCS/network 중 어디인지 분리되어 있지 않다. 그 다음 `v3_line2`의 두 갈래 contour를 morphology close와 projection run merge로 해결하고, 동시에 GCS video receive thread 분리와 video send FPS/quality 조절로 관제 영상을 안정화한다.

Pi Zero 2 W는 mission-critical 라인 추적과 Pixhawk 명령까지는 계속 시도할 수 있다. 다만 발열이 이미 심하므로 냉각과 video decimation은 필수이고, 고품질 관제 영상까지 동시에 요구하면 상위 Raspberry Pi로 전환하는 것이 더 안전하다.
