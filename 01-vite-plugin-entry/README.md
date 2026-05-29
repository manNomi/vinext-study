# 01 Vite Plugin Entry

## 한 줄 결론

vinext의 시작점은 CLI가 Vite를 실행하고, `vinext()` Vite plugin이 Next.js식 프로젝트를 Vite식 모듈 그래프로 번역하는 구조다.

## 이 파트가 존재하는 이유

Next.js 앱은 원래 `next dev`, `next build`, `next start`가 많은 암묵적 설정을 대신 처리한다. vinext는 그 자리를 `vinext` CLI와 Vite plugin으로 대체한다.

즉, 이 파트는 다음 책임을 가진다.

- 사용자가 `vinext dev/build/start/deploy/init/check/lint/typegen`을 실행할 수 있게 한다.
- Vite 설정이 없어도 기본 plugin 구성을 만든다.
- `next/*` import를 local shim으로 보낸다.
- `pages/`, `app/`, `middleware.ts`/`proxy.ts`, metadata file, public route를 감지한다.
- dev server middleware와 production build hook을 연결한다.
- App Router용 RSC/SSR/browser virtual entry와 Pages Router용 client/server entry를 생성한다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/cli.ts`](../../vinext/packages/vinext/src/cli.ts) | `vinext` 명령어 파싱, Vite dev/build/start 호출, deploy/init/check/lint/typegen 분기 |
| [`../../vinext/packages/vinext/src/index.ts`](../../vinext/packages/vinext/src/index.ts) | 메인 Vite plugin. alias, config, virtual module, dev middleware, build hook의 중심 |
| [`../../vinext/packages/vinext/src/init.ts`](../../vinext/packages/vinext/src/init.ts) | 기존 Next.js 프로젝트를 비파괴적으로 vinext 병행 실행 가능 상태로 변환 |
| [`../../vinext/packages/vinext/src/check.ts`](../../vinext/packages/vinext/src/check.ts) | migration 전 compatibility scanner |
| [`../../vinext/packages/vinext/src/deploy.ts`](../../vinext/packages/vinext/src/deploy.ts) | Cloudflare Workers 배포 자동화 |
| [`../../vinext/packages/vinext/src/config/next-config.ts`](../../vinext/packages/vinext/src/config/next-config.ts) | `next.config.*` 로드와 resolved config 생성 |
| [`../../vinext/packages/vinext/src/config/dotenv.ts`](../../vinext/packages/vinext/src/config/dotenv.ts) | Next.js식 `.env*` 로드 |
| [`../../vinext/packages/vinext/src/plugins`](../../vinext/packages/vinext/src/plugins) | font, CSS data URL, optimize imports, RSC client references 등 plugin 조각 |

## 요청/빌드/렌더 흐름

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

## Vite plugin stack에서의 위치

`vinext()`는 platform adapter가 아니라 Next.js compatibility adapter다. Vite plugin 배열에서 `vinext()`가 먼저 Next.js 앱을 Vite가 이해할 수 있는 module graph로 만들고, 그 뒤에 `nitro()`나 `cloudflare()` 같은 platform plugin이 output/runtime target을 맡는다.

```ts
// Cloudflare가 아닌 multi-platform target
import { defineConfig } from "vite";
import vinext from "vinext";
import { nitro } from "nitro/vite";

export default defineConfig({
  plugins: [vinext(), nitro()],
});
```

```ts
// Cloudflare Workers native target
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

정리하면 다음과 같다.

| Plugin | What it owns |
| --- | --- |
| `vinext()` | `next/*` shim, Next config/env, pages/app scanner, virtual RSC/SSR/browser entries, build/prerender metadata |
| `@vitejs/plugin-rsc` | RSC transform, `"use client"`/`"use server"` boundary, RSC multi-environment graph. App Router에서 `vinext()`가 자동 구성하거나 사용자가 직접 구성할 수 있다. |
| `nitro()` | Nitro-supported platforms의 server output. vinext가 만든 route/runtime 정보를 Nitro build에 싣는다. |
| `cloudflare()` | Workers build environment, bindings, assets, workerd runtime output. `vinext deploy`가 생성하는 native path다. |

더 자세히 말하면 `vinext()`가 먼저 "이 프로젝트는 Next.js 앱처럼 생겼다"는 사실을 Vite에게 알려준다. 이 단계에서 `next/link` 같은 import는 local shim으로 바뀌고, `pages/`와 `app/`는 route table/route graph가 되며, App Router라면 `virtual:vinext-rsc-entry`, `virtual:vinext-app-ssr-entry`, `virtual:vinext-app-browser-entry` 같은 virtual entry가 생긴다.

그 다음 `nitro()`나 `cloudflare()`가 "그렇게 만들어진 Vite server graph를 어디에 올릴 것인가"를 맡는다. Nitro는 Vercel/Netlify/AWS/Deno 같은 여러 preset의 server output을 만들고, Cloudflare plugin은 workerd/Workers 환경과 bindings를 만든다. 그래서 이 관계는 "둘 중 하나가 Next.js를 구현한다"가 아니라, `vinext()`가 Next.js compatibility layer를 만들고 platform plugin이 배포 target을 만든다는 층 구조다.

상세한 hook 흐름과 책임 분리는 [Appendix: Vite, Nitro, vinext Plugin Stack](../appendix-vite-nitro-vinext/README.md)에 길게 정리했다.

## 주요 함수와 책임

| Function / Hook | What to remember |
| --- | --- |
| `buildViteConfig()` in `cli.ts` | 사용자의 `vite.config.*` 유무에 따라 자동 config 또는 user config merge 경로를 선택한다. |
| `loadVite()` in `cli.ts` | linked package 환경에서도 프로젝트 루트의 Vite를 우선 resolve한다. |
| `vinext()` in `index.ts` | 여러 Vite plugin 조각을 반환하는 최상위 entry다. |
| Nitro detection in `index.ts` | plugin list에서 `nitro`/`nitro:*`를 감지해 bundled runtime에 맞는 build config와 routeRules hook을 적용한다. |
| `vinext:config` plugin | Next config, env, aliases, RSC, build output, page extensions 같은 큰 설정을 만든다. |
| `vinext:pages-router` plugin | dev server middleware에서 Pages/App 요청, middleware, config redirects/rewrites/headers를 연결한다. |
| `vinext:nitro-route-rules` plugin | Nitro의 `setup` hook에 ISR routeRules를 주입한다. |
| virtual module `load()` hooks | generated entry 코드를 Vite module graph에 주입한다. |
| build hooks | build id, server manifest, prerender manifest, precompress, Cloudflare build integration 등을 산출한다. |

## 수정할 때 깨지기 쉬운 지점

- CLI가 user `vite.config.*`를 발견했을 때 plugin을 중복 주입하면 RSC transform이 두 번 돌 수 있다.
- `next/*` alias와 `resolveId` hook은 environment별로 달라질 수 있다. RSC, SSR, client에서 같은 shim을 쓰면 안 되는 경우가 있다.
- `vinext()`는 `nitro()`/`cloudflare()`보다 먼저 있어야 virtual entries와 aliases를 platform plugin이 소비할 수 있다.
- dev server와 production server의 요청 처리 순서가 달라지면 Next.js parity가 쉽게 깨진다.
- middleware, config redirects/rewrites/headers, static asset 처리 순서는 request pipeline 전반에 영향을 준다.
- build hook은 실행 순서가 중요하다. manifest를 쓰는 hook과 읽는 hook의 순서가 바뀌면 prerender/start/deploy가 같이 깨진다.

## 관련 테스트

- [`../../vinext/tests/cli-args.test.ts`](../../vinext/tests/cli-args.test.ts)
- [`../../vinext/tests/init.test.ts`](../../vinext/tests/init.test.ts)
- [`../../vinext/tests/check.test.ts`](../../vinext/tests/check.test.ts)
- [`../../vinext/tests/deploy.test.ts`](../../vinext/tests/deploy.test.ts)
- [`../../vinext/tests/entry-templates.test.ts`](../../vinext/tests/entry-templates.test.ts)
- [`../../vinext/tests/next-config.test.ts`](../../vinext/tests/next-config.test.ts)
- [`../../vinext/tests/dotenv.test.ts`](../../vinext/tests/dotenv.test.ts)
- [`../../vinext/tests/build-optimization.test.ts`](../../vinext/tests/build-optimization.test.ts)

## 복기용 체크리스트

- CLI에서 Vite를 직접 import하지 않고 프로젝트 기준으로 resolve하는가?
- user Vite config가 있을 때 자동 plugin 주입과 충돌하지 않는가?
- Nitro/Cloudflare 같은 platform plugin이 있을 때 bundled runtime용 external/chunk 설정이 적용되는가?
- `next/*` alias가 RSC/SSR/client 환경 차이를 보존하는가?
- dev와 production 요청 순서가 같은 의미를 유지하는가?
- build 산출물의 소비자, 예를 들어 start/deploy/prerender가 필요한 manifest를 같은 위치에서 읽는가?
