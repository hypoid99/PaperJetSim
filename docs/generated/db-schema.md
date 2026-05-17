# DB Schema — PaperJetSim

SQLite 데이터베이스 스키마 정의.
파일 위치: `{APP_DATA_DIR}/records.db`

---

## 테이블 목록

| 테이블 | 역할 |
|--------|------|
| `records` | 비행 기록 저장 |
| `achievements` | 성취 달성 이력 |
| `schema_version` | 마이그레이션 버전 관리 |

---

## records 테이블

```sql
CREATE TABLE IF NOT EXISTS records (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,

    -- 비행 결과
    distance     REAL    NOT NULL,          -- 최종 거리 (m)
    flight_time  REAL    NOT NULL,          -- 비행 시간 (초)
    max_height   REAL    NOT NULL,          -- 최고 고도 (m)
    max_speed    REAL    NOT NULL,          -- 최고 속도 (m/s)

    -- 발사 조건
    launch_angle REAL    NOT NULL,          -- 발사 각도 (°, -10 ~ 45)
    launch_force REAL    NOT NULL,          -- 발사 힘 (%, 0 ~ 100)

    -- 환경 조건
    wind_speed   REAL    NOT NULL,          -- 바람 세기 (m/s)
    wind_dir     REAL    NOT NULL,          -- 바람 방향 (°)
    gravity      REAL    NOT NULL,          -- 중력 (m/s²)
    air_density  REAL    NOT NULL,          -- 공기밀도 (kg/m³)
    preset_name  TEXT    DEFAULT NULL,      -- 날씨 프리셋 이름 (커스텀이면 NULL)

    -- 메타
    created_at   TEXT    NOT NULL           -- ISO 8601, UTC
                         DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- 거리 기준 내림차순 조회 최적화
CREATE INDEX IF NOT EXISTS idx_records_distance
    ON records(distance DESC);

-- 날짜 기준 조회 최적화
CREATE INDEX IF NOT EXISTS idx_records_created
    ON records(created_at DESC);
```

---

## achievements 테이블

```sql
CREATE TABLE IF NOT EXISTS achievements (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    key          TEXT    NOT NULL UNIQUE,   -- 성취 식별자 (예: 'first_flight')
    unlocked_at  TEXT    NOT NULL           -- ISO 8601, UTC
                         DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
```

### 성취 키 목록

| key | 조건 |
|-----|------|
| `first_flight` | 첫 번째 착지 |
| `over_10m` | 거리 ≥ 10m |
| `over_50m` | 거리 ≥ 50m |
| `over_100m` | 거리 ≥ 100m |
| `airtime_10s` | 비행시간 ≥ 10초 |
| `speed_20ms` | 최고속도 ≥ 20 m/s |

---

## schema_version 테이블

```sql
CREATE TABLE IF NOT EXISTS schema_version (
    version      INTEGER NOT NULL,
    applied_at   TEXT    NOT NULL
                         DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- 초기 버전 삽입
INSERT INTO schema_version (version) VALUES (1);
```

---

## 마이그레이션 정책

스키마 변경 시:
1. `schema_version`의 버전을 증가
2. `data/migrations/v{N}.sql` 파일 작성
3. `RecordManager.__init__()` 에서 현재 버전 확인 후 순차 적용

```python
def _migrate(self):
    current = self._get_version()
    for v in range(current + 1, LATEST_VERSION + 1):
        self._apply_migration(v)
```

---

## 조회 쿼리 예시

```sql
-- 상위 10개 기록 조회
SELECT * FROM records ORDER BY distance DESC LIMIT 10;

-- 신기록 여부 확인
SELECT MAX(distance) FROM records;

-- 특정 프리셋 기록만 조회
SELECT * FROM records WHERE preset_name = '순풍' ORDER BY distance DESC;
```
