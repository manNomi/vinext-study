# 03. File-System Routing

## Goal

Vinext가 `pages/`와 `app/` 디렉터리를 읽어서 라우트 manifest로 바꾸는 과정을
이해합니다.

## Read

- `packages/vinext/src/routing/pages-router.ts`
- `packages/vinext/src/routing/app-router.ts`
- `packages/vinext/src/routing/route-matcher.ts`
- `tests/routing.test.ts`
- `tests/route-sorting.test.ts`

## Core Questions

1. Pages Router와 App Router scanner는 어떤 입력과 출력을 가지나?
2. Dynamic route, catch-all route, optional catch-all route는 어떻게 표현되나?
3. Route sorting은 왜 필요한가?
4. Next.js와 route priority가 다르면 어떤 문제가 생기나?

## Things To Trace

1. fixture 앱 하나를 고르고 실제 파일 경로가 route path로 변환되는 흐름을 따라갑니다.
2. `[id]`, `[...slug]`, `[[...slug]]`가 matcher로 바뀌는 코드를 찾습니다.
3. static route와 dynamic route가 충돌할 때 어떤 route가 먼저 매칭되는지 확인합니다.
4. `route-sorting` 테스트가 보호하는 회귀를 읽어봅니다.

## Small Exercise

아래 파일들이 어떤 route로 변환되는지 직접 적어봅니다.

1. `pages/index.tsx`
2. `pages/blog/[slug].tsx`
3. `app/users/[id]/page.tsx`
4. `app/docs/[[...slug]]/page.tsx`

## Done When

파일 하나가 요청 URL과 매칭되기까지 필요한 scanner, matcher, sorter의 역할을
구분해서 설명할 수 있습니다.
