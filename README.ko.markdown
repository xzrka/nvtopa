# nvtopa

[English](README.markdown) | 한국어 | [中文](README.zh.markdown) | [日本語](README.ja.markdown)

[syllo/nvtop](https://github.com/syllo/nvtop)의 포크로, **온도 계측**과
**터미널이 아무리 좁아도 잘리지 않는 디바이스 헤더**에 중점을 둡니다.

![nvtopa 메인 화면](screenshot/nvtopa_main.png)

---

## 목차

- [이 포크가 추가한 기능](#이-포크가-추가한-기능)
- [키보드 단축키](#키보드-단축키)
- [스크린샷](#스크린샷)
- [테스트 환경](#테스트-환경)
- [빌드 및 설치](#빌드-및-설치)
- [문서](#문서)
- [라이선스](#라이선스)

## 이 포크가 추가한 기능

### 1. 너비 적응형 디바이스 헤더

원본 nvtop은 디바이스 헤더를 약 120열 고정 폭으로 배치하기 때문에, 일반적인
터미널에서는 파워·팬·클록 필드가 오른쪽 끝에서 잘립니다. 이 포크는 레이아웃을
계산하기 전에 헤더를 콘솔 너비에 맞춥니다. **포기 사다리(give-up ladder)**를
순서대로 내려가며 팬 여백, 클록 레이블, ECC·NVLink 필드, 마지막으로 최소/최대
표시를 하나씩 내려놓아 행이 맞을 때까지 조정합니다. 코어 온도는 절대 완전히
사라지지 않습니다.

```
Device 1 [NVIDIA GeForce RTX 3090] PCIe GEN 4@16x   NVL3 4x 0.000 KiB/s
1455MHz 9501MHz  TEMP  63° 46/66   JCT  74°C   VRAM  74°C   FAN  59%  POW 215 / 220 W
```

pty 스윕 테스트로 **40열부터 200열까지** 어느 너비에서도 아무것도 잘리지 않는
것을 확인했습니다.

### 2. 세션 최소/최대 온도, 리셋 키

`TEMP`, `JCT`, `VRAM` 각각은 현재 값과 함께 프로그램 시작(또는 마지막 리셋) 이후
관측된 최소·최대 값을 표시합니다:

- `TEMP 63° 46/66` — 현재 63 °C, 최소 46 °C, 최대 66 °C
- **F7**을 누르면 히스토리가 리셋되고, 최소/최대 값이 현재 값으로 접힌 뒤 다시
  추적을 시작합니다.

최소/최대 값은 터미널 크기 변경에도 유지됩니다. 터미널 창이 아닌 디바이스 상태에
저장되기 때문입니다.

### 3. PCIe 옆의 NVLink 전송량

NVLink 전송량을 두 번째 헤더 행 끝(좁은 터미널에서 가장 먼저 잘리는 위치)에서
**1행, PCIe 전송량 바로 옆**으로 옮겼습니다:

```
Device 1 [NVIDIA GeForce RTX 3090] PCIe GEN 4@16x   NVL3 4x 0.000 KiB/s
```

PCIe 필드는 실제로 그리는 형태에 맞게 창 폭이 조정됩니다 — 상세형
`PCIe GEN 4@16x RX: ... TX: ...` (46열), 축약형 `GEN 4@16x RX 9.2G/TX 10.1G`
(30열), 또는 `PCIe GEN 4@16x` (14열) — 그래서 NVLink 필드가 빈칸 나열 뒤가
아니라 바로 붙어서 표시됩니다. 공간이 아예 없어지면(대략 60열 미만) NVLink를
생략하고 PCIe에 공간을 돌려줍니다. 브리지가 없는 보드는 이전과 마찬가지로 빈
슬롯을 유지합니다.

### 4. 모든 너비에서 유지되는 원본 전력 형식

원본은 좁은 터미널에서 `POW x / y W`(현재/최대)를 `POW x W`로 축소합니다. 이
포크는 모든 너비에서 원본 형식을 유지하고, 더 내려놓을 것이 없을 때만 필드
자체를 생략합니다.

### 5. GPU당 최대 10개의 그래프 라인

GPU별 차트가 한 번에 최대 10개 지표를 받습니다(원본은 4개). 기본값은 **기기별로
한 개의 차트에 전부 표시**하며, 각 라인은 고유한 색을 가집니다:

- GPU 사용률
- GPU 메모리 사용률
- GPU 코어 온도
- 메모리 저크션(junction)(핫스팟) 온도
- 메모리(VRAM) 온도

보드가 보고하지 않는 센서의 라인은 자동으로 생략됩니다 — RTX 3050에는 메모리
온도 센서가 없으므로 `vram(c)` 라인을 그리지 않습니다.

### 6. F8 — 그래프 항목 순환

기기별 차트는 모드에 따라 다른 지표를 보여주며, **F8**이 모드를 순환합니다.
모드 전환 시 플롯 버퍼를 비워 서로 섞이지 않게 합니다:

| 모드 | 표시되는 라인 |
| --- | --- |
| 전체 지표 (기본값) | 사용률, 메모리 사용률, 보고되는 모든 온도 |
| 원본 nvtop | GPU 사용률과 메모리 사용률만 |
| 온도만 | 코어·저크션(junction)·메모리 온도만 |

### 7. GDDR 레지스터에서 읽는 메모리 저크션(junction)·VRAM 온도

이 포크의 원래 기능으로,
[ThomasBaruzier/gddr6-core-junction-vram-temps](https://github.com/ThomasBaruzier/gddr6-core-junction-vram-temps)에서
가져왔습니다: `JCT`(메모리 저크션(junction), 핫스팟)와 `VRAM`(메모리 장치)은 지원되는
어떤 API도 보고하지 않는 GDDR6/GDDR6X/GDDR7 메모리 컨트롤러의 레지스터에서
읽습니다. Ampere(RTX 30) 세대 이상만 해당하며, 그보다 오래된 보드는 건드리지
않습니다. udev 규칙으로 GPU의 첫 BAR을 그룹에 넘겨주면 일반 사용자도 읽을 수
있습니다 — 이를 수행하는 스크립트는 [머신 설정
저장소](https://github.com/xzrka/nvtopa-setup)에 있습니다.

## 키보드 단축키

| 키 | 동작 |
| --- | --- |
| `F2` | 대화형 설정 창 (Chart 페이지에서 그래프 라인과 색상 설정) |
| `F6` | 프로세스 목록 정렬 |
| `F7` | **온도 히스토리(최소/최대) 리셋** |
| `F8` | **그래프 항목 순환** (전체 지표 → 원본 nvtop → 온도만) |
| `F9` | 프로세스 종료 |
| `F10` | 종료 |
| `F12` | 환경설정 저장 |
| `Ctrl+L` | 전체 화면 강제 다시 그리기 |

## 스크린샷

NVIDIA GPU 3대에서의 메인 화면 — 최소/최대 온도가 표시된 너비 적응형 헤더,
PCIe 옆의 NVLink, 기기별 5라인 차트:

![nvtopa 메인 화면](screenshot/nvtopa_main.png)

설정 창(`F2`) — Chart 페이지에서 개별 그래프 라인을 켜고 끄고 색상을 바꿀 수
있습니다:

![nvtopa 설정 창](screenshot/nvtopa_setup.png)

## 테스트 환경

이 포크는 다음 환경에서 개발·검증되었습니다:

| 항목 | 값 |
| --- | --- |
| GPU | NVIDIA GeForce RTX 3050 1개(GDDR6, 메모리 온도 센서 없음), NVIDIA GeForce RTX 3090 2개(GDDR6X, NVL3 4x로 NVLink 연결) |
| OS | Pop!_OS (Ubuntu 계열), Linux x86-64 |
| 드라이버 | NVIDIA 독점 드라이버 (NVML + BAR0 레지스터 접근) |
| 터미널 | GNOME 터미널, pty 헤니스로 비대면 검증 |

검증 방법:

- **`tests/temp_colors.py`** — 프로그램을 pty에서 재생하여 모든
  `TEMP`/`JCT`/`VRAM` 숫자에 적용된 SGR 색을 추출합니다. 온도 필드의 색상이
  자동으로 검사됩니다. `python3 tests/temp_colors.py 80 24 -- ./build/src/nvtop`
- **`tests/pty_capture.py`** — 임의 크기의 pty에서 프로그램을 실행하고 화면을
  출력합니다. 40열부터 200열까지 헤더를 스윕하여 각 너비에서 포기 사다리를
  검증하는 데 사용됩니다.
- **`ctest`** — 업스트림 테스트 모음에 더해, 센서 레지스터를 가짜 sysfs로
  구성하여 디코딩된 값을 검사하는 `nvidia_vram_temps`가 실행됩니다.
- **`tests/interfaceTests.cpp`** — PCIe 대역폭 표에 대한 단위 테스트.
- **40, 50, 60, 70, 80, 90, 100, 120, 160, 200열** 수동 pty 스윕으로 어느
  너비에서도 아무것도 잘리지 않고 전력 필드가 항상 `POW x / y W` 형식을 유지함을
  확인했습니다. F8의 세 그래프 모드와 F7 리셋은 pty와 GNOME 터미널 실제 화면
  양쪽에서 모두 시험했습니다.

## 빌드 및 설치

표준 nvtop 빌드:

```bash
git clone https://github.com/xzrka/nvtopa.git
cd nvtopa
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

이 포크가 개발된 머신에서는 [머신 설정 저장소](https://github.com/xzrka/nvtopa-setup)의
`install-nvtopa.sh`가 포크를 빌드하여 배포판의 `nvtop` 옆에 `nvtopa`로
설치하며, `unprivileged-temps.sh`가 일반 사용자도 메모리 온도 레지스터를 읽을 수
있는 udev 규칙을 설정합니다.

## 문서

- [`docs/UPSTREAM_README.markdown`](docs/UPSTREAM_README.markdown) — 업스트림
  전체 문서: 벤더 지원, 배포판별 설치, 문제 해결.
- 업스트림 프로젝트: <https://github.com/syllo/nvtop>
- 머신 설정(udev 규칙, 설치 스크립트, 가짜 sysfs): <https://github.com/xzrka/nvtopa-setup>

## 라이선스

업스트림 nvtop과 동일: BSD-3-Clause.
