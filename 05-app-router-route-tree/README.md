# 05. App Router Route Tree

## Goal

App Router의 `layout`, `template`, `page`, `loading`, `error`, parallel route가
Vinext에서 어떤 React tree로 연결되는지 이해합니다.

## Read

- `packages/vinext/src/routing/app-router.ts`
- `packages/vinext/src/entries/app-rsc-entry.ts`
- `packages/vinext/src/shims/slot.tsx`
- `tests/app-router.test.ts`
- `tests/fixtures/app-basic/app`

## Core Questions

1. App Router route tree는 어떤 자료구조로 표현되나?
2. `layout.tsx`와 `page.tsx`는 어떤 순서로 감싸지나?
3. Parallel route와 slot은 일반 child route와 무엇이 다른가?
4. Generated entry는 어디까지 하고, runtime helper는 어디부터 맡아야 하나?

## Things To Trace

1. `tests/fixtures/app-basic/app`에서 nested layout 예시를 하나 고릅니다.
2. 그 파일들이 scanner 결과에서 어떤 tree node가 되는지 봅니다.
3. RSC entry template 안에서 React element가 만들어지는 흐름을 따라갑니다.
4. `slot.tsx`가 slot context를 어떻게 공급하는지 확인합니다.

## Small Exercise

아래 App Router 구조를 직접 tree로 그려봅니다.

```txt
app/
  layout.tsx
  dashboard/
    layout.tsx
    page.tsx
  @modal/
    photo/[id]/page.tsx
```

## Done When

App Router에서 URL segment, layout nesting, slot이 각각 어떤 책임을 가지는지
분리해서 설명할 수 있습니다.
