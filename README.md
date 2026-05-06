# electronics-price-mcp

`electronics-price-mcp`는 한국 쇼핑몰 기준으로 전자기기와 PC 부품을 검색하고, 같은 모델끼리만 현재 가격을 비교하는 원격 MCP 서버입니다. 공개 production은 네이버 쇼핑 검색 API를 기본 source로 사용하고, Danawa provider는 canary 환경에서 먼저 검증하도록 분리했습니다.

## 문제의식

전자기기 가격 비교는 모델명이 조금만 달라도 다른 상품이 섞이기 쉽습니다. 이 프로젝트는 검색과 비교를 분리하고, `RTX 5070`과 `RTX 5070 Ti`처럼 다른 모델이 섞이면 비교를 거부하는 보수적인 가격 비교 MCP를 목표로 합니다.

## 주요 기능

- 한국 전자기기/PC 부품 검색
- 동일 모델 기준 현재 가격 비교
- Naver-only production provider
- Danawa canary provider와 static-catalog fallback
- HTTP API companion surface
- Durable Object 기반 route별 rate limit
- provider diagnostics와 품질 평가 리포트

## 원격 연결

원격 MCP 주소:

```text
https://electronics-price-mcp.jinhyuk9714.workers.dev/mcp
```

Codex:

```bash
codex mcp add electronics-price-mcp --url https://electronics-price-mcp.jinhyuk9714.workers.dev/mcp
```

Claude Code:

```bash
claude mcp add electronics-price-mcp https://electronics-price-mcp.jinhyuk9714.workers.dev/mcp --transport http
```

바로 확인할 수 있는 HTTP endpoint:

- `GET /prompt`
- `GET /api/search?query=그램 16`
- `GET /api/compare?query=RTX 5070`
- `GET /health`

공개 HTTP API는 읽기 전용이며 `search`, `compare`를 제공합니다.

## 현재 상태와 제한

- production 기본값은 `Naver-only`입니다.
- Danawa는 `danawa-canary` 환경에서 먼저 검증합니다.
- `static-catalog`는 canary/dev fallback용 보조 source이며 production 기본값은 비활성입니다.
- 실시간 재고, 배송 예정일, 역대 최저가는 다루지 않습니다.
- 공개 엔드포인트에는 운영 가드레일이 적용됩니다.
  - `/api/search`: 분당 60회
  - `/api/compare`: 분당 60회
  - `/mcp`: 분당 120회

Danawa canary URL:

```text
https://electronics-price-mcp-danawa-canary.jinhyuk9714.workers.dev
```

## 로컬 실행과 직접 배포

공개 원격 MCP를 그대로 쓰는 경우에는 환경변수가 필요 없습니다. 직접 실행하거나 배포할 때만 `.dev.vars.example`을 참고해 `.dev.vars`를 만듭니다.

아래 두 조합 중 하나 이상이 필요합니다.

- `NAVER_CLIENT_ID` + `NAVER_CLIENT_SECRET`
- `ENABLE_DANAWA=true` + `DANAWA_CLIENT_ID` + `DANAWA_CLIENT_SECRET`

```bash
npm install
npm test
npm run typecheck
npm run dev
```

기본 로컬 MCP 주소:

```text
http://127.0.0.1:8787/mcp
```

배포:

```bash
npm run deploy
npm run deploy:danawa-canary
```

운영용 smoke, canary 평가, GitHub Actions 승격 흐름은 `docs/OPERATIONS.md`에 정리되어 있습니다.

## 기술 스택

| 영역 | 기술 |
| --- | --- |
| Runtime | Node.js 22+, TypeScript |
| MCP/HTTP | `@modelcontextprotocol/sdk`, Hono |
| Validation | Zod |
| Deployment | Cloudflare Workers, Wrangler |
| Guardrail | Durable Object rate limiter |
| Quality | Vitest, TypeScript typecheck, service-quality eval scripts |

## 프로젝트 구조

```text
src/
├── domain/       # 모델명 정규화, price service, provider diagnostics
├── providers/    # Naver, Danawa, static catalog provider
├── server/       # MCP server와 price service 생성
├── runtime/      # rate limit runtime
├── pages/        # prompt, privacy, OpenAPI page
└── eval/         # 품질 평가 harness
eval-cases/       # service quality / multisource 평가 케이스
reports/          # 최신 평가 리포트
docs/             # 운영 런북
```

## 검증

```bash
npm test
npm run typecheck
npm run build
npm run eval:multisource-merge:strict
npm run eval:service-quality:static:strict
npm run eval:service-quality:advanced:static:strict
```

운영 리포트와 로컬 환경변수 예시는 다음 파일에서 확인할 수 있습니다.

- `docs/OPERATIONS.md`
- `.dev.vars.example`
