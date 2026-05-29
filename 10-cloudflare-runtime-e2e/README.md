# 10. Cloudflare Runtime And E2E Flow

## Goal

Vinext가 Cloudflare Workers를 primary target으로 삼는 이유와, E2E 테스트가 실제
호환성 회귀를 어떻게 잡는지 이해합니다.

## Read

- `packages/vinext/src/cloudflare`
- `packages/vinext/src/cloudflare/worker-entry.ts`
- `examples/app-router-cloudflare`
- `examples/pages-router-cloudflare`
- `.github/workflows/ci.yml`
- `.github/workflows/deploy-examples.yml`
- `tests/e2e`

## Core Questions

1. Cloudflare Worker entry는 어떤 request handler를 호출하나?
2. App Router와 Pages Router는 production에서 어떤 경로로 실행되나?
3. E2E 테스트는 unit test가 못 잡는 어떤 문제를 잡나?
4. Example deploy와 smoke test는 왜 중요한가?

## Things To Trace

1. Worker `fetch` handler에서 Vinext server로 이어지는 흐름을 따라갑니다.
2. App Router example과 Pages Router example의 config 차이를 봅니다.
3. Playwright E2E 하나를 골라 fixture, browser action, assertion을 연결해 봅니다.
4. CI workflow에서 check, unit, E2E가 어떻게 분리되는지 확인합니다.

## Small Exercise

새로운 Next.js compat bug를 발견했다고 가정하고, 어떤 테스트를 어디에 추가할지
계획합니다.

1. Unit test
2. Fixture page
3. E2E test
4. Next.js source or test reference link

## Done When

Vinext 기여에서 "Next.js 동작 확인", "작은 compatibility test", "Cloudflare runtime
관점"을 함께 보는 이유를 설명할 수 있습니다.
