# 작업 목록 (Tasks)

- 상태: v0.1 · 2026-09-17 · plan v0.1 기준. 완료 시 `[x]` + 커밋 해시
- 규칙: 태스크 1개 = 커밋 1개 이상. 의존 태스크 먼저. 각 태스크에 완료 기준(AC)

## M0 계획
- [x] T00 헌장·SRS·ADR·plan·tasks 작성 — 2026-09-17

## M1 스파이크 A — `analyze`
- [ ] T01 프로젝트 골격: uv 초기화, `pyproject.toml`(ruff·pyright·pytest), `src/tweet_sweep/`, `.python-version` — AC: `uv run python -c "import tweet_sweep"` 성공
- [ ] T02 `core/models.py` — AC: 단위 테스트
- [ ] T03 `core/archive.py` 파서(zip/폴더, part 병합, account.js) — AC: fixture(원글·답글·RT·미디어 혼합) 파싱, 3만 건 합성 데이터 < 10s
- [ ] T04 `core/inventory.py` SQLite 스키마·upsert·집계 — AC: 재실행 시 중복 0
- [ ] T05 `core/cost.py` + `core/report.py` analyze 보고서(MD/CSV) — AC: 유형·연도·미디어·비용 2시나리오·프리셋별 소요
- [ ] T06 `adapters/cli.py` `analyze` — AC: `uv run tweet-sweep analyze --archive data/<계정>` 보고서 생성
- [ ] T07 README 빠른 시작(clone → uv sync → analyze) — AC: 새 PC 3단계 재현
- 사용자 입력: `data/<계정명>/` 아카이브 zip

## M2 스파이크 B — 인증 + 1건 삭제
- [ ] T10 `infra/settings.py` TOML 로드·하드 상한 검증 — AC: 초과 설정 기동 거부 테스트
- [ ] T11 `infra/secrets.py` keyring 래퍼 — AC: 저장·조회·삭제, 로그 마스킹
- [ ] T12 `infra/oauth.py` PKCE + 콜백 서버 + 갱신(락) — AC: 모의 토큰 서버로 전 흐름, 갱신 직렬화 테스트
- [ ] T13 `infra/xapi.py` httpx 클라이언트·헤더 파싱·에러 매핑 — AC: respx 200/404/429/401/5xx
- [ ] T14 `auth login|status|logout` — AC: 실계정 로그인 1회, `users/me` 저장
- [ ] T15 `core/actions/delete_post.py` + 단건 실행 경로 — AC: 전용 테스트 게시물 1건 `deleted:true`, 감사 로그·DB 반영
- [ ] T16 콘솔 사용량에서 삭제 단가 확인 → SRS §8·cost 기본값 갱신
- 사용자 입력: console.x.com 앱(Client ID, 콜백 URI), 크레딧 $5, 테스트 게시물 1건

## M3 v1 CLI
- [ ] T20 `core/filters/` base + all/date_range/keyword/kind/keep_list/metric_threshold — AC: 조합 테스트
- [ ] T21 `core/pacing.py` 프리셋·워밍업·활동시간·예산 — AC: 속성 테스트(하드 상한 불변)
- [ ] T22 `core/breaker.py` — AC: 전이표 테스트
- [ ] T23 `core/runner.py` 루프·락·재개·스냅샷·Ctrl+C — AC: 강제 종료 후 재개 시 중복 요청 0
- [ ] T24 `plan`/`run`/`status`/`report`, dry-run 기본 — AC: `--live` 없이는 요청 0건
- [ ] T25 JSONL 감사 로그 + 진행바 — AC: 항목별 1행, 헤더 포함
- [ ] T26 계약 시나리오(429 쿨다운·401 종료·5xx 백오프·404 gone) — AC: 전부 통과
- [ ] T27 운영 문서 `docs/ops.md`(첫 주 cautious 절차, 잠금 징후 대응) — AC: 사용자 검토
- 라이브 실행은 사용자 명시 요청 시. 첫 실행 `cautious` ≤50건

## M4 v1.5 대시보드
- [ ] T30 `adapters/http/` FastAPI + OpenAPI — AC: `/docs` 스키마
- [ ] T31 HTMX 페이지: 진행률·최근 로그·설정 — AC: 브라우저 확인
- [ ] T32 대시보드 run 시작/중단(확인 단계) — AC: dry-run 기본 유지

## M5 확장 (항목별 ADR)
- [ ] T40 `unretweet` 액션 + RT 필터
- [ ] T41 `unlike` 액션
- [ ] T42 LLM 분류 필터(v2)
- [ ] T43 TS/React 프론트(OpenAPI 클라이언트)
