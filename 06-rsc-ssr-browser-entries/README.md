# 06. RSC, SSR, And Browser Entries

## Goal

Vinext App Router가 RSC environment, SSR environment, browser environment를
나누어 사용하는 이유를 이해합니다.

## Read

- `packages/vinext/src/entries/app-rsc-entry.ts`
- `packages/vinext/src/entries/app-ssr-entry.ts`
- `packages/vinext/src/entries/app-browser-entry.ts`
- `packages/vinext/src/server/app-browser-state.ts`
- `tests/app-browser-entry.test.ts`

## Core Questions

1. RSC entry와 SSR entry는 왜 같은 module state를 공유하지 못하나?
2. RSC stream은 어디서 만들어지고 어디서 HTML로 바뀌나?
3. Browser entry는 hydration 이후 어떤 상태를 소유하나?
4. Per-request state는 environment boundary를 어떻게 넘어가나?

## Things To Trace

1. App Router 요청이 RSC entry로 들어오는 지점을 찾습니다.
2. `handleSsr(rscStream, navContext)`처럼 state가 전달되는 흐름을 확인합니다.
3. Browser entry가 navigation payload를 받아 React tree를 갱신하는 위치를 봅니다.
4. `app-browser-state` 테스트가 어떤 history state 회귀를 막는지 읽습니다.

## Small Exercise

RSC, SSR, Browser 세 환경을 표로 비교합니다.

| Environment | Runs where | Owns | Cannot share |
| --- | --- | --- | --- |
| RSC |  |  |  |
| SSR |  |  |  |
| Browser |  |  |  |

## Done When

Server Component에서 만든 요청 상태가 Client Component SSR까지 전달되려면 왜
명시적인 bridge가 필요한지 설명할 수 있습니다.
