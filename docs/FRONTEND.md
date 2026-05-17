# Frontend (렌더링 컨벤션) — PaperJetSim

Pyglet + PyOpenGL 기반 렌더링 코드 작성 규칙과 컨벤션을 정의한다.
웹 프론트엔드가 아닌 게임 렌더링 레이어에 대한 가이드다.

---

## 1. 렌더링 레이어 구조

렌더링은 3개의 독립된 레이어로 나뉜다. 각 레이어는 서로 직접 참조하지 않는다.

```
Layer 3 (최상위): HUD — Pyglet Label, 2D 오버레이
Layer 2 (중간):   UI 패널 — 슬라이더, 버튼 (SETUP 상태에서만 표시)
Layer 1 (하위):   3D 씬 — OpenGL, 비행기, 궤적, 지형
```

렌더링 순서: Layer 1 → Layer 2 → Layer 3 (뒤에서 앞으로)

---

## 2. OpenGL 코드 규칙

### 상태 관리

OpenGL 상태는 사용 후 반드시 원래대로 복원한다.

```python
# ✅ 올바른 방식 — 상태 보존
glPushAttrib(GL_ALL_ATTRIB_BITS)
glEnable(GL_BLEND)
# ... 렌더링 ...
glPopAttrib()

# ❌ 잘못된 방식 — 다음 렌더러에 영향
glEnable(GL_BLEND)
# ... 렌더링 ...
# 복원 안 함
```

### VAO/VBO 네이밍

```python
self.terrain_vao = glGenVertexArrays(1)   # 용도_vao
self.trail_vbo   = glGenBuffers(1)        # 용도_vbo
self.plane_vbo   = glGenBuffers(1)
```

### 좌표계

- X축: 오른쪽 (+) / 왼쪽 (-)
- Y축: 위 (+) / 아래 (-)
- Z축: 화면 밖 (+) / 화면 안 (-) — OpenGL 기본

비행기는 X 방향으로 날아간다. Z축은 측풍 편향에만 사용.

---

## 3. 카메라 구현 규칙

카메라 행렬은 매 프레임 새로 계산한다. 캐싱하지 않는다.

```python
class Camera:
    def get_view_matrix(self) -> np.ndarray:
        """매 프레임 호출. 캐싱 없음."""
        return look_at(self.position, self.target, self.up)

    def follow(self, airplane: PaperAirplane) -> None:
        """추적 카메라: 비행기 뒤 고정 오프셋"""
        self.position = airplane.position + FOLLOW_OFFSET
        self.target   = airplane.position
```

---

## 4. HUD 컨벤션

HUD는 Pyglet의 2D 레이어를 사용한다. OpenGL을 직접 쓰지 않는다.

```python
# ✅ HUD는 pyglet.text.Label 사용
self.distance_label = pyglet.text.Label(
    "0.0m",
    font_size=28,
    bold=True,
    color=(255, 255, 255, 255),
    x=20, y=window.height - 40,
    anchor_x="left", anchor_y="top",
)

# 업데이트는 text 속성만 변경 (Label 재생성 금지)
self.distance_label.text = f"{distance:.1f}m"
```

---

## 5. 업데이트 루프 규칙

물리 업데이트와 렌더링을 분리한다.

```python
# main.py
PHYSICS_DT = 1 / 60   # 물리: 고정 60Hz

def update(self, dt: float) -> None:
    """물리 업데이트 — 고정 스텝"""
    accumulator = self._accumulator + dt
    while accumulator >= PHYSICS_DT:
        self.airplane.update(PHYSICS_DT)
        accumulator -= PHYSICS_DT
    self._accumulator = accumulator

def on_draw(self) -> None:
    """렌더링 — 가능한 빠르게 (vsync)"""
    self.clear()
    self.scene.draw()
    self.hud.draw()
```

물리가 느려져도 렌더링은 멈추지 않는다.

---

## 6. 파티클 시스템 규칙

파티클은 최대 개수를 초과하면 오래된 것부터 교체한다 (ring buffer).

```python
MAX_PARTICLES = 200

class ParticleSystem:
    def __init__(self):
        self.particles = [None] * MAX_PARTICLES
        self._head = 0   # ring buffer 포인터

    def emit(self, pos, vel, color, lifetime):
        self.particles[self._head] = Particle(pos, vel, color, lifetime)
        self._head = (self._head + 1) % MAX_PARTICLES
```

---

## 7. 금지 사항

- `on_draw()` 안에서 glGenBuffers / glGenVertexArrays 호출 금지 (매 프레임 할당)
- HUD 업데이트 시 Label 객체 재생성 금지 (`.text` 속성만 변경)
- 물리 계산을 `on_draw()` 안에서 수행 금지 (update와 분리)
- 전역 OpenGL 상태 변경 후 복원 없이 함수 종료 금지
