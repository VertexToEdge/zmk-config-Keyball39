# ZMK Keyball39 Configuration - Agent Reference Guide

> 이 문서는 AI 에이전트가 이 레포지토리에서 작업할 때 필요한 핵심 정보를 담고 있습니다.

## 프로젝트 개요

**Keyball39**: 39키 분리형 인체공학 키보드 + 통합 트랙볼
- **펌웨어**: ZMK (Zephyr Mechanical Keyboard)
- **컨트롤러**: Nice Nano v2 (nRF52840)
- **특징**: 무선 BLE, OLED 디스플레이, PMW3610 트랙볼 센서

## 디렉터리 구조

```
config/
├── keyball39.keymap              # 🔑 키맵 정의 (레이어, 바인딩)
├── keyball39.conf                # ⚙️ 메인 설정 (BLE, 디스플레이)
├── west.yml                      # 📦 의존성 (ZMK v0.2 + PMW3610 드라이버)
└── boards/shields/keyball_nano/
    ├── keyball39.dtsi            # 🔧 하드웨어 공통 정의
    ├── keyball39_left.overlay    # ⬅️ 왼쪽 하드웨어 오버레이
    ├── keyball39_right.overlay   # ➡️ 오른쪽 하드웨어 오버레이
    ├── keyball39_right.conf      # 🖱️ 트랙볼 전용 설정
    ├── Kconfig.shield            # Shield 옵션
    ├── Kconfig.defconfig         # 기본 설정
    └── keyball39.zmk.yml         # ZMK 메타데이터

build.yaml                        # 🏗️ 빌드 타겟 정의
.github/workflows/
├── build.yml                     # CI: 펌웨어 빌드
└── keymap_drawer.yml             # CI: 키맵 시각화
```

## 핵심 파일 설명

### 1. `config/keyball39.keymap`
**역할**: 키 레이아웃과 동작 정의

**주요 내용**:
- 7개 레이어 (DEFAULT, NUM, SYM, FUN, MOUSE, SCROLL, SNIPE)
- 커스텀 동작: mod-morph (콤마/물음표, 점/세미콜론)
- 매크로: 히라가나 (LC(SPACE))
- Layer-tap 타이밍: 240ms
- Mod-tap 타이밍: 200ms

**수정 시 주의**:
- Devicetree 구문 사용 (중괄호, 세미콜론 필수)
- 키코드는 ZMK 표준 따름 (예: `&kp A`, `&lt 1 SPACE`)
- 양쪽 키 수 일치 필요 (왼쪽 + 오른쪽 = 39키)

### 2. `config/keyball39.conf`
**역할**: 전역 펌웨어 설정

**중요 설정**:
```conf
# BLE 최적화
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y        # 최대 송신 출력
CONFIG_BT_PERIPHERAL_PREF_LATENCY=16  # 레이턴시 최적화

# Split 키보드
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y

# 디스플레이
CONFIG_ZMK_DISPLAY=y

# 성능
CONFIG_ZMK_BEHAVIORS_QUEUE_SIZE=512
```

### 3. `config/boards/shields/keyball_nano/keyball39_right.conf`
**역할**: 트랙볼 전용 설정 (오른쪽만)

**트랙볼 핵심 설정**:
```conf
# CPI (감도)
CONFIG_PMW3610_CPI=1200              # 일반 모드
CONFIG_PMW3610_SNIPE_CPI=400         # 저격 모드
CONFIG_PMW3610_CPI_DIVIDER=4         # 실제 CPI = CPI / divider

# 동작
CONFIG_PMW3610_POLLING_RATE_125_SW=y
CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=700
CONFIG_PMW3610_SMART_ALGORITHM=y
CONFIG_PMW3610_SCROLL_TICK=32

# 방향
CONFIG_PMW3610_ORIENTATION_180=y
CONFIG_PMW3610_INVERT_SCROLL_X=y
```

**수정 시 주의**:
- CPI 변경 시 실효 CPI = CPI / CPI_DIVIDER
- AUTOMOUSE_TIMEOUT은 자동으로 레이어 4로 전환되는 시간

### 4. `config/west.yml`
**역할**: 의존성 관리 (West 매니페스트)

**주요 의존성**:
```yaml
- name: zmk
  remote: zmkfirmware
  revision: v0.2                    # ZMK 안정 버전

- name: zmk-pmw3610-driver
  remote: kumamuk-git
  revision: main                    # 트랙볼 드라이버
  path: modules/drivers/sensor
```

**수정 시 주의**:
- ZMK 버전 업데이트 시 breaking changes 확인 필요
- PMW3610 드라이버는 외부 모듈 (표준 ZMK에 없음)

### 5. `config/boards/shields/keyball_nano/*.overlay`
**역할**: 하드웨어 핀 배치 및 디바이스 정의

**주요 디바이스**:
- `kscan0`: 키 매트릭스 (GPIO 행/열)
- `oled`: SSD1306 OLED (I2C 0x3C)
- `trackball`: PMW3610 (SPI1, 오른쪽만)

**수정 시 주의**:
- Devicetree 구문 (DTS)
- GPIO 핀 번호는 Nice Nano v2 핀맵 따름
- 오버레이는 기본 보드 정의에 추가되는 개념

### 6. `build.yaml`
**역할**: GitHub Actions 빌드 타겟 정의

**빌드 대상**:
```yaml
- board: nice_nano_v2
  shield: keyball39_left

- board: nice_nano_v2
  shield: keyball39_right
  snippet: studio-rpc-usb-uart    # Studio 지원

- board: nice_nano_v2
  shield: settings_reset           # 초기화용
```

## 레이어 구조 (빠른 참조)

| 레이어 | 이름 | 용도 | 활성화 방법 |
|--------|------|------|-------------|
| 0 | DEFAULT | QWERTY 기본 레이아웃 | 기본값 |
| 1 | NUM | 숫자, 화살표 | Layer-tap |
| 2 | SYM | 심볼, BT 제어 | Layer-tap (백스페이스) |
| 3 | FUN | F1-F12 | Momentary |
| 4 | MOUSE | 마우스 클릭, 네비게이션 | Layer-tap / Auto |
| 5 | SCROLL | 트랙볼 스크롤 모드 | Layer-tap |
| 6 | SNIPE | 트랙볼 정밀 모드 | Layer-tap (ESC) |

## 빌드 방법

### 로컬 빌드
```bash
# West 초기화
west init -l config/
west update

# 빌드
west build -p -b nice_nano_v2 -- -DSHIELD=keyball39_left
west build -p -b nice_nano_v2 -- -DSHIELD=keyball39_right -DSNIPPET=studio-rpc-usb-uart
```

### GitHub Actions
- Push/PR 시 자동 빌드
- Artifacts에서 `.uf2` 파일 다운로드
- Settings reset도 함께 생성됨

## 일반적인 수정 작업

### 키 바인딩 변경
1. `config/keyball39.keymap` 열기
2. 원하는 레이어의 `bindings` 배열 수정
3. 키코드 참조: [ZMK Keycodes](https://zmk.dev/docs/keymaps/list-of-keycodes)

### 트랙볼 감도 조정
1. `config/boards/shields/keyball_nano/keyball39_right.conf` 열기
2. `CONFIG_PMW3610_CPI` 또는 `CONFIG_PMW3610_CPI_DIVIDER` 수정
3. 실효 CPI = CPI / divider

### 블루투스 설정 변경
1. `config/keyball39.conf` 열기
2. `CONFIG_BT_*` 항목 수정

### 새 레이어 추가
1. `config/keyball39.keymap`의 `layers` 블록에 레이어 추가
2. 레이어 번호 순서대로 배치
3. 다른 레이어에서 `&mo N` 또는 `&lt N KEY`로 접근

## 트러블슈팅

### 빌드 실패
- `west.yml`의 의존성 버전 확인
- `west update` 실행 후 재빌드
- 에러 로그에서 파일명:라인 확인

### 키가 작동 안 함
- `keyball39.keymap`에서 키 수 확인 (39개 맞는지)
- Devicetree 구문 오류 (세미콜론, 중괄호) 확인
- 양쪽 펌웨어 모두 새로 플래시

### 트랙볼이 이상하게 움직임
- `CONFIG_PMW3610_ORIENTATION_*` 확인
- `CONFIG_PMW3610_INVERT_*` 토글
- CPI_DIVIDER가 너무 작으면 과민 반응

### 블루투스 연결 불안정
- `CONFIG_BT_CTLR_TX_PWR_*` 최대로 설정
- 양쪽 배터리 레벨 확인
- Settings reset으로 페어링 초기화

## 참고 링크

- [ZMK 공식 문서](https://zmk.dev/)
- [PMW3610 드라이버](https://github.com/kumamuk-git/zmk-pmw3610-driver)
- [Keyball 원본 프로젝트](https://github.com/Yowkees/keyball)
- [Nice Nano 핀맵](https://nicekeyboards.com/docs/nice-nano/pinout-schematic)
- [Devicetree 스펙](https://www.devicetree.org/)

## 주의사항

1. **펌웨어 플래시**: 양쪽 모두 업데이트 필요 (좌/우 별도 파일)
2. **Studio 사용**: 오른쪽만 USB 연결 (Central 역할)
3. **설정 리셋**: 문제 발생 시 settings_reset 먼저 플래시
4. **외부 드라이버**: PMW3610 드라이버는 별도 레포에서 가져옴 (west.yml 관리)
5. **레이어 순서**: keymap과 conf의 레이어 번호 일치 확인

---

**마지막 업데이트**: 2025-12-04
**ZMK 버전**: v0.2
**드라이버 버전**: kumamuk-git/zmk-pmw3610-driver@main
