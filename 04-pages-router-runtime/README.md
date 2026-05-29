# 04 Pages Router Runtime

## 한 줄 결론

Pages Router runtime은 `pages/` route table을 기반으로 SSR, API route, data route, `_app`, `_document`, ISR, client hydration을 Next.js 방식에 가깝게 재구현한다.

## 이 파트가 존재하는 이유

Pages Router는 App Router와 전혀 다른 데이터 모델을 가진다. `getServerSideProps`, `getStaticProps`, `getStaticPaths`, `_app`, `_document`, `__NEXT_DATA__`, `/_next/data/*` 요청을 모두 처리해야 하므로 별도의 runtime entry와 server handler가 필요하다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/entries/pages-server-entry.ts`](../../vinext/packages/vinext/src/entries/pages-server-entry.ts) | production/server bundle용 Pages SSR/API/data route entry codegen |
| [`../../vinext/packages/vinext/src/entries/pages-client-entry.ts`](../../vinext/packages/vinext/src/entries/pages-client-entry.ts) | client hydration entry와 page loader manifest |
| [`../../vinext/packages/vinext/src/server/dev-server.ts`](../../vinext/packages/vinext/src/server/dev-server.ts) | dev server의 Pages SSR request handler |
| [`../../vinext/packages/vinext/src/server/pages-page-response.ts`](../../vinext/packages/vinext/src/server/pages-page-response.ts) | HTML stream, `_document`, `__NEXT_DATA__`, ISR cache write |
| [`../../vinext/packages/vinext/src/server/pages-page-data.ts`](../../vinext/packages/vinext/src/server/pages-page-data.ts) | `getStaticProps`/`getServerSideProps` 결과 처리 |
| [`../../vinext/packages/vinext/src/server/pages-data-route.ts`](../../vinext/packages/vinext/src/server/pages-data-route.ts) | `/_next/data/<buildId>/*.json` normalization/response |
| [`../../vinext/packages/vinext/src/server/pages-api-route.ts`](../../vinext/packages/vinext/src/server/pages-api-route.ts) | Web Request 기반 production API route 처리 |
| [`../../vinext/packages/vinext/src/server/api-handler.ts`](../../vinext/packages/vinext/src/server/api-handler.ts) | Node dev server API route handler |
| [`../../vinext/packages/vinext/src/shims/router.ts`](../../vinext/packages/vinext/src/shims/router.ts) | client/server `next/router` state와 events |

## 요청/빌드/렌더 흐름

Pages SSR:

```text
Request
  -> middleware/config pipeline
  -> Pages route match
  -> page module import
  -> getServerSideProps 또는 getStaticProps 실행
  -> _app + Page element 구성
  -> renderToReadableStream()
  -> _document render
  -> __NEXT_DATA__ 삽입
  -> client entry가 hydrateRoot()
```

Pages data route:

```text
/_next/data/<buildId>/route.json
  -> page pathname으로 normalize
  -> 같은 route/data fetching machinery 실행
  -> JSON envelope 반환
```

Pages API route:

```text
pages/api/*
  -> API route match
  -> Edge API면 NextRequest/Response path
  -> Node-style API면 req/res helper 확장
  -> Response 또는 Node res write
```

## 주요 함수와 책임

| Function | What to remember |
| --- | --- |
| `generateServerEntry()` | route table과 page module imports를 production entry 코드로 굽는다. |
| `renderPage()` in generated entry | Web Request를 받아 Pages SSR Response를 만든다. |
| `createSSRHandler()` | dev server에서 Vite module runner를 사용해 page module을 로드한다. |
| `renderPagesPageResponse()` | streamed HTML, `_document`, asset tags, `__NEXT_DATA__`, cache headers를 합친다. |
| `parseNextDataPathname()` | data route URL을 원래 page URL로 되돌린다. |
| `handlePagesApiRoute()` | production/Web API route path를 처리한다. |
| `handleApiRoute()` | dev Node API route path를 처리한다. |
| `generateClientEntry()` | route pattern별 dynamic import loader를 만들고 hydration을 시작한다. |

## 수정할 때 깨지기 쉬운 지점

- dev server와 production Pages server가 같은 request order를 가져야 한다.
- `/_next/data` 요청은 middleware보다 먼저 page URL로 normalize되어야 하는 경우가 있다.
- `getStaticPaths`의 `fallback: true`, `"blocking"`, `false`는 SSR/data route/ISR 동작이 다르다.
- `_document`가 `__NEXT_DATA__`를 포함하지 않는 경우 fallback injection이 필요하다.
- Pages Router i18n은 URL locale prefix, cookie, Accept-Language와 맞물린다.
- API route는 Edge-style default handler와 Node-style req/res handler를 구분해야 한다.

## 관련 테스트

- [`../../vinext/tests/pages-router.test.ts`](../../vinext/tests/pages-router.test.ts)
- [`../../vinext/tests/pages-page-response.test.ts`](../../vinext/tests/pages-page-response.test.ts)
- [`../../vinext/tests/pages-page-data.test.ts`](../../vinext/tests/pages-page-data.test.ts)
- [`../../vinext/tests/pages-data-route.test.ts`](../../vinext/tests/pages-data-route.test.ts)
- [`../../vinext/tests/pages-api-route.test.ts`](../../vinext/tests/pages-api-route.test.ts)
- [`../../vinext/tests/pages-i18n.test.ts`](../../vinext/tests/pages-i18n.test.ts)
- [`../../vinext/tests/pages-router-concurrency.test.ts`](../../vinext/tests/pages-router-concurrency.test.ts)
- [`../../vinext/tests/e2e/pages-router/navigation.spec.ts`](../../vinext/tests/e2e/pages-router/navigation.spec.ts)
- [`../../vinext/tests/e2e/pages-router-prod/production.spec.ts`](../../vinext/tests/e2e/pages-router-prod/production.spec.ts)

## 복기용 체크리스트

- page render와 data route render가 같은 data fetching 결과 shape를 쓰는가?
- dev와 production에서 API route runtime 차이가 노출되지 않는가?
- `_app`, `_document`, page-specific assets가 SSR manifest에서 누락되지 않는가?
- fallback route에서 `useRouter().isFallback`과 data request 동작이 맞는가?
- middleware/config headers가 최종 HTML/API/data response에 병합되는가?
