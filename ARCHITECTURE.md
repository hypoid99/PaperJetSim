# Architecture — PaperJetSim

## 프로젝트 개요

종이비행기 물리 시뮬레이션 기반 데스크톱 게임.
발사 각도·힘·바람 등 환경 변수를 조절해 최장거리를 겨루는 게임형 시뮬레이터.

---

## 기술 스택

| 레이어 | 기술 | 선택 이유 |
|--------|------|-----------|
| Language | Python 3.11+ | 빠른 프로토타이핑, 풍부한 과학 라이브러리 |
| 3D 렌더링 | Pyglet 2.x + PyOpenGL | OpenGL 기반 3D, Python 네이티브 데스크톱 |
| 물리 엔진 | 자체 구현 (NumPy 기반) | 종이비행기 공기역학 커스텀 모델링 필요 |
| UI/HUD | Pyglet Label + 커스텀 위젯 | 게임 HUD 레이어 직접 구성 |
| 데이터 저장 | SQLite (sqlite3 내장) | 로컬 기록 저장, 의존성 없음 |
| 설정 관리 | JSON (configparser) | 게임 설정, 키 바인딩 저장 |
| 수치 계산 | NumPy | 벡터 연산, 물리 시뮬레이션 |

---

## 시스템 다이어그램

```
┌─────────────────────────────────────────────────────┐
│                   PaperJetSim App                   │
│                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌───────────┐ │
│  │  UI/HUD  │◄──►│  Game State  │◄──►│  Physics  │ │
│  │  Layer   │    │  Manager     │    │  Engine   │ │
│  └──────────┘    └──────┬───────┘    └───────────┘ │
│                         │                           │
│  ┌──────────┐    ┌──────▼───────┐    ┌───────────┐ │
│  │  3D      │◄──►│  Scene       │◄──►│  Record   │ │
│  │  Renderer│    │  Manager     │    │  Manager  │ │
│  └──────────┘    └──────────────┘    └───────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              SQLite (local DB)                │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 주요 모듈

| 모듈 | 위치 | 역할 |
|------|------|------|
| `main.py` | `/` | 진입점, Pyglet 윈도우 초기화 |
| `game/state.py` | `/game/` | 게임 상태 머신 (메뉴/발사준비/비행/결과) |
| `game/scene.py` | `/game/` | 3D 씬 구성, 카메라 제어 |
| `physics/airplane.py` | `/physics/` | 종이비행기 비행체 모델 |
| `physics/aerodynamics.py` | `/physics/` | 양력·항력·바람 계산 |
| `physics/environment.py` | `/physics/` | 중력, 바람, 공기밀도 환경 변수 |
| `render/renderer3d.py` | `/render/` | PyOpenGL 3D 렌더링 파이프라인 |
| `render/hud.py` | `/render/` | HUD (속도계, 각도, 거리 표시) |
| `render/trail.py` | `/render/` | 비행 궤적 라인 렌더링 |
| `ui/launch_panel.py` | `/ui/` | 발사 각도/힘 조절 UI |
| `ui/wind_panel.py` | `/ui/` | 바람/환경 설정 패널 |
| `ui/menu.py` | `/ui/` | 메인 메뉴, 기록판 |
| `data/record_manager.py` | `/data/` | SQLite 최장거리 기록 저장/조회 |
| `data/config.py` | `/data/` | 게임 설정 로드/저장 |
| `assets/` | `/assets/` | 3D 모델, 텍스처, 사운드 |

---

## 물리 모델 개요

종이비행기는 단순화된 공기역학 모델을 사용한다.

```
양력(Lift)  = 0.5 × ρ × v² × CL × A
항력(Drag)  = 0.5 × ρ × v² × CD × A
무게(Weight) = m × g

ρ = 공기밀도 (환경 변수로 조절)
v = 비행속도 (벡터)
CL = 양력계수 (받음각에 따라 변화)
CD = 항력계수 (받음각에 따라 변화)
A  = 날개 면적
m  = 비행기 질량
```

- 받음각(Angle of Attack)에 따라 CL/CD가 비선형 변화
- 바람은 속도 벡터에 더해지는 외력으로 처리
- 시뮬레이션 스텝: 60fps (dt = 1/60s)

---

## 게임 상태 머신

```
MENU ──► SETUP (발사 준비) ──► FLYING (비행 중) ──► RESULT (결과)
  ▲                                                      │
  └──────────────────────────────────────────────────────┘
```

---

## 디렉토리 구조

```
PaperJetSim/
├── main.py
├── requirements.txt
├── game/
│   ├── state.py
│   └── scene.py
├── physics/
│   ├── airplane.py
│   ├── aerodynamics.py
│   └── environment.py
├── render/
│   ├── renderer3d.py
│   ├── hud.py
│   └── trail.py
├── ui/
│   ├── launch_panel.py
│   ├── wind_panel.py
│   └── menu.py
├── data/
│   ├── record_manager.py
│   └── config.py
├── assets/
│   ├── models/
│   ├── textures/
│   └── sounds/
└── docs/
    ├── design-docs/
    ├── exec-plans/
    ├── product-specs/
    └── references/
```

---

## 환경 구성

- **개발**: Python venv + requirements.txt
- **실행**: `python main.py`
- **패키징**: PyInstaller로 단일 실행파일(.exe) 번들링 (Phase 2)

---

## 외부 의존성

```
pyglet>=2.0
PyOpenGL>=3.1
numpy>=1.24
```
