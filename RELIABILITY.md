# Reliability — PaperJetSim

코드의 안정성, 메모리 관리, 에러 처리 정책을 정의한다.
이 문서의 규칙은 모든 모듈에 적용된다.

---

## 1. 코드 품질 자동화 정책

### 원칙
> 코드가 수정될 때마다 불필요한 코드가 자동으로 정리된다.
> 정리되지 않은 코드는 커밋할 수 없다.

### 사용 도구

| 도구 | 역할 | 실행 시점 |
|------|------|-----------|
| `ruff --fix` | 미사용 import 제거, 코드 포맷 정리, 스타일 통일 | pre-commit (1순위) |
| `vulture` | 호출되지 않는 함수/클래스/변수 탐지 | pre-commit (2순위) |
| `pytest` | 단위 테스트 실행 | pre-commit (3순위) |

### 정리 범위 (ruff 기준)

- 미사용 import (`F401`)
- 미사용 변수 (`F841`)
- 정의됐지만 호출 안 되는 함수 → vulture가 별도 탐지
- 중복 코드 → 수동 리뷰 (자동화 범위 밖)

### 예외 허용

```python
# noqa: F401  ← 의도적으로 남긴 import는 주석으로 명시
import pyglet.gl  # noqa: F401  -- OpenGL 컨텍스트 초기화용
```

---

## 2. 에러 처리 정책

### 물리 엔진

수치 계산 중 비정상 값이 발생하면 시뮬레이션을 즉시 중단하고 안전 상태로 복귀한다.

```python
# physics/airplane.py 에서 반드시 적용
def update(self, dt: float) -> None:
    self._step(dt)
    if not self._is_valid_state():
        self._emergency_land()   # y=0, velocity=0으로 강제 착지
        logger.error(f"물리 수치 이상 감지: x={self.x}, y={self.y}, vx={self.vx}, vy={self.vy}")

def _is_valid_state(self) -> bool:
    values = [self.x, self.y, self.vx, self.vy]
    return all(math.isfinite(v) for v in values)
```

### UI / 입력

사용자 입력은 모두 범위 클램핑 후 사용한다. 예외를 발생시키지 않는다.

```python
# 잘못된 방식
def set_angle(self, value):
    if value < -10 or value > 45:
        raise ValueError("각도 범위 초과")  # ❌

# 올바른 방식
def set_angle(self, value):
    self.angle = max(-10, min(45, value))   # ✅ 클램핑
```

### 데이터 / 기록

DB 작업은 항상 try-except로 감싸고, 실패 시 게임 진행에 영향을 주지 않는다.

```python
def save_record(self, record: FlightRecord) -> bool:
    try:
        self._db.execute(INSERT_SQL, record.to_tuple())
        self._db.commit()
        return True
    except sqlite3.Error as e:
        logger.warning(f"기록 저장 실패 (게임 계속 진행): {e}")
        return False   # 게임은 중단하지 않음
```

---

## 3. OpenGL 메모리 관리 정책

Python GC는 GPU 메모리를 수거하지 못한다. 아래 규칙을 반드시 따른다.

### 규칙: 생성한 곳에서 삭제한다

```python
# render/trail.py
class TrailRenderer:
    def __init__(self):
        self.vbo = glGenBuffers(1)      # 생성

    def reset(self):
        """재발사 시 호출 — 기존 버퍼 명시 삭제 후 재생성"""
        glDeleteBuffers(1, [self.vbo])  # ✅ 명시 삭제
        self.vbo = glGenBuffers(1)      # 재생성

    def cleanup(self):
        """앱 종료 시 호출"""
        glDeleteBuffers(1, [self.vbo])  # ✅ 명시 삭제
```

### 리소스별 삭제 시점

| 리소스 | 삭제 시점 | 사용 함수 |
|--------|-----------|-----------|
| VBO (궤적 버퍼) | 재발사 시, 앱 종료 시 | `glDeleteBuffers()` |
| VAO (비행기 모델) | 씬 초기화 시, 앱 종료 시 | `glDeleteVertexArrays()` |
| 텍스처 | 씬 전환 시, 앱 종료 시 | `texture.delete()` |
| Pyglet 배치 | 씬 전환 시 | `batch.invalidate()` |

### 앱 종료 시 정리 순서

```python
# main.py
def on_close(self):
    self.trail_renderer.cleanup()
    self.scene_renderer.cleanup()
    self.hud.cleanup()
    super().on_close()
```

---

## 4. 로깅 정책

```python
import logging
logger = logging.getLogger(__name__)

# 레벨 사용 기준
logger.debug(...)    # 매 스텝 물리값 (개발 중에만 활성화)
logger.info(...)     # 게임 상태 전환, 기록 저장 성공
logger.warning(...)  # 기록 저장 실패, 비정상 입력 클램핑
logger.error(...)    # 물리 수치 이상, OpenGL 초기화 실패
```

개발 모드: `DEBUG` 레벨 활성화
릴리즈 모드: `INFO` 레벨부터만 출력

---

## 5. 향후 기능 추가 시 체크리스트

새 모듈을 추가하거나 기존 모듈을 수정할 때 반드시 확인한다.

- [ ] ruff + vulture 통과 확인
- [ ] 새 OpenGL 리소스는 `cleanup()` 메서드에 삭제 코드 추가
- [ ] 새 UI 입력값은 범위 클램핑 적용
- [ ] 새 DB 작업은 try-except 적용
- [ ] QUALITY_SCORE.md에 해당 모듈 테스트 항목 추가
- [ ] test-cases.md에 새 TC 추가
