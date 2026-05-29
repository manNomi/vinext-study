# Appendix: Vite, Nitro, vinext Plugin Stack

## 한 줄 결론

`vinext()`는 Next.js compatibility를 Vite plugin으로 구현하는 층이고, `nitro()`는 그 결과물을 여러 server platform에 올리기 위한 deployment/runtime adapter 층이다.

Cloudflare Workers로 갈 때는 `nitro()` 대신 `@cloudflare/vite-plugin`과 `vinext deploy`가 native path를 만든다. 이 세 plugin은 같은 문제를 푸는 경쟁자가 아니라 서로 다른 층의 책임을 갖는다.

```text
Next.js app source
  -> vinext(): Next.js semantics on Vite
  -> nitro(): generic multi-platform server output

Next.js app source
  -> vinext(): Next.js semantics on Vite
  -> cloudflare(): Workers-native output and bindings
```

## 왜 이 구분이 중요한가

Next.js를 Vite에서 돌린다는 말은 두 가지 문제를 동시에 포함한다.

첫 번째는 "Next.js 앱처럼 생긴 소스 코드를 어떻게 이해할 것인가"다. `next/link`, `next/navigation`, `next/server`, `pages/`, `app/`, `_app`, `_document`, route handlers, Server Components, metadata, ISR, middleware 같은 Next.js semantics를 해석해야 한다.

두 번째는 "이렇게 해석한 앱을 어디에 배포할 것인가"다. Node server로 띄울 수도 있고, Cloudflare Workers에 올릴 수도 있고, Vercel/Netlify/AWS/Deno 같은 server platform으로 보낼 수도 있다.

vinext와 Nitro의 위치는 여기서 갈린다.

| Layer | Question | Owner |
| --- | --- | --- |
| Next.js compatibility | 이 소스가 Next.js 앱으로서 무엇을 의미하는가? | `vinext()` |
| React/RSC transform | `"use client"`와 `"use server"` boundary를 어떻게 나눌 것인가? | `@vitejs/plugin-rsc` |
| Generic deployment output | Vercel/Netlify/AWS/Deno 같은 platform output을 어떻게 만들 것인가? | `nitro()` |
| Cloudflare-native deployment output | Workers, bindings, ASSETS, Images, KV를 어떻게 직접 연결할 것인가? | `cloudflare()` + `vinext deploy` |

그래서 `plugins: [vinext(), nitro()]`는 "Next.js 구현 plugin 두 개"가 아니라 "Next.js compatibility plugin 하나 + platform output plugin 하나"로 읽어야 한다.

## 최소 Vite config

Cloudflare가 아닌 multi-platform target에서는 README가 제시하는 최소 구성이 이렇게 생긴다.

```ts
import { defineConfig } from "vite";
import vinext from "vinext";
import { nitro } from "nitro/vite";

export default defineConfig({
  plugins: [vinext(), nitro()],
});
```

이 구성에서 각 줄의 의미는 다음과 같다.

- `vinext()`는 프로젝트 루트에서 `next.config.*`, `.env*`, `pages/`, `app/`, `public/`, middleware/proxy 파일, metadata file routes를 찾는다.
- `vinext()`는 `next/*` import를 `packages/vinext/src/shims/*`로 resolve하게 만든다.
- `vinext()`는 App Router가 있으면 RSC/SSR/browser virtual entry를 만들고, Pages Router가 있으면 pages server/client entry를 만든다.
- `nitro()`는 Vite build 결과를 Nitro가 지원하는 platform output으로 패키징한다.
- Nitro preset은 CI 환경에서 자동 감지되거나, 로컬에서는 `NITRO_PRESET`으로 지정할 수 있다.

로컬에서 target을 명시하고 싶으면 다음처럼 빌드한다.

```bash
NITRO_PRESET=vercel npx vite build
NITRO_PRESET=netlify npx vite build
NITRO_PRESET=deno_deploy npx vite build
```

## `vinext()`가 실제로 하는 일

`vinext()`의 중심 파일은 [`../../vinext/packages/vinext/src/index.ts`](../../vinext/packages/vinext/src/index.ts)다. 이 파일은 하나의 작은 plugin이라기보다 여러 Vite plugin 조각을 반환하는 orchestration layer에 가깝다.

큰 책임은 여섯 덩어리로 볼 수 있다.

| Responsibility | What happens |
| --- | --- |
| Config loading | `next.config.*`, inline `vinext({ nextConfig })`, dotenv, page extensions, basePath, assetPrefix, i18n 같은 설정을 읽는다. |
| Alias and resolve | `next/*` import와 일부 internal import를 vinext shim/runtime 파일로 보낸다. |
| Route discovery | `pages/`와 `app/`를 스캔해서 Pages route table, App route graph, metadata route를 만든다. |
| Virtual entries | Vite가 로드할 RSC/SSR/browser/pages entry source를 문자열로 생성한다. |
| Runtime wiring | dev server middleware, request pipeline, middleware/proxy execution, config redirects/rewrites/headers 연결을 만든다. |
| Build output | build id, server manifest, prerender manifest, Nitro routeRules, precompress, Cloudflare build glue 같은 산출물을 만든다. |

Vite hook 관점으로 보면 이렇게 이어진다.

```text
config()
  -> Next config/env/page extensions resolve
  -> plugin stack에 필요한 aliases/env/build options 주입
  -> App Router면 RSC 관련 environment 설정
  -> Nitro/Cloudflare plugin 존재 여부 감지

resolveId()
  -> next/* shim path로 resolve
  -> virtual:vinext-* module id resolve
  -> environment별로 다른 shim/entry를 고를 수 있게 함

load()
  -> virtual:vinext-rsc-entry
  -> virtual:vinext-app-ssr-entry
  -> virtual:vinext-app-browser-entry
  -> virtual:vinext-server-entry / pages entry

transform()
  -> JSX-in-JS, font, use cache, server export strip, instrumentation 등 보조 transform

configureServer()
  -> dev server request middleware
  -> route cache invalidation
  -> middleware/proxy execution

generateBundle/writeBundle/closeBundle()
  -> manifest emission
  -> build id
  -> route classification patch
  -> Nitro routeRules setup
  -> Cloudflare build glue
```

핵심 mental model은 이것이다.

```text
Vite는 "module graph와 plugin hook 실행기"다.
vinext는 "Next.js 앱을 Vite module graph로 번역하는 plugin"이다.
Nitro/Cloudflare는 "그 module graph를 platform output으로 만드는 plugin"이다.
```

## `nitro()`가 맡는 일

Nitro는 Next.js compatibility를 직접 만드는 계층이 아니다. Nitro는 server output과 deployment preset을 담당한다.

예를 들어 `vinext()` 없이 `nitro()`만 붙이면 Next.js의 `next/link`, `app/`, `pages/`, `generateMetadata`, `getStaticProps` 같은 semantics를 이해할 수 없다. 반대로 `vinext()`만 붙이면 Next.js semantics는 Vite에서 구성되지만, Vercel/Netlify/AWS/Deno 같은 Nitro target으로 패키징하는 기능은 없다.

그래서 Nitro path의 역할 분담은 다음처럼 읽는다.

```text
source semantics:
  Next.js app source -> vinext()

runtime/build packaging:
  Vite server graph -> nitro()

platform output:
  Nitro preset -> Vercel/Netlify/AWS/Deno/etc.
```

Nitro path에서 vinext가 특별히 해주는 일 중 하나는 ISR route 정보를 Nitro routeRules로 넘겨주는 것이다. 이 부분이 [`../../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../../vinext/packages/vinext/src/build/nitro-route-rules.ts)에 들어 있다.

## plugin 순서가 중요한 이유

권장 순서는 다음과 같다.

```ts
plugins: [vinext(), nitro()]
```

이 순서가 자연스러운 이유는 `nitro()`가 platform output을 만들기 전에 `vinext()`가 Next.js semantics를 Vite graph에 심어야 하기 때문이다.

`vinext()`가 먼저 해야 하는 일은 다음과 같다.

- `next/*` import를 shim으로 resolve 가능하게 만든다.
- virtual entry module id를 등록한다.
- App Router의 RSC/SSR/browser entry를 만들어 Vite가 빌드할 수 있게 한다.
- Pages Router의 server/client entry를 만들어 Vite가 빌드할 수 있게 한다.
- build config에서 bundled runtime에 맞는 external/chunk policy를 조정한다.
- Nitro plugin이 있으면 Nitro routeRules hook을 준비한다.

그 다음 `nitro()`가 Vite/Nitro build 흐름에서 server output을 만든다. `vinext()`가 없는 상태에서 Nitro가 먼저 platform output을 만들려고 하면, Next.js 앱을 어떤 entry로 실행해야 하는지, `next/*` import를 어디로 보낼지, route table이 무엇인지 알 수 없다.

## `vinext()`가 Nitro plugin을 감지하는 이유

`vinext()`는 plugin list에서 `nitro` 또는 `nitro:*` 이름을 가진 plugin을 감지한다. 이 감지는 단순 장식이 아니라 build 설정에 영향을 준다.

Nitro나 Cloudflare 같은 bundled runtime에서는 일반 Node server output과 다른 선택이 필요하다.

- server dependency externalization 정책이 달라진다.
- client/server chunking이 platform bundler와 충돌하지 않도록 조정된다.
- runtime build가 platform plugin에 의해 다시 패키징될 수 있다는 전제를 둔다.
- Nitro가 있으면 `vinext:nitro-route-rules` plugin이 Nitro setup convention을 통해 routeRules를 주입한다.

즉, `hasNitroPlugin`은 "Nitro 전용 기능을 켠다"라기보다 "이 build는 plain Node server target이 아니라 bundled/platform runtime target이다"라는 중요한 신호다.

## Nitro routeRules가 만들어지는 방식

Nitro routeRules는 Nitro가 route별 caching/ISR/SWR 같은 정책을 platform output에 반영할 때 쓰는 설정이다. vinext는 Next.js route에서 ISR 의미를 읽고, 이를 Nitro가 이해하는 `routeRules`로 바꾼다.

흐름은 다음과 같다.

```text
nitro.setup hook
  -> collectNitroRouteRules()
  -> appRouter()/pagesRouter()/apiRouter()로 route scan
  -> buildReportRows()로 route type 계산
  -> ISR route만 선택
  -> generateNitroRouteRules()
  -> convertToNitroPattern()
  -> mergeNitroRouteRules()
  -> nitro.options.routeRules 갱신
```

중요한 점은 `nitro.setup`이 build 초기에 실행된다는 것이다. 이 시점에는 speculative prerender 결과가 아직 없다. 그래서 Nitro routeRules는 prerender 결과가 아니라 filesystem scan과 static analysis를 기반으로 생성된다.

이 제약 때문에 다음 차이가 생긴다.

| Case | Build report with prerender | Nitro routeRules |
| --- | --- | --- |
| 명시적 ISR route | ISR로 분류 가능 | `swr` routeRule 생성 가능 |
| 명시적 dynamic route | dynamic으로 분류 | routeRule 생성 안 함 |
| speculative prerender로 static 확인된 route | prerender 후 static으로 승격 가능 | setup 시점에는 알 수 없어 routeRule 생성 안 함 |
| user routeRules가 이미 있는 route | build report와 무관 | 자동 rule이 덮어쓰지 않음 |

예시 변환은 이렇다.

```text
vinext internal pattern: /blog/:slug
Nitro rou3 pattern:      /blog/*

vinext internal pattern: /docs/:slug+
Nitro rou3 pattern:      /docs/**

vinext internal pattern: /docs/:slug*
Nitro rou3 pattern:      /docs/**
```

`generateNitroRouteRules()`는 ISR route 중 `revalidate`가 양의 finite number인 경우만 `{ swr: revalidate }` 형태로 만든다.

```text
Next-style route:
  /blog/:slug
  revalidate = 60

Nitro routeRules:
  /blog/* -> { swr: 60 }
```

`mergeNitroRouteRules()`는 사용자가 이미 다음 종류의 cache 관련 rule을 지정한 경우 자동 생성 rule을 건너뛴다.

- `swr`
- `cache`
- `static`
- `isr`
- `prerender`

이 설계는 사용자의 platform-specific tuning을 존중하기 위한 것이다. 자동 변환은 기본값을 채워주는 역할이지, 명시 설정을 덮는 역할이 아니다.

## Cloudflare native와 Nitro path 선택 기준

Cloudflare Workers로 배포할 때도 Nitro를 쓸 수는 있지만, vinext의 primary path는 native Cloudflare integration이다.

```text
Cloudflare native:
  vinext deploy
  -> @cloudflare/vite-plugin
  -> worker/index.ts generation
  -> wrangler.jsonc generation
  -> ASSETS binding
  -> Cloudflare Images
  -> KV-backed ISR
  -> TPR
```

```text
Nitro path:
  vite build
  -> plugins: [vinext(), nitro()]
  -> Nitro preset
  -> Vercel/Netlify/AWS/Deno/etc. output
```

선택 기준은 다음과 같다.

| Need | Better path |
| --- | --- |
| Cloudflare Workers first-class deploy | `vinext deploy` + `@cloudflare/vite-plugin` |
| `cloudflare:workers` binding access | Cloudflare native |
| KV-backed ISR with `KVCacheHandler` | Cloudflare native |
| Cloudflare Images-based local image optimization | Cloudflare native |
| TPR, traffic-aware KV warmup | Cloudflare native |
| Vercel/Netlify/AWS/Deno 등 다른 target | Nitro |
| 여러 hosting provider를 같은 app에서 실험 | Nitro |
| provider-specific Nitro preset 사용 | Nitro |

Cloudflare native path는 Cloudflare 기능을 깊게 활용한다. Nitro path는 여러 platform에 나갈 수 있는 범용성을 준다. 그래서 "Cloudflare에 배포할 수 있느냐"만 보면 둘 다 가능할 수 있지만, "Cloudflare 기능을 vinext가 의도한 방식으로 가장 잘 쓰느냐"를 보면 native path가 맞다.

## App Router에서 RSC plugin과의 관계

App Router는 React Server Components를 쓰므로 `@vitejs/plugin-rsc`가 중요하다.

vinext 문서의 Cloudflare custom Vite config 예시는 다음 구조를 보여준다.

```ts
plugins: [
  vinext(),
  rsc({
    entries: {
      rsc: "virtual:vinext-rsc-entry",
      ssr: "virtual:vinext-app-ssr-entry",
      client: "virtual:vinext-app-browser-entry",
    },
  }),
  cloudflare({
    viteEnvironment: { name: "rsc", childEnvironments: ["ssr"] },
  }),
]
```

여기서 중요한 것은 RSC plugin의 entry가 vinext virtual module을 가리킨다는 점이다. `vinext()`가 virtual entry를 만들고, RSC plugin은 그 entry들을 기준으로 RSC/SSR/client graph를 나눈다.

자동 구성 경로에서는 `vinext()`가 App Router 감지 후 필요한 RSC 설정을 구성한다. 사용자가 직접 Vite config를 만지는 경우에는 RSC plugin과 platform plugin 설정이 중복되거나 빠지지 않게 봐야 한다.

## 빌드 시나리오별 mental model

### 1. Vite config 없는 기본 dev

```text
vinext dev
  -> CLI가 Vite config를 자동 생성
  -> plugins: [vinext()]
  -> vinext dev middleware가 요청 처리
```

이 경로는 local development에 가장 가깝다. platform deployment output보다 Next.js compatibility와 HMR이 중요하다.

### 2. Vite config 없는 기본 build/start

```text
vinext build
  -> CLI가 Vite build 실행
  -> vinext build hooks
  -> dist/client, dist/server

vinext start
  -> Node production server
  -> dist/server output 실행
```

이 경로는 Node production testing에 가깝다. README에서도 Cloudflare Workers가 primary target이고 Node production server는 testing 성격이 강하다고 설명한다.

### 3. Nitro build

```text
vite build
  -> user vite.config.ts
  -> plugins: [vinext(), nitro()]
  -> vinext가 Next.js runtime graph 구성
  -> Nitro가 platform output 구성
  -> vinext가 Nitro routeRules 보조 주입
```

이 경로는 Cloudflare native가 아닌 target을 향한다.

### 4. Cloudflare native deploy

```text
vinext deploy
  -> project detection
  -> wrangler.jsonc / worker/index.ts / vite.config.ts 생성 또는 검증
  -> @cloudflare/vite-plugin 기반 build
  -> optional TPR
  -> wrangler deploy
```

이 경로는 Workers-specific behavior와 bindings가 중요하다. `deploy.ts`가 worker template, wrangler config, missing dependencies, native module stubs, Cloudflare plugin presence를 관리한다.

## 자주 헷갈리는 지점

### Nitro가 Next.js API를 구현하는가?

아니다. 이 노트 기준으로 Next.js API surface는 vinext가 구현한다. Nitro는 deployment/runtime adapter다.

### `vinext()`와 `nitro()` 중 하나만 쓰면 되는가?

목표에 따라 다르다. Next.js compatibility만 로컬에서 확인한다면 `vinext()`가 핵심이다. Nitro-supported platform output이 필요하면 `nitro()`를 함께 둔다.

### Cloudflare 배포에도 Nitro를 쓰는가?

쓸 수는 있지만, vinext에서 권장하는 Cloudflare path는 native integration이다. 이유는 `cloudflare:workers` bindings, KV cache, Images binding, TPR 같은 기능을 vinext가 직접 연결하기 때문이다.

### Nitro routeRules는 모든 static route를 포함하는가?

아니다. vinext의 Nitro routeRules 생성은 ISR route를 중심으로 한다. 또한 `nitro.setup` 시점에는 speculative prerender 결과가 없어서, build 후에야 static으로 확인되는 route를 routeRules에 반영하지 못할 수 있다.

### 사용자가 Nitro routeRules를 직접 쓰면 어떻게 되는가?

vinext는 사용자가 이미 cache 관련 rule을 지정한 route를 자동 생성 rule로 덮어쓰지 않는다. 자동 rule은 빈 곳을 채우는 보조 장치로 봐야 한다.

## 수정할 때 볼 파일

| Concern | Files |
| --- | --- |
| plugin detection, virtual entries, bundled runtime config | [`../../vinext/packages/vinext/src/index.ts`](../../vinext/packages/vinext/src/index.ts) |
| Nitro routeRules generation | [`../../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../../vinext/packages/vinext/src/build/nitro-route-rules.ts) |
| Cloudflare deploy config generation | [`../../vinext/packages/vinext/src/deploy.ts`](../../vinext/packages/vinext/src/deploy.ts) |
| Cloudflare cache handler | [`../../vinext/packages/vinext/src/cloudflare/kv-cache-handler.ts`](../../vinext/packages/vinext/src/cloudflare/kv-cache-handler.ts) |
| TPR | [`../../vinext/packages/vinext/src/cloudflare/tpr.ts`](../../vinext/packages/vinext/src/cloudflare/tpr.ts) |
| Nitro example | [`../../vinext/examples/app-router-nitro`](../../vinext/examples/app-router-nitro) |
| Cloudflare examples | [`../../vinext/examples/app-router-cloudflare`](../../vinext/examples/app-router-cloudflare), [`../../vinext/examples/pages-router-cloudflare`](../../vinext/examples/pages-router-cloudflare) |

## 관련 테스트

- [`../../vinext/tests/nitro-route-rules.test.ts`](../../vinext/tests/nitro-route-rules.test.ts)
- [`../../vinext/tests/deploy.test.ts`](../../vinext/tests/deploy.test.ts)
- [`../../vinext/tests/after-deploy.test.ts`](../../vinext/tests/after-deploy.test.ts)
- [`../../vinext/tests/kv-cache-handler.test.ts`](../../vinext/tests/kv-cache-handler.test.ts)
- [`../../vinext/tests/tpr-kv-keys.test.ts`](../../vinext/tests/tpr-kv-keys.test.ts)
- [`../../vinext/tests/e2e/cloudflare-workers/ssr.spec.ts`](../../vinext/tests/e2e/cloudflare-workers/ssr.spec.ts)
- [`../../vinext/tests/e2e/cloudflare-pages-router/ssr.spec.ts`](../../vinext/tests/e2e/cloudflare-pages-router/ssr.spec.ts)

## 복기용 체크리스트

- 지금 다루는 문제는 Next.js semantics 문제인가, platform output 문제인가?
- `vinext()`가 만들어야 할 virtual entry/alias/route graph가 먼저 존재하는가?
- `nitro()` target에서 Cloudflare-only binding에 의존하고 있지 않은가?
- Cloudflare native path라면 `@cloudflare/vite-plugin`, worker entry, wrangler config가 서로 같은 router mode를 보고 있는가?
- ISR route를 Nitro routeRules로 바꿀 때 user-defined cache rule을 덮어쓰지 않는가?
- catch-all/optional catch-all route pattern이 Nitro wildcard로 올바르게 변환되는가?
