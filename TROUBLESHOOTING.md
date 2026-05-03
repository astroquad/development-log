# Astroquad 트러블슈팅 및 개발 판단 로그

최종 업데이트: 2026-05-03

범위: `uav-gcs`, `uav-onboard` bring-up 과정에서 실제로 발생한 문제, 원인 분석, 해결 방법, 설계 판단을 보고서용 개발로그로 정리한다. 현재 기본 장치는 Raspberry Pi 4 + IMX519-78이며, Raspberry Pi Zero 2 W 관련 내용은 이전 bring-up 단계의 이력으로 남긴다.

## 1. GCS가 온보드 telemetry를 수신하지 못함

### 증상

Raspberry Pi에서 `uav_onboard`를 실행하면 다음처럼 telemetry가 정상 송신되는 것처럼 보였다.

```text
sent TELEMETRY seq=1 timestamp_ms=...
sent TELEMETRY seq=2 timestamp_ms=...
...
```

하지만 Windows GCS에서는 다음 메시지만 반복되었다.

```text
telemetry timeout after 2000 ms
```

### 원인

초기 `uav-onboard/config/network.toml`의 GCS 목적지 IP가 고정 IP로 설정되어 있었다.

```toml
[gcs]
ip = "192.168.1.100"
```

실제 노트북 IP는 네트워크마다 바뀌기 때문에, Pi가 존재하지 않거나 잘못된 IP로 UDP telemetry를 보내고 있었다.

### 해결

기본 목적지를 local broadcast로 변경했다.

```toml
[gcs]
ip = "255.255.255.255"
telemetry_port = 14550
video_port = 5600
```

또한 UDP sender에서 broadcast 송신을 허용하도록 `SO_BROADCAST`를 설정했다.

### 결과

GCS를 먼저 실행하고 Pi에서 `uav_onboard --config config --count 10`을 실행하면 GCS에서 증가하는 `seq`를 가진 telemetry를 수신할 수 있게 되었다.

## 2. Windows에서 OpenCV가 없어 `uav_gcs_video`가 빌드되지 않음

### 증상

Windows 로컬 빌드에서 다음 CMake warning이 발생했다.

```text
OpenCV was not found; uav_gcs_video will not be built.
```

그 결과 영상 수신 실행 파일이 생성되지 않았다.

### 원인

개발 환경은 MinGW/Ninja/g++ 조합이었다. OpenCV가 설치되어 있지 않았고, Windows에서 OpenCV를 설치하더라도 Visual Studio용 prebuilt OpenCV와 MinGW ABI가 맞지 않을 수 있었다.

### 해결

GCS 영상 창 backend를 두 경로로 분리했다.

- OpenCV가 있으면 `src/ui/VideoWindow.cpp` 사용
- Windows에서 OpenCV가 없으면 `src/ui/VideoWindowWin32.cpp` 사용

Win32 fallback backend는 Windows Imaging Component(WIC)로 JPEG를 decode하고, Win32/GDI로 화면을 그린다.

### 결과

OpenCV가 없는 Windows 환경에서도 `uav_gcs_video.exe`와 `uav_gcs_vision_debug.exe`를 빌드할 수 있게 되었다.

## 3. Pi에서 `video_streamer --source rpicam`이 시작되지 않음

### 증상

Pi에서 카메라는 인식되었다.

```text
rpicam-hello --list-cameras
0 : ov5647 ...
```

하지만 `video_streamer` 실행 시 다음 오류가 발생했다.

```text
failed to open rpicam source: failed to start rpicam-vid
```

### 원인

`RpicamMjpegSource`가 Linux에서도 `popen(command, "rb")`를 사용하고 있었다. Windows `_popen`에서는 `"rb"`가 자연스럽지만, POSIX `popen()`은 `"r"` 또는 `"w"`만 보장한다.

Pi에서 간단한 확인 결과 `"rb"` 모드가 invalid argument로 실패하는 것을 확인했다.

### 해결

플랫폼별 pipe read mode를 분리했다.

- Windows: `"rb"`
- Linux/POSIX: `"r"`

또한 `rpicam-vid`의 stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리하고, `--verbose 0`을 추가했다.

### 결과

Pi에서 `rpicam-vid` 기반 MJPEG frame을 읽고 UDP로 송신할 수 있게 되었다.

```text
sent video frame id=1 bytes=14952
sent video frame id=2 bytes=15038
sent video frame id=3 bytes=14999
```

## 4. `--count` 테스트 종료 후 `rpicam-vid` abort 로그가 보임

### 증상

`video_streamer --source rpicam --count 5`는 지정된 frame 수를 정상 송신했지만, `rpicam-vid`가 다음과 비슷한 로그를 남겼다.

```text
Received signal 13
terminate called after throwing an instance of 'std::runtime_error'
what(): failed to write output bytes
Aborted
```

### 원인

`video_streamer`가 지정된 frame 수를 읽은 뒤 pipe를 닫으면, stdout으로 MJPEG를 계속 쓰던 `rpicam-vid`가 SIGPIPE를 받고 종료한다. 이는 송신 실패라기보다는 테스트 종료 방식의 부작용이다.

### 해결

사용자 콘솔이 혼란스럽지 않도록 `rpicam-vid` stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리했다.

### 결과

터미널에는 송신 결과 중심의 로그만 보이게 되었다. `/tmp/astroquad_rpicam_vid.log`에는 SIGPIPE 종료 기록이 남을 수 있지만, `video_streamer`가 정상 종료하고 `sent video frame ...`이 출력되면 정상 테스트로 본다.

## 5. Broadcast 영상 스트리밍이 불안정함

### 증상

노트북 IP를 직접 지정하면 영상 수신이 안정적이었다.

```bash
./build/video_streamer --source rpicam --config config --gcs-ip 172.20.10.4
```

하지만 `255.255.255.255` broadcast로 영상 frame 자체를 보내면 frame 누락이 많거나 영상이 불안정했다.

### 원인

640x480 MJPEG frame은 대략 14-31 KB 수준이고, UDP payload를 1200 byte로 제한했기 때문에 frame 하나가 여러 UDP packet으로 분할된다. 이 중 packet 하나만 유실되어도 해당 JPEG frame은 완성되지 않는다.

Wi-Fi 환경에서는 broadcast packet이 낮은 전송률로 처리되거나 손실될 수 있다.

실제 측정 결과:

| 목적지 | 송신 frame | 완성 수신 frame | 완성률 |
|---|---:|---:|---:|
| `255.255.255.255` broadcast | 10 | 4 | 40% |
| 노트북 IP unicast | 10 | 10 | 100% |

### 해결

영상 본문은 broadcast로 계속 보내지 않고, GCS discovery 후 unicast로 전송하도록 변경했다.

동작 방식:

1. GCS가 UDP `5601`로 `AQGCS1 video_port=5600` beacon을 broadcast한다.
2. 온보드 `video_streamer` 또는 `vision_debug_node`가 시작 시 3초간 beacon을 기다린다.
3. beacon을 받으면 송신자 IP를 GCS IP로 사용한다.
4. 실제 영상은 해당 IP로 unicast 송신한다.
5. discovery 실패 시 기존 broadcast 주소로 fallback한다.

### 결과

Pi에서 매번 노트북 IP를 직접 입력하지 않아도 자동으로 GCS IP를 찾고, 실제 영상은 unicast로 보내게 되었다.

```text
discovering GCS video receiver for 3000 ms...
discovered GCS video receiver at <laptop-ip>:5600
```

## 6. GCS 영상 창이 중간중간 검은색으로 깜빡임

### 증상

영상이 수신되기는 하지만 GCS 영상 창이 중간중간 검은 화면으로 깜빡였다.

### 원인

두 가지가 겹쳤다.

1. 일정 시간 complete frame을 받지 못하면 `showStatus("waiting for video stream...")`가 호출되어 기존 frame 대신 검은 status 화면이 표시되었다.
2. Windows fallback 창에서 GDI paint 과정 중 배경 erase와 frame redraw가 분리되어 flicker가 보일 수 있었다.

UDP 특성상 incomplete frame drop은 정상적으로 발생할 수 있으므로, frame 하나가 빠질 때마다 검은 화면으로 돌아가면 사용자 경험이 나빠진다.

### 해결

GCS 표시 정책을 변경했다.

- 첫 frame을 받기 전에는 waiting 화면을 표시한다.
- 한 번이라도 frame을 받은 뒤에는 timeout이 발생해도 마지막 complete frame을 유지한다.
- JPEG decode 실패 시 화면을 지우지 않고 stderr warning만 출력한다.
- Win32 fallback backend에서 memory DC에 먼저 그리고 `BitBlt`로 한 번에 복사하는 double buffering을 적용했다.
- `WM_ERASEBKGND`를 처리해 배경 erase로 인한 flicker를 줄였다.

### 결과

일시적인 UDP frame drop이 있어도 마지막 정상 frame이 유지되므로 검은색 깜빡임이 줄었다.

## 7. OpenCV `camera_preview`와 Pi CSI camera 경로 차이

### 증상

초기 OpenCV `VideoCapture` 기반 `camera_preview`에서 frame read가 실패했다.

```text
frame read failed at index 0: failed to read a non-empty camera frame
```

### 원인

Pi CSI camera는 Raspberry Pi OS/libcamera 환경에서 OpenCV V4L2 경로로 항상 안정적으로 열리는 것이 아니다. 반면 `rpicam-hello`, `rpicam-vid`, `rpicam-still`은 libcamera/rpicam 경로를 사용하므로 정상 동작했다.

### 해결

실시간 Pi camera 경로는 OpenCV `VideoCapture`가 아니라 `rpicam-vid --codec mjpeg -o -` stdout을 읽는 방식으로 확정했다.

### 결과

`RpicamMjpegSource` 기반 `video_streamer`와 `vision_debug_node`가 Pi camera frame을 안정적으로 읽는다.

## 8. ArUco detector 빌드 시 OpenCV enum 이름 차이 발생

### 증상

로컬에서는 OpenCV가 없어 해당 target이 skip되었지만, Pi에서 빌드할 때 다음 오류가 발생했다.

```text
error: 'PREDEFINED_DICTIONARY_NAME' in namespace 'cv::aruco' does not name a type
```

### 원인

Pi에 설치된 OpenCV 4.10.0 header에서는 ArUco dictionary enum type이 `cv::aruco::PredefinedDictionaryType`이었다. 코드에서는 다른 OpenCV 버전에서 쓰이는 이름인 `PREDEFINED_DICTIONARY_NAME`을 사용하고 있었다.

### 해결

`ArucoDetector.cpp`의 dictionary mapping type을 Pi OpenCV 4.10.0에 맞게 수정했다.

```cpp
cv::aruco::PredefinedDictionaryType dictionaryFromName(const std::string& name)
```

### 결과

Pi에서 `onboard_vision`, `aruco_detector_tester`, `vision_debug_node`가 모두 빌드되었다.

## 9. `uav_gcs_vision_debug.exe`가 packet을 받지 못함

### 증상

Pi는 GCS beacon을 정상 발견했다.

```text
discovered GCS video receiver at 192.168.0.69:5600
```

하지만 `uav_gcs_vision_debug` 콘솔에는 telemetry packet이 계속 0개로 표시되었다.

```text
[marker] no telemetry packets yet packets=0 dropped=0
```

### 원인

Windows Defender Firewall에 `uav_gcs_vision_debug.exe` inbound rule이 생성되어 있었지만, Action이 `Block`이었다. 기존 `uav_gcs.exe`, `uav_gcs_video.exe`는 Allow였기 때문에 새 실행 파일만 막힌 상태였다.

### 해결

관리자 권한에서 `uav_gcs_vision_debug.exe` inbound UDP rule을 Allow로 변경해야 한다.

예시:

```powershell
Set-NetFirewallRule -DisplayName 'uav_gcs_vision_debug.exe' -Direction Inbound -Action Allow -Profile Private,Public
```

또는 Windows 보안 앱에서 해당 실행 파일의 Public/Private network 접근을 허용한다.

### 결과

방화벽이 허용된 실행 파일 경로로 테스트했을 때 GCS가 Pi에서 보낸 telemetry 20개를 모두 수신했다.

## 10. ArUco 오버레이를 온보드에서 그릴지 GCS에서 그릴지에 대한 설계 판단

### 고민 배경

ArUco marker 인식 후 결과를 영상 위에 표시하는 방법은 두 가지가 있었다.

1. 온보드에서 marker box, id, 방향 등을 영상에 직접 그린 뒤 그 결과 영상을 GCS로 송신한다.
2. 온보드는 marker id/corners/center/orientation만 계산해 telemetry로 보내고, GCS가 원본 영상 위에 overlay를 그린다.

온보드에서 바로 그려서 보내는 방식은 구현이 간단해 보였다. 특히 디버깅 단계에서는 `cv::aruco::drawDetectedMarkers`나 `cv::line`, `cv::putText`를 사용하면 빠르게 화면을 확인할 수 있다.

하지만 프로젝트 최종 목표와 Raspberry Pi Zero 2 W급 저사양 환경을 고려하면 단순 구현 편의성보다 역할 분리가 중요했다.

### 온보드에서 오버레이를 그리는 방식의 장점

- 구현이 직관적이다.
- GCS는 그냥 영상만 띄우면 된다.
- frame과 overlay가 영상 픽셀에 이미 합쳐져 있으므로 동기화 문제가 적어 보인다.
- 초기 데모에서는 빠르게 시각적 결과를 만들 수 있다.

### 온보드에서 오버레이를 그리는 방식의 단점

- 온보드 CPU/GPU 자원을 관제용 drawing에 사용한다.
- 영상에 그려진 overlay는 원본 영상 정보를 훼손한다.
- 실전에서는 overlay를 끄고 싶을 수 있는데, debug/competition 모드를 분리해야 한다.
- line tracing, intersection, marker map 등 mission-critical 계산과 관제용 drawing 코드가 섞일 위험이 있다.
- 나중에 GCS에서 overlay 스타일을 바꾸거나 log/GUI와 연동할 때 유연성이 떨어진다.
- Raspberry Pi Zero 2 W급 환경에서는 불필요한 drawing과 JPEG 재처리가 mission loop에 부담이 될 수 있다.

### GCS에서 오버레이를 그리는 방식의 장점

- 온보드는 인식과 판단에만 집중한다.
- 원본 카메라 영상을 그대로 보존해 GCS로 보낼 수 있다.
- 오버레이 스타일, 색상, 표시 항목, GUI 구성을 GCS에서 자유롭게 바꿀 수 있다.
- 최종 목표인 영상 창, 로그 창, 명령 창 구조와 잘 맞는다.
- ArUco뿐 아니라 line contour, intersection, grid state overlay를 같은 방식으로 확장할 수 있다.
- 온보드에서 관제용 GUI/drawing 연산을 하지 않는 원칙을 지킬 수 있다.

### GCS에서 오버레이를 그리는 방식의 단점

- video frame과 telemetry를 `frame_seq` 기준으로 맞춰야 한다.
- GCS 쪽에 overlay drawing backend가 필요하다.
- Windows에서 OpenCV가 없으므로 Win32/GDI overlay 구현이 추가로 필요하다.

### 최종 판단

GCS에서만 오버레이를 그리는 방식으로 결정했다.

결정 이유:

- 온보드의 mission-critical 비전 처리와 관제용 시각화를 분리해야 한다.
- 원본 영상은 보존하고, overlay는 GCS에서 동적으로 그리는 편이 확장성이 높다.
- Raspberry Pi Zero 2 W급 사양을 고려하면, 관제용 drawing은 노트북 GCS가 담당하는 것이 낫다.
- 최종 GCS 목표가 영상 창, 로그 창, 명령 창으로 분리되는 구조이므로 GCS overlay가 자연스럽다.
- line tracing, intersection detection, marker map, grid state도 모두 metadata telemetry + GCS overlay 구조로 확장할 수 있다.

### 구현 결과

온보드:

- `ArucoDetector`는 marker id, corners, center, orientation만 계산한다.
- `vision_debug_node`는 원본 JPEG frame과 marker telemetry를 송신한다.
- 온보드 코드에는 `drawDetectedMarkers`, `cv::line`, `cv::circle`, `cv::putText` 기반 marker overlay drawing이 없다.

GCS:

- `MarkerOverlay`가 marker telemetry를 overlay primitive로 변환한다.
- `VideoWindow`가 overlay primitive를 실제 영상 위에 그린다.
- OpenCV backend와 Win32/WIC backend 모두 overlay drawing을 지원한다.

### 보고서용 요약

초기에는 구현 편의성을 위해 온보드에서 ArUco marker overlay를 직접 그려 송신하는 방안도 검토했다. 그러나 최종 시스템에서는 온보드가 mission-critical 비전 판단에 집중해야 하고, GCS는 관제용 시각화와 로그를 담당해야 하므로 역할 분리를 우선했다. 이에 따라 온보드는 marker detection 결과만 telemetry로 전송하고, GCS가 원본 영상 위에 overlay를 그리는 구조로 설계했다.

## 11. 현재 권장 실행 순서

### ArUco vision debug

GCS:

```powershell
cd uav-gcs
git pull
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi, metadata-only 기본 실행:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config
```

이 기본 실행은 onboard 부담을 줄이기 위해 debug video를 켜지 않는다. GCS vision log에는 telemetry가 들어오지만 camera window는 `waiting for video stream...` 상태일 수 있다.

GCS camera window와 overlay까지 확인하려면 Pi 명령에 `--video`를 붙인다.

```bash
./build/vision_debug_node --config config --video
```

정상 동작 기준, metadata-only:

- Pi에서 `frame=N markers=M jpeg_bytes=...` 출력
- Pi startup line에 `video: off`, `telemetry: on` 출력
- GCS vision log에 증가하는 `frame`, `seq`, line/marker/system/camera log 출력
- GCS log의 video counters가 `video_sent=0`, `chunks_last=0`, `last_bytes=0`이면 video off 상태로 정상

정상 동작 기준, `--video`:

- Pi에서 `discovering GCS video receiver...` 이후 `discovered GCS video receiver at <laptop-ip>:5600` 또는 fallback 출력
- GCS에 raw camera 영상 창 표시
- ArUco marker가 보이면 GCS 영상 창에 GCS-side marker overlay 표시
- line이 검출되면 GCS 영상 창에 magenta contour와 green tracking point 표시

## 12. 다음에 비슷한 문제가 생겼을 때 확인할 것

1. GCS 실행 파일이 올바른 조합인지 확인한다.
   - telemetry만: `uav_gcs`
   - 영상만: `uav_gcs_video`
   - ArUco overlay: `uav_gcs_vision_debug`
2. Ninja generator 사용 시 실행 파일은 `build/Release/`가 아니라 `build/` 아래에 있다.
3. Pi camera가 `rpicam-hello --list-cameras`에서 보이는지 확인한다.
4. Metadata-only 실행이면 Pi에서 discovery가 출력되지 않는 것이 정상이다. `--video`를 켰을 때만 `discovered GCS video receiver...`를 확인한다.
5. discovery는 되는데 packet이 안 들어오면 Windows Firewall을 확인한다.
6. 영상이 불안정하면 `config/vision.toml`의 `debug_video.send_fps`, `camera.jpeg_quality`, 해상도를 낮춰본다.
7. rpicam 내부 오류는 Pi의 `/tmp/astroquad_rpicam_vid.log`를 확인한다.
8. ArUco overlay가 어긋나면 `camera.frame_seq`와 video `frame_id` 동기화를 먼저 의심한다.

## 13. 라인 contour 상단 절단, 낮은 tracking point, 밝은 바닥 반사 오검출

### 증상

실제 카메라 테스트 캡처 `line1.png`, `line2.png`, `line3.png`, `line4.png`에서 다음 현상을 확인했다.

- 어두운 조명 또는 검정 천 위에서는 흰색 라인이 비교적 잘 잡혔다.
- 밝은 조명이 바닥에 반사되는 환경에서는 흰색 라인뿐 아니라 반사광이 같은 밝은 contour로 붙으면서 magenta contour가 넓게 퍼졌다.
- 가상 격자선 이미지를 태블릿에 띄워 촬영한 경우 십자 형태는 잘 검출됐다.
- 모든 캡처에서 green tracking point가 화면 상하 기준 중점보다 아래쪽에 표시됐다.
- magenta contour가 하단까지는 내려오지만 상단은 잘린 것처럼 보였다.
- 카메라가 완전 탑다운에 가까울 때보다 약간 전방을 보도록 세웠을 때 라인이 더 잘 이어지는 느낌이 있었다.

### 원인

가장 큰 직접 원인은 line config 기본값이었다.

```toml
roi_top_ratio = 0.35
lookahead_y_ratio = 0.70
```

`roi_top_ratio = 0.35`는 영상 상단 35%를 라인 검출에서 제외한다. 따라서 실제 라인이 영상 상단까지 보여도 detector는 그 영역을 보지 못하고, GCS overlay의 contour도 ROI 시작 지점에서 잘린 것처럼 보인다.

`lookahead_y_ratio = 0.70`은 green tracking point를 의도적으로 화면 아래쪽 70% 위치에 둔다. 그래서 정상 검출이어도 tracking point가 중앙보다 낮게 보인다.

밝은 바닥 반사 문제는 별도 원인이 있다. 현재 라인 검출은 onboard 부담을 줄이기 위해 grayscale/local-contrast threshold 기반으로 동작한다. 흰색 라인과 밝은 반사광이 비슷한 밝기로 붙으면 OpenCV contour 단계에서 하나의 큰 component처럼 합쳐질 수 있다. 이 경우 line 자체보다 반사광이 magenta contour에 포함된다.

카메라 각도 문제는 설정값과 시야 특성이 겹친 결과다. 기존 ROI가 하단 위주였기 때문에, 카메라를 조금 전방으로 세워 라인이 하단 ROI를 길게 통과할 때 더 잘 잡히는 것처럼 보였다. 최종 주행에서는 카메라 장착 각도를 고정한 뒤 그 각도에서 `roi_top_ratio`와 `lookahead_y_ratio`를 맞추는 것이 중요하다.

### 적용한 수정

기본 ROI와 tracking point 위치를 다음처럼 변경했다.

```toml
roi_top_ratio = 0.08
lookahead_y_ratio = 0.55
```

효과:

- 영상 상단 대부분을 검출에 포함하므로 contour 상단 절단이 크게 줄어든다.
- green tracking point가 화면 중앙에 가까워져 현재 캡처에서 보였던 하단 치우침이 완화된다.
- 너무 먼 상단 0%부터 모두 쓰지는 않아, 카메라가 살짝 전방을 볼 때 생길 수 있는 비바닥 영역의 영향을 조금 줄인다.

밝은 반사광 대응으로 `LineDetector`에 line-branch filtering을 추가했다.

동작 방식:

1. 기존처럼 threshold mask와 contour 후보를 만든다.
2. `lookahead_y_ratio` 위치의 행에서 정상적인 라인 폭 후보만 통과시킨다.
3. 그 행의 tracking x를 기준으로 위/아래 행을 따라가며 정상 라인 폭 run만 추적한다.
4. 너무 넓은 반사광, 화면 가장자리 큰 물체, 십자 교차부처럼 폭이 넓은 span은 selected line branch contour에서 제외한다.
5. GCS에는 이 selected branch contour를 `vision.line.contour_px`로 보내고 magenta overlay로 그린다.

이 방식은 교차점 판단 로직이 아니다. 이번 단계에서는 라인을 따라갈 branch만 안정적으로 시각화하는 것이 목적이다. 따라서 십자 교차부의 가로선 전체를 저장하거나 교차점 좌표를 계산하지 않는다.

실행 전/현장 튜닝을 위해 다음 옵션도 추가했다.

```bash
./build/vision_debug_node --config config --line-only --line-roi-top 0.08 --line-lookahead 0.55
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-threshold 180
./build/line_detector_tuner --config config --image test_data/images/line_sample.jpg --mode light_on_dark --threshold 180 --roi-top 0.08 --lookahead 0.55
```

### 자연광/운동장 바닥에서의 예상

실내 나무 바닥이나 태블릿 화면처럼 반사가 강한 표면은 밝은 라인 검출에 불리하다. 운동장 흙바닥은 일반적으로 정반사가 적기 때문에 `line3.png` 같은 넓은 하이라이트 문제는 줄어들 가능성이 크다.

다만 자연광 환경도 완전히 안전하지는 않다.

- 구름/햇빛 변화로 auto exposure가 흔들릴 수 있다.
- UAV 그림자나 사람 그림자가 라인 근처에 생길 수 있다.
- 라인 재질이 흰색 테이프처럼 반짝이면 특정 각도에서 glare가 생길 수 있다.
- 흙바닥 색과 라인 색의 명암 차이가 작으면 grayscale threshold만으로는 불안정할 수 있다.

현장에서는 먼저 `--line-mode auto`로 확인하고, 라인이 흰색/밝은 색이면 `--line-mode light_on_dark`, 검정/어두운 색이면 `--line-mode dark_on_light`로 고정해서 비교한다. 반사광이 계속 붙으면 `--line-threshold`를 올려 라인보다 어두운 하이라이트를 제외한다.

### 현장 테스트 순서

GCS:

```powershell
cd uav-gcs
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode auto
```

흰색 라인으로 확정되면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

상단이 아직 잘리면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-roi-top 0.03
```

green tracking point를 더 위에서 보고 싶으면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-lookahead 0.45
```

주행 제어용으로 너무 먼 곳을 보고 흔들리면:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-lookahead 0.60
```

ArUco와 라인을 동시에 확인하려면 `--line-only`를 빼고 실행한다.

```bash
./build/vision_debug_node --config config --line-mode light_on_dark
```

## 14. 라인 branch filtering 되돌림과 교차점 표시 우선 결정

### 증상

`roi_top_ratio = 0.08`, `lookahead_y_ratio = 0.55`로 바꾼 뒤 green tracking point 위치와 contour 상단 절단 문제는 개선됐다. 하지만 반사광 억제를 위해 추가했던 line-branch filtering이 라인 인식을 너무 보수적으로 만들었다.

특히 `line4.png`처럼 십자 형태가 한 덩어리로 보여야 하는 장면에서, detector가 교차부 전체 contour를 보내지 않고 하단/상단의 일직선 branch 위주로 잘라서 보냈다. 이후 단계에서 교차점 판단을 붙이려면 십자 모양이 하나의 연결 contour로 보이는 것이 더 유리하다.

### 판단

이번 단계에서는 반사광을 과하게 줄이는 것보다 라인과 교차점을 적극적으로 검출하는 쪽이 우선이다.

유지:

- `roi_top_ratio = 0.08`
- `lookahead_y_ratio = 0.55`
- `--line-roi-top`, `--line-lookahead` 실행 옵션
- `auto`, `light_on_dark`, `dark_on_light`, `--line-threshold` 튜닝 옵션

되돌림:

- tracking row 주변의 좁은 branch만 남기던 line-branch filtering
- 교차부나 가로선 span을 magenta contour에서 제외하던 동작

### 현재 동작

`vision.line.contour_px`는 다시 detector가 선택한 연결 contour 전체를 단순화해서 보낸다. 따라서 십자 교차부가 threshold mask에서 하나의 connected component로 잡히면 GCS magenta overlay도 십자 형태로 표시된다.

반사광이 다시 문제가 되면 우선 다음 순서로 조정한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-threshold 180
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --line-roi-top 0.05 --line-lookahead 0.55
```

반사광 억제와 교차점 contour 보존을 동시에 만족해야 하는 상황이 반복되면, 다음 단계에서 단순 branch trimming이 아니라 `intersection candidate`를 별도 telemetry로 보내는 방식이 맞다. 즉, 라인 주행용 branch와 교차점 인식용 connected contour를 분리해서 둘 다 보존해야 한다.

## 15. 현재까지 문제 발생과 해결 흐름 요약

### 통신/영상 bring-up 단계

| 단계 | 문제 | 원인 | 해결 | 현재 상태 |
|---|---|---|---|---|
| Telemetry 수신 실패 | GCS가 Pi telemetry를 받지 못함 | 온보드 GCS IP가 고정되어 실제 노트북 IP와 불일치 | 기본 목적지를 broadcast로 변경하고 `SO_BROADCAST` 적용 | `uav_onboard` -> `uav_gcs` telemetry 수신 가능 |
| Windows 영상 빌드 실패 | OpenCV가 없어 `uav_gcs_video`가 빌드되지 않음 | Windows MinGW 환경에 OpenCV가 없고 ABI 부담이 있음 | OpenCV backend와 Win32/WIC fallback backend 분리 | OpenCV 없이도 GCS video/vision debug 실행 파일 빌드 가능 |
| Pi camera 실행 실패 | `video_streamer --source rpicam` 시작 실패 | Linux `popen()`에 `"rb"` mode 사용 | Windows는 `"rb"`, POSIX는 `"r"`로 분기 | Pi에서 `rpicam-vid` stdout MJPEG 읽기 가능 |
| `rpicam-vid` abort 로그 | `--count` 종료 후 SIGPIPE성 abort 로그가 보임 | 테스트 종료 시 parent가 pipe를 닫아 `rpicam-vid` stdout write 실패 | stderr를 `/tmp/astroquad_rpicam_vid.log`로 분리 | 사용자 콘솔에는 정상 송신 결과 중심으로 표시 |
| Broadcast 영상 불안정 | 영상 frame 완성률이 낮음 | MJPEG frame이 여러 UDP packet으로 나뉘고 broadcast 손실이 큼 | GCS beacon discovery 후 영상은 unicast로 전송 | Pi가 GCS IP 자동 발견 후 안정적인 unicast 송신 |
| GCS 영상 깜빡임 | packet drop 때 검은 화면이 보임 | timeout 때 status 화면으로 되돌아가고 Win32 paint flicker가 있음 | 마지막 complete frame 유지, double buffering, `WM_ERASEBKGND` 처리 | 영상 drop이 있어도 화면 유지 |

### 비전/오버레이 단계

| 단계 | 문제 | 원인 | 해결 | 현재 상태 |
|---|---|---|---|---|
| Pi ArUco 빌드 실패 | OpenCV enum type compile error | Pi OpenCV 4.10 header의 enum 이름이 코드와 다름 | `cv::aruco::PredefinedDictionaryType` 사용 | Pi에서 ArUco target 빌드 가능 |
| Vision debug packet 미수신 | Pi는 GCS를 발견하지만 GCS packet count가 0 | Windows Defender Firewall이 새 exe inbound UDP를 Block | `uav_gcs_vision_debug.exe` inbound allow rule 필요 | 방화벽 허용 시 telemetry/video 수신 가능 |
| Overlay 위치/역할 결정 | 온보드에서 그릴지 GCS에서 그릴지 결정 필요 | Pi Zero 2 W에서 drawing/JPEG 재처리는 mission-critical 자원 낭비 | 온보드는 metadata만 보내고 GCS가 overlay drawing | ArUco/line 모두 GCS-side overlay 구조 |
| 라인 오검출 | 발/검은 물체가 라인으로 잡힘 | 기존 polarity/후보 선택이 큰 어두운 contour에 취약 | `mode=auto`, `light_on_dark`, `dark_on_light`, threshold override 추가 | 라인 색이 확정되지 않은 환경에 대응 가능 |
| 라인 상단 절단/초록점 낮음 | contour 상단이 잘리고 tracking point가 아래에 치우침 | `roi_top_ratio=0.35`, `lookahead_y_ratio=0.70` | `roi_top_ratio=0.08`, `lookahead_y_ratio=0.55`로 조정 | 상단 절단 완화, tracking point 중앙화 |
| 반사광 억제 시도 | 밝은 바닥 반사가 contour에 섞임 | 흰색 라인과 반사광이 grayscale threshold에서 붙음 | line-branch filtering을 실험적으로 추가 | 교차점 표시에 불리해 되돌림 |
| 십자 교차점 contour 분리 | 십자가 하단/상단 branch로 쪼개짐 | branch filtering이 넓은 교차부를 제외 | branch filtering 제거, connected contour 전체 전송 | `line4.png` 같은 십자 contour 표시 우선 |

### 현재 설계 판단

현재 최종 판단은 다음과 같다.

- 온보드는 camera capture, ArUco/line detection, telemetry generation, raw JPEG debug streaming만 수행한다.
- GCS는 video display, marker/line overlay, vision log window, packet stats 표시를 담당한다.
- GCS video는 best-effort debug channel이며, 향후 제어/이동 판단 경로를 막지 않아야 한다.
- 라인트레이싱 MVP에서는 교차점 좌표 저장이나 grid update를 아직 하지 않는다.
- 교차점 단계에서는 라인 주행용 branch와 교차점 판단용 connected contour를 분리하는 것이 맞다.

### 현재 권장 실행 조합

GCS:

```powershell
cd uav-gcs
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
.\build\uav_gcs_vision_debug.exe --config config
```

Raspberry Pi, 라인만:

```bash
cd ~/astroquad/uav-onboard
git pull --ff-only
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Raspberry Pi, ArUco와 라인 동시:

```bash
./build/vision_debug_node --config config --line-mode light_on_dark
```

문제 발생 시 우선순위:

1. GCS가 packet을 못 받으면 Windows Firewall을 확인한다.
2. 영상이 끊기면 GCS discovery/unicast 여부와 network 품질을 확인한다.
3. 라인 상단이 잘리면 `--line-roi-top`을 더 낮춘다.
4. tracking point를 조정하려면 `--line-lookahead`를 조정한다.
5. 흰색 라인은 `--line-mode light_on_dark`, 어두운 라인은 `--line-mode dark_on_light`로 고정해서 비교한다.
6. 반사광이 심하면 `--line-threshold`와 camera angle을 먼저 튜닝한다.

## 16. 1.3 테스트 이후 라인 contour 갈라짐, GCS 프레임 드랍, Pi 발열

### 문제 상황

1.3 테스트에서는 약 2m 고도에서도 검은 천 위 흰색 라인, 아이패드 격자, 흰색 진열장 테두리처럼 대비가 충분한 대상은 이전보다 잘 잡혔다. 특히 십자 교차와 L자 형태도 connected contour로 유지되는 방향이 확인됐다.

반대로 방 바닥 위 휴지 라인은 여전히 잘 잡히지 않았다. 원인은 휴지 표면과 밝은 목재 바닥의 명도 차이가 작고, 바닥 반사와 카메라 자동 노출 때문에 라인과 배경의 local contrast가 약해지는 것으로 판단한다. 이 실내 테스트는 실제 운동장 흙바닥 대비 조건을 완전히 대표하지 못하므로, 경기장과 비슷한 흙색/무광 배경에서 별도 검증해야 한다.

검은 천 위 흰색 라인은 검출되지만 contour가 내부 edge 중심으로 갈라지거나, 라인 폭 전체가 아니라 한쪽 edge만 얇게 잡히는 경우가 있었다. 또한 latency는 줄었지만 GCS 영상 displayed FPS가 낮아져 관제용 영상이 끊겨 보였고, Pi Zero 2 W 발열이 커졌다.

### 원인 판단

- 라인 갈라짐: local contrast 기반 마스크가 라인 전체 면보다 양쪽 edge를 먼저 잡고, morphology 연결이 부족하면 하나의 라인이 두 줄처럼 분리된다.
- 원거리/고고도 검출 저하: 10cm 라인이 2m 고도에서 차지하는 픽셀 폭이 작아지고 JPEG 압축, 노출, 렌즈 왜곡 영향이 커진다.
- GCS 프레임 드랍: onboard latency를 낮추기 위해 낮은 품질/낮은 send FPS로 조정했지만, GCS 쪽에서 UDP 수신과 화면 표시가 같은 흐름에 묶이면 packet drain이 늦어져 incomplete frame이 늘 수 있다.
- CPU/발열: Pi Zero 2 W에서 camera capture, ArUco, line detection, JPEG packet 송신을 동시에 수행하면 미션 제어 여유가 줄어든다. 영상은 반드시 debug/best-effort 채널로 유지해야 한다.

### 적용한 해결

- 라인 마스크 후처리를 `morph_open_kernel`, `morph_close_kernel`, `morph_dilate_kernel`로 분리했다. 기본값은 작은 open, 큰 close로 잡아 얇은 노이즈는 줄이고 라인 내부 edge가 갈라지는 현상을 줄인다.
- projection 기반 run merge에 `line_run_merge_gap_px`를 추가했다. 가까운 밝은 run은 하나의 라인 후보로 합치되, 지나치게 먼 외곽 노이즈는 합치지 않는다.
- onboard debug video는 `fps`, `jpeg_quality`, `send_fps`, `chunk_pacing_us`를 분리했다. 현재 기본값은 12FPS capture, debug video 5FPS send, camera JPEG quality 45, chunk pacing 150us다. 다만 `debug_video.enabled = false`라서 `--video`를 켜지 않으면 video worker는 동작하지 않는다.
- onboard telemetry에 `video_send_ms`, `video_chunk_count`, `video_chunks_sent`, `video_skipped_frames`, `cpu_temp_c`를 추가했다.
- GCS는 UDP 수신을 별도 background thread에서 계속 drain하고, UI는 최신 complete JPEG만 표시하도록 변경했다.
- GCS reassembler에 `completed`, `incomplete`, `old_packets`, `chunk_mismatch_resets`, `last_chunk_count`, `last_frame_bytes` 통계를 추가했다.

### 검증 방법

로컬에서는 다음 검증을 통과했다.

```powershell
cmake --build uav-onboard/build
cmake --build uav-onboard/build-tests
ctest --test-dir uav-onboard/build-tests --output-on-failure
cmake --build uav-gcs/build
cmake --build uav-gcs/build-tests
ctest --test-dir uav-gcs/build-tests --output-on-failure
```

OpenCV가 있는 로컬 빌드에서는 `line_detector_tuner`와 `vision_debug_node`도 별도 Release 빌드로 확인했다. 단, 1.3 이미지 기반 튜너 검증은 GCS overlay가 이미 들어간 screenshot을 사용한 smoke test이므로 실제 raw camera frame 성능을 완전히 대체하지는 않는다.

### Pi 실기 테스트 체크리스트

GCS log에서 다음 항목을 같이 기록한다.

- `processing_latency_ms`, `read_frame_ms`, `jpeg_decode_ms`, `aruco_latency_ms`, `line_latency_ms`
- `display_fps`, `completed`, `incomplete`, `malformed`, `old_packets`, `mismatch_resets`
- `video_send_ms`, `video_chunk_count`, `video_skipped_frames`, `video_dropped_frames`
- `cpu_temp_c`

Pi에서 발열 또는 throttling이 의심되면 다음을 같이 확인한다.

```bash
vcgencmd measure_temp
vcgencmd get_throttled
```

테스트 우선순위는 1) 실제 경기장과 비슷한 흙색 무광 배경, 2) 1.8-2.0m 고도, 3) 흰색 라인과 검정 라인 각각, 4) line-only와 ArUco+line 동시 실행 비교 순서다.

### 추가 판단

Pi Zero 2 W와 Pi Camera만으로 MVP 검증은 가능하지만, ArUco, line tracing, mission 판단, Pixhawk 제어까지 모두 안정적으로 돌리려면 여유가 작다. 현재 기본 장치는 Pi 4 + IMX519로 바뀌었지만 설계 판단은 그대로다. 실제 비행 단계에서는 GCS 영상 FPS보다 제어 루프와 telemetry 안정성을 우선하고, 필요하면 debug video를 끄거나 line-only 모드로 비행 테스트를 시작한다.

## 17. Raspberry Pi 4 + IMX519-78 전환 체크

### 증상

Pi 4로 교체한 뒤 카메라가 보이지 않거나, `vision_debug_node`가 `failed to start rpicam-vid` 또는 frame read 실패로 종료될 수 있다.

### 우선 확인

```bash
rpicam-hello --version
rpicam-hello --list-cameras
rpicam-still -t 1000 --nopreview -o test_data/images/imx519_smoke.jpg
rpicam-vid -t 5000 --nopreview --codec mjpeg --width 640 --height 480 --framerate 12 -o /tmp/imx519_test.mjpeg
```

### 판단

- `rpicam-hello --list-cameras`에 IMX519가 없으면 Astroquad 코드 문제가 아니라 OS/kernel/rpicam/IMX519 driver 또는 CSI cable 문제부터 확인한다.
- IMX519-78은 autofocus camera이므로 `autofocus_mode`, `lens_position`, `exposure` 설정이 line confidence에 직접 영향을 준다.
- 실제 미션에서는 continuous autofocus가 focus hunting을 만들 수 있으므로, `manual + lens_position`과 `auto`를 실제 고도에서 비교한다.
- Pi 4에서도 debug video는 best-effort다. GCS 영상이 끊겨도 onboard line/marker telemetry와 추후 MAVLink control loop가 우선이다.

### 적용된 1차 대응

- `config/vision.toml`에 Pi 4 + IMX519용 `[camera]` 설정을 추가했다.
- `RpicamMjpegSource`가 rpicam autofocus/focus/exposure/AWB/denoise/orientation 옵션을 받을 수 있게 확장됐다.
- telemetry v1.5에 `system.*`, `camera.*`, `debug.capture_fps`, `debug.processing_fps`, `debug.video_send_failures`를 추가했다.
- GCS는 새 system/camera/debug field를 log window에 표시한다.
- 현재 성능 우선 기본값은 `camera.width=960`, `camera.height=720`, `camera.fps=12`, `camera.jpeg_quality=45`, `debug_video.enabled=false`, `debug_video.send_fps=5`, `debug_video.chunk_pacing_us=150`이다.
- GCS video latency/age 표시는 제거했다. GCS camera overlay는 `frame N`만 표시한다.

## 18. GCS 영상 창이 `waiting for video stream...`에 머무르지만 vision log는 계속 갱신됨

### 증상

Windows에서 다음처럼 GCS vision debug를 실행했다.

```powershell
.\build\uav_gcs_vision_debug.exe --config config
```

Pi에서는 line tracing telemetry가 정상으로 송신됐다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark
```

Pi console에는 `frame=N line=yes ... video_sent=0 video_chunks=0`처럼 처리 결과가 계속 출력되고, GCS log window도 frame/line/system/camera telemetry를 받았다. 하지만 GCS camera window는 검은 화면의 `waiting for video stream...` 상태였다.

### 원인

현재 `vision_debug_node` 기본값은 metadata-only다.

```toml
[debug_video]
enabled = false
```

따라서 `--video`를 붙이지 않으면 onboard는 ArUco/line 정보를 telemetry로만 보내고, raw MJPEG debug video는 보내지 않는다. 이때 GCS video receiver는 열려 있어도 받을 video packet이 없으므로 waiting 화면이 정상이다.

확인 신호:

- Pi startup line: `video: off`, `telemetry: on`
- Pi per-frame log: `video_sent=0`, `video_chunks=0`
- GCS packet/video log: `video_sent=0`, `chunks_last=0`, `last_bytes=0`

### 해결

영상과 overlay를 실제로 보고 싶을 때만 Pi 명령에 `--video`를 추가한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video
```

네트워크 discovery가 막히면 GCS IP를 직접 지정한다.

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --gcs-ip <laptop-ip>
```

### 설계 판단

이 동작은 버그가 아니라 의도된 기본값이다. 최종 시연에서는 GCS 없이 onboard만 돌릴 수도 있고, mission 판단과 MAVLink 제어가 최우선이다. GCS video는 디버그 관제용이므로 기본값은 꺼 두고 필요할 때만 켠다.

## 19. GCS 영상 latency/age가 음수거나 체감 지연과 맞지 않음

### 증상

GCS camera overlay 또는 log에 표시되던 latency/age가 음수로 나오거나, 양수 20-30ms처럼 표시되는데 실제 체감 지연은 약 500ms 수준이었다.

### 원인

초기 latency 표시는 onboard frame timestamp와 GCS 수신/표시 시각을 단순 비교했다. 하지만 다음 조건 때문에 이 값은 end-to-end video latency로 보기 어렵다.

- Raspberry Pi와 Windows laptop의 wall clock이 정확히 동기화되어 있다고 가정할 수 없다.
- UDP MJPEG video는 best-effort이며 incomplete frame drop, reassembly, UI paint delay가 섞인다.
- GCS는 최신 complete JPEG만 표시하므로 실제 사람이 보는 지연은 network packet timing만으로 설명되지 않는다.
- Debug video는 mission-critical 경로가 아니므로 정확한 latency 계측보다 onboard 처리 시간과 packet health가 더 중요하다.

### 해결

GCS camera overlay와 vision log에서 video latency/age 표시를 제거했다.

현재 표시 정책:

- Camera window overlay: `frame N`만 표시
- Vision log: onboard 처리 시간인 `processing`, `read`, `decode`, `aruco`, `line`, `json`, `tsend`, `vsubmit`, `vsend`를 표시
- Video health: `completed`, `incomplete`, `display_fps`, `chunks_last`, `last_bytes`, `video_sent`, `video_skipped`, `video_dropped`, `video_send_failures`로 판단

`config/ui.toml`에는 의도를 남기기 위해 다음 값을 둔다.

```toml
[video_window]
show_latency = false
```

### 설계 판단

현재 프로젝트에서 필요한 것은 정확한 관제 영상 latency 숫자가 아니라, onboard vision/control loop가 제때 돌고 있는지와 debug video가 관찰 가능한 수준인지다. 정확한 video latency가 필요해지는 경우에는 NTP/PTP 수준 clock sync 또는 GCS에서 송수신 round-trip 측정용 별도 protocol을 설계해야 한다.

## 20. IMX519 화질을 높여도 ArUco 인식 개선 대비 onboard 부담이 큼

### 증상

IMX519 영상이 GCS 화면에서 뿌옇게 보이고 ArUco marker 인식이 불안정해 보여, capture 해상도와 JPEG quality를 올리는 실험을 했다.

실험 방향:

- `camera.width/height`를 `1280x960`까지 올림
- `camera.jpeg_quality`를 `85-90` 수준까지 올림
- ArUco bench tuning용 CLI override 추가

```bash
./build/vision_debug_node --config config --aruco-only --video --camera-quality 90 --lens-position 1.0
```

### 관찰

GCS 영상의 JPEG artifact는 줄어들 수 있지만, 대회 조건에서는 ArUco marker가 50cm x 50cm이고 고도는 약 2m로 예상된다. 이 조건에서는 marker pixel 크기가 충분할 가능성이 높고, 고화질 video가 mission 성능을 결정하는 병목이라고 보기 어렵다.

반대로 고화질 설정은 다음 부담을 키운다.

- rpicam MJPEG frame 크기 증가
- onboard JPEG decode 시간 증가
- ArUco/line detector 입력 픽셀 증가
- optional debug video UDP chunk 수 증가
- Wi-Fi packet loss 가능성 증가
- 추후 mission logic/MAVLink control loop 여유 감소

### 해결

기본값을 성능 우선 설정으로 되돌렸다.

```toml
[camera]
width = 960
height = 720
fps = 12
jpeg_quality = 45

[debug_video]
enabled = false
send_fps = 5
jpeg_quality = 40
chunk_pacing_us = 150
```

Bench tuning이 필요할 때만 CLI override를 쓴다.

```bash
./build/vision_debug_node --config config --aruco-only --camera-quality 90 --lens-position 1.0
./build/vision_debug_node --config config --aruco-only --video --camera-width 1280 --camera-height 960 --camera-quality 85
```

### 설계 판단

현재 단계에서는 화질보다 onboard 처리 여유가 더 중요하다. ArUco가 실제 50cm marker/2m 조건에서 충분히 잡히는지 확인한 뒤, 정말 부족할 때만 해상도나 JPEG quality를 올린다. 기본값은 line tracing, ArUco, 추후 mission 판단, MAVLink 제어를 같이 얹을 수 있는 보수적 설정으로 유지한다.

## 21. Metadata-only 실행에서도 GCS discovery 때문에 startup이 3초 늦어짐

### 증상

이전 구현에서는 `vision_debug_node`를 telemetry-only로 돌려도 GCS video discovery beacon을 최대 3초 기다리는 흐름이 있었다. 최종 운용에서 video를 끄고 onboard만 빠르게 띄우는 경우에는 불필요한 대기였다.

### 원인

GCS discovery는 video destination IP/port를 찾기 위한 기능이다. 하지만 debug video가 꺼진 상태에서도 broadcast address이면 discovery를 시도하면, 실제로 보낼 video가 없는데 startup만 늦어진다.

### 해결

`vision_debug_node`에서 `send_video`가 true일 때만 GCS discovery를 수행하도록 변경했다.

현재 동작:

- 기본 `./build/vision_debug_node --config config`: `debug_video.enabled=false`이므로 discovery 대기 없음
- `--no-video`: discovery 대기 없음
- `--video`: broadcast/default GCS IP일 때 discovery를 최대 3초 수행
- `--video --gcs-ip <laptop-ip>`: 명시 IP가 있으므로 discovery 없이 바로 전송

### 설계 판단

Metadata-only 실행은 향후 mission run에 가장 가까운 형태다. 영상 관제 기능이 꺼져 있을 때는 startup과 처리 경로를 최대한 단순하게 유지한다.

## 22. 넓은 흰색 라인이 얇은 edge만 잡히고 교차점 판단이 흔들림

### 문제 상황

사용자가 제시한 GCS 캡처에서는 굵은 흰색 라인 전체가 magenta overlay로 감싸지지 않고, 한쪽 edge 또는 얇은 strip만 라인으로 잡히는 경향이 있었다. 특히 밝은 목재/흙색 배경에서는 `straight`, `T`, `+`가 `unknown`이나 `L`로 흔들렸다.

### 원인

당시 기본 마스크는 `local_contrast`였다. 이 방식은 배경 대비가 큰 어두운 천 위에서는 잘 동작하지만, 넓은 흰색 테이프/종이처럼 선 내부가 균일하고 배경도 밝은 경우에는 내부 면 전체보다 양쪽 경계 contrast를 더 강하게 잡는다.

그 결과:

- line contour가 실제 선 폭 전체가 아니라 edge 위주로 형성된다.
- intersection center 후보가 실제 교차점 중심이 아니라 한 branch 쪽으로 밀린다.
- 4방향 ray score 중 일부 branch가 threshold 주변에서 깜빡인다.
- `+ -> T`, `T -> L`, `straight -> unknown`처럼 branch 누락형 오분류가 생긴다.

### 해결

`LineMaskBuilder`에 `white_fill` 전략을 추가하고 기본값으로 전환했다.

```toml
[line]
mask_strategy = "white_fill"
white_v_min = 145
white_s_max = 90
fill_close_kernel = 11
fill_dilate_kernel = 3
```

이 전략은 HSV 기준으로 낮은 saturation, 높은 value 영역을 흰색 라인 후보로 잡고 close/dilate morphology로 내부를 채운다. `local_contrast`는 조명 비교 실험용 fallback으로 남겨 두었다.

또한 `LineDetector`는 전체 contour centroid가 아니라 하단 anchor/lookahead band의 X projection으로 tracking X를 계산하도록 바꿨다. L/T 교차점의 가로 branch가 라인 중심을 끌어당기는 문제를 줄이기 위한 조치다.

### 결과

사용자 제공 4개 라인 캡처에서 굵은 흰색 라인이 더 안정적으로 하나의 blob/contour로 잡혔다. 교차점 판단도 원시 detector 단계에서 branch score가 안정화되어 이전보다 `L`, `T`, `+`, `straight` 구분이 잘 유지된다.

## 23. 교차점 판단 정확도가 올라간 이유와 현재 로직

### 현재 교차점 판단 흐름

교차점 판단은 한 frame만 보고 바로 결정하지 않는다. 현재 흐름은 다음과 같다.

1. `IntersectionDetector`가 shared line mask에서 largest blob과 중심 후보를 찾는다.
2. 중심 후보 주변에서 camera-relative 4방향 ray score를 계산한다.
   - front
   - right
   - back
   - left
3. branch score가 `intersection_threshold` 이상이면 raw branch present로 본다.
4. `IntersectionStabilizer`가 raw type을 짧게 smoothing한다.
5. `IntersectionDecisionEngine`이 최근 frame window를 모아 branch evidence를 다시 계산한다.
6. window 안에서 branch별 `present_frames`, `max_score`, `average_score`를 보고 최종 accepted type을 정한다.

### type 결정 기준

`IntersectionDecisionEngine`은 우선순위를 `+ > T > L > straight > unknown`으로 두지만, 단일 frame에서 상위 type이 잠깐 튀는 것을 바로 채택하지 않는다.

- `+`: front/right/back/left 4방향 모두 충분한 frame 수 동안 보이고, 네 방향 모두 `high_confidence_score` 이상이어야 채택한다.
- `T`: 가능한 3방향 조합 중 branch evidence가 가장 좋은 mask를 고른다.
- `L`: 가능한 2방향 직각 조합 중 evidence가 가장 좋은 mask를 고른다.
- `straight`: front/back 또는 left/right 조합이 충분히 보이면 인정한다.
- `unknown`: branch evidence가 부족하거나 일관되지 않을 때 남긴다.

현재 주요 설정:

```toml
[intersection_decision]
cruise_window_frames = 6
turn_confirm_frames = 8
min_branch_score = 0.72
high_confidence_score = 0.85
min_cross_branch_frames = 2
min_t_branch_frames = 2
min_l_branch_frames = 3
record_node_once_frames = 18
```

### 정확도가 좋아진 이유

정확도 개선은 한 가지 수정 때문이 아니라 여러 변경이 겹친 결과다.

- `white_fill`로 넓은 흰색 라인의 내부가 채워져 branch ray가 끊기지 않는다.
- `+`는 네 번째 branch가 약할 때 바로 채택하지 않도록 `high_confidence_score` guard를 추가했다.
- `T`와 `L`은 단일 frame branch count가 아니라 최근 window의 branch evidence로 판단한다.
- line tracking point와 intersection classification을 분리했다. 라인 주행 중심은 anchor band에서 잡고, 교차점 판단은 connected mask topology를 본다.
- GCS overlay를 간소화해 실제 판단에 중요한 line center, offset, 접근 중인 branch 방향만 보이게 했다.

### 보고서용 요약

초기에는 한 frame의 ray score가 threshold 주변에서 흔들리면 교차점 type도 즉시 흔들렸다. 현재는 넓은 흰색 라인을 채워진 mask로 안정화하고, 최근 frame window에서 branch evidence를 누적해 topology를 결정한다. 특히 `+`는 네 방향 모두 높은 확신이 있을 때만 채택하도록 하여 약한 false branch 때문에 `T`가 `+`로 승격되는 문제를 줄였다.

## 24. GCS overlay 정보가 너무 많아 라인트레이싱 제어용 오차를 보기 어려움

### 문제 상황

기존 GCS 영상 overlay에는 line label, tracking point, intersection branch score, decision label 등 많은 정보가 동시에 표시됐다. 라인트레이싱 제어를 준비하려면 “현재 라인 중심이 카메라 중심에서 얼마나 벗어났는지”가 가장 잘 보여야 하는데, 화면이 복잡했다.

### 해결

Line overlay를 다음 세 요소 중심으로 단순화했다.

- magenta: 현재 선택된 line contour/border
- red circle: 현재 라인 중심 X
- green horizontal line: 카메라 중심에서 라인 중심까지의 lateral offset

중요한 표시 규칙은 red circle의 Y가 항상 camera center Y에 고정된다는 점이다. 실제 line tracking point의 세로 위치가 frame 아래쪽/위쪽에 있어도 GCS 표시에서는 같은 수평선 위에 두어, 좌우 오차만 직관적으로 보이게 했다.

교차점 overlay도 간소화했다.

- 화면 상단/현재 접근 영역의 교차점만 표시
- `IX <type>` 형태의 compact label만 표시
- branch score 숫자와 긴 decision label은 영상에서 제거하고 log에 남김

### 결과

영상 창에서 라인트레이싱 제어에 필요한 lateral error를 즉시 확인할 수 있게 되었다. 복잡한 판단 값은 vision log에서 확인하는 구조로 역할을 분리했다.

## 25. 빨간 라인 중심점이 좌우 이동에 조금 늦게 따라오는 느낌

### 문제 상황

라인 자체는 놓치지 않지만, 카메라를 좌우로 빠르게 움직였을 때 GCS 영상의 빨간 점과 초록 offset line이 약간 늦게 따라오는 느낌이 있었다.

### 원인

두 가지 지연 요인이 있었다.

1. onboard `LineStabilizer`가 EMA smoothing, velocity limit, jump rejection을 적용한다.
2. GCS debug video는 `--video`만 켜면 기본 5FPS이고, `--fps 12`를 지정해야 카메라 capture FPS에 가깝게 보인다.

기존 GCS overlay는 filtered `tracking_point_px`를 사용했기 때문에 제어 안정화에는 좋지만, 사람이 화면에서 보는 빨간 점은 raw detector보다 늦어 보일 수 있었다.

### 해결

GCS 영상 overlay의 red circle과 green offset line은 `raw_tracking_point_px`가 있으면 raw 값을 우선 사용하도록 변경했다. telemetry log에는 filtered 값과 raw 값을 모두 남겨, 제어 입력으로 어떤 값을 쓸지는 이후 control loop에서 결정할 수 있게 했다.

실시간 관찰용 권장 실행:

```bash
./build/vision_debug_node --config config --line-only --line-mode light_on_dark --video --fps 12
```

### 판단

관제 화면에서는 raw point가 더 직관적이고 빠르다. 반면 실제 제어에는 filtered point가 더 안전할 수 있다. 최종 제어 단계에서는 raw/filtered offset을 모두 기록하면서 lateral controller가 어느 값을 사용할지 별도 실험으로 결정한다.

## 26. GCS grid가 로그 아래에 묻혀 보이지 않음

### 문제 상황

`GridMapTracker`가 ASCII grid를 만들도록 구현했지만, GCS vision log에는 telemetry 값이 너무 많아 grid가 화면 아래로 밀렸다. 사용자가 보기에는 grid가 그려지는 전용 공간이 없는 것처럼 보였다.

### 원인

처음 구현은 `VisionLogFormatter`가 상세 telemetry 문자열 마지막에 `[grid-map]` 텍스트를 붙이는 방식이었다. log window는 하나의 multiline edit control만 사용했기 때문에 grid와 상세 telemetry가 같은 스크롤 영역에서 경쟁했다.

### 해결

GCS `VisionLogWindow`를 두 영역으로 분리했다.

- 상단 고정 pane: `[grid-map]` ASCII map
- 하단 pane: `[vision]`, `[line]`, `[intersection]`, `[intersection-decision]`, `[grid-node]`, `[video-rx]` 상세 telemetry

### 결과

grid는 항상 vision log window 상단에 남고, 상세 telemetry가 길어져도 grid 표시 영역이 밀리지 않는다.

## 27. GCS grid pane이 한 줄로 붙어서 보임

### 문제 상황

상단 grid pane이 생겼지만 Windows에서 다음처럼 줄바꿈 없이 한 줄로 이어져 보였다.

```text
[grid-map] nodes=1 current=(0,0) heading=unknowns|@
```

하단 telemetry도 `[vision] ... [camera] ... [system] ...`이 한 줄로 이어져 표시됐다.

### 원인

Win32 `EDIT` control은 `\n`만 있는 문자열을 안정적인 줄바꿈으로 처리하지 않는다. Windows multiline edit에는 `\r\n` line ending을 넣어야 한다.

### 해결

`VisionLogWindow`에서 UI에 문자열을 넣기 전에 line ending을 정규화했다.

- 입력 문자열의 단독 `\n` 앞에 `\r`을 추가
- UTF-8 -> UTF-16 변환 전에 정규화

### 결과

상단 grid와 하단 telemetry가 의도한 여러 줄 형태로 표시된다.

## 28. GCS grid가 `nodes=1 current=(0,0) heading=unknown`에서 더 이상 확장되지 않음

### 문제 상황

상단 grid pane 자체는 표시되지만 처음부터 끝까지 다음과 비슷한 상태에 머물렀다.

```text
[grid-map] nodes=1 current=(0,0) heading=unknown
```

교차점 판단과 하단 vision telemetry는 계속 갱신되는데, grid는 새 칸을 추가하지 않았다.

### 원인

현재 단계에는 실제 드론 상태머신, IMU heading, MAVLink turn completion event가 없다. 따라서 onboard `GridCoordinateTracker`가 처음 grid node를 local `(0,0)`으로 저장한 뒤에도 `current_heading_ = Unknown` 상태를 유지했다.

기존 좌표 갱신 로직은 heading이 알려져 있을 때만 다음 좌표로 advance한다.

```cpp
if (current_heading_ != GridHeading::Unknown) {
    current_coord_ = advance(current_coord_, current_heading_);
}
```

heading이 unknown이면 이후 node event가 들어와도 좌표가 계속 `(0,0)`이 된다. GCS `GridMapTracker`는 좌표를 key로 map에 저장하므로 같은 `(0,0)`이 반복되어도 node count가 늘지 않는다.

### 해결

임시 vision-only grid smoke용 heading 추정 로직을 추가했다.

- 첫 node에 heading이 없으면 launch line에서 grid로 진입했다고 보고 local heading을 `north`로 둔다.
- 첫 node에서 side branch가 보이면 첫 row/column으로 들어가기 위해 side branch를 우선 선택한다.
- 이후에는 front branch가 있으면 직진한다.
- front branch가 없고 right/left branch가 있으면 해당 방향으로 90도 회전한다고 가정한다.
- row 끝에서 한 칸 올라간 뒤 같은 방향으로 한 번 더 회전해야 하는 snake 특성을 위해 `pending_second_turn`을 둔다.

이 로직은 실제 비행 제어가 아니라 수동 카메라 테스트와 GCS grid 렌더링을 위한 임시 추정이다. 향후 MAVLink/IMU/state machine이 들어오면 `notifyTurnCompleted()`나 mission state가 실제 heading을 공급해야 한다.

### 결과

드론 없이 손으로 카메라를 들고 snake 방식으로 이동하는 테스트에서도 node event가 들어올 때마다 local coordinate가 확장될 수 있게 되었다.

### 남은 리스크

- 초기 진입 방향이 bottom-entry가 아니라 다른 방향이면 local map이 회전된 형태로 그려질 수 있다.
- 실제 드론이 회전 완료를 알려주지 않는 한 heading은 vision-only 추정이다.
- 공식 좌표계 변환은 아직 구현 전이다.
- 같은 교차점을 다시 방문하지 않는 실제 snake policy와 visited guard는 mission layer에서 별도로 완성해야 한다.
- 확실히 회전 완료를 판단하기 애매해서 그리드가 완벽하게 그려지지 않음

## 29. `dark_on_light`에서 굵은 검정 라인이 하나의 라인으로 잡히지 않음

### 문제 상황

2m 고도 탑다운뷰를 가정한 굵은 검정 라인 테스트에서 `--line-mode dark_on_light`로 실행했는데도 검정 라인 전체가 하나의 넓은 contour로 잡히지 않았다. Magenta overlay가 검정 라인의 양쪽 edge 위주로 생기고, red line-center point도 실제 라인 중앙보다 한쪽 edge에 가까웠다.

사용자 캡처 기준 문제 양상:

- 검정 라인 폭 전체가 mask로 채워지지 않음
- contour 후보가 배경 노이즈와 edge 조각으로 많이 쪼개짐
- line offset이 실제 중앙보다 크게 치우침
- `light_on_dark`의 흰색 라인 정확도는 이미 만족스러우므로, 이 경로에는 영향을 주면 안 됨

### 원인

기존 기본 `mask_strategy = "white_fill"`은 이름 그대로 밝은 라인을 채우는 경로가 중심이었다. `dark_on_light`에서는 어두운 선 내부를 직접 채우는 전용 fill 경로가 부족해, local contrast/edge 성분이 더 강하게 남았다.

또한 line 후보 폭 제한이 밝은 라인 기준과 사실상 공유되어 있었다. 2m 고도에서 10cm 정도의 굵은 검정 라인은 화면에서 폭이 넓게 보이므로, 어두운 라인에는 별도 폭 허용치가 필요했다.

정리하면 원인은 두 가지다.

- 검정 라인 내부를 value 기준으로 직접 채우는 `dark_fill` 경로가 없었다.
- `light_on_dark`와 `dark_on_light`의 morphology/width 튜닝이 완전히 분리되어 있지 않았다.

### 해결

`LineMaskBuilder`와 `LineDetector`를 polarity별로 분리했다.

`light_on_dark` 경로:

- HSV low-saturation/high-value 기반 `white_fill`
- `white_v_min`, `white_s_max`
- `fill_close_kernel`, `fill_dilate_kernel`
- `max_line_width_ratio`

`dark_on_light` 경로:

- low-value 기반 `dark_fill`
- `dark_v_max`
- `dark_fill_close_kernel`, `dark_fill_dilate_kernel`
- `dark_max_line_width_ratio`

현재 핵심 설정:

```toml
[line]
mask_strategy = "white_fill"

white_v_min = 145
white_s_max = 90
fill_close_kernel = 11
fill_dilate_kernel = 3
max_line_width_ratio = 0.22

dark_v_max = 85
dark_fill_close_kernel = 13
dark_fill_dilate_kernel = 3
dark_max_line_width_ratio = 0.34
```

`mask_strategy = "white_fill"`이더라도 `dark_on_light`에서는 내부적으로 dark fill 경로로 분기한다. 따라서 기본 전략 이름 때문에 검정 라인이 흰색 fill 로직을 타는 구조가 아니다.

`line_detector_tuner`에도 dark 전용 override를 추가했다.

```text
--dark-v-max
--dark-fill-close
--dark-fill-dilate
--dark-max-width
```

그리고 회귀 테스트를 추가했다.

- 넓은 검정 라인이 `mask_strategy = "white_fill"` 상태에서도 `dark_on_light`로 정상 검출되는지 확인
- 기존 넓은 흰색 라인 `light_on_dark` 검출이 그대로 유지되는지 확인

이 테스트는 한쪽 mode 튜닝이 다른 mode 정확도를 깨는 상황을 막기 위한 guard다.

### 결과

사용자 제공 검정 라인 캡처 기준으로 다음처럼 개선됐다.

```text
before: offset=-62.32px, contours_found=427
after:  offset=-14.07px, contours_found=8
```

수정 후 summary:

```text
detected=true
tracking_point=(468.43,404.54)
center_overlay_y=367
offset=-14.07px
angle=83.78deg
confidence=0.83
contour_points=48
contour_area=83362.50
contour_bounds=(350,59,222,626)
mask_count=1
contours_found=8
candidates_evaluated=3
```

검정 라인이 edge 조각이 아니라 하나의 넓은 line blob으로 잡히며, 배경 노이즈 contour도 크게 줄었다. `uav-onboard` OpenCV tests도 통과했다.

### line mode 해석

`light_on_dark`:

- 어두운 배경 위 밝은 라인을 검출하는 mode
- 흰색, 연노랑, 연한 회색처럼 saturation이 낮고 value가 높은 라인에 유리

`dark_on_light`:

- 밝은 배경 위 어두운 라인을 검출하는 mode
- 검정, 진한 회색, 남색처럼 value가 낮은 라인에 유리

`auto`:

- 밝은 라인 후보와 어두운 라인 후보를 모두 만들고, score가 더 좋은 contour를 선택한다.
- 라인 색이 확정되지 않은 초기 현장 점검에는 유용하다.
- 실제 대회 주행처럼 라인 색과 배경이 정해져 있으면 `light_on_dark` 또는 `dark_on_light`로 고정하는 편이 더 예측 가능하다.

line mode 옵션을 주지 않으면 현재 config 기본값을 따른다. 현재 기본 config는 `mode = "light_on_dark"`이므로, 옵션을 생략한다고 항상 `auto`가 되는 것은 아니다. `auto`를 쓰려면 CLI나 config에서 명시해야 한다.

### 유색 라인 대응

현재 구현은 특정 색 이름을 직접 인식하는 방식이 아니라, 배경 대비와 brightness polarity를 이용한다.

- 연노랑/연두/하늘색처럼 배경보다 충분히 밝고 value가 높으면 `light_on_dark`로 잡힐 가능성이 있다. 단, 채도가 높으면 `white_s_max` 조건에서 빠질 수 있다.
- 진한 초록/남색처럼 배경보다 충분히 어두우면 `dark_on_light`로 잡힐 가능성이 있다.
- 빨강, 진한 초록, 채도 높은 파랑처럼 밝기만으로 분리하기 애매한 색은 현재 light/dark mode만으로는 불안정할 수 있다.

향후 라인 색이 흰색/검정 계열이 아니라 특정 유색으로 확정되면 `red_on_dark`, `green_on_light`처럼 색마다 mode를 늘리기보다, HSV hue range를 설정으로 받는 `color_fill` 전략을 추가하는 편이 낫다. 그래야 색 추가마다 코드를 늘리지 않고 config만 바꿔 현장 색상에 대응할 수 있다.
