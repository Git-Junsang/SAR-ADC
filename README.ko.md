# SAR-ADC

[English](README.md) | **한국어**

---

**Cadence Virtuoso 기반 SAR ADC용 C-DAC 및 부트스트랩 스위치 설계**

---

> ⚠️ **본 저장소는 IP(파운드리 기밀 자산)를 제거한 버전입니다.**
> 파운드리 PDK 기밀 자산 — 공정 테크놀로지 파일(`.tf`), 디스플레이 리소스(`.drf`), Calibre 셀맵, SPICE 모델(`.scs`), DRC/LVS/PEX 룰덱 — 과 시뮬레이션 원시 출력은 모두 제거했습니다.
> 저장소에는 **직접 설계한 스키매틱·심볼·테스트벤치(OpenAccess `.oa`)만** 포함되며, 파운드리 소자는 **이름으로만** 참조됩니다.
> PDK가 없으므로 Cadence에서 그대로 재오픈되지 않는 **아카이브/포트폴리오용** 스냅샷입니다.

---

## 1. 배경

SAR(Successive Approximation Register) ADC는 비트마다 아날로그 시행을 한 번씩 반복해 입력을 디지털로 변환합니다 — 입력을 샘플링하고, DAC 기준값과 비교한 뒤, 비트 단위로 DAC 코드를 좁혀 갑니다. 비교기·커패시터 DAC·SAR 로직만 있으면 되고 단계마다 증폭기가 필요 없어, 저전력·중해상도 데이터 변환에서 널리 쓰입니다. 달성 정확도는 두 아날로그 블록이 좌우하며, 본 프로젝트는 그 둘을 설계합니다.

- **샘플링 스위치**: 일반 MOS 스위치는 온-저항이 신호에 따라 변해 트래킹 중 고조파 왜곡을 유발합니다. **부트스트랩 스위치**는 입력 스윙 전 구간에서 V_GS ≈ V_DD 를 일정하게 유지해 R_on 을 거의 고정 → 샘플링 선형성(ENOB) 향상.
- **커패시터 DAC(C-DAC)**: 전하 재분배로 각 비트 시행의 기준 전압을 만듭니다. 매칭과 총 커패시턴스가 면적/전력과 상충하며, **스플릿(브릿지 커패시터) 배열**은 동일 해상도에서 총/단위 커패시턴스 비를 줄입니다.
- 모든 블록은 Cadence Virtuoso에서 **스키매틱 레벨**로 설계·시뮬레이션했으며, 동적 비교기와 2단 OP앰프를 함께 구현했습니다.

---

## 2. 시스템 구성

```
 Vin ─►[ 샘플링 스위치 ]─►[   C-DAC   ]─►[  비교기  ]─► 비트 판정
        Bootstrap / TG      Conv/Split      Dynamic        │
              ▲                                             │
              └────────── DAC 코드 피드백 (SAR 로직*) ───────┘
                          * 디지털 SAR 로직은 본 저장소 범위 밖
```

| 구성 요소                         | 역할                                                                   |
| --------------------------------- | ---------------------------------------------------------------------- |
| 샘플링 스위치 (Bootstrap / TG)    | Vin 을 C-DAC 상판에 트랙-앤-홀드, 온-저항을 거의 일정하게 유지           |
| C-DAC (Conv / Split / Diff-split) | 이진 가중 전하 재분배 DAC — 비트별 기준 전압 생성                        |
| 동적 비교기 (`COMP`)              | 매 비트 시행마다 DAC 출력 vs 공통모드를 비교하는 클럭 래치               |
| 2단 OP앰프 (`2stage_Op_amp`)      | 기준/보조 증폭기 (버퍼, 이득)                                           |

> 디지털 SAR 제어 로직(축차근사 레지스터)은 **본 저장소 범위 밖**이며, 아날로그 프런트엔드와 DAC에 집중합니다.

---

## 3. 회로 아키텍처

### 3.1 샘플링 프런트엔드 — 부트스트랩 스위치 · TG

![부트스트랩 스위치 스키매틱: M0–M9 + 부트 커패시터 C0, Clks/Clksb 제어, IN→OUT 샘플링 경로](documents/figures/fig1_bootstrapped_switch.png)

- **부트스트랩 스위치** (`SWITCH/BSSW`) — Razavi 구조: 오프 구간에 부트 커패시터 C0 를 V_DD 로 충전하고 트래킹 중 입력 위에 올려, 샘플링 소자가 일정한 V_GS 를 보게 합니다. → 입력 무관 온-저항, 높은 선형성.
- **트랜스미션 게이트** (`SWITCH/TG`) — NMOS+PMOS 병렬 CMOS 패스게이트, 비교용 기본 샘플링 스위치 *(구 `CJH` 라이브러리에서 병합)*.

### 3.2 전하 재분배 DAC — C-DAC

![6-bit 스플릿 C-DAC: C_dum / C_lsb0..2 / C_bridge / C_msb0..2, 단위 커패시터 3.38 fF pip-cap](documents/figures/fig5_cdac_split_6bit.png)

- **기본 C-DAC** (`CDAC/CONV_CDAC`) — 이진 가중 커패시터 배열.
- **스플릿 C-DAC** (`CDAC/SPLIT_CDAC`) — **브릿지 커패시터**로 배열을 MSB/LSB 서브 배열로 분할해 동일 해상도에서 총 커패시턴스를 줄입니다. **6-bit** (3-bit MSB + 3-bit LSB); 단위 커패시터 **3.38 fF** pip-cap (2.6 µm × 2.6 µm).
- **차동 스플릿 C-DAC** (`CDAC/DIFF_SPLIT_CDAC`) — 스플릿 배열의 차동(V_op / V_on) 구조로 PSRR·선형성 개선.

### 3.3 비교기 · OP앰프

![동적 래치 비교기 스키매틱](documents/figures/fig2_comparator.png)

- **동적 비교기** (`CDAC/COMP`) — 매 비트 시행마다 한 번 판정하는 클럭 구동 동적 래치.
- **2단 OP앰프** (`2stage_Op_amp`) — 차동 입력단 + 출력단, 기준/버퍼용.

**설계 사양**

| 항목                     | 사양                                                                                  |
| ------------------------ | ------------------------------------------------------------------------------------- |
| 설계 도구                | Cadence Virtuoso (Schematic + ADE Spectre), OpenAccess DB                              |
| 검증                     | Spectre 트랜지언트 (ADE state 포함), Calibre DRC / LVS / PEX 런셋                      |
| 공정 (참조만)            | ETRI/NSPL 0.5 µm Analog CMOS PDK (설계 라이브러리); HL18G 0.18 µm, TSMC 0.18 µm 셋업   |
| C-DAC 해상도             | 6-bit 스플릿 (브릿지 커패시터 기준 3-bit MSB + 3-bit LSB)                              |
| 단위 커패시터            | 3.38 fF pip-cap (2.6 µm × 2.6 µm)                                                      |
| 샘플링 스위치            | 부트스트랩 (Razavi) + CMOS 트랜스미션 게이트                                           |
| 주요 지표                | 스위치 온-저항 (`TB_R_ON`), 샘플링 선형성 / ENOB (`TB_BSSW_2`, `TB_DIFF_BSSW`)         |
| 설계 수준                | 스키매틱 + 심볼 + 테스트벤치 (레이아웃 / GDS 없음)                                     |

**셀 맵** (`design/ETRI/`)

| 그룹              | 셀                                                                                              |
| ----------------- | ----------------------------------------------------------------------------------------------- |
| 샘플링 스위치     | `SWITCH/BSSW` (부트스트랩), `SWITCH/TG` (트랜스미션 게이트)                                       |
| C-DAC             | `CDAC/CONV_CDAC`, `CDAC/SPLIT_CDAC`, `CDAC/DIFF_SPLIT_CDAC`                                       |
| 비교기 / 증폭기   | `CDAC/COMP` (동적 비교기), `2stage_Op_amp`                                                        |
| 테스트벤치        | `TB_BSSW`, `TB_BSSW_2`, `TB_DIFF_BSSW`, `TB_R_ON`, `TB_TG`, `TB_CONV_CDAC_2`, `TB_SPLIT_CDAC`, `TB_SPLIT_CDAC_2`, `TB_DIFF_SPLIT_CDAC` |

---

## 4. 디렉토리 구조

```
SAR-ADC/
├── design/                          # Cadence Virtuoso 설계 (OpenAccess)
│   ├── cds.lib / .cdsinit / .cshrc / .libmgr   # 라이브러리·환경 설정
│   ├── calview.cellmap                          # Calibre 뷰 매핑 (analogLib)
│   ├── DRC_runset / LVS_runset / PEX_runset     # 물리 검증 런셋
│   ├── ETRI/                        # 설계 라이브러리 모음
│   │   ├── 2stage_Op_amp/           #   2단 연산증폭기
│   │   ├── CDAC/                    #   커패시터 DAC + 비교기
│   │   │   ├── COMP/                #     동적 비교기
│   │   │   ├── CONV_CDAC/           #     기본 C-DAC
│   │   │   ├── SPLIT_CDAC/          #     스플릿 C-DAC
│   │   │   ├── DIFF_SPLIT_CDAC/     #     차동 스플릿 C-DAC
│   │   │   └── TB_*/                #     테스트벤치 (Spectre state)
│   │   └── SWITCH/                  #   샘플링 스위치
│   │       ├── BSSW/                #     부트스트랩 스위치
│   │       ├── TG/                  #     트랜스미션 게이트 (구 CJH 병합)
│   │       └── TB_*/                #     테스트벤치 (Spectre state)
│   └── TSMC180nm/                   # TSMC 0.18 µm 라이브러리 셋업 (경로 참조만)
│
└── documents/
    ├── figures/                     # 블록 회로 그림 (fig1~6)
    ├── papers/                      # 참고 논문 · 설계 보고서
    └── Presentation/                # 발표자료 · 포스터 (PPTX)
```

---

## 5. 시작하기

> ⚠️ 본 저장소는 **IP를 제거한 아카이브**입니다 — PDK·모델·룰덱이 포함되지 않아 셀이 Cadence에서 소자 완결 상태로 열리지 않습니다. 아래 절차는 호환 PDK를 로컬에 직접 갖춘 경우를 가정합니다.

### 요구 사항

- Cadence Virtuoso (IC6.1.8 / ICADV) + Spectre 시뮬레이터
- 호환 CMOS PDK — NSPL 0.5 µm Analog CMOS (설계 라이브러리), 또는 HL18G / TSMC 0.18 µm (**미포함**)
- Calibre (선택) — DRC / LVS / PEX

### 라이브러리 설정

```tcl
# design/ETRI/cds.lib — DEFINE 경로를 로컬에 맞추고
# 사용하는 PDK 의 cds.lib 를 INCLUDE
INCLUDE <your-pdk>/cds.lib
DEFINE  CDAC    <path>/ETRI/CDAC
DEFINE  SWITCH  <path>/ETRI/SWITCH
```

```bash
cd design
virtuoso &          # Library Manager 실행
```

### 시뮬레이션 (ADE Spectre)

1. `TB_*` 셀의 `schematic` 뷰를 엽니다.
2. **ADE** 실행 → *Session ▸ Load State* → 저장된 `spectre_state*` 선택.
3. *Netlist and Run*. ENOB / 온-저항 출력은 저장된 state에 이미 세팅돼 있습니다
   (`SWITCH/TB_BSSW_2`, `SWITCH/TB_DIFF_BSSW` 는 ENOB 파형 셋업 포함, `SWITCH/TB_R_ON` 은 온-저항 측정).

### 물리 검증 (Calibre)

`design/DRC_runset` / `LVS_runset` / `PEX_runset` 은 Calibre 런셋 템플릿입니다 — 실행 전 룰덱·레이아웃 경로를 각자 환경으로 다시 지정하세요. *(레이아웃 / GDS 는 본 아카이브에 없습니다.)*

---

## 6. 개발 현황

**현재 상태 — 스키매틱 레벨 설계·시뮬레이션 완료.** 핵심 아날로그 블록(부트스트랩 스위치, 트랜스미션 게이트, 동적 비교기, 기본/스플릿/차동 스플릿 C-DAC, 2단 OP앰프)을 스키매틱 레벨로 설계하고 블록별 Spectre 테스트벤치를 구성했습니다. 샘플링 선형성(ENOB)·스위치 온-저항 분석은 ADE에 세팅돼 있습니다. 물리 레이아웃 / GDS 는 범위 밖입니다.

### 샘플링 스위치

- [X] 부트스트랩 스위치 (`SWITCH/BSSW`) — Razavi 구조, 스키매틱 + 심볼
- [X] 트랜스미션 게이트 (`SWITCH/TG`) — CMOS 패스게이트 (구 `CJH` 라이브러리에서 병합)
- [X] `TB_R_ON` — 온-저항 측정
- [X] `TB_BSSW` / `TB_BSSW_2` / `TB_DIFF_BSSW` — ENOB / 선형성 스윕 (단일 + 차동)
- [X] `TB_TG` — 트랜스미션 게이트 테스트벤치

### C-DAC

- [X] `CONV_CDAC` — 이진 가중 배열 (스키매틱 + 심볼)
- [X] `SPLIT_CDAC` — 6-bit 브릿지-캡 스플릿 배열, 3.38 fF 단위 pip-cap
- [X] `DIFF_SPLIT_CDAC` — 차동 스플릿 배열 (V_op / V_on)
- [X] `TB_CONV_CDAC_2` / `TB_SPLIT_CDAC` / `TB_SPLIT_CDAC_2` / `TB_DIFF_SPLIT_CDAC`

### 비교기 · 증폭기

- [X] `COMP` — 클럭 구동 동적 래치 비교기
- [X] `2stage_Op_amp` — 2단 OP앰프 (스키매틱 + 심볼)

### 문서

- [X] 블록 스키매틱 캡처 (`documents/figures/` fig1~6)
- [X] 설계 보고서 (`documents/papers/`)
- [X] 발표자료 + 포스터 (`documents/Presentation/`)

### 범위 밖

- [ ] 디지털 SAR 제어 로직 / 레지스터
- [ ] 물리 레이아웃, DRC/LVS 클린 GDS, PEX 백어노테이션

---

## 7. 기타

### 문서

| 경로                        | 내용                                                          |
| --------------------------- | ------------------------------------------------------------- |
| `documents/figures/`        | 블록 스키매틱 — 부트스트랩 스위치·비교기·C-DAC (fig1~6)        |
| `documents/papers/`         | 참고 논문 + 설계 보고서 (PDF)                                 |
| `documents/Presentation/`   | 발표자료 + 포스터 (PPTX)                                       |

### 설계 노트

- **IP 제거**: `.oa` 셀에는 파운드리 소자 **이름**(`nch`/`pch`, `nmos4`/`pmos4`, `pipcap` 등)과 공정 식별자만 남아 있고, 모델 파라미터·레이아웃·룰덱은 포함되지 않습니다.
- **레이아웃 / GDS 없음**: 스키매틱 + 시뮬레이션 아카이브입니다.
- **셀은 기능 기준으로 명명**했으며, 트랜스미션 게이트는 구 `CJH` 라이브러리에서 병합했습니다.
- **PDK 셋업 3종**이 경로로 참조됩니다 (`design/ETRI` → NSPL 0.5 µm, `design/cds.lib` → HL18G 0.18 µm, `design/TSMC180nm` → TSMC 0.18 µm). 실제 설계 라이브러리는 `ETRI/` 입니다.

---

## 8. 참고 문헌

- B. Razavi, *"The Design of a Bootstrapped Switch Circuit,"* IEEE Solid-State Circuits Magazine — 부트스트랩 스위치 설계 참고 *(`documents/papers/`, 제3자 저작물로 로컬 보관)*
- *"SAR ADC용 C-DAC 및 Bootstrap Switch 설계"* — 프로젝트 설계 보고서 *(`documents/papers/`)*
- 스플릿 / 브릿지 커패시터 DAC 및 SAR ADC 프런트엔드 설계 문헌
