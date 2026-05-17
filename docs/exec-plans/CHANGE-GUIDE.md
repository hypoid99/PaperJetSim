# Change Guide — PaperJetSim

코딩 완료 후 기존 기능을 변경할 때 따라야 할 절차.
**반드시 이 순서를 지킨다. 코드를 먼저 수정하지 않는다.**

---

## 기본 원칙

```
문서 수정 → 테스트 케이스 수정 → 코드 수정 → 훅 자동 검증 → 완료 처리
```

문서가 항상 코드보다 먼저 바뀐다. 문서가 진실의 기준이기 때문이다.
코드를 먼저 바꾸면 문서와 코드가 어긋나고, 이후 AI 도구가 문서를 보고
엉뚱한 방향으로 코드를 다시 작성하는 문제가 생긴다.

---

## 전체 변경 절차 (5단계)

### Step 1 — 변경 계획 파일 작성

`docs/exec-plans/active/` 에 변경 계획 파일을 만든다.
파일명: `change-{변경대상}-{날짜}.md`

```
예) change-launch-angle-max-2026-05-17.md
    change-physics-integrator-2026-05-20.md
    change-db-schema-add-preset-2026-05-22.md
```

아래 템플릿을 사용한다 → [변경 계획 템플릿](#변경-계획-템플릿) 참고

---

### Step 2 — 영향 범위 파악

변경 전에 어디까지 영향이 미치는지 확인한다.

```bash
# 변경할 함수/상수가 어디서 참조되는지 확인
grep -rn "ANGLE_MAX" .
grep -rn "set_angle" .
grep -rn "launch_angle" .
```

영향 받는 파일 목록을 Step 1의 계획 파일에 기록한다.

---

### Step 3 — 문서 수정 (코드 전)

영향 받는 문서를 먼저 수정한다.

| 변경 내용 | 수정해야 할 문서 |
|-----------|-----------------|
| 기능 동작 변경 | `docs/product-specs/[기능].md` |
| 수치/범위 변경 | `docs/product-specs/[기능].md` + `docs/references/test-cases.md` |
| DB 컬럼 변경 | `docs/generated/db-schema.md` |
| UI 레이아웃 변경 | `docs/DESIGN.md` |
| 렌더링 방식 변경 | `docs/FRONTEND.md` |
| 물리 모델 변경 | `docs/product-specs/physics.md` |
| 새 기술 부채 발생 | `docs/exec-plans/tech-debt-tracker.md` |

---

### Step 4 — 테스트 케이스 수정

`docs/references/test-cases.md` 에서 영향 받는 TC를 수정한다.

```
변경 전: TC-U01 | set_angle(90°) → 기대값: angle == 45°
변경 후: TC-U01 | set_angle(90°) → 기대값: angle == 60°
         TC-U01 | set_angle(60°) → 기대값: angle == 60°  ← 새 케이스 추가
```

기존 TC를 삭제하지 않는다. 동작이 바뀐 경우 기대값만 수정하고,
새로운 경계값이 생기면 TC를 추가한다.

---

### Step 5 — 코드 수정

문서와 TC가 확정된 후에 코드를 수정한다.

**규칙: 숫자는 상수로, 한 곳에서만 정의한다**

```python
# ❌ 잘못된 방식 — 여러 곳에 숫자 직접 사용
def set_angle(self, v): return max(-10, min(45, v))   # 45 하드코딩
def validate(self, v): return -10 <= v <= 45           # 또 45 하드코딩

# ✅ 올바른 방식 — 상수 한 곳에서 정의
ANGLE_MIN = -10
ANGLE_MAX = 60   # 여기만 바꾸면 전체 반영

def set_angle(self, v): return max(ANGLE_MIN, min(ANGLE_MAX, v))
def validate(self, v):  return ANGLE_MIN <= v <= ANGLE_MAX
```

---

### Step 6 — 훅 자동 검증 (커밋 시 자동 실행)

```bash
git add .
git commit -m "feat: 발사 각도 최대값 45° → 60° 변경"

# 자동 실행:
# [1] ruff --fix   → 변경 과정에서 생긴 미사용 코드 정리
# [2] vulture      → 삭제된 코드가 어딘가에 아직 참조되는지 확인
# [3] pytest       → 수정된 TC 포함 전체 테스트 재실행
```

하나라도 실패하면 커밋 중단. 모두 통과해야 커밋 허용.

---

### Step 7 — 완료 처리

```bash
# 변경 계획 파일을 completed 로 이동
mv docs/exec-plans/active/change-launch-angle-max-2026-05-17.md \
   docs/exec-plans/completed/
```

---

## 변경 계획 템플릿

`docs/exec-plans/active/change-{대상}-{날짜}.md` 에 아래 내용으로 작성한다.

```markdown
# 변경: {무엇을 바꾸는가}

## 변경 내용
{변경 전} → {변경 후}

## 변경 이유
왜 바꾸는가. 어떤 문제를 해결하는가.

## 영향 받는 코드 파일
- {파일경로}: {어떤 부분이 바뀌는가}

## 영향 받는 문서
- {문서경로}: {어떤 내용이 바뀌는가}

## 영향 받는 테스트 케이스
- {TC-ID}: {기대값이 어떻게 바뀌는가}

## 완료 기준
- [ ] 관련 문서 수정 완료
- [ ] TC 수정/추가 완료
- [ ] 코드 수정 완료
- [ ] 훅 파이프라인 전체 통과
- [ ] 수동 동작 확인 완료
```

---

## 변경 유형별 위험도 & 주의사항

### 🟢 낮음 — 수치/범위 조정

예시: 각도 범위 변경, HUD 색상 변경, 슬라이더 기본값 변경

```
주의사항:
- 관련 상수가 여러 파일에 흩어져 있지 않은지 grep으로 확인
- TC 기대값 수정 필수
- 특별한 회귀 테스트 없음
```

---

### 🟡 중간 — UI/렌더링 변경

예시: 레이아웃 재배치, 카메라 동작 변경, HUD 항목 추가/제거

```
주의사항:
- docs/DESIGN.md 또는 docs/FRONTEND.md 먼저 수정
- on_draw() 안에서 새 할당(glGenBuffers 등) 하지 않도록 주의
- 수동 확인 체크리스트 반드시 실행 (자동화 어려운 영역)
```

---

### 🟠 높음 — 물리 엔진 변경

예시: 적분 방식 교체(Euler → RK4), 양력 모델 수정, 실속각 변경

```
주의사항:
- docs/product-specs/physics.md 수정 후 설계 검토
- TC-P01~TC-P09 전체 재실행 필수
- 변경 전후 비행거리 비교 수동 확인 (같은 조건에서 결과가 달라짐)
- 기존 최고기록이 새 물리 모델과 달라질 수 있음 → 기록 초기화 여부 판단
```

---

### 🔴 매우 높음 — DB 스키마 변경

예시: 컬럼 추가/삭제, 테이블 구조 변경

```
주의사항:
- docs/generated/db-schema.md 수정 필수
- 마이그레이션 파일 작성 (data/migrations/v{N}.sql)
- 기존 records.db 백업 후 마이그레이션 테스트
- TC-D01~TC-D06 전체 재실행 필수
- schema_version 테이블 버전 증가 확인
```

---

### 🔴 매우 높음 — 모듈 삭제 / 대규모 리팩터링

예시: 파일 삭제, 함수 이름 변경, 모듈 통합

```
주의사항:
- grep으로 삭제할 대상의 참조 전체 확인 후 제거
- vulture 결과로 잔존 참조 없는지 재확인
- 전체 pytest 통과 필수
- ARCHITECTURE.md 모듈 목록 업데이트
```

---

## 자주 하는 실수

| 실수 | 결과 | 예방 |
|------|------|------|
| 코드를 먼저 수정 | 문서-코드 불일치, AI 도구 혼란 | 항상 문서 먼저 |
| TC 수정 없이 코드 변경 | pytest 실패로 커밋 막힘 | Step 4 먼저 |
| 상수 여러 곳에 직접 입력 | 일부만 변경되어 버그 발생 | 상수 파일 한 곳에 |
| OpenGL 버퍼 재생성 후 미삭제 | GPU 메모리 누수 | RELIABILITY.md 규칙 준수 |
| 완료 파일 이동 생략 | active/ 폴더에 완료된 항목 쌓임 | Step 7 반드시 실행 |

---

## 빠른 참조

```bash
# 변경 영향 범위 파악
grep -rn "{함수명 또는 상수명}" .

# 수동 훅 전체 실행 (커밋 없이)
pre-commit run --all-files

# 특정 모듈 테스트만 실행
pytest physics/ -v
pytest data/ -v
pytest ui/ -v

# 진행 중인 변경 계획 목록 확인
ls docs/exec-plans/active/

# 완료된 변경 이력 확인
ls docs/exec-plans/completed/
```
