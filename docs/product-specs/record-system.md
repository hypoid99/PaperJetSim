# 최장거리 기록 시스템 스펙 — PaperJetSim

## 개요

플레이어의 최장거리 기록을 로컬에 저장하고 기록판(리더보드)을 통해 보여준다.
경쟁 심리를 자극해 반복 플레이를 유도하는 핵심 게임 루프.

---

## 기록 데이터 구조

```sql
CREATE TABLE records (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    distance    REAL    NOT NULL,        -- 최종 거리 (m)
    flight_time REAL    NOT NULL,        -- 비행 시간 (초)
    max_height  REAL    NOT NULL,        -- 최고 고도 (m)
    max_speed   REAL    NOT NULL,        -- 최고 속도 (m/s)
    launch_angle REAL   NOT NULL,        -- 발사 각도 (°)
    launch_force REAL   NOT NULL,        -- 발사 힘 (%)
    wind_speed  REAL    NOT NULL,        -- 바람 세기 (m/s)
    wind_dir    REAL    NOT NULL,        -- 바람 방향 (°)
    gravity     REAL    NOT NULL,        -- 중력 (m/s²)
    air_density REAL    NOT NULL,        -- 공기밀도 (kg/m³)
    preset_name TEXT,                    -- 날씨 프리셋 이름 (있으면)
    created_at  TEXT    NOT NULL         -- ISO 8601 타임스탬프
);
```

---

## 기록판(리더보드) UI

```
┌──────────────────────────────────────────────┐
│              🏆 최장거리 기록                  │
├────┬─────────┬──────┬─────────┬──────────────┤
│ 순위│ 거리    │ 시간  │ 조건    │ 일시          │
├────┼─────────┼──────┼─────────┼──────────────┤
│ 1  │ 41.3m  │ 6.2s │ 순풍    │ 2026-05-17   │
│ 2  │ 38.7m  │ 5.8s │ 평온    │ 2026-05-16   │
│ 3  │ 34.2m  │ 5.1s │ 커스텀  │ 2026-05-16   │
│ …  │ …      │ …    │ …       │ …            │
└────┴─────────┴──────┴─────────┴──────────────┘
│  필터: [전체 ▼]   정렬: [거리순 ▼]            │
│                          [기록 초기화]        │
└──────────────────────────────────────────────┘
```

---

## 기록 갱신 흐름

```
착지 → 거리 계산
     → DB 조회 (현재 최고기록)
     → 신기록이면?
          YES → "🏆 신기록!" 연출 (파티클 + 사운드)
                DB INSERT
          NO  → "아쉽네요! 기록: XX.Xm" 표시
                DB INSERT (기록은 쌓임)
     → 결과 화면 표시
```

---

## 성취 시스템 (간단 버전)

| 성취 | 조건 |
|------|------|
| 첫 비행 | 처음 착지 |
| 10미터 돌파 | 거리 ≥ 10m |
| 반백 돌파 | 거리 ≥ 50m |
| 백미터 클럽 | 거리 ≥ 100m |
| 체공왕 | 비행시간 ≥ 10초 |
| 로켓맨 | 최고속도 ≥ 20 m/s |

성취 달성 시 첫 비행에서 한 번만 알림 표시.

---

## 구현 파일

- `data/record_manager.py` — SQLite CRUD, 기록 조회/저장
- `ui/menu.py` — 기록판 UI 렌더링
