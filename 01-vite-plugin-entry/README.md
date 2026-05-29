# 01. Vite Plugin Entry

## Goal

Vinext가 Vite plugin으로 어디서 시작되는지 이해합니다. Next.js 형태의 앱이
Vite module graph 안으로 들어오는 첫 관문입니다.

## Read

- `packages/vinext/src/index.ts`
- `packages/vinext/src/cli.ts`
- `package.json`
- `packages/vinext/package.json`

## Core Questions

1. `vinext()` plugin은 어떤 Vite hook을 사용하나?
2. `next/*` import는 어디서 Vinext shim으로 바뀌나?
3. `virtual:` module은 어디서 만들어지고 어디서 load 되나?
4. dev 전용 동작과 build 전용 동작은 어떻게 갈라지나?

## Things To Trace

1. `index.ts`에서 `resolveId`를 찾습니다.
2. `index.ts`에서 `load(`를 찾습니다.
3. `virtual:vinext`로 시작하는 module id를 따라갑니다.
4. App Router와 Pages Router scanner가 호출되는 위치를 확인합니다.

## Small Exercise

아래 문장을 직접 채워봅니다.

1. Vinext는 Vite에 `...`로 등록된다.
2. `next/navigation`은 `...`를 통해 Vinext shim으로 연결된다.
3. 서버 엔트리는 `...` virtual module로 생성된다.
4. 라우트 스캔은 `...` 모듈로 위임된다.
5. Next.js 호환성은 `...` 레이어에서 만들어진다.

## Done When

`import { useRouter } from "next/navigation"`이 실제 Next.js package가 아니라
Vinext shim으로 연결되는 흐름을 설명할 수 있습니다.
