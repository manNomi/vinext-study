# 11 vinext vs Next

이 챕터는 "vinext와 Next.js가 무엇이 다른가"를 기능 목록으로 비교하지 않는다. 핵심 질문은 더 아래에 있다.

- 왜 vinext라는 별도 구현이 나왔는가?
- Next.js 코드를 그대로 가져오면 될 것 같은데, 왜 vinext는 Vite 방식으로 다시 짜는가?
- vinext는 Next.js와 같아야 하는가, 달라야 하는가?
- 기능을 구현하거나 버그를 고칠 때 Next.js 소스를 어디까지 따라가야 하는가?

한 줄로 정리하면 이렇다.

> Next.js는 Next.js API의 원본 구현이고, vinext는 그 공개 API surface를 Vite 위에서 다시 실행시키는 대체 구현이다.

여기서 중요한 단어는 "공개 API surface"와 "대체 구현"이다. vinext는 사용자가 작성한 Next.js 앱 코드를 최대한 유지하려고 한다. 하지만 Next.js 프레임워크 내부 코드를 그대로 복사하려고 하지는 않는다. 목표가 "Next.js 내부 구조를 보존하는 것"이 아니라 "Next.js 앱이 기대하는 외부 동작을 Vite/Nitro/Cloudflare 런타임에서 재현하는 것"이기 때문이다.

## 비교의 기준

vinext와 Next.js를 비교할 때는 세 층을 분리해야 한다.

```text
사용자 앱 코드
  pages/, app/, next/link, next/navigation, next/server, next.config.js

공개 프레임워크 API
  file-system routing, SSR, RSC, route handler, middleware, metadata, cache

프레임워크 내부 구현
  compiler, bundler, manifest, route module, request runtime, deploy output
```

vinext가 맞추려는 것은 첫 번째와 두 번째 층이다. 사용자의 앱 코드가 `next/*` 모듈을 import하고, `pages/` 또는 `app/`에 파일을 두고, `getServerSideProps`, `generateMetadata`, `cookies()`, `headers()`, route handler, server action 같은 Next.js 공개 API를 사용할 때, 그 코드가 비슷하게 동작하도록 만드는 것이 목표다.

반대로 vinext가 그대로 가져오지 않으려는 것은 세 번째 층이다. Next.js 내부의 webpack/Turbopack/SWC 기반 빌드 파이프라인, manifest 구조, route module 클래스, app render runtime, Vercel 중심 최적화는 Next.js라는 구현체에 깊게 결합되어 있다. vinext는 이 내부 구조를 가져오는 대신 Vite plugin hook, Vite module graph, `@vitejs/plugin-rsc`, Web API, Cloudflare Workers, Nitro에 맞는 구조로 다시 만든다.

## 목적 차이

Next.js는 원본 프레임워크다. React 애플리케이션을 위한 routing, rendering, data fetching, bundling, deployment, compiler transform을 하나의 제품으로 제공한다. App Router, Pages Router, RSC, server action, image/font/script 최적화, middleware, metadata, cache, dynamic/static rendering 판단 같은 기능은 Next.js 안에서 설계되고 확장된다.

vinext는 원본 프레임워크가 아니다. vinext는 "Next.js 앱을 Next.js compiler toolchain 대신 Vite 위에서 실행할 수 있는가"라는 실험에서 출발한다. 그래서 새 프레임워크 문법을 만드는 것보다 기존 Next.js 앱의 공개 API를 재현하는 데 집중한다. 사용자는 `next/link`, `next/navigation`, `next/server` 같은 import를 계속 쓰고, `pages/`와 `app/` 디렉터리도 계속 쓴다. 다만 그 import와 파일 시스템 routing이 Next.js 내부가 아니라 vinext의 shim, scanner, virtual entry, server runtime으로 연결된다.

이 차이를 문장으로 압축하면 다음과 같다.

| 관점 | Next.js | vinext |
| --- | --- | --- |
| 정체성 | 원본 full-stack React framework | Next.js 공개 API의 Vite 기반 대체 구현 |
| 목표 | Next.js 생태계와 기능 자체를 설계하고 제공 | 기존 Next.js 앱을 다른 toolchain/runtime에서 실행 |
| toolchain | Next.js compiler pipeline, webpack/Turbopack, SWC, 자체 manifest | Vite plugin stack, `@vitejs/plugin-rsc`, virtual module |
| runtime | Next.js server/app render runtime, Vercel/Node/Edge/self-hosting 경로 | Web API 중심 server runtime, Cloudflare native, Nitro adapter |
| 호환성 철학 | 원본이므로 내부 구현과 공개 동작이 함께 움직임 | 공개 동작은 맞추되 내부 구현은 Vite에 맞게 재작성 |
| 범위 | 방대한 기능과 장기 호환성 | 최신 Next.js API 중심의 pragmatic compatibility |

## vinext가 나온 근본 이유

vinext가 나온 이유를 "Next.js를 싫어해서"라고 이해하면 방향을 놓친다. 더 정확한 이유는 "Next.js의 API는 강력하지만, 그 API가 반드시 Next.js의 기존 compiler/runtime으로만 실행되어야 하는가"를 실험하기 위해서다.

Vite는 현대 프론트엔드에서 사실상 기본 빌드 도구 중 하나가 되었다. 빠른 HMR, native ESM 기반 dev server, 명확한 plugin API, Rollup/Rolldown으로 이어지는 build 흐름, 풍부한 plugin 생태계를 갖고 있다. 여기에 `@vitejs/plugin-rsc`가 React Server Components를 다룰 수 있게 되면서, Vite 위에서도 App Router와 비슷한 multi-environment build를 구성할 수 있는 가능성이 생겼다.

vinext의 문제의식은 여기서 시작한다.

```text
Next.js 앱 코드
  -> Next.js compiler/runtime에서만 실행

이 구조를

Next.js 앱 코드
  -> Vite plugin/runtime에서 실행

으로 바꿀 수 있는가?
```

이 질문의 실용적 가치는 세 가지다.

1. 기존 Next.js 앱 코드를 최대한 보존하면서 Vite toolchain을 사용할 수 있다.
2. Cloudflare Workers 같은 Web API 중심 runtime에 더 직접적인 배포 경로를 만들 수 있다.
3. Next.js API가 "특정 구현"이 아니라 "재현 가능한 public contract"인지 검증할 수 있다.

이 때문에 vinext는 단순한 deploy adapter가 아니다. OpenNext가 `next build` 결과물을 다른 플랫폼에 맞게 적응시키는 쪽이라면, vinext는 Next.js API를 Vite 위에서 다시 구현한다. 이 차이는 매우 크다. OpenNext는 Next.js output을 신뢰하고 그 위에 플랫폼 변환을 붙인다. vinext는 Next.js output을 만들지 않고, Vite가 이해하는 module graph와 build output을 만든다.

## "Next 코드 그대로 가져오면 되지 않나"에 대한 답

이 질문은 두 가지 의미로 나뉜다. 둘을 섞으면 vinext의 설계를 오해하기 쉽다.

첫 번째는 사용자 앱 코드다. 이 코드는 최대한 그대로 가져오는 것이 vinext의 목표다. 사용자가 이미 작성한 `app/page.tsx`, `pages/index.tsx`, `next/link`, `next/navigation`, route handler, middleware, `next.config.js`를 최대한 유지하게 하는 것이 vinext의 가치다.

두 번째는 Next.js 프레임워크 내부 코드다. 이 코드를 그대로 가져오는 것은 vinext의 목표와 맞지 않는다. 이유는 다음과 같다.

### 1. Next.js 내부 코드는 Next.js toolchain을 전제로 한다

Next.js 내부 구현은 독립적인 유틸리티 함수들의 모음이 아니다. build, server, client, route module, manifest, loader, compiler transform이 서로 맞물려 있다. 예를 들어 Next.js의 build 쪽은 webpack/Turbopack/SWC와 강하게 연결되어 있고, server render 쪽은 build 단계에서 생성된 manifest와 route module 정보를 전제로 움직인다.

vinext는 이 전제를 공유하지 않는다. vinext의 중심은 `packages/vinext/src/index.ts`의 Vite plugin이다. 여기서 Vite의 `config`, `resolveId`, `load`, `transform`, `configureServer`, `generateBundle`, `closeBundle` 같은 hook을 사용해 alias, virtual module, route graph, build metadata를 만든다. 즉 같은 기능을 구현하더라도 시작점이 다르다.

```text
Next.js 내부 구현
  compiler/loaders/templates/manifests
  -> Next server runtime
  -> Next deployment output

vinext 내부 구현
  Vite plugin hooks
  -> next/* shims
  -> filesystem route scan
  -> virtual RSC/SSR/browser entries
  -> Vite/Nitro/Cloudflare output
```

Next.js 코드를 그대로 복사하면, 그 코드는 여전히 Next.js manifest, Next.js loader, Next.js compiler state, Next.js route module을 찾는다. vinext가 필요한 것은 그 내부 의존성까지 끌고 오는 것이 아니라, 동일한 공개 동작을 Vite의 module graph 안에서 만드는 것이다.

### 2. 공개 API와 내부 계약은 다르다

`next/link`를 사용하는 사용자는 `Link` 컴포넌트가 어떻게 prefetch하고 navigation하는지를 기대한다. 하지만 Next.js 내부의 router state 구조, manifest field, prefetch scheduler, bundler-specific chunk id는 공개 계약이 아니다.

vinext가 맞춰야 하는 것은 사용자에게 보이는 결과다.

- `next/link`가 렌더링되고 클릭 시 navigation이 일어나는가?
- `next/navigation`의 `useRouter`, `usePathname`, `useSearchParams`가 기대한 값을 주는가?
- `cookies()`와 `headers()`가 request context에서 올바르게 읽히는가?
- route handler의 `NextRequest`, `NextResponse`가 Web API처럼 동작하는가?
- App Router에서 RSC payload, SSR HTML, client hydration이 이어지는가?

반대로 Next.js 내부의 private module, private manifest shape, private class hierarchy까지 맞추는 것은 vinext의 핵심 목표가 아니다. 내부까지 맞추면 vinext는 Vite 대체 구현이 아니라 Next.js 내부 fork에 가까워진다.

### 3. Vite의 장점을 잃는다

vinext가 Next.js 내부를 많이 가져올수록 Vite plugin으로서의 장점이 줄어든다. Vite를 쓰는 이유는 빠른 dev server, 명확한 plugin pipeline, native ESM, Vite 생태계와의 결합이다. 그런데 Next.js 내부 build/runtime을 그대로 가져오면 Vite는 단순 wrapper가 되고, 실제 핵심은 다시 Next.js toolchain으로 돌아간다.

vinext는 `next/*` import를 shim으로 해석하고, `pages/`와 `app/`를 직접 scan하고, RSC/SSR/browser entry를 virtual module로 생성한다. 이 방식은 Vite가 모듈을 해석하고 변환하고 번들링하는 방식과 잘 맞는다. Next.js 내부 코드를 붙이는 방식보다 더 작고 명확한 경로를 만들 수 있다.

### 4. 배포 runtime의 전제가 다르다

Next.js는 self-hosting도 가능하고 Edge runtime도 지원한다. 하지만 vinext의 핵심 배포 축은 Cloudflare Workers native와 Nitro를 통한 multi-platform output이다. 특히 Workers는 Node.js server process보다 Web API 중심 runtime에 가깝다. request/response, cache, bindings, KV, assets, image optimization 같은 관심사가 다르다.

그래서 vinext는 `Request`, `Response`, `Headers`, Web Streams 같은 표준 Web API와 Cloudflare/Nitro adapter에 맞는 방식으로 request pipeline을 구성한다. Next.js 내부 코드를 그대로 가져오면 Node/Vercel/Next build output에 맞춰진 전제가 함께 들어올 수 있다. 그 결과 Workers와 Nitro에서 오히려 더 많은 우회 코드가 필요해진다.

### 5. 유지보수 비용이 폭발한다

Next.js 내부 코드를 많이 복사하면, Next.js가 바뀔 때마다 복사한 코드도 따라가야 한다. 이때 어려운 점은 단순히 파일을 업데이트하는 것이 아니다. 복사한 코드가 의존하는 주변 내부 구조도 함께 바뀌기 때문이다.

vinext는 이 문제를 다른 방식으로 푼다.

- Next.js 소스와 테스트를 참조해 공개 동작을 확인한다.
- 관련 Next.js 테스트가 있으면 vinext 테스트로 port한다.
- 구현은 vinext의 shim/server/routing/entries/build 계층에 맞춰 작성한다.
- 의도적인 차이는 문서화하고, 실수로 생긴 차이는 테스트로 잡는다.

즉 vinext의 호환성 전략은 "source copy"가 아니라 "behavior mapping"이다.

### 6. bug-for-bug parity가 항상 좋은 목표는 아니다

Next.js는 원본이므로 Next.js가 하는 동작이 기준이다. 하지만 vinext는 README에서 명시하듯 "pragmatic compatibility"를 목표로 한다. 일반적인 실사용 앱에서 중요한 동작은 맞추되, undocumented Vercel behavior나 오래된 deprecated API까지 모두 bug-for-bug로 복제하려는 프로젝트는 아니다.

물론 이것이 "대충 비슷하면 된다"는 뜻은 아니다. vinext의 작업 규칙은 오히려 엄격하다. 기능을 추가하거나 버그를 고칠 때 먼저 Next.js 소스와 테스트를 찾아보고, Next.js에 테스트가 있으면 그 케이스를 vinext 테스트로 가져오는 것이 기본이다. 다만 최종 구현은 Next.js 내부 구조를 복사하는 방식이 아니라 vinext의 구조에 맞는 방식이어야 한다.

## 왜 vinext는 Next와 다르게 코딩하는가

vinext 코드가 Next.js 코드와 다르게 생긴 이유는 "기능을 몰라서"가 아니라 "동일한 공개 동작을 다른 substrate 위에 올리기 때문"이다. 여기서 substrate는 코드를 떠받치는 기반 구조를 의미한다. Next.js의 substrate는 Next.js compiler/runtime이고, vinext의 substrate는 Vite plugin/runtime이다.

### Next.js의 내부 사고방식

Next.js 내부는 거대한 framework product로 움직인다. 빌드 과정에서 app/pages route를 분석하고, route module을 만들고, 여러 manifest를 생성하고, webpack/Turbopack/SWC transform을 거쳐 client/server bundle을 구성한다. server runtime은 이 manifest와 module을 읽어 request를 처리한다.

단순화하면 다음과 같다.

```text
source files
  -> Next compiler pipeline
  -> route modules
  -> manifests
  -> app render / pages render runtime
  -> deployment output
```

이 구조에서는 manifest와 compiler output이 매우 중요하다. request runtime은 "이미 Next build가 만들어둔 정보"를 전제로 움직인다.

### vinext의 내부 사고방식

vinext는 Vite plugin으로 시작한다. Vite plugin은 module resolution, transformation, virtual module loading, dev server middleware, build bundle hook에 개입할 수 있다. vinext는 이 hook들을 이용해 Next.js 앱을 Vite가 이해하는 앱으로 바꾼다.

```text
source files
  -> vinext Vite plugin
  -> next/* shim alias
  -> pages/app route scan
  -> virtual RSC/SSR/browser entries
  -> Vite dev/build
  -> Node / Cloudflare / Nitro runtime
```

그래서 vinext에서 자주 보이는 패턴은 Next.js 내부와 다르다.

| vinext 패턴 | 이유 |
| --- | --- |
| `next/*`를 local shim으로 alias | 사용자 import는 보존하고 구현만 Vite-compatible하게 바꾸기 위해 |
| file-system scanner로 route table/graph 생성 | Next.js manifest 대신 vinext가 직접 request matching 데이터를 만들기 위해 |
| virtual module로 entry 생성 | Vite가 RSC/SSR/browser 환경별 entry를 빌드하도록 만들기 위해 |
| Web API 중심 request pipeline | Cloudflare Workers/Nitro/Node 경로를 함께 다루기 위해 |
| AsyncLocalStorage 기반 request context | `headers()`, `cookies()`, server component context를 공개 API처럼 재현하기 위해 |
| platform plugin 감지 | Cloudflare plugin, Nitro plugin 여부에 따라 output/runtime 전제를 조정하기 위해 |

이것이 vinext가 Next.js와 다르게 코딩되는 가장 근본적인 이유다. Next.js가 "자체 compiler가 만든 결과물을 자체 runtime이 소비하는 구조"라면, vinext는 "Vite가 소비할 수 있는 entry와 module graph를 만들어 Next.js 공개 API를 재현하는 구조"다.

## 구조별 깊은 비교

### 1. Build toolchain

Next.js의 build toolchain은 Next.js 자체의 핵심이다. webpack/Turbopack/SWC, route analysis, static/dynamic 판단, manifest 생성, client/server chunk 관리, font/image/script 최적화, middleware build, app/pages runtime bundle이 하나의 pipeline으로 엮인다.

vinext는 이 pipeline을 그대로 쓰지 않는다. 대신 `vinext()`가 Vite plugin stack 안에서 동작한다. Vite plugin hook으로 config를 확장하고, import resolution을 가로채고, virtual module을 제공하고, build bundle을 보정하고, platform plugin과 상호작용한다.

이 차이 때문에 vinext의 build 코드는 Next.js build 코드보다 "Vite가 어느 hook에서 무엇을 요구하는가"에 더 민감하다. 예를 들어 Next.js에서는 build manifest가 중심 데이터라면, vinext에서는 route scan 결과, generated entry source, Vite manifest, Nitro routeRules, Cloudflare output이 각각 다른 시점에 만들어진다.

### 2. Module resolution

Next.js에서 `next/link`, `next/navigation`, `next/server`는 Next.js 패키지 내부 구현으로 연결된다. 사용자는 이 내부 위치를 신경 쓰지 않는다.

vinext에서는 이 import들이 `packages/vinext/src/shims`의 local module로 resolve된다. 이 shim 계층은 vinext의 핵심이다. 사용자의 import는 그대로지만 실제 구현은 달라진다.

```text
import Link from "next/link";

Next.js 실행:
  next/link -> Next.js package implementation

vinext 실행:
  next/link -> vinext shims/link.tsx
```

이 방식은 vinext가 Next.js API를 "외부 모양은 유지하고 내부 구현은 교체"할 수 있게 해준다. `next/navigation`, `next/headers`, `next/cookies`, `next/server`, `next/image`, `next/font`, `next/script` 같은 API도 같은 원리로 접근한다.

### 3. Routing

Next.js는 pages/app 파일을 분석해 내부 manifest와 route module을 구성한다. route matching, dynamic segment, catch-all, optional catch-all, route groups, parallel routes, intercepting routes 같은 개념은 build/runtime 전체와 연결되어 있다.

vinext도 같은 routing convention을 지원해야 하지만, 구현 방식은 다르다. `routing/` 계층이 `pages/`와 `app/` 디렉터리를 scan해서 vinext가 사용할 route table과 app route graph를 만든다. 그 결과는 virtual entry 생성과 request matching에 사용된다.

즉 사용자 관점의 route convention은 Next.js와 같아야 하지만, 내부 자료구조는 vinext의 request pipeline이 읽기 좋은 형태여야 한다.

### 4. App Router와 RSC

App Router는 가장 큰 차이를 만든다. Next.js의 App Router는 Next.js app render runtime, loader tree, flight manifest, client reference manifest, server action handling, cache boundary, metadata, streaming과 깊게 결합되어 있다.

vinext는 이 영역을 Vite의 multi-environment build로 다시 구성한다. 핵심은 세 entry다.

- RSC entry: server component tree를 실행하고 RSC payload를 만든다.
- SSR entry: RSC payload와 React tree를 HTML stream으로 렌더링한다.
- Browser entry: client hydration과 navigation runtime을 연결한다.

이 분리는 `@vitejs/plugin-rsc`가 제공하는 RSC 환경과도 연결된다. Next.js 내부 App Router runtime을 복사하는 대신, vinext는 Vite가 이해하는 entry를 생성하고 그 entry 안에서 route graph, layout tree, request context, metadata, action handling을 연결한다.

### 5. Pages Router

Pages Router는 App Router보다 오래된 구조지만, 여전히 복잡하다. `_app`, `_document`, `_error`, `getStaticProps`, `getServerSideProps`, API routes, data routes, dynamic routes, ISR이 얽혀 있다.

Next.js에서는 이 기능들이 Next.js pages runtime과 build output에 묶여 있다. vinext는 `pages-server-entry`, `pages-client-entry`, `server/dev-server.ts`, `server/prod-server.ts`, `server/api-handler.ts`, ISR cache 계층으로 나눠 구현한다.

여기서도 목표는 같다. 사용자가 작성한 Pages Router 코드는 최대한 그대로 동작해야 한다. 하지만 내부 구현은 Next.js pages runtime을 복사하는 것이 아니라 vinext server runtime으로 재구성된다.

### 6. Request context

`headers()`, `cookies()`, middleware, route handler, server component는 모두 request context를 필요로 한다. Next.js는 자체 AsyncLocalStorage 계층과 work store/request store 계층을 갖는다.

vinext도 request 단위 상태가 필요하다. 하지만 vinext는 이를 Web API와 Vite 환경에 맞게 다시 만든다. `shims/headers.ts`, `shims/cookies.ts`, `shims/request-context.ts`, `shims/unified-request-context.ts`, middleware header propagation 관련 server 파일들이 이 역할을 맡는다.

이 영역에서 Next.js와의 동작 차이는 매우 미묘하게 드러난다. 예를 들어 middleware가 request header를 override할 때 어떤 header를 내부 전파용으로 쓰는지, dev/prod/Workers 경로에서 같은 context가 유지되는지, RSC stream 소비 시 context가 사라지지 않는지 같은 문제가 생길 수 있다.

그래서 vinext에서 request context를 고칠 때는 단일 파일만 보면 위험하다. App Router dev, Pages dev, Pages prod, Workers entry를 함께 확인해야 한다.

### 7. Deployment

Next.js는 Vercel과 강하게 맞물려 있지만 self-hosting과 static export도 제공한다. OpenNext는 Next.js의 build output을 가져와 AWS/Cloudflare 등에서 실행되게 바꾼다.

vinext의 배포 전략은 다르다.

- Cloudflare Workers는 native path다. `vinext deploy`, `@cloudflare/vite-plugin`, bindings, KV cache, Workers assets 같은 요소와 직접 연결된다.
- Cloudflare 외 플랫폼은 Nitro plugin을 통해 간다. `vinext()`가 Next.js 호환 runtime을 만들고, `nitro()`가 Vercel/Netlify/AWS/Deno Deploy 같은 platform output을 맡는다.

이 때문에 vinext의 build/deploy 코드는 "Next.js output을 어떻게 변환할까"보다 "Vite output과 vinext route/runtime metadata를 어떤 platform output에 맞출까"에 가깝다.

## OpenNext와의 차이

vinext를 이해할 때 OpenNext와 비교하면 선명해진다.

```text
OpenNext
  Next.js app
  -> next build
  -> Next.js output
  -> platform adapter

vinext
  Next.js app
  -> Vite + vinext()
  -> Vite output
  -> Cloudflare native or Nitro adapter
```

OpenNext는 Next.js output을 기반으로 하기 때문에 더 많은 Next.js long-tail API를 자연스럽게 가져갈 수 있다. 성숙도와 coverage 측면에서 유리하다.

vinext는 Next.js output을 만들지 않는다. 그래서 coverage 측면에서는 더 어려운 길을 간다. 대신 Vite 기반 dev/build, 더 작은 bundle, Cloudflare native integration, Nitro와의 조합, Vite plugin 생태계라는 장점을 얻는다.

따라서 "안전하게 production Next.js를 다른 플랫폼에 올리고 싶다"면 OpenNext가 더 보수적인 선택일 수 있다. "Next.js API를 Vite 위에서 얼마나 재현할 수 있는가, 그리고 그 결과로 더 가벼운 toolchain을 만들 수 있는가"가 관심이라면 vinext가 의미 있다.

## vinext가 같아야 하는 것과 달라도 되는 것

vinext의 어려움은 "무엇을 반드시 같게 해야 하는가"와 "무엇은 달라도 되는가"를 계속 구분하는 데 있다.

### 반드시 같아야 하는 것

사용자 앱에서 관찰 가능한 공개 동작은 같아야 한다.

- `pages/`와 `app/`의 route convention
- dynamic/catch-all route matching 결과
- `next/link`, `next/navigation`, `next/router`의 사용자 관찰 동작
- `NextRequest`, `NextResponse`, `redirect`, `notFound` 같은 공개 API
- `headers()`, `cookies()`의 request scope 동작
- SSR HTML, RSC payload, hydration의 연결
- `getStaticProps`, `getServerSideProps`, API route, data route의 결과
- metadata, static params, route handler, middleware의 일반적 동작

이 영역에서 Next.js 테스트가 존재하면 vinext도 그 케이스를 참고해야 한다. vinext의 `AGENTS.md`가 강조하는 것처럼, 기능 추가나 버그 수정 전에는 `.nextjs-ref/test`와 `.nextjs-ref/packages/next/src`를 검색해 Next.js의 실제 동작을 확인하는 것이 기본이다.

### 달라도 되는 것

사용자에게 보이지 않는 내부 구현은 달라도 된다. 오히려 달라야 한다.

- Next.js 내부 manifest shape
- webpack/Turbopack loader 구조
- private route module class
- Next.js 내부 file layout
- Vercel-specific undocumented behavior
- deprecated API나 오래된 Next.js version behavior
- Vite/Nitro/Cloudflare에 맞게 달라져야 하는 build output

이 구분이 없으면 vinext는 두 방향 중 하나로 흔들린다. 너무 다르게 만들면 Next.js 앱 호환성이 깨진다. 너무 똑같이 만들려고 하면 Vite 대체 구현이라는 의미가 사라진다.

## 실제 작업할 때의 판단 순서

vinext에서 어떤 기능을 구현하거나 버그를 고칠 때는 다음 순서가 안전하다.

1. 이 문제가 공개 API 문제인지 내부 구현 문제인지 나눈다.
2. 사용자 앱에서 관찰되는 증상을 먼저 적는다.
3. `.nextjs-ref/test`에서 관련 Next.js 테스트를 찾는다.
4. `.nextjs-ref/packages/next/src`에서 Next.js 구현을 읽고 의도를 확인한다.
5. vinext에서 책임 계층을 고른다.
6. Next.js 테스트를 vinext 테스트로 port하거나 동등한 fixture를 만든다.
7. 구현은 vinext의 shim/routing/entries/server/build 구조에 맞춰 작성한다.
8. dev/prod/Workers/Nitro 경로 중 영향을 받는 경로를 함께 확인한다.
9. 의도적으로 Next.js와 다르게 처리했다면 이유를 문서나 테스트명에 남긴다.

책임 계층은 대략 이렇게 고른다.

| 증상 | 먼저 볼 vinext 계층 |
| --- | --- |
| `next/link`, `next/navigation`, `next/server` import 문제 | `shims/` |
| route matching, dynamic params, route priority 문제 | `routing/` |
| App Router layout/page/slot/intercepting route 문제 | `routing/app-router*`, `entries/app-*` |
| RSC payload, SSR HTML, hydration 문제 | `entries/app-rsc-entry.ts`, `entries/app-ssr-entry.ts`, browser entry |
| Pages Router SSR/API/data route 문제 | `entries/pages-*`, `server/dev-server.ts`, `server/prod-server.ts`, `server/api-handler.ts` |
| `headers()`, `cookies()`, middleware context 문제 | `shims/*context*`, `server/middleware*`, request pipeline |
| build/prerender/static export 문제 | `build/`, generated entry, route classification |
| Cloudflare deploy 문제 | `deploy.ts`, `cloudflare/`, Workers E2E |
| Nitro deploy/cache routeRules 문제 | `build/nitro-route-rules.ts`, Nitro plugin setup |

## 예시로 보는 차이

### `next/link`

사용자 코드는 동일하다.

```tsx
import Link from "next/link";

export default function Page() {
  return <Link href="/dashboard">Dashboard</Link>;
}
```

Next.js에서는 이 import가 Next.js client router와 내부 prefetch/navigation runtime으로 연결된다. vinext에서는 `next/link`가 shim으로 resolve되고, vinext의 client navigation/runtime state와 연결된다.

사용자가 기대하는 것은 같다. 링크가 렌더링되고, prefetch가 가능한 상황에서 prefetch가 일어나고, 클릭하면 route가 바뀌어야 한다. 하지만 그 동작을 만드는 내부 router state와 module graph는 다르다.

### `cookies()`와 `headers()`

사용자 코드는 동일하다.

```tsx
import { cookies, headers } from "next/headers";

export default async function Page() {
  const theme = (await cookies()).get("theme")?.value;
  const ua = (await headers()).get("user-agent");
  return <pre>{JSON.stringify({ theme, ua })}</pre>;
}
```

Next.js에서는 자체 request async storage에서 이 값을 읽는다. vinext에서는 shim과 request context 계층이 현재 요청의 `Request`, `Headers`, cookie state를 보관하고 읽는다.

여기서 중요한 것은 API 모양과 request scope 동작이다. 내부 storage class 이름이나 store shape가 같을 필요는 없다. 하지만 stream 렌더링 중에도 같은 요청 context가 유지되어야 하고, middleware가 변경한 header/cookie가 올바르게 전파되어야 한다.

### App Router rendering

Next.js App Router는 자체 app render runtime과 manifest를 기반으로 RSC payload와 HTML을 만든다.

vinext는 app route graph를 만들고, RSC/SSR/browser virtual entry를 생성한다. RSC entry가 server component tree를 실행하고, SSR entry가 HTML stream을 만들고, browser entry가 hydration/navigation을 맡는다.

사용자는 동일한 `app/layout.tsx`, `app/page.tsx`, `loading.tsx`, `error.tsx`, `route.ts` convention을 쓴다. 하지만 내부에서는 Next.js loader tree를 그대로 쓰는 것이 아니라 vinext가 생성한 route graph와 entry source가 중심이 된다.

### Nitro routeRules

Next.js 자체에는 Nitro routeRules라는 개념이 없다. 이것은 Nitro deployment layer의 개념이다. vinext가 Nitro와 함께 사용될 때는 Next.js의 `revalidate` 같은 route-level cache 정보를 Nitro가 이해하는 `routeRules`로 바꿔야 한다.

이 예시는 vinext가 왜 Next.js 내부 구현과 다르게 코딩될 수밖에 없는지 잘 보여준다. 사용자 API는 Next.js 방식이다. 하지만 deployment output은 Nitro 방식이다. 따라서 vinext는 중간에서 Next.js 의미를 Nitro 의미로 변환해야 한다.

## 위험 지점

vinext와 Next.js의 차이를 다룰 때 특히 위험한 지점은 다음과 같다.

### 1. 문서화되지 않은 Next.js 동작

Next.js 내부 동작 중 일부는 public contract가 아니다. 그런데 실제 앱이 그 동작에 기대고 있을 수 있다. vinext가 모든 private behavior를 복제할 수는 없지만, 일반적인 앱에서 자주 밟는 경로라면 테스트로 확인하고 지원해야 한다.

### 2. dev/prod 경로 차이

Vite dev server에서 동작하는 것과 production bundle에서 동작하는 것은 다를 수 있다. 특히 vinext는 App Router dev, Pages dev, Pages prod, Cloudflare Workers entry, Nitro build 경로가 나뉜다. request pipeline을 수정할 때 한 경로만 고치면 다른 경로에서 차이가 생길 수 있다.

### 3. RSC 환경 분리

RSC, SSR, browser는 서로 다른 환경이다. 같은 module import라도 환경에 따라 다르게 resolve되어야 할 수 있다. `next/navigation`, server-only/client-only, client reference, action reference 같은 영역은 이 분리를 잘못 다루면 hydration mismatch나 runtime error로 이어진다.

### 4. Next.js release drift

vinext가 최신 Next.js를 target한다고 해도, Next.js는 계속 바뀐다. public API가 바뀌거나 내부 테스트가 추가되면 vinext도 따라가야 한다. 복사한 내부 코드가 아니라 ported tests와 behavior mapping으로 추적하는 이유가 여기에 있다.

### 5. platform-specific behavior

Cloudflare Workers, Nitro, Node production은 같은 JavaScript를 실행하더라도 runtime capability가 다르다. Node API 사용 가능 여부, stream 처리, cache, environment variable, bindings, asset serving, compression, image optimization이 다를 수 있다. vinext는 이 차이를 platform plugin과 build/deploy 계층에서 흡수해야 한다.

## 결론

vinext는 Next.js를 대체하려는 새 문법의 프레임워크라기보다, Next.js 공개 API를 Vite 기반 구현으로 다시 해석하는 프로젝트다. 그래서 사용자 앱 코드는 Next.js처럼 유지하려고 하지만, 프레임워크 내부 코드는 Next.js처럼 유지하지 않는다.

가장 중요한 판단 기준은 이것이다.

```text
사용자가 관찰하는 Next.js 공개 동작인가?
  -> 최대한 Next.js와 같아야 한다.

Next.js 내부 구현 세부사항인가?
  -> vinext 구조에 맞게 달라도 된다.

Vite/Nitro/Cloudflare runtime에 필요한 변환인가?
  -> Next.js와 다르게 코딩하는 것이 맞다.
```

따라서 "Next 코드를 그대로 가져오면 되지 않나"에 대한 최종 답은 이렇다.

사용자 앱 코드는 최대한 그대로 가져오는 것이 맞다. 하지만 Next.js 프레임워크 내부 코드를 그대로 가져오면 vinext가 얻고자 하는 Vite-native 구조, Cloudflare/Nitro deployment flexibility, 작은 compatibility layer, 명확한 behavior mapping을 잃는다. vinext의 본질은 Next.js를 복사하는 것이 아니라, Next.js 앱이 기대하는 공개 동작을 다른 기반 위에서 다시 성립시키는 것이다.

## References

- [`../vinext/README.md`](../../vinext/README.md): vinext의 목적, design principles, OpenNext 비교, architecture 개요.
- [`../vinext/AGENTS.md`](../../vinext/AGENTS.md): Next.js 동작 검증, 테스트 port, dev/prod parity 작업 규칙.
- [`../vinext/packages/vinext/src/index.ts`](../../vinext/packages/vinext/src/index.ts): `vinext()` Vite plugin의 중앙 진입점.
- [`../vinext/packages/vinext/src/shims`](../../vinext/packages/vinext/src/shims): `next/*` 공개 API shim 계층.
- [`../vinext/packages/vinext/src/routing`](../../vinext/packages/vinext/src/routing): pages/app 파일 시스템 routing scan.
- [`../vinext/packages/vinext/src/entries`](../../vinext/packages/vinext/src/entries): RSC/SSR/browser/pages virtual entry 생성.
- [`../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../../vinext/packages/vinext/src/build/nitro-route-rules.ts): Next.js cache 의미를 Nitro `routeRules`로 변환하는 계층.
- [`../vinext/.nextjs-ref/packages/next/src`](../../vinext/.nextjs-ref/packages/next/src): 로컬 Next.js 참조 소스.
- [`../vinext/.nextjs-ref/test`](../../vinext/.nextjs-ref/test): vinext 호환성 판단에 사용하는 로컬 Next.js 참조 테스트.
