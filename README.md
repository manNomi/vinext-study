# vinext Structure Notes

이 저장소는 `vinext`를 처음부터 따라 배우는 학습가이드가 아니라, 구조를 빠르게 복기하고 수정 지점을 찾기 위한 총정리 노트다.

기준 소스는 형제 디렉터리의 [`../vinext`](../vinext) 저장소다. 이 노트는 `vinext` 소스를 수정하지 않고, 관찰한 구조를 챕터별로 압축한다.

## Ultra Goal

`vinext`의 전체 구조를 "어떤 기능이 어디에서 시작되고, 어떤 파일을 지나며, 어떤 테스트로 검증되는가"라는 관점으로 재구성한다.

완성된 노트는 다음 질문에 답해야 한다.

- `next/*` API 호환성은 어디에서 만들어지는가?
- `pages/`와 `app/` 라우팅은 언제 스캔되고 어떤 형태로 매칭되는가?
- App Router 요청은 RSC, SSR, browser entry를 어떻게 통과하는가?
- Pages Router 요청은 SSR, API route, data route, ISR을 어디에서 처리하는가?
- dev, build, start, deploy에서 같은 책임이 어떤 파일로 나뉘는가?
- Cloudflare Workers 전용 경로와 일반 Node production 경로는 어디서 갈라지는가?
- 특정 기능을 수정할 때 우선 봐야 할 소스와 테스트는 무엇인가?

## One Screen Model

vinext는 Next.js 앱을 그대로 두고, Next.js 컴파일러/런타임 대신 Vite 기반 런타임을 끼우는 프로젝트다.

```text
사용자 Next.js 앱
  -> vinext CLI
  -> vinext Vite plugin
  -> next/* shim alias
  -> pages/app 파일 시스템 라우터
  -> virtual entry 생성
  -> dev server 또는 production bundle
  -> Node server / Cloudflare Worker
```

핵심은 네 가지다.

- `packages/vinext/src/index.ts`: Vite plugin의 중앙 허브. alias, config, virtual module, build hook을 묶는다.
- `packages/vinext/src/routing/*`: `pages/`와 `app/` 파일 시스템을 route table/graph로 바꾼다.
- `packages/vinext/src/entries/*`: Vite가 실행할 가상 entry 코드를 생성한다.
- `packages/vinext/src/server/*`와 `packages/vinext/src/shims/*`: 실제 요청 처리와 `next/*` 공개 API 호환성을 구현한다.

## Vite Plugin Stack

vinext는 단독 framework runtime처럼 보이지만, 실제로는 Vite plugin stack 위에서 움직인다.

```ts
import { defineConfig } from "vite";
import vinext from "vinext";
import { nitro } from "nitro/vite";

export default defineConfig({
  plugins: [vinext(), nitro()],
});
```

역할 구분은 이렇게 잡으면 된다.

| Plugin | Responsibility |
| --- | --- |
| `vinext()` | Next.js API 호환성. `next/*` shim, file-system routing, virtual entries, RSC/SSR/browser wiring, build/prerender metadata를 만든다. |
| `nitro()` | Cloudflare가 아닌 Vercel, Netlify, AWS, Deno Deploy 같은 platform output을 담당하는 deployment/runtime adapter다. |
| `cloudflare()` | Cloudflare Workers native target. `cloudflare:workers` bindings, Workers build environment, ASSETS/Images/KV 같은 Cloudflare 기능을 담당한다. |

요약하면 `vinext()`는 "Next.js 앱을 Vite 앱처럼 실행 가능하게 만드는 plugin"이고, `nitro()`나 `cloudflare()`는 "그 Vite 앱을 특정 runtime에 배포 가능하게 만드는 platform plugin"이다.

## Chapter Index

| Chapter | Purpose |
| --- | --- |
| [01 Vite Plugin Entry](./01-vite-plugin-entry/README.md) | `vinext()` plugin과 CLI가 전체 시스템을 부팅하는 방식 |
| [02 Next Shims](./02-next-shims/README.md) | `next/*` import를 Vite 환경에서 동작하게 만드는 shim 계층 |
| [03 File System Routing](./03-file-system-routing/README.md) | Pages/App Router의 파일 스캔, route pattern, trie matching |
| [04 Pages Router Runtime](./04-pages-router-runtime/README.md) | Pages SSR, API route, data route, `_app`, `_document`, ISR |
| [05 App Router Route Tree](./05-app-router-route-tree/README.md) | App Router route graph, layout tree, slots, intercepting routes |
| [06 RSC SSR Browser Entries](./06-rsc-ssr-browser-entries/README.md) | App Router의 RSC, SSR, browser entry 분리와 연결 |
| [07 Client Navigation History](./07-client-navigation-history/README.md) | client navigation, prefetch, history, scroll, refresh |
| [08 Request Context Server Shims](./08-request-context-server-shims/README.md) | AsyncLocalStorage 기반 request context와 middleware header propagation |
| [09 Build Prerender Static Export](./09-build-prerender-static-export/README.md) | build, prerender, static export, standalone, route classification |
| [10 Cloudflare Runtime E2E](./10-cloudflare-runtime-e2e/README.md) | Workers deploy, KV cache, image optimization, TPR, E2E |

## Reading Order

처음 한 번만 읽는다면 이 순서가 가장 짧다.

1. `01`에서 CLI와 Vite plugin의 진입점을 잡는다.
2. `03`에서 route table/graph가 어떤 데이터로 만들어지는지 본다.
3. `06`에서 App Router 요청이 RSC/SSR/browser로 쪼개지는 이유를 잡는다.
4. `04`에서 Pages Router가 별도 runtime인 이유를 확인한다.
5. `02`, `07`, `08`로 API 호환성과 runtime state를 보강한다.
6. `09`, `10`으로 build/deploy/Workers까지 연결한다.

## Source Map

| Area | Primary files |
| --- | --- |
| CLI | [`../vinext/packages/vinext/src/cli.ts`](../vinext/packages/vinext/src/cli.ts), [`../vinext/packages/vinext/src/init.ts`](../vinext/packages/vinext/src/init.ts), [`../vinext/packages/vinext/src/deploy.ts`](../vinext/packages/vinext/src/deploy.ts) |
| Vite plugin | [`../vinext/packages/vinext/src/index.ts`](../vinext/packages/vinext/src/index.ts), [`../vinext/packages/vinext/src/plugins`](../vinext/packages/vinext/src/plugins) |
| Routing | [`../vinext/packages/vinext/src/routing`](../vinext/packages/vinext/src/routing) |
| Entries | [`../vinext/packages/vinext/src/entries`](../vinext/packages/vinext/src/entries) |
| Server runtime | [`../vinext/packages/vinext/src/server`](../vinext/packages/vinext/src/server) |
| Shims | [`../vinext/packages/vinext/src/shims`](../vinext/packages/vinext/src/shims) |
| Build | [`../vinext/packages/vinext/src/build`](../vinext/packages/vinext/src/build) |
| Nitro integration | [`../vinext/packages/vinext/src/build/nitro-route-rules.ts`](../vinext/packages/vinext/src/build/nitro-route-rules.ts), [`../vinext/examples/app-router-nitro`](../vinext/examples/app-router-nitro) |
| Cloudflare | [`../vinext/packages/vinext/src/cloudflare`](../vinext/packages/vinext/src/cloudflare), [`../vinext/packages/vinext/src/deploy.ts`](../vinext/packages/vinext/src/deploy.ts) |
| Tests | [`../vinext/tests`](../vinext/tests), [`../vinext/tests/e2e`](../vinext/tests/e2e) |

## Modification Compass

| Symptom | First place to inspect |
| --- | --- |
| `next/link`, `next/navigation`, `next/server` behavior mismatch | `02`, then matching file in `shims/` |
| route priority or dynamic param mismatch | `03`, then `routing/*` and route tests |
| App Router layout, slot, intercepting route issue | `05`, then `app-route-graph.ts` and `app-page-route-wiring.tsx` |
| RSC payload, SSR HTML, hydration mismatch | `06`, then generated app entries and `server/app-*` files |
| Pages Router SSR/API/data route issue | `04`, then `pages-server-entry.ts`, `dev-server.ts`, `prod-server.ts` |
| navigation, prefetch, scroll, history issue | `07`, then `shims/navigation.ts`, `shims/link.tsx`, `server/app-browser-*` |
| `headers()`, `cookies()`, middleware request override issue | `08`, then `shims/headers.ts`, `unified-request-context.ts` |
| static export, prerender, build report issue | `09`, then `build/prerender.ts`, `run-prerender.ts`, `static-export.ts` |
| Nitro deployment or ISR routeRules issue | `09`, then `build/nitro-route-rules.ts` and `examples/app-router-nitro` |
| Workers deploy, KV ISR, image optimization issue | `10`, then `deploy.ts`, `cloudflare/*`, worker E2E |

## Verification Habit

노트를 바탕으로 실제 코드를 수정할 때는 전체 테스트보다 관련 테스트를 먼저 고른다.

```bash
vp test run tests/app-router.test.ts
vp test run tests/pages-router.test.ts
vp test run tests/navigation-planner.test.ts
vp test run tests/prerender.test.ts
vp test run tests/deploy.test.ts
```

넓은 변경이면 `pnpm run check`로 formatting, lint, type check까지 확인한다.
