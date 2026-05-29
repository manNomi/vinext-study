# 08 Request Context Server Shims

## 한 줄 결론

request context 계층은 `headers()`, `cookies()`, middleware override, cache scope, Cloudflare `waitUntil()`을 같은 request lifetime 안에서 안전하게 공유하게 만든다.

## 이 파트가 존재하는 이유

Next.js API는 많은 값을 함수 인자로 넘기지 않는다. Server Component 안에서 `headers()`를 호출하고, route handler 안에서 `cookies()`를 읽고, cache handler에서 `waitUntil()`을 쓰며, middleware가 request headers를 바꾼 결과가 뒤쪽 render 단계에 보인다. vinext는 이를 AsyncLocalStorage 기반 context로 재현한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/shims/unified-request-context.ts`](../../vinext/packages/vinext/src/shims/unified-request-context.ts) | request headers/cache/execution context 등을 하나의 ALS store로 통합 |
| [`../../vinext/packages/vinext/src/shims/request-context.ts`](../../vinext/packages/vinext/src/shims/request-context.ts) | Cloudflare-like `ExecutionContext` accessor |
| [`../../vinext/packages/vinext/src/shims/headers.ts`](../../vinext/packages/vinext/src/shims/headers.ts) | `headers()`, `cookies()`, `draftMode()`, dynamic usage tracking |
| [`../../vinext/packages/vinext/src/shims/cache-for-request.ts`](../../vinext/packages/vinext/src/shims/cache-for-request.ts) | request-scoped factory cache |
| [`../../vinext/packages/vinext/src/shims/internal/work-unit-async-storage.ts`](../../vinext/packages/vinext/src/shims/internal/work-unit-async-storage.ts) | Next internal work unit storage compatibility |
| [`../../vinext/packages/vinext/src/server/middleware.ts`](../../vinext/packages/vinext/src/server/middleware.ts) | `proxy.ts`/`middleware.ts` discovery and execution |
| [`../../vinext/packages/vinext/src/server/middleware-request-headers.ts`](../../vinext/packages/vinext/src/server/middleware-request-headers.ts) | middleware request header override protocol decode/encode |
| [`../../vinext/packages/vinext/src/server/middleware-response-headers.ts`](../../vinext/packages/vinext/src/server/middleware-response-headers.ts) | middleware response headers merge |
| [`../../vinext/packages/vinext/src/server/app-post-middleware-context.ts`](../../vinext/packages/vinext/src/server/app-post-middleware-context.ts) | middleware 이후 request context 재구성 |

## 요청/빌드/렌더 흐름

```text
Request 시작
  -> createRequestContext()
  -> runWithRequestContext()
  -> headersContextFromRequest()
  -> middleware 실행
  -> x-middleware-request-* override decode
  -> headers/cookies snapshot invalidate + rebuild
  -> Server Component / Route Handler / Server Action render
  -> headers(), cookies(), draftMode(), cacheForRequest()가 같은 context 조회
  -> cache writes나 background work가 ctx.waitUntil()에 등록
```

## 주요 개념

| Concept | What to remember |
| --- | --- |
| Unified context | 여러 ALS를 흩어 쓰지 않고 request 단위 state를 하나의 store로 묶는 방향이다. |
| Headers context | request headers/cookies를 lazy snapshot으로 제공하고 middleware override 후 invalidation한다. |
| Dynamic usage | `headers()`, `cookies()`, `connection()`, `noStore()` 등은 render가 static인지 dynamic인지에 영향을 준다. |
| Cache scope guard | `"use cache"`나 `unstable_cache()` 안에서 request-specific API를 읽으면 에러로 막아야 한다. |
| ExecutionContext | Workers `ctx.waitUntil()`을 deep call stack에서 꺼내 쓸 수 있게 한다. |
| Middleware protocol | `x-middleware-request-*`, `x-middleware-override-headers` 같은 internal header를 실제 request로 반영한다. |

## 수정할 때 깨지기 쉬운 지점

- middleware가 `headers()`를 먼저 읽은 뒤 request header override를 반환하면, 기존 sealed snapshot을 무효화해야 한다.
- `cookies().set()`/`draftMode().enable()`은 pending Set-Cookie를 최종 response에 모아야 한다.
- request context가 stream consumption보다 먼저 사라지면 RSC/SSR streaming 중 API 호출이 실패한다.
- cache scope 안에서 request API를 허용하면 static cache correctness가 깨진다.
- Cloudflare `waitUntil()`은 request context를 통해 background KV write 같은 작업에 전달되어야 한다.
- dev Node server에서는 execution context가 없을 수 있으므로 `null` path를 안전하게 처리해야 한다.

## 관련 테스트

- [`../../vinext/tests/unified-request-context.test.ts`](../../vinext/tests/unified-request-context.test.ts)
- [`../../vinext/tests/request-context.test.ts`](../../vinext/tests/request-context.test.ts)
- [`../../vinext/tests/app-request-context.test.ts`](../../vinext/tests/app-request-context.test.ts)
- [`../../vinext/tests/app-post-middleware-context.test.ts`](../../vinext/tests/app-post-middleware-context.test.ts)
- [`../../vinext/tests/middleware-runtime.test.ts`](../../vinext/tests/middleware-runtime.test.ts)
- [`../../vinext/tests/middleware-runtime-trailing-slash.test.ts`](../../vinext/tests/middleware-runtime-trailing-slash.test.ts)
- [`../../vinext/tests/nextjs-compat/als-isolation.test.ts`](../../vinext/tests/nextjs-compat/als-isolation.test.ts)
- [`../../vinext/tests/nextjs-compat/request-apis.test.ts`](../../vinext/tests/nextjs-compat/request-apis.test.ts)
- [`../../vinext/tests/nextjs-compat/set-cookies.test.ts`](../../vinext/tests/nextjs-compat/set-cookies.test.ts)

## 복기용 체크리스트

- 이 값은 request마다 달라지는가, render마다 달라지는가, process 전역인가?
- middleware 전/후 어느 시점의 headers/cookies를 읽어야 하는가?
- snapshot cache를 invalidate해야 하는 mutation이 있는가?
- cache scope에서 호출될 수 있는 API인가?
- Workers와 Node dev 양쪽에서 execution context 부재를 처리하는가?
