# 07 Client Navigation History

## 한 줄 결론

client navigation은 Link click 하나가 아니라 prefetch cache, RSC fetch, visible commit, history state, scroll restoration이 함께 움직이는 runtime이다.

## 이 파트가 존재하는 이유

Next.js App Router의 navigation은 단순 `pushState`가 아니다. soft navigation 가능 여부를 판단하고, RSC payload를 가져오고, layout persistence를 지키며, URL/history/scroll을 화면 commit과 맞춰야 한다. Pages Router도 기존 `next/router` events와 data route loading을 유지해야 한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/shims/link.tsx`](../../vinext/packages/vinext/src/shims/link.tsx) | `<Link>` click/prefetch/status, App/Pages navigation delegation |
| [`../../vinext/packages/vinext/src/shims/navigation.ts`](../../vinext/packages/vinext/src/shims/navigation.ts) | App Router public hooks/router, client URL state, history patching |
| [`../../vinext/packages/vinext/src/shims/router.ts`](../../vinext/packages/vinext/src/shims/router.ts) | Pages Router singleton, router events, `beforePopState` |
| [`../../vinext/packages/vinext/src/client/navigation-runtime.ts`](../../vinext/packages/vinext/src/client/navigation-runtime.ts) | browser global navigation runtime registry |
| [`../../vinext/packages/vinext/src/client/pages-router-link-navigation.ts`](../../vinext/packages/vinext/src/client/pages-router-link-navigation.ts) | Pages Router Link navigation bridge |
| [`../../vinext/packages/vinext/src/server/app-browser-navigation-controller.ts`](../../vinext/packages/vinext/src/server/app-browser-navigation-controller.ts) | App Router visible commit and navigation lifecycle controller |
| [`../../vinext/packages/vinext/src/server/navigation-planner.ts`](../../vinext/packages/vinext/src/server/navigation-planner.ts) | soft/hard navigation, cache proof, persistence decision planner |
| [`../../vinext/packages/vinext/src/server/app-browser-popstate.ts`](../../vinext/packages/vinext/src/server/app-browser-popstate.ts) | App Router popstate coordination |
| [`../../vinext/packages/vinext/src/server/app-history-state.ts`](../../vinext/packages/vinext/src/server/app-history-state.ts) | history state shape helpers |

## 요청/빌드/렌더 흐름

App Router link navigation:

```text
<Link> visible
  -> prefetch route eligibility check
  -> RSC prefetch cache key 생성
  -> click
  -> navigateClientSide()
  -> navigation runtime navigate()
  -> .rsc request
  -> navigation planner decision
  -> visible commit
  -> URL/history/scroll update
```

Pages Router navigation:

```text
<Link> click or router.push()
  -> next/router shim
  -> page loader / _next/data fetch
  -> Router state update
  -> events emit
  -> React render/hydration 유지
```

## 주요 개념

| Concept | What to remember |
| --- | --- |
| Prefetch cache key | URL뿐 아니라 interception context/cache variant가 같이 영향을 줄 수 있다. |
| Visible commit | RSC payload가 도착했다고 바로 URL을 바꾸지 않고, 화면에 반영 가능한 commit인지 판단한다. |
| Navigation planner | current visible snapshot과 target route manifest를 비교해 soft navigation 가능성을 결정한다. |
| History patching | 외부 `pushState`/`replaceState`도 hooks snapshot과 충돌하지 않게 metadata를 보존한다. |
| Scroll restoration | popstate와 RSC render settle 타이밍을 맞춰야 old content scroll flash를 줄일 수 있다. |
| Pages Router events | `routeChangeStart`, `routeChangeComplete`, `beforePopState` 같은 legacy contract를 유지한다. |

## 수정할 때 깨지기 쉬운 지점

- URL을 먼저 바꾸고 RSC commit이 실패하면 화면과 주소가 갈라진다.
- hash-only navigation은 RSC fetch를 건너뛰어야 한다.
- refresh는 이전 prefetch/navigation cache를 무효화해야 stale payload를 재사용하지 않는다.
- intercepted route에서 cache proof가 없으면 보이는 UI를 잘못 재사용할 수 있다.
- App Router와 Pages Router가 모두 있을 때 `<Link>`의 대상 runtime 선택이 중요하다.
- history state에 vinext metadata를 보존하지 않으면 back/forward나 scroll restoration이 깨진다.

## 관련 테스트

- [`../../vinext/tests/navigation-runtime.test.ts`](../../vinext/tests/navigation-runtime.test.ts)
- [`../../vinext/tests/navigation-planner.test.ts`](../../vinext/tests/navigation-planner.test.ts)
- [`../../vinext/tests/link-navigation.test.ts`](../../vinext/tests/link-navigation.test.ts)
- [`../../vinext/tests/prefetch-cache.test.ts`](../../vinext/tests/prefetch-cache.test.ts)
- [`../../vinext/tests/app-optimistic-routing.test.ts`](../../vinext/tests/app-optimistic-routing.test.ts)
- [`../../vinext/tests/app-browser-entry.test.ts`](../../vinext/tests/app-browser-entry.test.ts)
- [`../../vinext/tests/e2e/app-router/navigation.spec.ts`](../../vinext/tests/e2e/app-router/navigation.spec.ts)
- [`../../vinext/tests/e2e/app-router/navigation-flows.spec.ts`](../../vinext/tests/e2e/app-router/navigation-flows.spec.ts)
- [`../../vinext/tests/e2e/app-router/nextjs-compat/hash-popstate-scroll.spec.ts`](../../vinext/tests/e2e/app-router/nextjs-compat/hash-popstate-scroll.spec.ts)
- [`../../vinext/tests/e2e/pages-router/router-events.spec.ts`](../../vinext/tests/e2e/pages-router/router-events.spec.ts)

## 복기용 체크리스트

- 이 navigation은 App Router인가 Pages Router인가?
- URL update 권한은 click handler가 아니라 commit path에 있는가?
- prefetch cache와 refresh invalidation이 같은 cache key 체계를 쓰는가?
- popstate에서 RSC render가 끝나기 전에 scroll을 복원하지 않는가?
- hard navigation fallback이 무한 루프를 만들지 않는가?
