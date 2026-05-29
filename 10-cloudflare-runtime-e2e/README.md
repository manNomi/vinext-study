# 10 Cloudflare Runtime E2E

## 한 줄 결론

Cloudflare 계층은 vinext의 primary deployment target으로, Workers entry, wrangler config, KV-backed ISR, image optimization, TPR, E2E 검증을 묶는다.

## 이 파트가 존재하는 이유

vinext는 "Next.js API surface를 Vite로 재구현"하는 것에서 끝나지 않고 Cloudflare Workers에 자연스럽게 올리는 것을 핵심 목표로 둔다. Workers는 Node server와 request/asset/cache model이 다르므로 별도 deploy generator와 runtime adapter가 필요하다.

## 핵심 파일 지도

| File | Role |
| --- | --- |
| [`../../vinext/packages/vinext/src/deploy.ts`](../../vinext/packages/vinext/src/deploy.ts) | `vinext deploy` 전체 pipeline, wrangler/worker/vite config 생성 |
| [`../../vinext/packages/vinext/src/cloudflare/index.ts`](../../vinext/packages/vinext/src/cloudflare/index.ts) | public Cloudflare exports |
| [`../../vinext/packages/vinext/src/cloudflare/kv-cache-handler.ts`](../../vinext/packages/vinext/src/cloudflare/kv-cache-handler.ts) | Cloudflare KV-backed CacheHandler |
| [`../../vinext/packages/vinext/src/cloudflare/tpr.ts`](../../vinext/packages/vinext/src/cloudflare/tpr.ts) | Traffic-aware Pre-Rendering pipeline |
| [`../../vinext/packages/vinext/src/server/worker-utils.ts`](../../vinext/packages/vinext/src/server/worker-utils.ts) | Worker entry shared utilities |
| [`../../vinext/packages/vinext/src/server/image-optimization.ts`](../../vinext/packages/vinext/src/server/image-optimization.ts) | `/_vinext/image` handling and Cloudflare Images binding integration |
| [`../../vinext/packages/vinext/src/server/request-pipeline.ts`](../../vinext/packages/vinext/src/server/request-pipeline.ts) | shared request normalization/security/header helpers |
| [`../../vinext/examples/app-router-cloudflare`](../../vinext/examples/app-router-cloudflare) | minimal App Router Workers example |
| [`../../vinext/tests/e2e/cloudflare-workers`](../../vinext/tests/e2e/cloudflare-workers) | App Router Workers E2E |
| [`../../vinext/tests/e2e/cloudflare-pages-router`](../../vinext/tests/e2e/cloudflare-pages-router) | Pages Router Workers E2E |

## 요청/빌드/렌더 흐름

Deploy:

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

Worker request:

```text
Cloudflare Worker fetch(request, env, ctx)
  -> image optimization path short-circuit if /_vinext/image
  -> static asset signal via ASSETS binding
  -> middleware/config/request pipeline
  -> App Router handler or Pages Router handler
  -> KV cache handler uses ctx.waitUntil for background writes
```

TPR:

```text
Cloudflare analytics
  -> traffic path ranking
  -> route selection by coverage/limit
  -> local production server prerender
  -> rendered responses serialized
  -> KV bulk upload with runtime-compatible keys
```

## 주요 개념

| Concept | What to remember |
| --- | --- |
| Generated worker entry | App Router와 Pages Router worker templates가 다르고, image/asset/middleware path가 포함된다. |
| `@cloudflare/vite-plugin` | Workers build에서는 Vite config에 Cloudflare plugin이 필요하다. |
| KV ISR | `KVCacheHandler`가 cache entry와 tag invalidation timestamp를 KV에 저장한다. |
| `waitUntil()` | cache writes/delete 같은 background work가 request 이후에도 살아남게 한다. |
| Cloudflare Images | local image optimization은 `env.IMAGES` binding이 있을 때 resize/transcode 가능하다. |
| TPR | 전체 사이트를 prerender하지 않고 traffic이 있는 path만 KV에 미리 채운다. |
| E2E | Workers behavior는 Node dev/prod와 다르므로 Playwright + wrangler dev 경로가 따로 있다. |

## 수정할 때 깨지기 쉬운 지점

- Workers build에 Cloudflare plugin이 빠지면 RSC/workerd 환경과 bindings가 맞지 않는다.
- Pages Router ISR KV detection은 제한이 있어 수동 설정 gap을 고려해야 한다.
- generated worker template과 `server/worker-utils.ts`, `prod-server.ts`의 header merge/static asset logic이 어긋나기 쉽다.
- KV key format을 바꾸면 runtime cache와 TPR bulk upload가 동시에 깨진다.
- `ctx.waitUntil()`이 ALS를 통해 전달되지 않으면 background KV 작업이 response 이후 중단될 수 있다.
- image optimization fallback은 binding이 없거나 unsupported content type일 때 원본 passthrough가 가능해야 한다.

## 관련 테스트

- [`../../vinext/tests/deploy.test.ts`](../../vinext/tests/deploy.test.ts)
- [`../../vinext/tests/kv-cache-handler.test.ts`](../../vinext/tests/kv-cache-handler.test.ts)
- [`../../vinext/tests/tpr-kv-keys.test.ts`](../../vinext/tests/tpr-kv-keys.test.ts)
- [`../../vinext/tests/image-optimization-parity.test.ts`](../../vinext/tests/image-optimization-parity.test.ts)
- [`../../vinext/tests/request-pipeline.test.ts`](../../vinext/tests/request-pipeline.test.ts)
- [`../../vinext/tests/after-deploy.test.ts`](../../vinext/tests/after-deploy.test.ts)
- [`../../vinext/tests/e2e/cloudflare-workers/ssr.spec.ts`](../../vinext/tests/e2e/cloudflare-workers/ssr.spec.ts)
- [`../../vinext/tests/e2e/cloudflare-workers/navigation.spec.ts`](../../vinext/tests/e2e/cloudflare-workers/navigation.spec.ts)
- [`../../vinext/tests/e2e/cloudflare-pages-router/ssr.spec.ts`](../../vinext/tests/e2e/cloudflare-pages-router/ssr.spec.ts)
- [`../../vinext/tests/e2e/cloudflare-pages-router/middleware.spec.ts`](../../vinext/tests/e2e/cloudflare-pages-router/middleware.spec.ts)

## 복기용 체크리스트

- 이 동작은 Node `vinext start`와 Workers 모두에서 같아야 하는가, Workers 전용인가?
- generated worker와 shared utility가 같은 header/static asset policy를 쓰는가?
- KV entry shape와 TPR upload shape가 같은가?
- `waitUntil()`이 request context를 통해 deep call stack까지 전달되는가?
- Wrangler config, Vite config, worker entry가 서로 같은 router mode를 가리키는가?
