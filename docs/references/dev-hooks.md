# Dev Hooks 설계 — PaperJetSim

코드 수정 후 자동으로 실행되는 훅 파이프라인 전체를 정의한다.

---

## 전체 실행 순서

코드를 수정하고 `git commit`을 실행하면 아래 순서로 자동 실행된다.
**하나라도 실패하면 커밋이 중단된다.**

```
git commit
    │
    ▼
[STEP 1] ruff --fix        ← 불필요한 코드 자동 정리
    │   실패 시 → 커밋 중단, 수정 필요 항목 출력
    ▼
[STEP 2] vulture           ← 죽은 코드(호출 안 되는 함수) 탐지
    │   실패 시 → 커밋 중단, 죽은 코드 목록 출력
    ▼
[STEP 3] pytest            ← 단위 테스트 실행
    │   실패 시 → 커밋 중단, 실패 케이스 출력
    ▼
[STEP 4] 커밋 허용 ✅
```

---

## 설정 파일: `.pre-commit-config.yaml`

프로젝트 루트에 아래 파일을 생성한다.

```yaml
# .pre-commit-config.yaml
repos:
  # STEP 1: 코드 자동 정리 (ruff)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]          # 자동 수정 적용
        name: "코드 정리 (ruff)"
      - id: ruff-format
        name: "포맷 통일 (ruff-format)"

  # STEP 2: 죽은 코드 탐지 (vulture)
  - repo: local
    hooks:
      - id: vulture
        name: "죽은 코드 탐지 (vulture)"
        entry: vulture . --min-confidence 80
        language: system
        types: [python]
        pass_filenames: false

  # STEP 3: 단위 테스트 (pytest)
  - repo: local
    hooks:
      - id: pytest
        name: "단위 테스트 (pytest)"
        entry: pytest physics/ data/ ui/ -q --tb=short
        language: system
        pass_filenames: false
```

---

## ruff 세부 설정: `ruff.toml`

```toml
# ruff.toml
[lint]
select = [
    "F",    # pyflakes — 미사용 import, 미사용 변수
    "E",    # pycodestyle — 스타일 오류
    "W",    # pycodestyle — 경고
    "I",    # isort — import 순서
    "UP",   # pyupgrade — Python 버전별 최신 문법
]
ignore = [
    "E501", # 줄 길이 제한 무시 (긴 수식 허용)
]

[lint.per-file-ignores]
"tests/*" = ["F811"]   # 테스트 파일에서 재정의 허용

[format]
indent-width = 4
quote-style = "double"
```

---

## vulture 예외 처리: `whitelist.py`

vulture가 "죽은 코드"로 잘못 탐지하는 항목을 예외 등록한다.

```python
# whitelist.py  (vulture 예외 목록)

# Pyglet 이벤트 핸들러 — 직접 호출하지 않지만 프레임워크가 호출
_.on_draw          # used by pyglet
_.on_key_press     # used by pyglet
_.on_mouse_drag    # used by pyglet
_.on_close         # used by pyglet

# Observer 훅 — 테스트에서만 등록
_.add_observer     # used in tests
```

---

## 초기 설치 방법

프로젝트 처음 세팅 시 1회 실행한다.

```bash
# 1. 가상환경 생성 및 활성화
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 2. 의존성 설치
pip install pre-commit ruff vulture pytest
pip install -r requirements.txt

# 3. pre-commit 훅 등록 (이후 자동 실행)
pre-commit install

# 4. 훅 동작 확인 (전체 파일 대상 1회 테스트)
pre-commit run --all-files
```

---

## 수동 실행 명령어

커밋 없이 개발 중에 수동으로 각 단계를 실행할 때 사용한다.

```bash
# 코드 정리만 실행
ruff check . --fix
ruff format .

# 죽은 코드 탐지만 실행
vulture . whitelist.py --min-confidence 80

# 테스트만 실행
pytest physics/ -v                  # 물리 엔진만
pytest data/ -v                     # 기록 시스템만
pytest -v                           # 전체

# 전체 훅 수동 실행 (커밋 없이)
pre-commit run --all-files
```

---

## 훅 실행 결과 예시

### 성공 케이스

```
코드 정리 (ruff)..............................................Passed
포맷 통일 (ruff-format).......................................Passed
죽은 코드 탐지 (vulture).....................................Passed
단위 테스트 (pytest)..........................................Passed

✅ 모든 훅 통과 → 커밋 허용
```

### 실패 케이스 — 미사용 import 발견

```
코드 정리 (ruff)..............................................Failed
- physics/aerodynamics.py:3:1: F401 'math.sqrt' imported but unused
  → ruff --fix 로 자동 수정됨, 수정 파일을 다시 git add 후 커밋

❌ 커밋 중단
```

### 실패 케이스 — 죽은 코드 발견

```
죽은 코드 탐지 (vulture).....................................Failed
- physics/airplane.py:87: unused function 'debug_print' (80% confidence)
  → 실제로 안 쓰는 함수면 삭제, 의도적이면 whitelist.py에 등록

❌ 커밋 중단
```

### 실패 케이스 — 테스트 실패

```
단위 테스트 (pytest)..........................................Failed
FAILED physics/test_aerodynamics.py::test_tc_p06_stall
  AssertionError: CL at 25° (1.18) is not less than CL_max * 0.5 (0.60)
  → 실속 모델 구현 수정 필요

❌ 커밋 중단
```

---

## 향후 기능 추가 시 훅 확장 방법

새 기능을 추가할 때 `.pre-commit-config.yaml`에 단계를 추가한다.

```yaml
# 예시: 타입 검사 추가 (Phase 2 이후)
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: v1.9.0
  hooks:
    - id: mypy
      name: "타입 검사 (mypy)"
      args: [--ignore-missing-imports]
```

현재 계획된 추가 예정 훅:

| 훅 | 추가 시점 | 목적 |
|----|-----------|------|
| `mypy` | Phase 2 | 타입 힌트 검사 |
| `pytest --cov` | Phase 2 | 커버리지 리포트 |
| `pyinstaller --dry-run` | Phase 2 완료 시 | 패키징 가능 여부 확인 |
