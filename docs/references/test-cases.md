# Test Cases — PaperJetSim

물리 엔진 및 주요 모듈의 예상 동작을 입력값 / 기대값 형태로 정의한다.
pytest로 구현할 때 이 문서를 기준으로 작성한다.

---

## 표기 규칙

```
입력  → 테스트에 넣는 값
기대값 → assert로 확인할 조건
허용오차 → 부동소수점 비교 시 ±범위
```

---

## P — 물리 엔진 테스트

### TC-P01 | 기본 조건 비행거리 범위 확인

```
설명:   기본 설정에서 비행거리가 현실적인 범위에 들어오는지 확인
입력:   angle=15°, force=50%, wind=0, gravity=9.81, air_density=1.225
기대값: 20m ≤ distance ≤ 35m
허용오차: ±0.01m
```

### TC-P02 | 발사각이 거리에 영향을 줌

```
설명:   각도가 높을수록(최적각 범위 내) 더 멀리 날아가야 함
입력A:  angle=5°,  force=50%, 나머지 기본값
입력B:  angle=20°, force=50%, 나머지 기본값
기대값: distance(B) > distance(A)
```

### TC-P03 | 발사 힘이 거리에 영향을 줌

```
설명:   힘이 강할수록 더 멀리 날아가야 함
입력A:  angle=15°, force=10%
입력B:  angle=15°, force=90%
기대값: distance(B) > distance(A)
```

### TC-P04 | 순풍 vs 역풍

```
설명:   순풍(뒤에서 부는 바람)이 역풍보다 유리해야 함
입력A:  wind_speed=5, wind_dir=0°  (순풍, 비행 방향과 동일)
입력B:  wind_speed=5, wind_dir=180° (역풍)
기대값: distance(A) > distance(B)
```

### TC-P05 | 중력 0에서 항력에 의해 멈춤

```
설명:   중력이 0이어도 항력 때문에 언젠가는 멈춰야 함 (무한 비행 불가)
입력:   gravity=0, force=50%, 나머지 기본값
기대값: flight_time < 300초 (5분 이내 착지 or 정지)
        y >= 0 (지면 아래로 뚫리지 않음)
```

### TC-P06 | 실속각 초과 시 CL 감소

```
설명:   받음각이 실속각(15°)을 넘으면 양력계수가 감소해야 함
입력:   alpha=10°, alpha=16°, alpha=25°
기대값: CL(10°) > CL(15°) ≥ CL(16°) > CL(25°)
        CL(25°) < CL_max × 0.5  (실속 후 급격한 감소)
```

### TC-P07 | NaN / Inf 발생 없음

```
설명:   어떤 입력 조합에서도 수치 폭발이 없어야 함
입력:   [경계값 조합 20개]
        - force=0, angle=-10°, wind=10, gravity=20, air_density=1.5
        - force=100, angle=45°, wind=10, gravity=1.0, air_density=0.5
        - gravity=0, air_density=0 (달+진공)
        - 그 외 랜덤 조합 17개
기대값: 전 케이스에서 math.isnan(), math.isinf() 모두 False
        비행 중 매 스텝 x, y, vx, vy 값이 유한수(finite)
```

### TC-P08 | 착지 후 업데이트 중단

```
설명:   y < 0이 된 이후에 update()를 추가로 호출해도 위치가 변하지 않아야 함
입력:   비행기를 착지시킨 후 update() 10회 추가 호출
기대값: 10회 후의 x, y 값 == 착지 직후 x, y 값
        is_landed == True
```

### TC-P09 | 발사 힘 0% → 즉시 착지

```
설명:   힘이 0이면 초기 속도가 없으므로 거의 즉시 착지해야 함
입력:   force=0, angle=0°, gravity=9.81
기대값: distance < 1.0m
        flight_time < 1.0초
```

---

## U — UI / 컨트롤 테스트

### TC-U01 | 각도 슬라이더 범위 클램핑

```
설명:   허용 범위 밖으로 드래그해도 클램핑되어야 함
입력:   set_angle(-20°)  →  기대값: angle == -10°  (최솟값 클램핑)
        set_angle(90°)   →  기대값: angle == 45°   (최댓값 클램핑)
        set_angle(20°)   →  기대값: angle == 20°   (정상 범위)
```

### TC-U02 | 힘 슬라이더 범위 클램핑

```
입력:   set_force(-10)  →  기대값: force == 0
        set_force(150)  →  기대값: force == 100
        set_force(50)   →  기대값: force == 50
```

### TC-U03 | 스페이스바 → 상태 전환

```
설명:   SETUP 상태에서 스페이스를 누르면 FLYING으로 전환
입력:   game_state == SETUP, key_press(SPACE)
기대값: game_state == FLYING
        airplane.is_launched == True
```

### TC-U04 | R키 → 기본값 복원

```
입력:   set_angle(40°), set_force(80%), key_press(R)
기대값: angle == 15°  (기본값)
        force == 50%  (기본값)
```

### TC-U05 | 프리셋 선택 → 슬라이더 일괄 변경

```
입력:   select_preset("달 표면")
기대값: gravity == 1.62
        air_density == 0.0
        wind_speed == 0
```

### TC-U06 | 환경 슬라이더 범위 클램핑

```
입력:   set_gravity(-5)   →  기대값: gravity == 1.0  (최솟값)
        set_gravity(100)  →  기대값: gravity == 20.0 (최댓값)
```

### TC-U07 | 환경 설정 적용

```
설명:   적용 버튼 후 physics/environment 객체에 값이 반영되어야 함
입력:   wind_panel.set_wind(5.0, 45°), wind_panel.apply()
기대값: environment.wind_speed == 5.0
        environment.wind_dir   == 45°
```

---

## R — 렌더링 테스트

### TC-R01 | 궤적 버퍼 최대 크기 제한

```
설명:   3600개를 초과하면 오래된 포인트가 삭제되어 메모리 무한 증가 방지
입력:   trail.add_point() 를 4000회 호출
기대값: len(trail.points) == 3600
        trail.points[-1] == 가장 최근에 추가한 포인트
```

### TC-R02 | 카메라 모드 순환

```
설명:   C키를 3번 누르면 원래 모드로 돌아와야 함
입력:   camera_mode == FOLLOW, press(C) × 3
기대값: camera_mode == FOLLOW  (3번 후 원점 복귀)
```

### TC-R03 | 60fps 유지 확인

```
설명:   비행 시뮬레이션 중 평균 프레임레이트가 목표치를 유지해야 함
입력:   10초 비행 시뮬레이션 실행, 프레임 시간 측정
기대값: average_fps >= 58.0  (60fps ±3% 허용)
        min_fps >= 45.0       (일시적 드롭은 허용)
```

---

## D — 데이터/기록 테스트

### TC-D01 | DB 자동 생성

```
입력:   DB 파일이 없는 상태에서 RecordManager() 초기화
기대값: records.db 파일 생성됨
        records 테이블 존재
```

### TC-D02 | 기록 저장 → 조회 왕복

```
입력:   save_record(distance=34.7, flight_time=5.3, ...)
기대값: get_records()[0].distance == 34.7
        get_records()[0].flight_time == 5.3
        저장 전후 레코드 수 차이 == 1
```

### TC-D03 | 신기록 판별

```
입력:   기존 최고기록 = 30.0m
        save_record(distance=35.0m)
기대값: is_new_record(35.0) == True
        save_record(distance=25.0m)
        is_new_record(25.0) == False
```

### TC-D04 | 기록 초기화

```
입력:   5개 기록 저장 후 clear_records() 호출
기대값: len(get_records()) == 0
        DB 파일은 남아있음 (삭제 아닌 초기화)
```

### TC-D05 | 거리 0m 기록 저장 (예외 아님)

```
설명:   힘 0%로 발사해 즉시 착지한 경우도 기록으로 저장되어야 함
입력:   save_record(distance=0.0m, ...)
기대값: 예외 없이 저장됨
        get_records() 에 포함됨
```

### TC-D06 | 앱 재시작 후 기록 유지

```
설명:   RecordManager를 새로 생성해도 이전 기록이 조회되어야 함
입력:   save_record(35.0) → RecordManager 객체 소멸 → 새 RecordManager 생성
기대값: get_records()[0].distance == 35.0
```

---

## 경계값 요약표

| 파라미터 | 최솟값 | 최댓값 | 경계 테스트 필수 |
|---------|--------|--------|----------------|
| 발사 각도 | -10° | 45° | ✓ |
| 발사 힘 | 0% | 100% | ✓ |
| 바람 세기 | 0 m/s | 10 m/s | ✓ |
| 중력 | 1.0 m/s² | 20.0 m/s² | ✓ |
| 공기밀도 | 0.0 kg/m³ | 1.5 kg/m³ | ✓ |
| 궤적 포인트 | 0 | 3600 | ✓ |
| 비행거리 | 0 m | 이론상 무제한 | ✓ (0m 케이스) |
