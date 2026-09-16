# tweet-sweep

X(구 Twitter) 계정의 내 게시물(원글·답글·미디어)을 **계정 안전을 최우선**으로, 천천히 자동 삭제하는 개인용 CLI.

- 저장소 https://github.com/Dolmaeng/tweet-sweep · MIT

## 상태
- M0 계획 완료(2026-09-17). 코드 없음. 다음: M1 `analyze` 스파이크
- 확정/미정 항목 `docs/spec/01-requirements.md` §10 · 진행 `docs/spec/03-tasks.md`

## 핵심 원칙 (요약)
- 공식 X API v2만 사용. 브라우저 자동화·비공식 엔드포인트 금지 → `docs/adr/0001`
- 속도 제한의 일부만 사용, 이상 응답 시 즉시 중단. 기본 동작은 dry-run
- 열람은 X 데이터 아카이브, 삭제는 API → `docs/adr/0003`
- 모든 상태는 로컬 SQLite(계정별). 중단 후 재개 가능

## 문서 지도
| 경로 | 내용 |
|---|---|
| `docs/spec/00-constitution.md` | 헌장(불변 원칙) |
| `docs/spec/01-requirements.md` | 요구사항 명세(SRS) |
| `docs/spec/02-plan.md` | 구현 계획(구조·데이터·안전 엔진·마일스톤) |
| `docs/spec/03-tasks.md` | 작업 목록·완료 기준 |
| `docs/adr/` | 아키텍처 결정 기록 |
| `docs/log/` | 작업 로그(일자별) |
| `docs/learn/` | 조사·학습 노트 |

## 빠른 시작
- M1 완료 후 작성. 목표: `git clone` → `uv sync` → `uv run tweet-sweep analyze --archive data/<계정명>` 3단계
- 아카이브 zip은 `data/<계정명>/`에 둔다(`data/`는 커밋되지 않음)
