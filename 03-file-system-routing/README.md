# 03 File System Routing

## 한 줄 결론

vinext의 routing 계층은 파일 시스템을 Next.js식 route pattern으로 변환하고, 요청 시 trie 기반 matcher로 빠르게 route와 params를 찾는다.

## 이 파트가 존재하는 이유

Next.js 호환성의 절반은 파일 이름 규칙이다. `pages/blog/[id].tsx`, `app/(marketing)/docs/[...slug]/page.tsx`, `app/@modal/(.)photo/page.tsx` 같은 경로가 URL, params, layout ownership, route priority로 바뀌어야 한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/routing/pages-router.ts`](../../vinext/packages/vinext/src/routing/pages-router.ts) | `pages/` route와 `pages/api/` route 스캔 |
| [`../../vinext/packages/vinext/src/routing/app-router.ts`](../../vinext/packages/vinext/src/routing/app-router.ts) | App Router route graph 캐시와 request-time match entry |
| [`../../vinext/packages/vinext/src/routing/app-route-graph.ts`](../../vinext/packages/vinext/src/routing/app-route-graph.ts) | App Router의 route, layout, slot, interception metadata 구성 |
| [`../../vinext/packages/vinext/src/routing/route-trie.ts`](../../vinext/packages/vinext/src/routing/route-trie.ts) | static/dynamic/catch-all 우선순위 기반 trie |
| [`../../vinext/packages/vinext/src/routing/route-matching.ts`](../../vinext/packages/vinext/src/routing/route-matching.ts) | Pages/App 공통 matching preamble |
| [`../../vinext/packages/vinext/src/routing/route-pattern.ts`](../../vinext/packages/vinext/src/routing/route-pattern.ts) | route pattern fill/match/static params normalization |
| [`../../vinext/packages/vinext/src/routing/file-matcher.ts`](../../vinext/packages/vinext/src/routing/file-matcher.ts) | page extensions 기준 valid file detection |
| [`../../vinext/packages/vinext/src/routing/route-validation.ts`](../../vinext/packages/vinext/src/routing/route-validation.ts) | route conflict/invalid pattern validation |

## 요청/빌드/렌더 흐름

Pages Router:

```text
pages/ scan
  -> fileToRoute()
  -> pattern, patternParts, isDynamic 생성
  -> compareRoutes()로 static > dynamic > catch-all 정렬
  -> buildRouteTrie()
  -> request pathname match
```

App Router:

```text
app/ scan
  -> app-route-graph 구성
  -> page.tsx / route.ts / layout.tsx / template.tsx / loading/error/not-found 등 발견
  -> route groups, parallel slots, intercepting routes metadata 반영
  -> appRouter()가 request matching용 route list 제공
  -> app-rsc-entry가 route graph/manifest를 codegen에 사용
```

## 주요 데이터 모델

| Model | Meaning |
| --- | --- |
| Pages `Route` | route pattern, file path, dynamic 여부, pattern parts를 가진 request match 단위 |
| App `AppRoute` | page/route handler/layout/boundary/slot metadata까지 포함한 App Router match 단위 |
| `RouteManifest` | browser navigation planner가 현재 route topology를 판단할 수 있게 만든 client-visible graph |
| `patternParts` | trie matching을 위해 route pattern을 segment 단위로 미리 쪼갠 값 |
| params | `[id]`, `[...slug]`, `[[...slug]]`에서 추출되는 `string | string[]` 값 |

## 수정할 때 깨지기 쉬운 지점

- route priority는 Next.js와 맞아야 한다. static route가 dynamic route보다 먼저 이겨야 한다.
- `pages/api`는 page route와 같은 스캐너 계열이지만 runtime은 API handler로 간다.
- App Router route group `(group)`과 parallel slot `@slot`은 URL에는 보이지 않지만 layout ownership에는 영향을 준다.
- intercepting routes는 "어떤 URL을 가로채는가"와 "어느 slot에 렌더되는가"를 동시에 추적해야 한다.
- optional catch-all은 빈 segment와 여러 segment를 모두 처리해야 한다.
- route graph cache invalidation이 dev HMR과 맞지 않으면 새 파일/삭제 파일이 반영되지 않는다.

## 관련 테스트

- [`../../vinext/tests/routing.test.ts`](../../vinext/tests/routing.test.ts)
- [`../../vinext/tests/route-trie.test.ts`](../../vinext/tests/route-trie.test.ts)
- [`../../vinext/tests/route-pattern.test.ts`](../../vinext/tests/route-pattern.test.ts)
- [`../../vinext/tests/route-sorting.test.ts`](../../vinext/tests/route-sorting.test.ts)
- [`../../vinext/tests/file-matcher.test.ts`](../../vinext/tests/file-matcher.test.ts)
- [`../../vinext/tests/app-route-graph.test.ts`](../../vinext/tests/app-route-graph.test.ts)
- [`../../vinext/tests/app-rsc-route-matching.test.ts`](../../vinext/tests/app-rsc-route-matching.test.ts)
- [`../../vinext/tests/page-extensions-routing.test.ts`](../../vinext/tests/page-extensions-routing.test.ts)
- [`../../vinext/tests/intercepting-routes-build.test.ts`](../../vinext/tests/intercepting-routes-build.test.ts)

## 복기용 체크리스트

- 파일 경로가 URL-visible segment와 invisible segment로 올바르게 나뉘는가?
- route pattern이 Pages/App 양쪽에서 같은 의미로 해석되는가?
- 동적 route params 이름과 값 shape가 Next.js와 맞는가?
- HMR이나 build 재실행에서 cache가 stale하지 않은가?
- App Router route graph의 layout/slot/boundary metadata가 request-time render까지 보존되는가?
