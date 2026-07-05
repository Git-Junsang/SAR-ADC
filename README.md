# SAR-ADC

> ⚠️ **본 저장소는 IP(파운드리 기밀 자산)를 제거한 버전입니다.**
> 파운드리 PDK 기밀 자산 — 공정 테크놀로지 파일(`.tf`), 디스플레이 리소스(`.drf`), Calibre 셀맵, SPICE 모델(`.scs`), DRC/LVS/PEX 룰덱 — 과 시뮬레이션 원시 출력은 모두 제거했습니다.
> 저장소에는 **직접 설계한 스키매틱·심볼·테스트벤치(OpenAccess `.oa`)만** 포함되며, 파운드리 소자는 이름으로만 참조됩니다.
> PDK가 없으므로 Cadence에서 그대로 재오픈되지 않는 **아카이브/포트폴리오용**입니다.

**SAR ADC용 C-DAC 및 부트스트랩 스위치 설계 (Cadence Virtuoso)**

SAR(Successive Approximation Register) ADC의 핵심 아날로그 블록 — 샘플링 스위치, 커패시터 DAC(C-DAC), 동적 비교기 — 를 스키매틱 레벨로 설계·검증한 프로젝트입니다.
중앙대학교 CMOS 집적회로 수업 프로젝트(2026).

---

## 개요

SAR ADC는 아래 아날로그 경로를 반복(bit-by-bit)하여 입력을 디지털로 변환합니다. 본 프로젝트는 그중 **샘플링 스위치**와 **C-DAC**를 중심으로 설계하고, 비교기·연산증폭기를 함께 구현했습니다.

```
 Vin ─►[ 샘플링 스위치 ]─►[   C-DAC   ]─►[ 비교기 ]─► 비교 결과
        Bootstrap / TG      Conv/Split      Dynamic        │
              ▲                                             │
              └────────── DAC 코드 피드백 (SAR 로직*) ───────┘
                          * 디지털 SAR 로직은 본 저장소 범위 밖
```

| 항목 | 내용 |
| --- | --- |
| 설계 도구 | Cadence Virtuoso (Schematic / ADE Spectre), OpenAccess DB |
| 검증 | Spectre 시뮬레이션(ADE state 포함), Calibre DRC/LVS/PEX 런셋 |
| 공정(참조만) | ETRI/NSPL 0.5µm Analog CMOS, HL18G 0.18µm, TSMC 0.18µm 셋업 |
| 설계 범위 | 샘플링 스위치, C-DAC(기본/스플릿/차동스플릿), 동적 비교기, 2단 OP앰프 |

---

## 설계 블록

각 셀은 회로 내용에 맞춰 명명했습니다. (라이브러리: `design/ETRI/`)

| 블록 | 셀 이름 | 뷰 | 설명 |
| --- | --- | --- | --- |
| 부트스트랩 스위치 | `SWITCH/BSSW` | sch+sym | 게이트-소스 전압을 일정하게 유지하는 샘플링 스위치 (선형성↑) |
| 트랜스미션 게이트 | `SWITCH/TG` | sch+sym | NMOS+PMOS 병렬 CMOS 패스게이트 샘플링 스위치 *(구 `CJH` 라이브러리에서 병합)* |
| 동적 비교기 | `CDAC/COMP` | sch | 클럭 구동 동적 래치 비교기 |
| 기본 C-DAC | `CDAC/CONV_CDAC` | sch+sym | 이진 가중 커패시터 배열 DAC |
| 스플릿 C-DAC | `CDAC/SPLIT_CDAC` | sch+sym | 브릿지 커패시터로 배열을 분할해 총 커패시턴스↓ |
| 차동 스플릿 C-DAC | `CDAC/DIFF_SPLIT_CDAC` | sch+sym | 스플릿 C-DAC의 차동 구조 |
| 2단 연산증폭기 | `2stage_Op_amp` | sch+sym | 차동 입력 + 출력단 2단 OP앰프 |

**테스트벤치** (`TB_*`): `TB_BSSW`, `TB_BSSW_2`, `TB_DIFF_BSSW`, `TB_R_ON`(스위치 온-저항 측정), `TB_TG`, `TB_CONV_CDAC_2`, `TB_SPLIT_CDAC`, `TB_SPLIT_CDAC_2`, `TB_DIFF_SPLIT_CDAC` — 각 블록별 Spectre 시뮬레이션 셋업 포함.

---

## 디렉토리 구조

```
SAR-ADC/
├── design/                          # Cadence Virtuoso 설계 (OpenAccess)
│   ├── cds.lib / .cdsinit / .cshrc / .libmgr   # 라이브러리·환경 설정
│   ├── calview.cellmap                          # Calibre 뷰 매핑 (analogLib)
│   ├── DRC_runset / LVS_runset / PEX_runset     # 물리 검증 런셋
│   ├── ETRI/                        # 설계 라이브러리 모음
│   │   ├── 2stage_Op_amp/           # 2단 연산증폭기
│   │   ├── CDAC/                    # 커패시터 DAC + 비교기
│   │   │   ├── COMP/                #   동적 비교기
│   │   │   ├── CONV_CDAC/           #   기본 C-DAC
│   │   │   ├── SPLIT_CDAC/          #   스플릿 C-DAC
│   │   │   ├── DIFF_SPLIT_CDAC/     #   차동 스플릿 C-DAC
│   │   │   └── TB_*/                #   테스트벤치
│   │   └── SWITCH/                  # 샘플링 스위치
│   │       ├── BSSW/                #   부트스트랩 스위치
│   │       ├── TG/                  #   트랜스미션 게이트 (구 CJH 병합)
│   │       └── TB_*/                #   테스트벤치
│   └── TSMC180nm/                   # TSMC 0.18µm 라이브러리 셋업 (경로 참조만)
│
└── documents/
    ├── figures/                     # 블록 회로 그림 (fig1~6)
    ├── papers/                      # 참고 논문 · 설계 보고서
    └── Presentation/                # 발표자료 · 포스터
```

---

## 문서

| 경로 | 내용 |
| --- | --- |
| `documents/figures/` | 부트스트랩 스위치·비교기·C-DAC 등 블록 회로 그림 |
| `documents/papers/` | 참고 논문 및 설계 보고서 (PDF) |
| `documents/Presentation/` | 발표자료·포스터 (PPTX) |

---

## 사용 참고

- 본 저장소는 **PDK/파운드리 IP가 제거**되어 있어 Cadence에서 바로 열리지 않습니다(소자·테크 미참조). 설계 의도·구조 확인용 아카이브입니다.
- `.oa` 파일 내부에는 파운드리 소자 **이름**(`nch`/`pch`, `nmos4`/`pmos4` 등)과 공정 식별자만 남아 있으며, 모델 파라미터·레이아웃·룰덱 등 실제 PDK 콘텐츠는 포함되지 않습니다.
- 레이아웃/GDS는 포함하지 않습니다.

---

## 참고 문헌

- B. Razavi, *"The Design of a Bootstrapped Switch Circuit,"* IEEE Solid-State Circuits Magazine — 부트스트랩 스위치 설계 참고
