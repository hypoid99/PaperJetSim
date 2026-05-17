# Phase 1 Week 1 — 물리 엔진 & 렌더링 기반

## 목표

비행기가 날아가는 물리 시뮬레이션과 3D 화면을 완성한다.
UI 없이도 콘솔에서 물리 계산 결과를 확인할 수 있는 상태.

## 작업 목록

- [ ] `requirements.txt` 작성 (pyglet, PyOpenGL, numpy)
- [ ] `main.py` — Pyglet 윈도우 초기화
- [ ] `physics/environment.py` — Environment 데이터 클래스
- [ ] `physics/aerodynamics.py` — CL/CD 함수 구현
- [ ] `physics/airplane.py` — PaperAirplane 클래스 (상태 + update)
- [ ] 물리 단위 테스트 (pytest) — 실속각, 착지 조건
- [ ] `render/renderer3d.py` — 기본 OpenGL 씬 (바닥 평면, 하늘)
- [ ] `render/trail.py` — 궤적 라인 렌더링
- [ ] 통합 테스트: 발사 → 비행 → 착지 전체 루프 확인

## 완료 기준

- 각도 15°, 힘 50%, 평온한 날씨 조건에서 비행거리 20~30m 범위 내
- 60fps 유지 (비행 중 CPU 30% 이하)
- 궤적이 물리적으로 자연스러운 포물선 형태

## 의존성

없음 (첫 번째 태스크)
