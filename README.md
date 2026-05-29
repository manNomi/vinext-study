# Vinext Structure Study

Vinext 구조를 작은 단위로 읽기 위한 10단계 학습 노트입니다.

목표는 Next.js 앱이 Vinext 위에서 어떻게 Vite, React Server Components,
Cloudflare Workers 런타임으로 연결되는지 직접 코드로 추적하는 것입니다.

## Roadmap

1. [Vite Plugin Entry](./01-vite-plugin-entry/)
2. [`next/*` Shim Resolution](./02-next-shims/)
3. [File-System Routing](./03-file-system-routing/)
4. [Pages Router Runtime](./04-pages-router-runtime/)
5. [App Router Route Tree](./05-app-router-route-tree/)
6. [RSC, SSR, And Browser Entries](./06-rsc-ssr-browser-entries/)
7. [Client Navigation And History State](./07-client-navigation-history/)
8. [Request Context And Server Shims](./08-request-context-server-shims/)
9. [Build, Prerender, And Static Export](./09-build-prerender-static-export/)
10. [Cloudflare Runtime And E2E Flow](./10-cloudflare-runtime-e2e/)

## How To Study

1. 먼저 각 폴더의 `README.md`에 적힌 파일만 읽습니다.
2. `Core Questions`에 답하면서 코드 흐름을 손으로 정리합니다.
3. 관련 테스트를 하나 찾아서 실제 동작을 확인합니다.
4. 이해한 내용을 짧게 요약해서 개인 노트로 남깁니다.

## Useful Commands

```sh
rg -n "resolveId|load\\(" packages/vinext/src/index.ts
rg -n "next/navigation|next/link" packages/vinext/src/shims tests
vp test run tests/shims.test.ts -t "navigation"
vp test run tests/app-router.test.ts -t "router"
```

## Mental Model

Vinext는 Next.js 자체를 실행하는 도구가 아닙니다. Next.js의 공개 API와
라우팅 모델을 Vite, React Server Components, Cloudflare Workers 중심의
런타임 위에 다시 구현하는 호환성 레이어입니다.

코드를 읽을 때는 계속 이 질문을 기준으로 보면 좋습니다.

1. 이 코드는 Next.js 공개 API를 흉내 내는가?
2. 이 코드는 파일 시스템 라우트를 찾는가?
3. 이 코드는 서버 요청을 처리하는가?
4. 이 코드는 브라우저 네비게이션 상태를 다루는가?
5. 이 코드는 Cloudflare 배포나 런타임과 연결되는가?
