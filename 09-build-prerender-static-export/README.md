# 09. Build, Prerender, And Static Export

## Goal

Production build, prerender, static export가 Vinext에서 어떤 순서와 책임으로
동작하는지 이해합니다.

## Read

- `packages/vinext/src/build/prerender.ts`
- `packages/vinext/src/build/run-prerender.ts`
- `packages/vinext/src/build/static-export.ts`
- `packages/vinext/src/server/metadata-routes.ts`
- `tests/static-export.test.ts`
- `tests/metadata-routes.test.ts`

## Core Questions

1. Production build는 왜 Vite `build()`가 아니라 builder flow를 사용해야 하나?
2. Prerender 대상 route는 어디서 결정되나?
3. Static export는 어떤 asset과 HTML을 생성하나?
4. Metadata routes는 일반 page route와 무엇이 다른가?

## Things To Trace

1. `vp run vinext#build`가 내부적으로 어떤 build path를 타는지 확인합니다.
2. `run-prerender`가 route별 결과를 어떻게 수집하는지 봅니다.
3. Static export test에서 output file assertion을 읽습니다.
4. Metadata route가 static output으로 연결되는지 확인합니다.

## Small Exercise

Build 결과물을 세 그룹으로 나누어 정리합니다.

1. Client assets
2. Server entries
3. Prerendered or static files

## Done When

Build와 prerender가 runtime request handling과 어떻게 다른 책임을 가지는지 설명할
수 있습니다.
