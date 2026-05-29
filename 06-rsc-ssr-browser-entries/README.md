# 06 RSC SSR Browser Entries

## 한 줄 결론

App Router는 하나의 entry가 아니라 RSC entry, SSR entry, browser entry가 역할을 나누고 RSC stream을 매개로 이어진다.

## 이 파트가 존재하는 이유

React Server Components는 server-only graph, SSR HTML graph, browser hydration graph를 분리한다. vinext는 Vite와 `@vitejs/plugin-rsc` 위에서 이 세 환경을 생성하고, generated virtual entry가 request dispatch와 render orchestration을 맡게 한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/entries/app-rsc-entry.ts`](../../vinext/packages/vinext/src/entries/app-rsc-entry.ts) | App Router RSC entry generator. route match, RSC render, server actions, prerender endpoints |
| [`../../vinext/packages/vinext/src/entries/app-ssr-entry.ts`](../../vinext/packages/vinext/src/entries/app-ssr-entry.ts) | SSR environment entry generator |
| [`../../vinext/packages/vinext/src/entries/app-browser-entry.ts`](../../vinext/packages/vinext/src/entries/app-browser-entry.ts) | browser hydration/navigation bootstrap generator |
| [`../../vinext/packages/vinext/src/entries/app-rsc-manifest.ts`](../../vinext/packages/vinext/src/entries/app-rsc-manifest.ts) | route/module manifest codegen helper |
| [`../../vinext/packages/vinext/src/server/app-rsc-handler.ts`](../../vinext/packages/vinext/src/server/app-rsc-handler.ts) | RSC request handling primitives |
| [`../../vinext/packages/vinext/src/server/app-page-render.ts`](../../vinext/packages/vinext/src/server/app-page-render.ts) | App page render orchestration |
| [`../../vinext/packages/vinext/src/server/app-ssr-stream.ts`](../../vinext/packages/vinext/src/server/app-ssr-stream.ts) | RSC payload를 HTML stream으로 변환하는 SSR path |
| [`../../vinext/packages/vinext/src/server/app-browser-entry.ts`](../../vinext/packages/vinext/src/server/app-browser-entry.ts) | browser runtime bootstrap and hydration logic |
| [`../../vinext/packages/vinext/src/server/app-rsc-response-finalizer.ts`](../../vinext/packages/vinext/src/server/app-rsc-response-finalizer.ts) | RSC response finalization |
| [`../../vinext/packages/vinext/src/server/app-server-action-execution.ts`](../../vinext/packages/vinext/src/server/app-server-action-execution.ts) | server action execution |

## 요청/빌드/렌더 흐름

Initial HTML request:

```text
HTTP request
  -> RSC entry route match
  -> page/layout element tree build
  -> renderToReadableStream(RSC payload)
  -> SSR entry consumes RSC payload
  -> renderToReadableStream(HTML)
  -> browser entry hydrates from embedded/streamed RSC state
```

RSC navigation request:

```text
client navigation
  -> .rsc fetch
  -> RSC entry renders new payload
  -> browser runtime decodes payload
  -> visible tree commit
  -> history/url/scroll update
```

Server action:

```text
form/action request
  -> action metadata resolve
  -> action execute
  -> redirect/revalidate/result 처리
  -> follow-up RSC render
```

## 주요 함수와 책임

| Function / Area | What to remember |
| --- | --- |
| `generateRscEntry()` | route graph, module imports, action dispatch, prerender helper를 포함한 RSC entry source를 만든다. |
| `generateSsrEntry()` | SSR environment에서 HTML rendering을 위한 entry를 만든다. |
| `generateBrowserEntry()` | route manifest와 prefetch route 정보를 browser bootstrap에 심는다. |
| `buildPageElements()` in generated RSC entry | matched route metadata를 React element tree로 바꾼다. |
| `matchRoute()` / `findIntercept()` in generated RSC entry | request URL과 interception context를 route graph에 매칭한다. |
| `seedMemoryCacheFromPrerender()` | pre-rendered routes를 runtime memory cache로 seed한다. |

## 수정할 때 깨지기 쉬운 지점

- generated code는 TypeScript 파일처럼 보이지 않아도 runtime source다. 문자열 codegen 변경은 테스트가 특히 중요하다.
- RSC/SSR/browser environment에서 import 가능한 모듈이 다르다.
- RSC payload에 필요한 client reference manifest가 dedupe/resolve되지 않으면 hydration이 깨진다.
- server action 후 redirect, cache revalidation, RSC rerender 순서가 틀리면 UI와 URL이 어긋난다.
- prerender endpoints는 build-time HTTP fetch 경로와 production server secret에 의존한다.
- browser entry가 route manifest를 잘못 받으면 navigation planner가 soft navigation 가능성을 오판한다.

## 관련 테스트

- [`../../vinext/tests/app-router.test.ts`](../../vinext/tests/app-router.test.ts)
- [`../../vinext/tests/app-browser-entry.test.ts`](../../vinext/tests/app-browser-entry.test.ts)
- [`../../vinext/tests/app-rsc-handler.test.ts`](../../vinext/tests/app-rsc-handler.test.ts)
- [`../../vinext/tests/app-rsc-response-finalizer.test.ts`](../../vinext/tests/app-rsc-response-finalizer.test.ts)
- [`../../vinext/tests/app-ssr-stream.test.ts`](../../vinext/tests/app-ssr-stream.test.ts)
- [`../../vinext/tests/app-server-action-execution.test.ts`](../../vinext/tests/app-server-action-execution.test.ts)
- [`../../vinext/tests/rsc-streaming.test.ts`](../../vinext/tests/rsc-streaming.test.ts)
- [`../../vinext/tests/client-reference-dedup.test.ts`](../../vinext/tests/client-reference-dedup.test.ts)
- [`../../vinext/tests/e2e/app-router/hydration.spec.ts`](../../vinext/tests/e2e/app-router/hydration.spec.ts)
- [`../../vinext/tests/e2e/app-router/server-actions.spec.ts`](../../vinext/tests/e2e/app-router/server-actions.spec.ts)

## 복기용 체크리스트

- 변경한 모듈이 RSC, SSR, browser 중 어느 환경에서 실행되는가?
- generated entry가 route graph와 manifest를 같은 version/shape로 소비하는가?
- initial render와 client navigation render가 같은 boundary/cache semantics를 공유하는가?
- server action 이후 response가 action result와 rerender payload를 모두 포함하는가?
- prerender용 internal endpoint가 runtime public endpoint처럼 노출되지 않는가?
