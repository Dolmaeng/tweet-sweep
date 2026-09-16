# tweet-sweep

X(구 Twitter) 계정의 내 게시물(포스트·답글·미디어·리포스트)을 **계정 안전을 최우선**으로, 천천히 자동 삭제하는 개인용 CLI.

## 상태
- 단계: 기획·명세 (2026-09-17 시작). 코드 없음
- 확정/미정 항목: `docs/spec/01-requirements.md` §10

## 핵심 원칙 (요약)
- 공식 X API v2만 사용. 브라우저 자동화·비공식 엔드포인트 금지 → `docs/adr/0001`
- 속도 제한의 일부만 사용, 이상 응답 시 즉시 중단. 기본 동작은 dry-run
- 열람은 X 데이터 아카이브, 삭제는 API → `docs/adr/0003`
- 모든 상태는 로컬 SQLite. 중단 후 재개 가능

## 문서 지도
| 경로 | 내용 |
|---|---|
| `docs/spec/00-constitution.md` | 헌장(불변 원칙) |
| `docs/spec/01-requirements.md` | 요구사항 명세(SRS) |
| `docs/adr/` | 아키텍처 결정 기록 |
| `docs/log/` | 작업 로그(일자별) |
| `docs/learn/` | 조사·학습 노트 |

## 빠른 시작
- 구현 후 작성. 목표: `git clone` → 의존성 설치 → 인증 → `plan` 실행의 3단계
