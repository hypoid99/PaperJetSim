# 물리 엔진 스펙 — PaperJetSim

## 개요

종이비행기의 실제 비행을 단순화된 공기역학 모델로 시뮬레이션한다.
수치적 정확성보다 "게임적 재미"와 "직관적 피드백"을 우선한다.

---

## 물리 변수

### 비행기 고정 파라미터 (v1.0 기본값)

| 파라미터 | 기호 | 기본값 | 단위 |
|---------|------|--------|------|
| 질량 | m | 0.005 | kg (5g) |
| 날개 면적 | A | 0.04 | m² |
| 날개 가로세로비 | AR | 4.0 | - |
| 최소 항력계수 | CD₀ | 0.05 | - |
| 최대 양력계수 | CL_max | 1.2 | - |
| 실속각 | α_stall | 15° | deg |

### 환경 파라미터 (사용자 조절 가능)

| 파라미터 | 기호 | 범위 | 기본값 |
|---------|------|------|--------|
| 공기밀도 | ρ | 0.5 ~ 1.5 | 1.225 kg/m³ |
| 중력가속도 | g | 1.0 ~ 20.0 | 9.81 m/s² |
| 바람 속도 | v_w | -10 ~ +10 | 0 m/s |
| 바람 방향 | θ_w | 0 ~ 360 | 0° |

---

## 양력/항력 모델

### 받음각(AoA)에 따른 계수 변화

```
CL(α) = CL_max × sin(2α)          (α < α_stall)
CL(α) = CL_max × (α_stall - α)/10 (α ≥ α_stall, 실속 구간)

CD(α) = CD₀ + CL² / (π × AR × e)  (오즈왈드 효율 e = 0.8)
```

### 매 시뮬레이션 스텝 (dt = 1/60s)

```python
# 속도 벡터 기준 받음각 계산
alpha = atan2(-vy, vx) + pitch_angle

# 힘 계산
lift = 0.5 * rho * speed² * CL(alpha) * A
drag = 0.5 * rho * speed² * CD(alpha) * A
weight = m * g

# 가속도 = (합력) / m
ax = (-drag * cos(theta) + lift * sin(theta)) / m + wind_x
ay = (-drag * sin(theta) - lift * cos(theta) - weight) / m + wind_y

# 속도/위치 업데이트 (Euler integration)
vx += ax * dt
vy += ay * dt
x  += vx * dt
y  += vy * dt
```

---

## 종료 조건

- `y < 0` (지면 충돌) → 비행 종료, 착지 지점 X가 최종 거리
- `speed < 0.5 m/s` (실속 후 낙하) → 비행 종료

---

## 구현 파일

- `physics/airplane.py` — PaperAirplane 클래스 (상태, 업데이트)
- `physics/aerodynamics.py` — CL/CD 함수, 힘 계산
- `physics/environment.py` — 환경 변수 컨테이너
