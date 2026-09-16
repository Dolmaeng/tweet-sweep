# 구현 계획 (Plan)

- 상태: v0.1 · 2026-09-17 · SRS v0.3 기준. 변경은 ADR 또는 이 문서 개정으로
- 산출 순서: M1 `analyze` 스파이크 → M2 인증+1건 삭제 스파이크 → M3 v1 CLI → M4 대시보드 → M5 확장

## 1. 저장소 구조
```
tweet-sweep/
  pyproject.toml            uv 관리. 의존성·스크립트·ruff·pyright·pytest 설정
  uv.lock  .python-version  재현성(≥3.12, 로컬 3.14)
  src/tweet_sweep/
    core/                   프레임워크 무의존(표준 라이브러리만). 테스트 중심
      models.py             Post, PostKind, Account, Job, JobItem, DeleteResult
      archive.py            tweets.js 파서, account.js 검증, zip/폴더 입력
      inventory.py          SQLite 저장소(스키마·upsert·집계)
      filters/              base.py + 필터 1개 = 파일 1개
      actions/              base.py + delete_post.py (unretweet/unlike는 M5)
      pacing.py             프리셋·간격·15분/일일 예산·활동시간·워밍업·하드 상한
      breaker.py            회로차단 상태기계
      runner.py             plan/run 루프(락·재개·스냅샷·Ctrl+C)
      cost.py               단가 시나리오·비용 상한
      report.py             analyze/plan/status 보고서(MD/CSV)
    infra/                  외부 세계
      xapi.py               httpx 클라이언트, x-rate-limit 헤더 파싱, 에러 매핑
      oauth.py              PKCE 흐름, 127.0.0.1 콜백 서버, 토큰 갱신(락)
      secrets.py            keyring 래퍼(계정 id별 항목)
      settings.py           pydantic-settings + TOML, 하드 상한 검증
      logging.py            JSONL 감사 로그 + 콘솔
    adapters/
      cli.py                typer: analyze / auth / plan / run / status / report
      http/                 FastAPI + HTMX (M4)
  tests/unit  tests/contract  tests/fixtures   합성·익명 아카이브 샘플, 모의 서버
  config/default.toml  config/example.toml
  data/<계정명>/            gitignore. 아카이브·state.db·logs
```
- 의존 방향: `adapters → core ← infra`. `core`는 `infra`를 인터페이스(Protocol)로만 참조

## 2. 데이터 모델 (`data/<계정명>/state.db`)
| 테이블 | 열 | 메모 |
|---|---|---|
| `account` | user_id PK, username, archive_generated_at, imported_at | account.js + users/me |
| `post` | id PK, created_at, kind(post/reply/retweet), text, in_reply_to, has_media, media_paths, like_count, rt_count, raw_json | 아카이브 원본 스냅샷 겸용(P3) |
| `job` | id PK, action, preset, filter_spec(json), created_at, status | plan 1회 = job 1개 |
| `job_item` | job_id, post_id, status(pending/done/gone/failed/skipped), attempts, last_code, done_at | PK(job_id, post_id) |
| `audit` | ts, job_id, post_id, method, url, status, rl_limit, rl_remaining, rl_reset, cost_est, err | JSONL과 이중 기록 |
| `budget` | day PK, count, cost_est | 일일 상한·비용 상한 |
- 요청 전 `attempts+1` 커밋 → 응답 후 상태 커밋. 강제 종료로 `attempts>0 & pending`이면 재개 시 재시도, 404는 gone

## 3. CLI 명령
| 명령 | 동작 | 부작용 |
|---|---|---|
| `analyze --archive data/<계정>` | 파싱 → 인벤토리 → 보고서(유형·연도·미디어·비용 2시나리오·프리셋별 소요) | 로컬만 |
| `auth login|status|logout` | PKCE 로그인, keyring 저장, `users/me` 1회 | API 읽기 1회 |
| `plan [필터]` | 대상 목록 CSV + 요약, job 생성 | 없음 |
| `run --job <id> [--live --confirm]` | `--live` 없으면 dry-run(요청·대기 없음). 있으면 확인 후 실행 | **API 쓰기** |
| `status` / `report` | 진행률·비용·다음 실행 가능 시각 / 세션 요약 | 없음 |
- 공통 옵션 `--config` `--account`(기본: 로그인 계정) `--log-level`

## 4. 안전 엔진
- `pacing.py`: 프리셋 → 간격(min,max)·시간당·일일 상한·활동 시간대·워밍업 계수. `next_delay()` = 균등 무작위 + 5% 확률 긴 휴식(5~15분)
- 하드 상한 상수 `HARD_MAX_PER_15MIN=30`, `HARD_MAX_PER_DAY=1600`. 설정 초과 시 `settings.py`가 기동 거부
- `breaker.py`: CLOSED →(429)→ COOLING(reset+60s) → CLOSED. 세션 내 429 2회 → OPEN(종료). 401/403 → OPEN 즉시. 5xx 연속 3 → OPEN. 본문 `locked|suspended|limited` → OPEN
- `runner.py` 루프: 락 → 예산 확인 → 항목 선택 → 스냅샷 확인 → 액션 → 감사 기록 → 예산 갱신 → 대기. Ctrl+C는 대기 중 즉시, 요청 중이면 응답 후 종료
- 비용: 요청마다 설정 단가(기본 $0.010 보수)로 누적. 상한 도달 시 종료. M2에서 실제 단가 확인 후 기본값 갱신

## 5. 인증 (OAuth 2.0 PKCE)
- `code_verifier` → S256 challenge → 브라우저 `https://x.com/i/oauth2/authorize` → `http://127.0.0.1:<port>/callback` 로컬 서버가 code 수신 → `POST /2/oauth2/token` → keyring 항목 `tweet-sweep/<user_id>`
- 갱신: 만료 5분 전 선제. 파일 락으로 직렬화(refresh 1회용). 실패 시 재로그인 안내
- 클라이언트 ID는 설정 파일(공개 클라이언트, 시크릿 없음)

## 6. 아카이브 파서
- 입력 zip/폴더. `data/tweets.js`, `data/tweets-part*.js`, `data/account.js`만 읽음
- 첫 `=` 이후를 `json.loads`. `tweet` 객체 → `Post`. `RT @` 접두 → retweet, `in_reply_to_status_id` → reply, 그 외 post. `extended_entities.media` → has_media
- `account.js` accountId를 저장 → `run` 전 `users/me` id와 비교(FR-16)
- 3만 건 < 10s(NFR-03). 스트리밍 불필요

## 7. 테스트
- 단위: pacing(상한·워밍업·범위) · breaker(전이표) · filters(조합) · archive(fixture) · cost
- 계약: `respx`로 200/404/429(+헤더)/401/5xx 모의. 회로차단·재개 시나리오
- 속성: 어떤 설정에서도 하드 상한 초과 불가(`hypothesis`, 선택)
- 실계정: M2에서 전용 테스트 게시물 1건만. 라이브 실행 테스트는 자동화하지 않음

## 8. 개발 워크플로우·도구 역할 (선택 이유)
| 도구 | 역할 | 이유 |
|---|---|---|
| Claude Code | 계획→구현→테스트→커밋 주도. 서브에이전트로 조사·리뷰 병렬화. 사용자 레벨 훅으로 ruff/pyright 자동 실행 | 설치·인증된 유일한 에이전틱 CLI. 도구 호출·장기 작업에 강함 |
| ChatGPT/Codex | 마일스톤·PR 단위 독립 코드 리뷰(안전 엔진 중심), 명세 비판 | 다른 모델 계열의 교차 검증이 자기 리뷰보다 결함을 더 찾음. Codex CLI 있으면 diff 직접, 없으면 웹에 diff |
| Perplexity | 라이브 실행 전 X API 가격·한도·정책 재확인(출처 첨부) | 웹 근거·인용 강점. 정책 변동 위험 대응 |
- 브랜치: M0~M1은 `main` 직접 커밋(문서·스파이크). M2부터 마일스톤 브랜치 + PR → 교차 리뷰 후 병합. 이유: PR이 리뷰 표면과 GitHub 협업 학습을 동시에 제공
- 검증 루프: 편집 → ruff format/check → pyright → pytest → 커밋. 강제는 훅, 문서는 안내(하네스: 가이드+센서)
- 문서 우선: 동작 변경은 SRS/plan 수정 → 코드. ADR은 결정 즉시

## 9. 마일스톤
| 단계 | 산출물 | 완료 기준 |
|---|---|---|
| M0 계획 | 헌장·SRS·ADR·plan·tasks | 사용자 승인 |
| M1 스파이크 A | `analyze` | 실제 아카이브로 보고서 생성 → 예산·RT 방침 확정 |
| M2 스파이크 B | `auth login` + 테스트 게시물 1건 삭제 | `deleted:true`, 감사 로그 1행, 콘솔에서 단가 확인 → SRS §8 갱신 |
| M3 v1 CLI | plan/run/status/report + 안전 엔진 + 테스트 | 계약 시나리오 전부 통과, 첫 주 `cautious` 운영 무사고 |
| M4 v1.5 대시보드 | FastAPI+HTMX, OpenAPI | 브라우저에서 진행률·로그 확인 |
| M5 확장 | unretweet/unlike 액션, LLM 필터, TS 프론트 | 항목별 ADR |

## 10. 사용자 준비 항목
- `data/<계정명>/`에 아카이브 zip 배치(계정별 폴더)
- console.x.com: 프로젝트·앱 생성 → OAuth 2.0 설정(Type: Native/Public, 콜백 `http://127.0.0.1:8765/callback`, 웹사이트 URL 임의) → Client ID 확보 → 크레딧 $5
- 테스트용 게시물 1건 작성(M2에서 삭제)
