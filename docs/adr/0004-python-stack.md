# ADR-0004: 구현 스택 — Python 3.12+ / uv / 표준 라이브러리 우선

- 상태: 확정(질의응답 Q4, 2026-09-17). 진화 경로 포함

## 맥락
- 실행 PC에 Python 3.14만 설치. Node·Docker 없음
- 학습 목표: 백엔드·인프라·AI 엔지니어링 기본기. 사용자 의향: 1단계 Python CLI → FastAPI+HTMX 대시보드, 2단계 TS/React 프론트로 리팩터링, 3단계 TS 풀스택 전환 가능

## 결정
- 1단계: Python ≥3.12 · `uv` · CLI `typer` · HTTP `httpx` · 설정 `pydantic-settings`+TOML · 저장 `sqlite3`(표준) · 비밀 `keyring` · 로그 표준 `logging` JSON · 테스트 `pytest`+`respx` · 품질 `ruff`+`pyright`
- OAuth 2.0 PKCE·요청은 SDK(tweepy) 대신 `httpx`로 직접 구현
- 대시보드는 `FastAPI` + HTMX(서버 렌더링, Node 불필요). HTTP API는 OpenAPI 스키마를 노출해 2단계 TS 프론트가 그대로 소비(ADR-0005)

## 이유
- Python: LLM 내부 실습(PyTorch·Hugging Face)과 AI 직무 채용 언어. 이미 설치. 학습 목표 ③④ 직결
- 직접 구현: OAuth·속도 제한·백오프를 이해하고 통제하는 것이 안전 목표의 핵심. SDK는 재시도·페이싱을 숨김
- 표준 라이브러리 우선: 의존성 최소 → 다른 PC clone 재현 용이
- uv: 잠금 파일 재현성, 단일 바이너리, 2026 사실상 표준
- HTMX 먼저: Node 없이 웹 기본기(HTTP·서버 렌더링·인증)부터. 프론트 프레임워크는 API 계약이 안정된 뒤

## 대안과 기각 사유
| 대안 | 기각 사유 |
|---|---|
| TS 풀스택 즉시 | Node 미설치. LLM 내부 실습 경로 부족. 웹·에이전트 앱 개발에서는 동급이라 3단계 후보로 유지 |
| tweepy | 내부 동작 불투명, 페이싱 통제 어려움. 참고용만 |
| 서버 DB(PostgreSQL 등) | 개인 도구에 과함, 이식성 저하 |

## 결과·후속
- `02-plan.md`에 패키지 구조·의존성 확정. 2단계 진입 조건: CLI v1 완료 + HTTP API 계약 안정
