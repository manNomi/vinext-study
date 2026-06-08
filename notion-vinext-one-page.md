# vinext 구조 총정리 - Notion One Page

이 페이지는 `vinext-study`의 모든 챕터를 Notion 한 페이지에 붙여 넣거나 Markdown import하기 좋게 재구성한 단일 문서다.

목표는 학습 순서 안내가 아니라, `vinext`를 수정하거나 리뷰할 때 "어떤 기능이 어디에서 시작되고, 어떤 파일을 지나며, 어떤 테스트로 검증되는가"를 한 페이지에서 판단하게 만드는 것이다.

## 페이지 사용법

- 처음 읽을 때는 `Executive Summary -> One Screen Model -> Core Layer Map -> Chapter Notes` 순서로 읽는다.
- 실제 코드를 고칠 때는 `Modification Compass`에서 증상을 찾고, 해당 챕터의 `깨지기 쉬운 지점`과 `테스트`를 확인한다.
- Next.js와 vinext의 차이가 애매할 때는 `vinext vs Next` 섹션을 먼저 보고, `.nextjs-ref/test`와 `.nextjs-ref/packages/next/src`를 확인한다.
- 배포 target이 헷갈릴 때는 `Vite, Nitro, Cloudflare Plugin Stack`과 `Deployment Path Decision`을 먼저 본다.

## Executive Summary

vinext는 Next.js 앱을 Vite 기반 런타임에서 실행하기 위한 compatibility layer다. 사용자의 앱 코드는 `pages/`, `app/`, `next/link`, `next/navigation`, `next/server`, `next.config.js` 같은 Next.js 표면을 유지한다. 하지만 내부 구현은 Next.js compiler/runtime을 그대로 쓰지 않고, Vite plugin, local shim, filesystem route scanner, virtual entry, Web API 기반 server runtime으로 다시 구성한다.

핵심은 네 가지다.

| Core | Meaning |
| --- | --- |
| `vinext()` | Next.js API compatibility를 Vite plugin으로 구현하는 중앙 허브 |
| `shims/` | `next/*` import를 local implementation으로 바꾸는 공개 API 호환 계층 |
| `routing/` | `pages/`와 `app/` 파일 시스템을 route table/route graph로 바꾸는 계층 |
| `entries/` + `server/` | RSC, SSR, browser, Pages runtime, request pipeline을 실제 실행하는 계층 |

한 줄 모델은 다음과 같다.

```text
사용자 Next.js 앱
  -> vinext CLI
  -> vinext() Vite plugin
  -> next/* shim alias
  -> pages/app filesystem routing
  -> virtual RSC/SSR/browser/pages entries
  -> dev server 또는 production build
  -> Node server / Cloudflare Workers / Nitro-supported platform
```

## One Screen Model

vinext는 "Next.js 앱처럼 생긴 소스"와 "Vite가 실행할 수 있는 module graph" 사이에 끼는 번역기다.

```text
Next.js semantics
  pages/
  app/
  next/*
  next.config.*
  middleware.ts / proxy.ts
  metadata files
  SSR / RSC / ISR / route handlers

        |
        v

vinext compatibility layer
  CLI
  Vite plugin hooks
  shims
  route scanner
  virtual entries
  request context
  build/prerender/deploy glue

        |
        v

Runtime/output targets
  Vite dev server
  Node production server
  Cloudflare Workers native
  Nitro platform output
```

## Core Layer Map

| Layer | Main Question | Primary Owner | First Files |
| --- | --- | --- | --- |
| CLI/config | 사용자가 어떤 명령으로 vinext를 실행하는가? | `cli.ts`, `index.ts` | `../vinext/packages/vinext/src/cli.ts`, `../vinext/packages/vinext/src/index.ts` |
| Next API shim | `next/*` import를 어떻게 보존하는가? | `shims/` | `../vinext/packages/vinext/src/shims` |
| Routing | 파일 이름을 URL과 params로 어떻게 바꾸는가? | `routing/` | `pages-router.ts`, `app-route-graph.ts`, `route-trie.ts` |
| Pages runtime | `pages/`의 SSR/API/data/ISR을 어떻게 처리하는가? | `entries/pages-*`, `server/pages-*` | `pages-server-entry.ts`, `dev-server.ts`, `prod-server.ts` |
| App route tree | layout, slot, boundary, metadata를 어떻게 보존하는가? | App route graph and wiring | `app-route-graph.ts`, `app-page-route-wiring.tsx` |
| RSC/SSR/browser | App Router를 어떤 entry들로 나눠 실행하는가? | `entries/app-*` | `app-rsc-entry.ts`, `app-ssr-entry.ts`, `app-browser-entry.ts` |
| Client navigation | soft navigation, history, scroll을 어떻게 맞추는가? | navigation shims/runtime | `navigation.ts`, `link.tsx`, `navigation-planner.ts` |
| Request context | `headers()`, `cookies()`, middleware state를 어디에 보관하는가? | ALS context shims | `unified-request-context.ts`, `headers.ts`, `middleware.ts` |
| Build/prerender | Vite output을 Next.js식 산출물로 어떻게 바꾸는가? | `build/` | `prerender.ts`, `run-prerender.ts`, `static-export.ts` |
| Cloudflare/Nitro | 어떤 platform output으로 배포하는가? | `deploy.ts`, `cloudflare/`, Nitro route rules | `deploy.ts`, `kv-cache-handler.ts`, `nitro-route-rules.ts` |

## vinext vs Next

### 핵심 결론

Next.js는 Next.js API의 원본 구현이고, vinext는 그 공개 API surface를 Vite 위에서 다시 실행시키는 대체 구현이다.

vinext가 맞추려는 것은 사용자에게 보이는 공개 동작이다.

- `pages/`와 `app/` 파일 규칙
- `next/*` public imports
- SSR, RSC, route handler, middleware, metadata
- `getStaticProps`, `getServerSideProps`, `generateStaticParams`
- `headers()`, `cookies()`, `NextRequest`, `NextResponse`
- client navigation, prefetch, history, hydration

vinext가 그대로 복사하지 않으려는 것은 Next.js 내부 구현이다.

- webpack/Turbopack/SWC 중심 build pipeline
- Next.js internal manifest shape
- private route module class
- app render runtime의 private structure
- Vercel-specific undocumented behavior
- 오래된 deprecated API behavior

### 왜 vinext가 나왔는가

vinext의 질문은 "Next.js API가 반드시 Next.js compiler/runtime에서만 실행되어야 하는가"다.

Vite는 빠른 HMR, native ESM dev server, 명확한 plugin API, 풍부한 생태계를 갖고 있다. 여기에 `@vitejs/plugin-rsc`가 React Server Components를 지원하면서, Vite 위에서도 RSC framework를 구성할 수 있는 기반이 생겼다.

vinext는 이 가능성을 바탕으로 기존 Next.js 앱 코드를 유지하면서 Vite toolchain, Cloudflare Workers native path, Nitro multi-platform output을 사용할 수 있는지 실험한다.

### 왜 Next.js 코드를 그대로 가져오지 않는가

사용자 앱 코드는 최대한 그대로 가져오는 것이 vinext의 목표다. 하지만 Next.js framework 내부 코드를 그대로 가져오는 것은 목표와 맞지 않는다.

| Reason | Explanation |
| --- | --- |
| Toolchain mismatch | Next.js 내부 코드는 webpack/Turbopack/SWC, loader, manifest, route module을 전제로 한다. vinext는 Vite plugin hook과 module graph를 전제로 한다. |
| Public API vs private contract | 사용자가 기대하는 것은 공개 API 결과이지, Next.js private manifest/class 구조가 아니다. |
| Vite advantage loss | Next 내부 runtime을 많이 가져올수록 Vite는 단순 wrapper가 되고, Vite-native 구조의 장점이 사라진다. |
| Runtime target mismatch | vinext는 Cloudflare Workers, Nitro, Web API 중심 runtime을 중요하게 본다. Next 내부 코드는 Node/Vercel/Next output 전제를 끌고 올 수 있다. |
| Maintenance cost | 내부 코드를 복사하면 Next.js upstream 변경을 계속 따라가야 한다. vinext는 source copy보다 behavior mapping을 택한다. |
| Pragmatic compatibility | vinext는 일반 실사용 Next 앱을 목표로 하며, undocumented Vercel behavior를 전부 bug-for-bug로 복제하려 하지 않는다. |

### 같아야 하는 것과 달라도 되는 것

| Must Match | Can Differ |
| --- | --- |
| 사용자 앱에서 관찰 가능한 Next.js 공개 API 동작 | Next.js 내부 manifest shape |
| route convention과 params shape | webpack/Turbopack loader 구조 |
| SSR/RSC/hydration 결과 | private route module class |
| request-scoped API behavior | Vercel-specific undocumented behavior |
| middleware/header/cookie 전파 | Vite/Nitro/Cloudflare에 맞춘 build output |
| Next.js 테스트가 규정한 behavior | deprecated API나 오래된 version behavior |

### 작업 판단 순서

1. 이 문제가 공개 API 문제인지 내부 구현 문제인지 나눈다.
2. 사용자 앱에서 관찰되는 증상을 먼저 적는다.
3. `.nextjs-ref/test`에서 관련 Next.js 테스트를 찾는다.
4. `.nextjs-ref/packages/next/src`에서 Next.js 구현을 읽고 의도를 확인한다.
5. vinext의 책임 계층을 고른다.
6. 가능한 경우 Next.js 테스트를 vinext 테스트로 port한다.
7. 구현은 vinext의 `shims`, `routing`, `entries`, `server`, `build` 구조에 맞춰 작성한다.
8. dev/prod/Workers/Nitro 중 영향을 받는 경로를 함께 확인한다.
9. 의도적 divergence는 테스트명이나 문서에 남긴다.

## Vite, Nitro, Cloudflare Plugin Stack

### 한 줄 결론

`vinext()`는 Next.js compatibility를 Vite plugin으로 구현하는 층이고, `nitro()`는 그 결과물을 여러 server platform에 올리기 위한 deployment/runtime adapter다. Cloudflare Workers로 갈 때는 `nitro()` 대신 `@cloudflare/vite-plugin`과 `vinext deploy`가 native path를 만든다.

```text
Next.js app source
  -> vinext(): Next.js semantics on Vite
  -> nitro(): generic multi-platform server output

Next.js app source
  -> vinext(): Next.js semantics on Vite
  -> cloudflare(): Workers-native output and bindings
```

### Plugin role

| Plugin | Responsibility |
| --- | --- |
| `vinext()` | `next/*` shim, Next config/env, filesystem routing, virtual entries, RSC/SSR/browser wiring, build/prerender metadata |
| `@vitejs/plugin-rsc` | `"use client"`와 `"use server"` boundary, RSC multi-environment graph |
| `nitro()` | Vercel, Netlify, AWS, Deno Deploy 등 Nitro-supported platform output |
| `cloudflare()` | Workers build environment, bindings, ASSETS, Images, workerd runtime |

### 최소 Vite config

Cloudflare가 아닌 multi-platform target에서는 다음 구성이 기본 mental model이다.

```ts
import { defineConfig } from "vite";
import vinext from "vinext";
import { nitro } from "nitro/vite";

export default defineConfig({
  plugins: [vinext(), nitro()],
});
```

Cloudflare Workers native target에서는 Cloudflare plugin과 Workers environment가 중요하다.

```ts
import { defineConfig } from "vite";
import vinext from "vinext";
import { cloudflare } from "@cloudflare/vite-plugin";

export default defineConfig({
  plugins: [
    vinext(),
    cloudflare({
      viteEnvironment: { name: "rsc", childEnvironments: ["ssr"] },
    }),
  ],
});
```

### 왜 순서가 중요한가

권장 순서는 `plugins: [vinext(), nitro()]`다. `vinext()`가 먼저 Next.js semantics를 Vite graph에 심고, `nitro()`가 그 결과를 platform output으로 패키징한다.

`vinext()`가 먼저 하는 일은 다음과 같다.

- `next/*` import를 shim으로 resolve한다.
- virtual entry module id를 등록한다.
- App Router의 RSC/SSR/browser entry를 만든다.
- Pages Router의 server/client entry를 만든다.
- bundled runtime에 맞는 external/chunk policy를 조정한다.
- Nitro plugin이 있으면 Nitro routeRules hook을 준비한다.

### Nitro routeRules

Nitro routeRules는 Nitro가 route별 caching/ISR/SWR 정책을 반영할 때 쓰는 설정이다. vinext는 Next.js의 ISR 의미를 Nitro가 이해하는 `routeRules`로 변환한다.

```text
nitro.setup hook
  -> collectNitroRouteRules()
  -> appRouter()/pagesRouter()/apiRouter() route scan
  -> buildReportRows()
  -> ISR route만 선택
  -> generateNitroRouteRules()
  -> convertToNitroPattern()
  -> mergeNitroRouteRules()
  -> nitro.options.routeRules 갱신
```

예시는 다음과 같다.

```text
vinext internal pattern: /blog/:slug
Nitro rou3 pattern:      /blog/*

vinext internal pattern: /docs/:slug+
Nitro rou3 pattern:      /docs/**

Next-style route:
  /blog/:slug
  revalidate = 60

Nitro routeRules:
  /blog/* -> { swr: 60 }
```

중요한 제약은 `nitro.setup`이 build 초기에 실행된다는 점이다. speculative prerender 결과는 아직 없으므로, Nitro routeRules는 prerender 결과가 아니라 filesystem scan과 static analysis 기반으로 생성된다.

사용자가 이미 `swr`, `cache`, `static`, `isr`, `prerender` 같은 cache 관련 rule을 지정한 route는 자동 생성 rule이 덮어쓰지 않는다.

### Deployment Path Decision

| Need | Better Path |
| --- | --- |
| Cloudflare Workers first-class deploy | `vinext deploy` + `@cloudflare/vite-plugin` |
| `cloudflare:workers` binding access | Cloudflare native |
| KV-backed ISR with `KVCacheHandler` | Cloudflare native |
| Cloudflare Images optimization | Cloudflare native |
| Traffic-aware prerendering, TPR | Cloudflare native |
| Vercel/Netlify/AWS/Deno 등 다른 target | Nitro |
| 여러 hosting provider 실험 | Nitro |
| provider-specific Nitro preset | Nitro |

## End-to-End Runtime Flows

### CLI to Vite plugin

```text
vinext CLI
  -> command 분기
  -> Vite createServer/build 또는 vinext 자체 start/deploy/init/check
  -> vinext() plugin 설치
  -> config hook에서 Next config, aliases, build config, RSC 설정 구성
  -> resolveId/load/transform hook에서 next/* shim과 virtual module 제공
  -> dev: configureServer middleware가 routing/runtime 처리
  -> build: generateBundle/writeBundle/closeBundle hook이 manifest, prerender, deploy 준비
```

### Pages Router SSR

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

### App Router initial HTML

```text
HTTP request
  -> RSC entry route match
  -> page/layout element tree build
  -> renderToReadableStream(RSC payload)
  -> SSR entry consumes RSC payload
  -> renderToReadableStream(HTML)
  -> browser entry hydrates from embedded/streamed RSC state
```

### App Router client navigation

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

### Request context

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

### Build and prerender

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

### Cloudflare deploy

```text
vinext deploy
  -> project detection
  -> missing deps check/install
  -> wrangler.jsonc generation if needed
  -> worker/index.ts generation if needed
  -> vite.config.ts generation/check
  -> Vite build with Cloudflare plugin
  -> optional TPR
  -> wrangler deploy
```

## Chapter Notes

### 01 Vite Plugin Entry

한 줄 결론: vinext의 시작점은 CLI가 Vite를 실행하고, `vinext()` Vite plugin이 Next.js식 프로젝트를 Vite식 module graph로 번역하는 구조다.

핵심 책임:

- `vinext dev/build/start/deploy/init/check/lint/typegen` 명령을 제공한다.
- Vite 설정이 없어도 기본 plugin 구성을 만든다.
- `next/*` import를 local shim으로 보낸다.
- `pages/`, `app/`, middleware/proxy, metadata file, public route를 감지한다.
- dev server middleware와 production build hook을 연결한다.
- App Router용 RSC/SSR/browser virtual entry와 Pages Router용 client/server entry를 생성한다.

주요 파일:

| File | Role |
| --- | --- |
| `cli.ts` | 명령어 파싱, Vite dev/build/start 호출, deploy/init/check/lint/typegen 분기 |
| `index.ts` | alias, config, virtual module, dev middleware, build hook 중심 |
| `init.ts` | 기존 Next.js 프로젝트를 vinext 병행 실행 가능 상태로 변환 |
| `check.ts` | migration 전 compatibility scanner |
| `deploy.ts` | Cloudflare Workers 배포 자동화 |
| `config/next-config.ts` | `next.config.*` 로드와 resolved config 생성 |
| `plugins/` | font, CSS data URL, optimize imports, RSC client references 등 plugin 조각 |

깨지기 쉬운 지점:

- user `vite.config.*`가 있을 때 plugin 중복 주입으로 RSC transform이 두 번 돌 수 있다.
- `next/*` alias와 `resolveId` hook은 RSC, SSR, client 환경별로 다를 수 있다.
- `vinext()`는 `nitro()`/`cloudflare()`보다 먼저 있어야 virtual entries와 aliases를 platform plugin이 소비할 수 있다.
- dev server와 production server의 요청 처리 순서가 달라지면 Next.js parity가 깨진다.
- build hook은 실행 순서가 중요하다. manifest를 쓰는 hook과 읽는 hook 순서가 바뀌면 prerender/start/deploy가 같이 깨진다.

관련 테스트:

- `tests/cli-args.test.ts`
- `tests/init.test.ts`
- `tests/check.test.ts`
- `tests/deploy.test.ts`
- `tests/entry-templates.test.ts`
- `tests/next-config.test.ts`
- `tests/dotenv.test.ts`
- `tests/build-optimization.test.ts`

### 02 Next Shims

한 줄 결론: `shims/`는 vinext가 Next.js API surface를 Vite/Web API/React primitive 위에 다시 구현하는 호환성 계층이다.

핵심 책임:

- 사용자 앱의 `next/link`, `next/navigation`, `next/server`, `next/headers`, `next/cache` import를 보존한다.
- Next.js 내부 runtime 대신 local shim으로 resolve한다.
- App Router, Pages Router, route handler, middleware, RSC/SSR/client 환경별 차이를 흡수한다.

주요 파일:

| File | Role |
| --- | --- |
| `shims/link.tsx` | `<Link>`, prefetch, App/Pages Router navigation delegation |
| `shims/navigation.ts` | App Router hooks, router instance, redirects, notFound, client navigation |
| `shims/router.ts` | Pages Router `next/router` singleton, events, client navigation |
| `shims/server.ts` | `NextRequest`, `NextResponse`, `NextURL`, cookies, userAgent, after |
| `shims/headers.ts` | async `headers()`, `cookies()`, `draftMode()` |
| `shims/cache.ts` | `next/cache`, ISR/cache handler, tags, `"use cache"` integration |
| `shims/image.tsx` | `next/image` 호환 컴포넌트와 `/_vinext/image` URL |
| `shims/script.tsx` | `next/script` strategy handling |
| `shims/document.tsx` | Pages Router `_document` primitives |
| `shims/internal` | Next internal import 호환용 최소 구현 |

깨지기 쉬운 지점:

- shim은 공개 API처럼 보여야 하므로 error message, 반환 shape, sync/async 호환성이 중요하다.
- App Router와 Pages Router가 같은 shim을 공유해도 내부 상태 source는 다를 수 있다.
- client shim에서 `window` 접근은 SSR/RSC import 시점에 터지지 않아야 한다.
- `headers()`/`cookies()`는 request context뿐 아니라 dynamic rendering classification에도 영향을 준다.
- cache scope 안에서 request-specific API를 허용하면 persisted cache에 request 값이 얼어붙을 수 있다.

관련 테스트:

- `tests/shims.test.ts`
- `tests/link.test.ts`
- `tests/link-navigation.test.ts`
- `tests/router-javascript-urls.test.ts`
- `tests/image-component.test.ts`
- `tests/script.test.ts`
- `tests/head.test.ts`
- `tests/document.test.ts`
- `tests/cache-for-request.test.ts`
- `tests/nextjs-compat/navigation.test.ts`
- `tests/nextjs-compat/request-apis.test.ts`

### 03 File System Routing

한 줄 결론: routing 계층은 파일 시스템을 Next.js식 route pattern으로 변환하고, 요청 시 trie 기반 matcher로 route와 params를 찾는다.

핵심 책임:

- `pages/blog/[id].tsx` 같은 Pages route를 URL pattern으로 변환한다.
- `app/(marketing)/docs/[...slug]/page.tsx` 같은 App route를 route graph로 변환한다.
- static, dynamic, catch-all, optional catch-all 우선순위를 Next.js와 맞춘다.
- route groups, parallel slots, intercepting routes처럼 URL에는 보이지 않지만 layout ownership에 영향을 주는 metadata를 보존한다.

주요 파일:

| File | Role |
| --- | --- |
| `routing/pages-router.ts` | `pages/` route와 `pages/api/` route scan |
| `routing/app-router.ts` | App Router graph cache와 request-time match API |
| `routing/app-route-graph.ts` | route, layout, slot, interception metadata 구성 |
| `routing/route-trie.ts` | static/dynamic/catch-all 우선순위 기반 trie |
| `routing/route-matching.ts` | Pages/App 공통 matching preamble |
| `routing/route-pattern.ts` | route pattern fill/match/static params normalization |
| `routing/file-matcher.ts` | page extensions 기준 valid file detection |
| `routing/route-validation.ts` | route conflict/invalid pattern validation |

깨지기 쉬운 지점:

- static route가 dynamic route보다 먼저 이겨야 한다.
- `pages/api`는 page route와 같은 scan 계열이지만 runtime은 API handler로 간다.
- route group `(group)`과 parallel slot `@slot`은 URL에 보이지 않지만 layout ownership에 영향을 준다.
- intercepting routes는 target URL과 slot ownership을 동시에 추적해야 한다.
- optional catch-all은 빈 segment와 여러 segment를 모두 처리해야 한다.
- route graph cache invalidation이 dev HMR과 맞지 않으면 새 파일/삭제 파일이 반영되지 않는다.

관련 테스트:

- `tests/routing.test.ts`
- `tests/route-trie.test.ts`
- `tests/route-pattern.test.ts`
- `tests/route-sorting.test.ts`
- `tests/file-matcher.test.ts`
- `tests/app-route-graph.test.ts`
- `tests/app-rsc-route-matching.test.ts`
- `tests/page-extensions-routing.test.ts`
- `tests/intercepting-routes-build.test.ts`

### 04 Pages Router Runtime

한 줄 결론: Pages Router runtime은 `pages/` route table을 기반으로 SSR, API route, data route, `_app`, `_document`, ISR, client hydration을 Next.js 방식에 가깝게 재구현한다.

핵심 책임:

- `getServerSideProps`, `getStaticProps`, `getStaticPaths`를 처리한다.
- `_app`, `_document`, `_error`, `__NEXT_DATA__`를 구성한다.
- `/_next/data/<buildId>/*.json` data route를 page route와 같은 data fetching machinery로 처리한다.
- `pages/api/*`를 Node-style 또는 Edge-style API route로 처리한다.
- dev server와 production entry가 같은 의미의 request order를 유지해야 한다.

주요 파일:

| File | Role |
| --- | --- |
| `entries/pages-server-entry.ts` | production/server bundle용 Pages SSR/API/data route entry codegen |
| `entries/pages-client-entry.ts` | client hydration entry와 page loader manifest |
| `server/dev-server.ts` | dev server Pages SSR request handler |
| `server/pages-page-response.ts` | HTML stream, `_document`, `__NEXT_DATA__`, ISR cache write |
| `server/pages-page-data.ts` | `getStaticProps`/`getServerSideProps` 결과 처리 |
| `server/pages-data-route.ts` | `/_next/data` normalization/response |
| `server/pages-api-route.ts` | Web Request 기반 production API route 처리 |
| `server/api-handler.ts` | Node dev server API route handler |
| `shims/router.ts` | client/server `next/router` state와 events |

깨지기 쉬운 지점:

- dev server와 production Pages server가 같은 request order를 가져야 한다.
- `/_next/data` 요청은 middleware보다 먼저 page URL로 normalize되어야 하는 경우가 있다.
- `getStaticPaths`의 `fallback: true`, `"blocking"`, `false`는 SSR/data route/ISR 동작이 다르다.
- `_document`가 `__NEXT_DATA__`를 포함하지 않는 경우 fallback injection이 필요하다.
- Pages Router i18n은 URL locale prefix, cookie, Accept-Language와 맞물린다.
- API route는 Edge-style default handler와 Node-style req/res handler를 구분해야 한다.

관련 테스트:

- `tests/pages-router.test.ts`
- `tests/pages-page-response.test.ts`
- `tests/pages-page-data.test.ts`
- `tests/pages-data-route.test.ts`
- `tests/pages-api-route.test.ts`
- `tests/pages-i18n.test.ts`
- `tests/pages-router-concurrency.test.ts`
- `tests/e2e/pages-router/navigation.spec.ts`
- `tests/e2e/pages-router-prod/production.spec.ts`

### 05 App Router Route Tree

한 줄 결론: App Router의 핵심은 URL match 하나가 아니라, matched page를 둘러싼 layout/template/loading/error/not-found/slot/interception metadata를 route graph로 보존하는 것이다.

핵심 책임:

- nested layouts, templates, loading/error/not-found boundaries를 route tree로 보존한다.
- route groups, parallel routes, intercepting routes, metadata routes를 scan한다.
- segment config의 `dynamic`, `revalidate`, `dynamicParams`를 static/dynamic classification과 render policy에 연결한다.
- route handler `route.ts`를 page tree가 아닌 HTTP method dispatch로 보낸다.

주요 파일:

| File | Role |
| --- | --- |
| `routing/app-route-graph.ts` | App Router route graph 생성 중심 |
| `routing/app-router.ts` | graph cache와 request match API |
| `server/app-page-route-wiring.tsx` | graph metadata를 React element tree로 wiring |
| `server/app-elements.ts` | App element tree transport 구조 |
| `server/app-page-boundary.ts` | App Router boundary model |
| `server/app-page-boundary-render.ts` | boundary rendering |
| `server/metadata-routes.ts` | file-based metadata route scanner |
| `server/file-based-metadata.ts` | metadata file handling |
| `server/app-segment-config.ts` | segment config parsing/normalization |

깨지기 쉬운 지점:

- URL-transparent segment 제거 타이밍과 layout ownership 보존 타이밍을 섞으면 안 된다.
- slot의 `default.tsx`와 mirrored slot page 선택은 navigation persistence에 영향을 준다.
- intercepting route의 source/target 관계가 틀리면 client navigation에서 hard navigation으로 떨어질 수 있다.
- boundary wrapping 순서는 사용자 눈에 바로 보이는 차이를 만든다.
- metadata route는 일반 page route와 다른 response/build data path를 가진다.
- browser-facing route manifest는 navigation planner와 맞아야 한다.

관련 테스트:

- `tests/app-route-graph.test.ts`
- `tests/app-page-route-wiring.test.ts`
- `tests/app-elements.test.ts`
- `tests/app-page-boundary.test.ts`
- `tests/app-page-boundary-render.test.ts`
- `tests/app-segment-config.test.ts`
- `tests/metadata-routes.test.ts`
- `tests/file-based-metadata.test.ts`
- `tests/e2e/app-router/inherited-parallel-slots.spec.ts`
- `tests/e2e/app-router/metadata.spec.ts`

### 06 RSC SSR Browser Entries

한 줄 결론: App Router는 하나의 entry가 아니라 RSC entry, SSR entry, browser entry가 역할을 나누고 RSC stream을 매개로 이어진다.

핵심 책임:

- RSC entry가 server component tree를 실행하고 RSC payload를 만든다.
- SSR entry가 RSC payload를 HTML stream으로 바꾼다.
- browser entry가 hydration과 client navigation runtime을 bootstrap한다.
- server action, prerender endpoint, route manifest, client reference manifest를 entry codegen과 연결한다.

주요 파일:

| File | Role |
| --- | --- |
| `entries/app-rsc-entry.ts` | RSC entry generator, route match, RSC render, server actions, prerender endpoints |
| `entries/app-ssr-entry.ts` | SSR environment entry generator |
| `entries/app-browser-entry.ts` | browser hydration/navigation bootstrap generator |
| `entries/app-rsc-manifest.ts` | route/module manifest codegen helper |
| `server/app-rsc-handler.ts` | RSC request handling primitives |
| `server/app-page-render.ts` | App page render orchestration |
| `server/app-ssr-stream.ts` | RSC payload를 HTML stream으로 변환 |
| `server/app-browser-entry.ts` | browser runtime bootstrap and hydration logic |
| `server/app-rsc-response-finalizer.ts` | RSC response finalization |
| `server/app-server-action-execution.ts` | server action execution |

깨지기 쉬운 지점:

- generated code는 문자열 codegen이므로 테스트가 특히 중요하다.
- RSC/SSR/browser environment에서 import 가능한 module이 다르다.
- RSC payload에 필요한 client reference manifest가 dedupe/resolve되지 않으면 hydration이 깨진다.
- server action 후 redirect, cache revalidation, RSC rerender 순서가 틀리면 UI와 URL이 어긋난다.
- prerender endpoint는 build-time HTTP fetch 경로와 production server secret에 의존한다.
- browser entry가 route manifest를 잘못 받으면 navigation planner가 soft navigation 가능성을 오판한다.

관련 테스트:

- `tests/app-router.test.ts`
- `tests/app-browser-entry.test.ts`
- `tests/app-rsc-handler.test.ts`
- `tests/app-rsc-response-finalizer.test.ts`
- `tests/app-ssr-stream.test.ts`
- `tests/app-server-action-execution.test.ts`
- `tests/rsc-streaming.test.ts`
- `tests/client-reference-dedup.test.ts`
- `tests/e2e/app-router/hydration.spec.ts`
- `tests/e2e/app-router/server-actions.spec.ts`

### 07 Client Navigation History

한 줄 결론: client navigation은 Link click 하나가 아니라 prefetch cache, RSC fetch, visible commit, history state, scroll restoration이 함께 움직이는 runtime이다.

핵심 책임:

- `<Link>` prefetch/click/status를 App Router와 Pages Router로 위임한다.
- App Router navigation에서 `.rsc` fetch, soft/hard 판단, visible commit, URL/history/scroll update를 조율한다.
- Pages Router에서는 `next/router` singleton, events, data route loading, `beforePopState`를 유지한다.
- 외부 `pushState`/`replaceState`와 vinext metadata가 충돌하지 않게 history state를 patch한다.

주요 파일:

| File | Role |
| --- | --- |
| `shims/link.tsx` | `<Link>` click/prefetch/status, App/Pages delegation |
| `shims/navigation.ts` | App Router public hooks/router, client URL state, history patching |
| `shims/router.ts` | Pages Router singleton, router events, `beforePopState` |
| `client/navigation-runtime.ts` | browser global navigation runtime registry |
| `client/pages-router-link-navigation.ts` | Pages Router Link navigation bridge |
| `server/app-browser-navigation-controller.ts` | visible commit and lifecycle controller |
| `server/navigation-planner.ts` | soft/hard navigation, cache proof, persistence decision |
| `server/app-browser-popstate.ts` | App Router popstate coordination |
| `server/app-history-state.ts` | history state shape helpers |

깨지기 쉬운 지점:

- URL을 먼저 바꾸고 RSC commit이 실패하면 화면과 주소가 갈라진다.
- hash-only navigation은 RSC fetch를 건너뛰어야 한다.
- refresh는 prefetch/navigation cache를 무효화해야 stale payload를 재사용하지 않는다.
- intercepted route에서 cache proof가 없으면 보이는 UI를 잘못 재사용할 수 있다.
- App Router와 Pages Router가 모두 있을 때 `<Link>`의 target runtime 선택이 중요하다.
- history state에 vinext metadata를 보존하지 않으면 back/forward나 scroll restoration이 깨진다.

관련 테스트:

- `tests/navigation-runtime.test.ts`
- `tests/navigation-planner.test.ts`
- `tests/link-navigation.test.ts`
- `tests/prefetch-cache.test.ts`
- `tests/app-optimistic-routing.test.ts`
- `tests/app-browser-entry.test.ts`
- `tests/e2e/app-router/navigation.spec.ts`
- `tests/e2e/app-router/navigation-flows.spec.ts`
- `tests/e2e/app-router/nextjs-compat/hash-popstate-scroll.spec.ts`
- `tests/e2e/pages-router/router-events.spec.ts`

### 08 Request Context Server Shims

한 줄 결론: request context 계층은 `headers()`, `cookies()`, middleware override, cache scope, Cloudflare `waitUntil()`을 같은 request lifetime 안에서 안전하게 공유하게 만든다.

핵심 책임:

- Server Component, route handler, server action에서 request-scoped API를 인자 없이 쓸 수 있게 한다.
- middleware가 request headers/cookies를 바꾼 결과를 뒤쪽 render 단계에 반영한다.
- `"use cache"`나 `unstable_cache()` 안에서 request-specific API 사용을 guard한다.
- Workers `ctx.waitUntil()`을 deep call stack에서 사용할 수 있게 한다.

주요 파일:

| File | Role |
| --- | --- |
| `shims/unified-request-context.ts` | request headers/cache/execution context 등을 하나의 ALS store로 통합 |
| `shims/request-context.ts` | Cloudflare-like `ExecutionContext` accessor |
| `shims/headers.ts` | `headers()`, `cookies()`, `draftMode()`, dynamic usage tracking |
| `shims/cache-for-request.ts` | request-scoped factory cache |
| `shims/internal/work-unit-async-storage.ts` | Next internal work unit storage compatibility |
| `server/middleware.ts` | `proxy.ts`/`middleware.ts` discovery and execution |
| `server/middleware-request-headers.ts` | middleware request header override protocol decode/encode |
| `server/middleware-response-headers.ts` | middleware response headers merge |
| `server/app-post-middleware-context.ts` | middleware 이후 request context 재구성 |

깨지기 쉬운 지점:

- middleware가 `headers()`를 먼저 읽은 뒤 request header override를 반환하면 기존 sealed snapshot을 무효화해야 한다.
- `cookies().set()`/`draftMode().enable()`은 pending `Set-Cookie`를 최종 response에 모아야 한다.
- request context가 stream consumption보다 먼저 사라지면 RSC/SSR streaming 중 API 호출이 실패한다.
- cache scope 안에서 request API를 허용하면 static cache correctness가 깨진다.
- Cloudflare `waitUntil()`은 request context를 통해 background KV write 같은 작업에 전달되어야 한다.
- Node dev에서는 execution context가 없을 수 있으므로 `null` path를 안전하게 처리해야 한다.

관련 테스트:

- `tests/unified-request-context.test.ts`
- `tests/request-context.test.ts`
- `tests/app-request-context.test.ts`
- `tests/app-post-middleware-context.test.ts`
- `tests/middleware-runtime.test.ts`
- `tests/middleware-runtime-trailing-slash.test.ts`
- `tests/nextjs-compat/als-isolation.test.ts`
- `tests/nextjs-compat/request-apis.test.ts`
- `tests/nextjs-compat/set-cookies.test.ts`

### 09 Build Prerender Static Export

한 줄 결론: build 계층은 Vite output을 Next.js식 배포 산출물로 바꾸고, static/dynamic route 판단, prerender, static export, standalone output을 조율한다.

핵심 책임:

- Vite multi-environment build 결과를 Next.js식 runtime 산출물로 재구성한다.
- static, ISR, dynamic, API, internal route를 classification한다.
- `output: 'export'`, prerender, ISR cache seed, build report, standalone output을 처리한다.
- Nitro 사용 시 ISR route를 Nitro `routeRules`로 변환한다.

주요 파일:

| File | Role |
| --- | --- |
| `build/prerender.ts` | Pages/App prerender 실제 구현 |
| `build/run-prerender.ts` | build/deploy 공통 prerender orchestration |
| `build/static-export.ts` | `output: 'export'` wrapper |
| `build/report.ts` | build report route classification |
| `build/layout-classification.ts` | App layout static/dynamic classification |
| `build/route-classification-manifest.ts` | route classification manifest codegen |
| `build/route-classification-injector.ts` | generated RSC chunk에 build-time classification 주입 |
| `build/standalone.ts` | `output: 'standalone'` 산출물 생성 |
| `build/nitro-route-rules.ts` | Nitro route rules 생성 |
| `build/precompress.ts` | build assets precompression |

깨지기 쉬운 지점:

- `output: 'export'`와 default prerender는 허용 정책이 다르다.
- catch-all/optional catch-all params를 URL로 채울 때 output path가 틀어지기 쉽다.
- App Router `generateStaticParams`는 parent dynamic params와 top-down으로 연결된다.
- prerender manifest 위치는 runtime start/deploy/cache seed와 맞아야 한다.
- route classification은 build report 표시가 아니라 RSC entry runtime decision에도 연결된다.
- Nitro routeRules는 build 전 filesystem/static analysis 기반이라 speculative prerender 결과와 차이가 날 수 있다.
- 사용자가 명시한 Nitro cache rule을 자동 생성 rule이 덮어쓰면 안 된다.

관련 테스트:

- `tests/prerender.test.ts`
- `tests/run-prerender-concurrency.test.ts`
- `tests/static-export.test.ts`
- `tests/app-static-generation.test.ts`
- `tests/build-report.test.ts`
- `tests/layout-classification.test.ts`
- `tests/route-classification-manifest.test.ts`
- `tests/route-classification-injector.test.ts`
- `tests/nitro-route-rules.test.ts`
- `tests/standalone-build.test.ts`
- `tests/e2e/static-export/app-router.spec.ts`
- `tests/e2e/standalone-output/basic.spec.ts`

### 10 Cloudflare Runtime E2E

한 줄 결론: Cloudflare 계층은 vinext의 primary deployment target으로, Workers entry, wrangler config, KV-backed ISR, image optimization, TPR, E2E 검증을 묶는다.

핵심 책임:

- `vinext deploy`로 wrangler config, worker entry, Vite config를 생성/검증한다.
- Cloudflare Workers request에서 image optimization, assets, middleware/config pipeline, App/Pages runtime을 연결한다.
- KV-backed ISR과 tag invalidation을 처리한다.
- `ctx.waitUntil()`을 cache write/delete 같은 background work에 연결한다.
- Cloudflare analytics 기반 TPR로 traffic 있는 path를 KV에 미리 채운다.

주요 파일:

| File | Role |
| --- | --- |
| `deploy.ts` | `vinext deploy` pipeline, wrangler/worker/vite config 생성 |
| `cloudflare/index.ts` | public Cloudflare exports |
| `cloudflare/kv-cache-handler.ts` | Cloudflare KV-backed CacheHandler |
| `cloudflare/tpr.ts` | Traffic-aware Pre-Rendering pipeline |
| `server/worker-utils.ts` | Worker entry shared utilities |
| `server/image-optimization.ts` | `/_vinext/image` handling and Images binding integration |
| `server/request-pipeline.ts` | shared request normalization/security/header helpers |
| `examples/app-router-cloudflare` | minimal App Router Workers example |
| `tests/e2e/cloudflare-workers` | App Router Workers E2E |
| `tests/e2e/cloudflare-pages-router` | Pages Router Workers E2E |

깨지기 쉬운 지점:

- Workers build에 Cloudflare plugin이 빠지면 RSC/workerd 환경과 bindings가 맞지 않는다.
- Nitro target에서는 Cloudflare-only binding, KV handler, Images binding에 기대면 안 된다.
- Pages Router ISR KV detection에는 제한이 있어 수동 설정 gap을 고려해야 한다.
- generated worker template과 shared utility의 header merge/static asset logic이 어긋나기 쉽다.
- KV key format을 바꾸면 runtime cache와 TPR bulk upload가 동시에 깨진다.
- `ctx.waitUntil()`이 ALS를 통해 전달되지 않으면 background KV 작업이 response 이후 중단될 수 있다.
- image optimization fallback은 binding이 없거나 unsupported content type일 때 원본 passthrough가 가능해야 한다.

관련 테스트:

- `tests/deploy.test.ts`
- `tests/kv-cache-handler.test.ts`
- `tests/tpr-kv-keys.test.ts`
- `tests/image-optimization-parity.test.ts`
- `tests/request-pipeline.test.ts`
- `tests/after-deploy.test.ts`
- `tests/e2e/cloudflare-workers/ssr.spec.ts`
- `tests/e2e/cloudflare-workers/navigation.spec.ts`
- `tests/e2e/cloudflare-pages-router/ssr.spec.ts`
- `tests/e2e/cloudflare-pages-router/middleware.spec.ts`

### 11 vinext vs Next

한 줄 결론: vinext는 Next.js를 fork하지 않고, Next.js 공개 API surface를 Vite 위에서 다시 실행시키는 대체 구현이다.

핵심 책임:

- "사용자 앱 코드는 유지한다"와 "프레임워크 내부 구현은 Vite-native로 다시 만든다"를 구분한다.
- Next.js behavior를 `.nextjs-ref/test`와 `.nextjs-ref/packages/next/src`로 검증한다.
- source copy가 아니라 behavior mapping으로 compatibility를 쌓는다.
- Next와 같아야 하는 공개 API와 달라도 되는 내부 구현을 분리한다.

실무 기준:

| Question | Answer |
| --- | --- |
| 사용자 앱에서 관찰 가능한 Next.js 공개 동작인가? | 최대한 Next.js와 같아야 한다. |
| Next.js 내부 구현 세부사항인가? | vinext 구조에 맞게 달라도 된다. |
| Vite/Nitro/Cloudflare runtime에 필요한 변환인가? | Next.js와 다르게 코딩하는 것이 맞다. |

주요 비교:

| Area | Next.js | vinext |
| --- | --- | --- |
| Build toolchain | Next compiler pipeline, webpack/Turbopack/SWC, internal manifests | Vite plugin hooks, virtual modules, Vite output |
| Module resolution | `next/*` package implementation | local `shims/*` |
| Routing | Next internal manifests and route modules | `routing/` scan result and route graph |
| App Router | Next app render runtime and loader tree | RSC/SSR/browser virtual entries |
| Pages Router | Next pages runtime and build output | Pages server/client entries and Web API handlers |
| Request context | Next async storage/work store | vinext ALS context and Web API snapshots |
| Deployment | Vercel/self-hosting/static export | Cloudflare native, Nitro, Node testing |

위험 지점:

- undocumented Next.js behavior에 실제 앱이 기대고 있을 수 있다.
- dev/prod/Workers/Nitro 경로가 달라서 한 경로만 고치면 divergence가 생길 수 있다.
- RSC, SSR, browser environment 분리를 잘못하면 hydration mismatch가 난다.
- Next.js release drift를 tests와 behavior mapping으로 계속 추적해야 한다.
- platform-specific behavior는 Node, Workers, Nitro에서 다르게 나타날 수 있다.

### Appendix Vite Nitro vinext

한 줄 결론: `vinext()`와 `nitro()`는 경쟁 plugin이 아니다. `vinext()`가 Next.js semantics를 만들고, `nitro()`가 generic deployment output을 만든다. Cloudflare Workers는 native path가 우선이다.

자주 헷갈리는 답:

| Question | Answer |
| --- | --- |
| Nitro가 Next.js API를 구현하는가? | 아니다. Next.js API surface는 vinext가 구현한다. Nitro는 deployment/runtime adapter다. |
| `vinext()`와 `nitro()` 중 하나만 쓰면 되는가? | Next.js compatibility만 확인하면 `vinext()`가 핵심이고, Nitro-supported platform output이 필요하면 `nitro()`를 함께 둔다. |
| Cloudflare 배포에도 Nitro를 쓰는가? | 쓸 수는 있지만 권장 path는 native integration이다. |
| Nitro routeRules는 모든 static route를 포함하는가? | 아니다. ISR route 중심이고, `nitro.setup` 시점에는 speculative prerender 결과가 없다. |
| 사용자가 Nitro routeRules를 직접 쓰면 어떻게 되는가? | vinext는 user-defined cache rule을 자동 rule로 덮어쓰지 않는다. |

관련 테스트:

- `tests/nitro-route-rules.test.ts`
- `tests/deploy.test.ts`
- `tests/after-deploy.test.ts`
- `tests/kv-cache-handler.test.ts`
- `tests/tpr-kv-keys.test.ts`
- `tests/e2e/cloudflare-workers/ssr.spec.ts`
- `tests/e2e/cloudflare-pages-router/ssr.spec.ts`

## Modification Compass

| Symptom | First Place to Inspect |
| --- | --- |
| `next/link`, `next/navigation`, `next/server` behavior mismatch | `02 Next Shims`, then matching file in `shims/` |
| route priority or dynamic param mismatch | `03 File System Routing`, then `routing/*` and route tests |
| App Router layout, slot, intercepting route issue | `05 App Router Route Tree`, then `app-route-graph.ts`, `app-page-route-wiring.tsx` |
| RSC payload, SSR HTML, hydration mismatch | `06 RSC SSR Browser Entries`, then generated app entries and `server/app-*` |
| Pages Router SSR/API/data route issue | `04 Pages Router Runtime`, then `pages-server-entry.ts`, `dev-server.ts`, `prod-server.ts` |
| navigation, prefetch, scroll, history issue | `07 Client Navigation History`, then `shims/navigation.ts`, `shims/link.tsx`, browser navigation files |
| `headers()`, `cookies()`, middleware override issue | `08 Request Context Server Shims`, then `shims/headers.ts`, `unified-request-context.ts` |
| static export, prerender, build report issue | `09 Build Prerender Static Export`, then `build/prerender.ts`, `run-prerender.ts`, `static-export.ts` |
| Nitro deployment or ISR routeRules issue | `09` and Appendix, then `build/nitro-route-rules.ts`, `examples/app-router-nitro` |
| Workers deploy, KV ISR, image optimization issue | `10 Cloudflare Runtime E2E`, then `deploy.ts`, `cloudflare/*`, worker E2E |
| Next.js와 같아야 할지 vinext답게 달라야 할지 애매한 issue | `11 vinext vs Next`, then `AGENTS.md`, `.nextjs-ref/test`, `.nextjs-ref/packages/next/src` |

## Verification Habit

관련 테스트를 먼저 고른다. 전체 suite보다 변경 계층에 맞는 targeted test가 우선이다.

```bash
vp test run tests/app-router.test.ts
vp test run tests/pages-router.test.ts
vp test run tests/navigation-planner.test.ts
vp test run tests/prerender.test.ts
vp test run tests/deploy.test.ts
```

넓은 변경이면 formatting, lint, type check까지 확인한다.

```bash
pnpm run check
```

Next.js behavior 확인이 필요한 경우:

```bash
rg -rn "middleware" .nextjs-ref/test/ --include "*.test.*"
rg -rn "x-middleware-override-headers" .nextjs-ref/packages/next/src/
```

## Layer Checklists

### CLI and plugin

- CLI에서 Vite를 프로젝트 기준으로 resolve하는가?
- user Vite config가 있을 때 자동 plugin 주입과 충돌하지 않는가?
- Nitro/Cloudflare plugin이 있을 때 bundled runtime용 external/chunk 설정이 적용되는가?
- `next/*` alias가 RSC/SSR/client 환경 차이를 보존하는가?
- dev와 production 요청 순서가 같은 의미를 유지하는가?
- start/deploy/prerender가 필요한 manifest를 같은 위치에서 읽는가?

### Shims

- 해당 API가 App Router, Pages Router, route handler, middleware 중 어디에서 호출되는가?
- RSC/SSR/client 중 어느 environment에서 import되는가?
- sync처럼 보이는 Next API가 Promise/decorated object 형태를 요구하는가?
- middleware request header override 이후에도 stale snapshot을 보고 있지 않은가?
- test fixture가 실제 사용자 import 경로인 `next/*`를 통해 검증하는가?

### Routing

- 파일 경로가 URL-visible segment와 invisible segment로 올바르게 나뉘는가?
- route pattern이 Pages/App 양쪽에서 같은 의미로 해석되는가?
- 동적 route params 이름과 값 shape가 Next.js와 맞는가?
- HMR이나 build 재실행에서 cache가 stale하지 않은가?
- App Router route graph의 layout/slot/boundary metadata가 request-time render까지 보존되는가?

### Pages runtime

- page render와 data route render가 같은 data fetching 결과 shape를 쓰는가?
- dev와 production에서 API route runtime 차이가 노출되지 않는가?
- `_app`, `_document`, page-specific assets가 SSR manifest에서 누락되지 않는가?
- fallback route에서 `useRouter().isFallback`과 data request 동작이 맞는가?
- middleware/config headers가 최종 HTML/API/data response에 병합되는가?

### App route tree

- route graph가 request matching, RSC rendering, browser navigation에서 같은 route identity를 공유하는가?
- slot/default/interception metadata가 layout boundary와 같은 index 체계를 쓰는가?
- metadata file routes가 static export/prerender와 연결되는가?
- segment config가 build report, prerender, runtime render policy에 모두 반영되는가?
- boundary wrapping 순서가 Next.js와 맞는가?

### RSC/SSR/browser entries

- 변경한 module이 RSC, SSR, browser 중 어느 환경에서 실행되는가?
- generated entry가 route graph와 manifest를 같은 version/shape로 소비하는가?
- initial render와 client navigation render가 같은 boundary/cache semantics를 공유하는가?
- server action 이후 response가 action result와 rerender payload를 모두 포함하는가?
- prerender용 internal endpoint가 runtime public endpoint처럼 노출되지 않는가?

### Navigation

- 이 navigation은 App Router인가 Pages Router인가?
- URL update 권한은 click handler가 아니라 commit path에 있는가?
- prefetch cache와 refresh invalidation이 같은 cache key 체계를 쓰는가?
- popstate에서 RSC render가 끝나기 전에 scroll을 복원하지 않는가?
- hard navigation fallback이 무한 루프를 만들지 않는가?

### Request context

- 이 값은 request마다 달라지는가, render마다 달라지는가, process 전역인가?
- middleware 전/후 어느 시점의 headers/cookies를 읽어야 하는가?
- snapshot cache를 invalidate해야 하는 mutation이 있는가?
- cache scope에서 호출될 수 있는 API인가?
- Workers와 Node dev 양쪽에서 execution context 부재를 처리하는가?

### Build/prerender

- 이 route는 static, ISR, dynamic, API, internal 중 무엇인가?
- export mode에서 이 route가 허용되는가?
- prerender output path와 runtime lookup path가 같은 규칙을 쓰는가?
- App+Pages hybrid build에서 manifest가 덮어써지지 않고 merge되는가?
- standalone output에 외부화된 runtime dependency가 빠지지 않는가?
- Nitro routeRules 변환이 dynamic/catch-all/optional catch-all pattern을 올바르게 보존하는가?

### Cloudflare/Nitro

- 이 동작은 Node `vinext start`와 Workers 모두에서 같아야 하는가, Workers 전용인가?
- Nitro target에서도 필요한 기능인가, Cloudflare native 기능인가?
- generated worker와 shared utility가 같은 header/static asset policy를 쓰는가?
- KV entry shape와 TPR upload shape가 같은가?
- `waitUntil()`이 request context를 통해 deep call stack까지 전달되는가?
- Wrangler config, Vite config, worker entry가 서로 같은 router mode를 가리키는가?

## Source Map

| Area | Primary Files |
| --- | --- |
| CLI | `../vinext/packages/vinext/src/cli.ts`, `../vinext/packages/vinext/src/init.ts`, `../vinext/packages/vinext/src/deploy.ts` |
| Vite plugin | `../vinext/packages/vinext/src/index.ts`, `../vinext/packages/vinext/src/plugins` |
| Routing | `../vinext/packages/vinext/src/routing` |
| Entries | `../vinext/packages/vinext/src/entries` |
| Server runtime | `../vinext/packages/vinext/src/server` |
| Shims | `../vinext/packages/vinext/src/shims` |
| Build | `../vinext/packages/vinext/src/build` |
| Nitro integration | `../vinext/packages/vinext/src/build/nitro-route-rules.ts`, `../vinext/examples/app-router-nitro` |
| Cloudflare | `../vinext/packages/vinext/src/cloudflare`, `../vinext/packages/vinext/src/deploy.ts` |
| Next comparison | `../vinext/README.md`, `../vinext/AGENTS.md`, `../vinext/.nextjs-ref/packages/next/src`, `../vinext/.nextjs-ref/test` |
| Tests | `../vinext/tests`, `../vinext/tests/e2e` |

## Final Mental Model

```text
사용자 앱 코드는 Next.js처럼 유지한다.
프레임워크 내부 구현은 Vite-native로 다시 만든다.

Next.js public behavior:
  match as closely as possible

Next.js private internals:
  study, test, and map behavior
  but do not blindly copy structure

Platform output:
  Cloudflare native when Workers features matter
  Nitro when generic multi-platform deployment matters
```

vinext를 이해하는 가장 짧은 문장은 이것이다.

> vinext는 Next.js를 복사하는 프로젝트가 아니라, Next.js 앱이 기대하는 공개 동작을 Vite, Web API, Cloudflare, Nitro라는 다른 기반 위에서 다시 성립시키는 프로젝트다.
