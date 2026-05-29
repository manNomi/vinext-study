# 02 Next Shims

## 한 줄 결론

`shims/`는 vinext가 Next.js API 표면을 Vite/Web API/React primitive 위에 다시 구현하는 호환성 계층이다.

## 이 파트가 존재하는 이유

사용자 앱은 `next/link`, `next/navigation`, `next/server`, `next/headers`, `next/cache` 같은 import를 그대로 사용한다. vinext는 Next.js 내부 런타임을 쓰지 않으므로, 이 import들을 local shim으로 resolve해서 같은 공개 API처럼 보이게 해야 한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/shims/link.tsx`](../../vinext/packages/vinext/src/shims/link.tsx) | `<Link>`, prefetch, App/Pages Router navigation delegation |
| [`../../vinext/packages/vinext/src/shims/navigation.ts`](../../vinext/packages/vinext/src/shims/navigation.ts) | App Router hooks, router instance, redirects, notFound, client navigation |
| [`../../vinext/packages/vinext/src/shims/router.ts`](../../vinext/packages/vinext/src/shims/router.ts) | Pages Router `next/router` singleton, events, client navigation |
| [`../../vinext/packages/vinext/src/shims/server.ts`](../../vinext/packages/vinext/src/shims/server.ts) | `NextRequest`, `NextResponse`, `NextURL`, cookies, userAgent, after |
| [`../../vinext/packages/vinext/src/shims/headers.ts`](../../vinext/packages/vinext/src/shims/headers.ts) | async `headers()`, `cookies()`, `draftMode()` |
| [`../../vinext/packages/vinext/src/shims/cache.ts`](../../vinext/packages/vinext/src/shims/cache.ts) | `next/cache`, ISR/cache handler, tags, `"use cache"` integration |
| [`../../vinext/packages/vinext/src/shims/image.tsx`](../../vinext/packages/vinext/src/shims/image.tsx) | `next/image` 호환 컴포넌트와 `/_vinext/image` URL |
| [`../../vinext/packages/vinext/src/shims/script.tsx`](../../vinext/packages/vinext/src/shims/script.tsx) | `next/script` 전략별 실행/삽입 |
| [`../../vinext/packages/vinext/src/shims/document.tsx`](../../vinext/packages/vinext/src/shims/document.tsx) | Pages Router `_document` primitives |
| [`../../vinext/packages/vinext/src/shims/internal`](../../vinext/packages/vinext/src/shims/internal) | Next 내부 import 호환용 최소 구현 |

## 요청/빌드/렌더 흐름

```text
사용자 코드: import Link from "next/link"
  -> Vite resolveId / alias
  -> packages/vinext/src/shims/link.tsx
  -> 현재 runtime 감지
  -> App Router면 RSC navigation runtime 사용
  -> Pages Router면 next/router shim 사용
```

Server API도 비슷하다.

```text
Server Component / Route Handler
  -> headers(), cookies(), draftMode()
  -> AsyncLocalStorage 기반 request context 조회
  -> middleware가 수정한 request headers/cookies 반영
  -> dynamic usage / cache scope guard 기록
```

## 주요 책임 묶음

| Group | What to remember |
| --- | --- |
| Navigation shims | `next/link`, `next/navigation`, `next/router`가 App Router와 Pages Router를 런타임에서 분기한다. |
| Request shims | `next/server`, `next/headers`가 Web Request/Response와 ALS context를 Next.js처럼 보이게 한다. |
| Render shims | `next/head`, `next/document`, `next/script`, metadata 관련 shim이 SSR head/script 상태를 수집한다. |
| Asset shims | `next/image`, `next/font/google`, `next/font/local`, `next/og`가 Vite/Workers 환경에 맞는 대체 경로를 제공한다. |
| Cache shims | `next/cache`와 `"use cache"`가 ISR, tag invalidation, request API guard와 맞물린다. |
| Internal shims | Next ecosystem 패키지가 import하는 내부 경로를 최소한으로 받아준다. |

## 수정할 때 깨지기 쉬운 지점

- shim은 공개 API처럼 보여야 하므로 에러 메시지, 반환 shape, sync/async 호환성이 중요하다.
- App Router와 Pages Router가 같은 shim을 공유해도 내부 상태 소스는 다를 수 있다.
- client shim에서 `window` 접근은 SSR/RSC import 시점에 터지지 않게 방어해야 한다.
- `headers()`/`cookies()`는 request context뿐 아니라 dynamic rendering classification에도 영향을 준다.
- cache scope 안에서 request-specific API를 허용하면 persisted cache에 request 값이 얼어붙을 수 있다.
- ecosystem 패키지가 Next 내부 경로를 import하는 경우, 내부 shim을 너무 넓게 구현하면 유지보수 비용이 커진다.

## 관련 테스트

- [`../../vinext/tests/shims.test.ts`](../../vinext/tests/shims.test.ts)
- [`../../vinext/tests/link.test.ts`](../../vinext/tests/link.test.ts)
- [`../../vinext/tests/link-navigation.test.ts`](../../vinext/tests/link-navigation.test.ts)
- [`../../vinext/tests/router-javascript-urls.test.ts`](../../vinext/tests/router-javascript-urls.test.ts)
- [`../../vinext/tests/image-component.test.ts`](../../vinext/tests/image-component.test.ts)
- [`../../vinext/tests/script.test.ts`](../../vinext/tests/script.test.ts)
- [`../../vinext/tests/head.test.ts`](../../vinext/tests/head.test.ts)
- [`../../vinext/tests/document.test.ts`](../../vinext/tests/document.test.ts)
- [`../../vinext/tests/cache-for-request.test.ts`](../../vinext/tests/cache-for-request.test.ts)
- [`../../vinext/tests/nextjs-compat/navigation.test.ts`](../../vinext/tests/nextjs-compat/navigation.test.ts)
- [`../../vinext/tests/nextjs-compat/request-apis.test.ts`](../../vinext/tests/nextjs-compat/request-apis.test.ts)

## 복기용 체크리스트

- 해당 API가 App Router, Pages Router, route handler, middleware 중 어디에서 호출되는가?
- RSC/SSR/client 중 어느 environment에서 import되는가?
- sync처럼 보이는 Next API가 실제로는 Promise+decorated object 형태를 요구하는가?
- middleware request header override 이후에도 같은 snapshot을 보고 있지 않은가?
- test fixture가 실제 사용자 import 경로인 `next/*`를 통해 검증하는가?
