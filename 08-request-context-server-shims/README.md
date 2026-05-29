# 08. Request Context And Server Shims

## Goal

`headers()`, `cookies()`, `params`, `searchParams` 같은 request-scoped API가
Vinext에서 어떻게 유지되는지 이해합니다.

## Read

- `packages/vinext/src/shims/headers.ts`
- `packages/vinext/src/shims/cookies.ts`
- `packages/vinext/src/server/request-context.ts`
- `packages/vinext/src/server/async-storage.ts`
- `tests/headers.test.ts`
- `tests/cookies.test.ts`

## Core Questions

1. Request-scoped data는 module global이 아니라 어디에 저장되나?
2. AsyncLocalStorage가 필요한 이유는 무엇인가?
3. Server shim은 request lifecycle이 끝난 뒤 어떤 상태를 정리해야 하나?
4. Next.js 15+ thenable params와 backward compatibility는 어떻게 맞추나?

## Things To Trace

1. Request가 시작될 때 context가 생성되는 위치를 찾습니다.
2. `headers()` 또는 `cookies()` 호출이 현재 request state를 읽는 흐름을 따라갑니다.
3. SSR과 RSC environment 사이에서 request state가 어떻게 전달되는지 확인합니다.
4. 테스트에서 concurrent request 격리를 어떻게 검증하는지 봅니다.

## Small Exercise

`headers()`를 module global로 구현하면 생길 수 있는 버그를 3개 적어봅니다.

## Done When

Request-scoped API를 구현할 때 AsyncLocalStorage와 cleanup이 왜 중요한지 설명할 수
있습니다.
