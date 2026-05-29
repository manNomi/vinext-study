# 05 App Router Route Tree

## 한 줄 결론

App Router의 핵심은 URL match 하나가 아니라, matched page를 둘러싼 layout/template/loading/error/not-found/slot/interception metadata를 route graph로 보존하는 것이다.

## 이 파트가 존재하는 이유

App Router는 URL 하나를 컴포넌트 하나로 렌더하지 않는다. nested layouts, templates, route segment config, parallel routes, intercepting routes, metadata, route handlers, loading/error boundaries가 한 route tree 안에서 조합된다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/routing/app-route-graph.ts`](../../vinext/packages/vinext/src/routing/app-route-graph.ts) | App Router route graph 생성의 중심 |
| [`../../vinext/packages/vinext/src/routing/app-router.ts`](../../vinext/packages/vinext/src/routing/app-router.ts) | graph cache와 request match API |
| [`../../vinext/packages/vinext/src/server/app-page-route-wiring.tsx`](../../vinext/packages/vinext/src/server/app-page-route-wiring.tsx) | graph metadata를 실제 React element tree로 wiring |
| [`../../vinext/packages/vinext/src/server/app-elements.ts`](../../vinext/packages/vinext/src/server/app-elements.ts) | App element tree transport 구조 |
| [`../../vinext/packages/vinext/src/server/app-page-boundary.ts`](../../vinext/packages/vinext/src/server/app-page-boundary.ts) | App Router boundary 모델 |
| [`../../vinext/packages/vinext/src/server/app-page-boundary-render.ts`](../../vinext/packages/vinext/src/server/app-page-boundary-render.ts) | boundary rendering |
| [`../../vinext/packages/vinext/src/server/metadata-routes.ts`](../../vinext/packages/vinext/src/server/metadata-routes.ts) | file-based metadata route scanner |
| [`../../vinext/packages/vinext/src/server/file-based-metadata.ts`](../../vinext/packages/vinext/src/server/file-based-metadata.ts) | metadata file handling |
| [`../../vinext/packages/vinext/src/server/app-segment-config.ts`](../../vinext/packages/vinext/src/server/app-segment-config.ts) | segment config parsing/normalization |

## 요청/빌드/렌더 흐름

```text
app/ directory
  -> app-route-graph scan
  -> route segments -> URL pattern + params
  -> layouts/templates/errors/loading/not-found discovery
  -> parallel slots and intercepting routes discovery
  -> routeManifest 생성
  -> app-rsc-entry codegen
  -> app-page-route-wiring이 React tree 구성
  -> RSC stream render
```

## 주요 개념

| Concept | What to remember |
| --- | --- |
| Route groups | URL에는 안 보이지만 layout/boundary discovery에는 남는다. |
| Parallel routes | `@slot` 디렉터리가 layout props와 default/page fallback에 영향을 준다. |
| Intercepting routes | 현재 URL과 intercepted target URL을 동시에 고려한다. |
| Layout tree positions | layout별 boundary/slot ownership를 안정적으로 연결하기 위한 위치 정보다. |
| Segment config | `dynamic`, `revalidate`, `dynamicParams` 등이 static/dynamic classification과 render policy에 영향을 준다. |
| Route handlers | `route.ts`는 page tree가 아니라 HTTP method dispatch로 간다. |
| Metadata routes | sitemap, robots, manifest, favicon, OG image 같은 file convention이 별도 route로 materialize된다. |

## 수정할 때 깨지기 쉬운 지점

- URL-transparent segment를 제거하는 타이밍과 layout ownership을 보존하는 타이밍을 섞으면 안 된다.
- slot의 `default.tsx`와 mirrored slot page 선택은 navigation persistence에 영향을 준다.
- intercepting route는 source route와 target route의 관계가 틀리면 client navigation에서 hard navigation으로 떨어질 수 있다.
- error/not-found/forbidden/unauthorized boundary wrapping 순서는 사용자 눈에 바로 보이는 차이를 만든다.
- metadata route는 일반 page route와 다른 response/build data path를 가진다.
- route graph의 browser-facing manifest는 navigation planner와 맞아야 한다.

## 관련 테스트

- [`../../vinext/tests/app-route-graph.test.ts`](../../vinext/tests/app-route-graph.test.ts)
- [`../../vinext/tests/app-page-route-wiring.test.ts`](../../vinext/tests/app-page-route-wiring.test.ts)
- [`../../vinext/tests/app-elements.test.ts`](../../vinext/tests/app-elements.test.ts)
- [`../../vinext/tests/app-page-boundary.test.ts`](../../vinext/tests/app-page-boundary.test.ts)
- [`../../vinext/tests/app-page-boundary-render.test.ts`](../../vinext/tests/app-page-boundary-render.test.ts)
- [`../../vinext/tests/app-segment-config.test.ts`](../../vinext/tests/app-segment-config.test.ts)
- [`../../vinext/tests/metadata-routes.test.ts`](../../vinext/tests/metadata-routes.test.ts)
- [`../../vinext/tests/file-based-metadata.test.ts`](../../vinext/tests/file-based-metadata.test.ts)
- [`../../vinext/tests/e2e/app-router/inherited-parallel-slots.spec.ts`](../../vinext/tests/e2e/app-router/inherited-parallel-slots.spec.ts)
- [`../../vinext/tests/e2e/app-router/metadata.spec.ts`](../../vinext/tests/e2e/app-router/metadata.spec.ts)

## 복기용 체크리스트

- route graph가 request matching, RSC rendering, browser navigation에서 같은 route identity를 공유하는가?
- slot/default/interception metadata가 layout boundary와 같은 index 체계를 쓰는가?
- metadata file routes가 static export/prerender와 연결되는가?
- segment config가 build report, prerender, runtime render policy에 모두 반영되는가?
- boundary wrapping 순서가 Next.js와 맞는가?
