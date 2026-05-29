# 04. Pages Router Runtime

## Goal

Pages Router 요청이 dev server와 production server에서 어떻게 처리되는지
이해합니다.

## Read

- `packages/vinext/src/server/dev-server.ts`
- `packages/vinext/src/server/prod-server.ts`
- `packages/vinext/src/server/response.ts`
- `tests/pages-router.test.ts`
- `tests/middleware.test.ts`

## Core Questions

1. Dev server와 production server는 어떤 로직을 공유하고 어떤 로직이 분리되어 있나?
2. Pages Router에서 middleware, redirects, rewrites, filesystem routes의 순서는 어떻게 적용되나?
3. SSR page와 API route는 어디서 갈라지나?
4. Response header와 status는 어디서 최종 조립되나?

## Things To Trace

1. `/pages-basic` fixture 요청 하나가 어떤 handler를 타는지 따라갑니다.
2. Middleware가 request를 rewrite할 때 URL과 header가 어떻게 바뀌는지 봅니다.
3. `getServerSideProps` 또는 API route 처리 지점을 찾습니다.
4. Prod server 쪽에도 같은 동작이 있는지 비교합니다.

## Small Exercise

Pages Router 요청 순서를 7단계 이하로 요약합니다.

1. Request enters...
2. Config redirects...
3. Middleware...
4. Route match...
5. Render or route handler...
6. Response...

## Done When

Pages Router 버그를 고칠 때 dev와 prod server 둘 다 봐야 하는 이유를 설명할 수
있습니다.
