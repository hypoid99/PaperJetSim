# Security — PaperJetSim

로컬 데스크톱 앱이지만 파일 접근, 데이터 저장, 사용자 입력에 대한 보안 정책을 정의한다.

---

## 1. 데이터 보호

### 저장 데이터

| 데이터 | 저장 위치 | 민감도 | 암호화 |
|--------|-----------|--------|--------|
| 비행 기록 | `records.db` (SQLite) | 낮음 | 불필요 |
| 게임 설정 | `config.json` | 낮음 | 불필요 |
| 로그 | `paperjetsim.log` | 낮음 | 불필요 |

개인정보, 계정 정보, 결제 정보를 일절 수집하지 않는다.

### 파일 저장 위치

```python
# 운영체제별 표준 앱 데이터 경로 사용
import platformdirs
APP_DATA_DIR = platformdirs.user_data_dir("PaperJetSim", "PaperJetSim")
# Windows: C:\Users\<user>\AppData\Local\PaperJetSim\PaperJetSim
# macOS:   ~/Library/Application Support/PaperJetSim
# Linux:   ~/.local/share/PaperJetSim
```

시스템 디렉토리(`C:\Windows`, `/etc` 등)에 파일을 쓰지 않는다.

---

## 2. SQLite 인젝션 방지

사용자 입력을 SQL에 직접 삽입하지 않는다. 반드시 파라미터 바인딩을 사용한다.

```python
# ❌ 잘못된 방식
cursor.execute(f"SELECT * FROM records WHERE preset='{preset_name}'")

# ✅ 올바른 방식
cursor.execute("SELECT * FROM records WHERE preset=?", (preset_name,))
```

---

## 3. 입력값 검증

모든 사용자 입력은 사용 전에 범위 검증 및 타입 변환을 거친다.

```python
# config.json 로드 시 검증 예시
def load_config(path: str) -> Config:
    with open(path) as f:
        raw = json.load(f)
    return Config(
        angle=max(-10, min(45, float(raw.get("angle", 15)))),   # 범위 강제
        force=max(0, min(100, float(raw.get("force", 50)))),
    )
```

config.json이 손상되거나 이상한 값이 있을 경우 기본값으로 대체하고 게임을 정상 실행한다.

---

## 4. 금지 사항

- 네트워크 요청 금지 (v1.0은 완전 오프라인)
- 외부 스크립트 실행 금지 (`subprocess`, `eval`, `exec` 사용 금지)
- 절대 경로 하드코딩 금지 (`platformdirs` 사용)
- API 키 또는 시크릿을 코드에 하드코딩 금지 (현재는 해당 없음)
- 로그에 파일 시스템 전체 경로 출력 금지 (상대 경로만)

---

## 5. 향후 온라인 기능 추가 시 (Phase 3)

글로벌 리더보드 등 온라인 기능 추가 시 아래 정책을 별도로 수립해야 한다.

- HTTPS 통신만 허용
- 사용자 식별은 익명 UUID 방식 (이름/이메일 수집 안 함)
- 서버 측 입력값 재검증 필수
- 이 문서에 인증/인가 섹션 추가
