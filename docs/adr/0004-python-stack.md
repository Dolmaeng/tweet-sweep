# ADR-0004: 구현 스택 — Python 3.12+ / uv / 표준 라이브러리 우선

- 상태: 제안(SRS Q4 확정 대기) · 2026-09-17

## 맥락
- 실행 PC에 Python 3.14만 설치. Node·Docker 없음
- 학습 목표: 백엔드·인프라·AI 엔지니어링 기본기. 소규모 개인 도구지만 필터·대시보드 확장 예정

## 결정(안)
- Python ≥3.12 · 패키지/가상환경 `uv` · CLI `typer` · HTTP `httpx` · 설정 `pydantic-settings`+TOML · 저장 `sqlite3`(표준) · 비밀 `keyring`(Windows Credential Manager) · 로그 표준 `logging` JSON 포매터 · 테스트 `pytest`+`respx` · 품질 `ruff`+`pyright`
- OAuth 2.0 PKCE·요청은 SDK(tweepy) 대신 `httpx`로 직접 구현
- 대시보드(선택)는 `FastAPI` + 서버 렌더링(HTMX)으로 Node 없이 시작

## 이유
- Python: AI/데이터 생태계 표준, 이미 설치, 학습 목표 ③④와 정합
- 직접 구현: OAuth·속도 제한·백오프를 이해하고 통제하는 것이 안전 목표의 핵심. SDK는 재시도·페이싱을 숨김
- 표준 라이브러리 우선: 의존성 최소 → 다른 PC clone 재현 용이
- uv: 잠금 파일 재현성, 단일 바이너리 설치, 2026 사실상 표준

## 대안과 기각 사유
| 대안 | 기각 사유 |
|---|---|
| TypeScript/Node | Node 미설치. 프론트 학습엔 유리하나 백엔드·AI 목표 우선 |
| tweepy | 내부 동작 불투명, 페이싱 통제 어려움. 참고용만 |
| 서버 DB(PostgreSQL 등) | 개인 도구에 과함, 이식성 저하 |

## 결과·후속
- Q4 확정 시 상태→확정, `02-plan.md` 작성
