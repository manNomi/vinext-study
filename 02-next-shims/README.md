# 02. `next/*` Shim Resolution

## Goal

Vinext가 `next/link`, `next/navigation`, `next/headers` 같은 Next.js 공개 API를
어떻게 직접 구현하는지 이해합니다.

## Read

- `packages/vinext/src/shims/navigation.ts`
- `packages/vinext/src/shims/link.tsx`
- `packages/vinext/src/shims/headers.ts`
- `packages/vinext/src/shims/next-shims.d.ts`
- `tests/shims.test.ts`
- `tests/link.test.ts`

## Core Questions

1. Runtime shim과 type declaration shim은 왜 둘 다 필요한가?
2. Client shim과 server shim의 책임은 어떻게 다르나?
3. `next/navigation.js`처럼 `.js` 확장자가 붙은 import는 어떻게 처리되나?
4. Shim은 어디까지 Next.js와 같아야 하고, 어디부터 Vinext 구현 세부사항인가?

## Things To Trace

1. `next/navigation` import가 어떤 파일로 resolve 되는지 확인합니다.
2. `useRouter`, `usePathname`, `useSearchParams`의 상태 출처를 따라갑니다.
3. `next-shims.d.ts`가 fixture type check에 어떻게 영향을 주는지 봅니다.
4. `tests/shims.test.ts`에서 public API surface를 검증하는 방식을 확인합니다.

## Small Exercise

`useRouter()`가 반환하는 값들을 표로 정리합니다.

| Field or method | Source | Notes |
| --- | --- | --- |
| `push` |  |  |
| `replace` |  |  |
| `refresh` |  |  |
| `bfcacheId` |  |  |

## Done When

새로운 `next/*` API가 추가됐을 때 runtime shim, type declaration, fixture declaration,
test 중 어디를 수정해야 하는지 판단할 수 있습니다.
