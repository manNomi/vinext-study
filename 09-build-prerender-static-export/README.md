# 09 Build Prerender Static Export

## 한 줄 결론

build 계층은 Vite output을 Next.js식 배포 산출물로 바꾸고, static/dynamic route 판단, prerender, static export, standalone output을 조율한다.

## 이 파트가 존재하는 이유

Next.js의 `next build`는 단순 번들링이 아니라 route classification, static generation, ISR cache seed, build report, standalone server output, static export를 함께 수행한다. vinext는 Vite build 결과 위에서 이 산출물을 재구성해야 한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/build/prerender.ts`](../../vinext/packages/vinext/src/build/prerender.ts) | Pages/App prerender의 실제 구현 |
| [`../../vinext/packages/vinext/src/build/run-prerender.ts`](../../vinext/packages/vinext/src/build/run-prerender.ts) | build/deploy에서 공통으로 쓰는 prerender orchestration |
| [`../../vinext/packages/vinext/src/build/static-export.ts`](../../vinext/packages/vinext/src/build/static-export.ts) | `output: 'export'`용 wrapper |
| [`../../vinext/packages/vinext/src/build/report.ts`](../../vinext/packages/vinext/src/build/report.ts) | build report route classification |
| [`../../vinext/packages/vinext/src/build/layout-classification.ts`](../../vinext/packages/vinext/src/build/layout-classification.ts) | App layout static/dynamic classification |
| [`../../vinext/packages/vinext/src/build/route-classification-manifest.ts`](../../vinext/packages/vinext/src/build/route-classification-manifest.ts) | route classification manifest codegen |
| [`../../vinext/packages/vinext/src/build/route-classification-injector.ts`](../../vinext/packages/vinext/src/build/route-classification-injector.ts) | generated RSC chunk에 build-time classification 주입 |
| [`../../vinext/packages/vinext/src/build/standalone.ts`](../../vinext/packages/vinext/src/build/standalone.ts) | `output: 'standalone'` 산출물 생성 |
| [`../../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../../vinext/packages/vinext/src/build/nitro-route-rules.ts) | Nitro route rules 생성 |
| [`../../vinext/packages/vinext/src/build/precompress.ts`](../../vinext/packages/vinext/src/build/precompress.ts) | build assets precompression |

## 요청/빌드/렌더 흐름

Normal build:

```text
vinext build
  -> Vite multi-environment build
  -> client/server/RSC bundles
  -> route classification
  -> prerender static/ISR candidates
  -> dist/server/vinext-prerender.json
  -> build report
  -> optional standalone/precompress/nitro rules
```

Nitro build:

```text
vite.config.ts
  -> plugins: [vinext(), nitro()]
  -> vinext() creates Next-compatible route/runtime graph
  -> nitro() creates platform server output
  -> vinext:nitro-route-rules runs during nitro.setup
  -> ISR routes become Nitro routeRules with swr values
```

Static export:

```text
next.config output: "export"
  -> export mode prerender
  -> Pages dynamic routes require getStaticPaths fallback false
  -> App dynamic routes require generateStaticParams
  -> HTML/JSON/RSC output written for static hosting
  -> SSR/API/dynamic-only routes become build errors or skipped warnings
```

Prerender internals:

```text
production server start
  -> internal /__vinext/prerender/* endpoint with secret
  -> getStaticPaths / generateStaticParams enumeration
  -> route URL render via HTTP
  -> HTML/RSC/data files written
  -> manifest merged for App + Pages hybrid projects
```

## 주요 개념

| Concept | What to remember |
| --- | --- |
| Prerender manifest | `vinext-prerender.json` is runtime's index for pre-rendered routes and cache seeding. |
| Export mode | static hosting을 위해 dynamic server behavior를 허용하지 않는다. |
| Route classification | segment config, module graph, speculative prerender 결과가 static/dynamic 판단에 쓰인다. |
| Internal prerender secret | build-time HTTP endpoint가 외부처럼 열리지 않게 보호한다. |
| Hybrid projects | `app/`와 `pages/`가 같이 있으면 prerender result를 하나로 merge해야 한다. |
| Standalone output | server bundle, client assets, runtime deps, `server.js`를 self-hosting 폴더로 복사한다. |
| Nitro routeRules | vinext의 ISR route를 Nitro의 `routeRules`로 변환한다. 내부 `:param` pattern은 Nitro/rou3의 `*`/`**` wildcard로 바뀐다. |

## Nitro 연동 요약

Nitro는 Cloudflare가 아닌 여러 deployment target을 Vite plugin으로 감싸는 쪽이다. vinext는 Nitro 자체를 직접 dependency로 두지 않고, Nitro plugin convention인 `nitro.setup` hook에 맞춰 routeRules를 주입한다.

핵심 구현은 [`../../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../../vinext/packages/vinext/src/build/nitro-route-rules.ts)다.

- `collectNitroRouteRules()`가 `app/`, `pages/`, `pages/api/`를 스캔한다.
- `buildReportRows()` 결과 중 ISR route만 골라낸다.
- `generateNitroRouteRules()`가 `{ "/blog/*": { swr: 60 } }` 같은 Nitro routeRules를 만든다.
- `convertToNitroPattern()`이 vinext 내부 route pattern을 Nitro의 rou3 wildcard로 변환한다.
- `mergeNitroRouteRules()`는 사용자가 이미 cache/static/isr/prerender rule을 지정한 route를 덮어쓰지 않는다.

예시:

```text
vinext pattern: /blog/:slug
Nitro pattern:  /blog/*

vinext pattern: /docs/:slug+
Nitro pattern:  /docs/**
```

## 수정할 때 깨지기 쉬운 지점

- `output: 'export'`와 default prerender는 비슷해 보여도 허용 정책이 다르다.
- dynamic route params를 URL로 채울 때 catch-all/optional catch-all shape를 잘못 다루면 build output path가 틀어진다.
- App Router는 `generateStaticParams`가 parent dynamic params와 top-down으로 연결된다.
- Pages Router `getStaticPaths`는 production server의 internal endpoint를 통해 가져오는 경로가 있다.
- prerender manifest 위치는 runtime start/deploy/cache seed와 맞아야 한다.
- route classification은 build report용 표시가 아니라 RSC entry runtime decision에도 연결된다.
- Nitro routeRules는 prerender 결과가 아니라 build 전 filesystem/static analysis 기반이므로 speculative prerender로만 static 판정된 route와 차이가 날 수 있다.
- 사용자가 명시한 Nitro cache rule을 자동 생성 rule이 덮어쓰면 안 된다.

## 관련 테스트

- [`../../vinext/tests/prerender.test.ts`](../../vinext/tests/prerender.test.ts)
- [`../../vinext/tests/run-prerender-concurrency.test.ts`](../../vinext/tests/run-prerender-concurrency.test.ts)
- [`../../vinext/tests/static-export.test.ts`](../../vinext/tests/static-export.test.ts)
- [`../../vinext/tests/app-static-generation.test.ts`](../../vinext/tests/app-static-generation.test.ts)
- [`../../vinext/tests/build-report.test.ts`](../../vinext/tests/build-report.test.ts)
- [`../../vinext/tests/layout-classification.test.ts`](../../vinext/tests/layout-classification.test.ts)
- [`../../vinext/tests/route-classification-manifest.test.ts`](../../vinext/tests/route-classification-manifest.test.ts)
- [`../../vinext/tests/route-classification-injector.test.ts`](../../vinext/tests/route-classification-injector.test.ts)
- [`../../vinext/tests/nitro-route-rules.test.ts`](../../vinext/tests/nitro-route-rules.test.ts)
- [`../../vinext/tests/standalone-build.test.ts`](../../vinext/tests/standalone-build.test.ts)
- [`../../vinext/tests/e2e/static-export/app-router.spec.ts`](../../vinext/tests/e2e/static-export/app-router.spec.ts)
- [`../../vinext/tests/e2e/standalone-output/basic.spec.ts`](../../vinext/tests/e2e/standalone-output/basic.spec.ts)

## 복기용 체크리스트

- 이 route는 static, ISR, dynamic, API, internal 중 무엇인가?
- export mode에서 이 route가 허용되는가?
- prerender output path와 runtime lookup path가 같은 규칙을 쓰는가?
- App+Pages hybrid build에서 manifest가 덮어써지지 않고 merge되는가?
- standalone output에 외부화된 runtime dependency가 빠지지 않는가?
- Nitro routeRules 변환이 dynamic/catch-all/optional catch-all pattern을 올바르게 보존하는가?
