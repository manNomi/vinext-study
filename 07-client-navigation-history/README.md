# 07. Client Navigation And History State

## Goal

Client navigation, `window.history.state`, scroll restoration, `bfcacheId` 같은
브라우저 상태가 Vinext에서 어떻게 관리되는지 이해합니다.

## Read

- `packages/vinext/src/shims/navigation.ts`
- `packages/vinext/src/server/app-browser-state.ts`
- `packages/vinext/src/entries/app-browser-entry.ts`
- `tests/shims.test.ts`
- `tests/e2e/app-router/nextjs-compat`

## Core Questions

1. `router.push`, `router.replace`, `<Link>`는 어디서 browser navigation으로 연결되나?
2. Hash-only navigation은 왜 full navigation과 다르게 처리되나?
3. `history.state`에는 어떤 Vinext 내부 key가 들어가나?
4. `bfcacheId`는 어떤 사용자 문제를 해결하기 위한 escape hatch인가?

## Things To Trace

1. `useRouter().push()`가 URL 변경과 RSC payload 요청으로 이어지는 흐름을 따라갑니다.
2. `scrollToHash` 또는 hash navigation 처리 코드를 찾습니다.
3. Back/forward traversal에서 이전 상태가 복원되는 지점을 확인합니다.
4. Next.js compat E2E 중 navigation 관련 테스트를 하나 읽습니다.

## Small Exercise

아래 navigation에서 어떤 상태가 유지되고 어떤 상태가 바뀌어야 하는지 적어봅니다.

1. Fresh `router.push("/x/2")`
2. Browser back
3. Hash-only navigation
4. `router.refresh()`

## Done When

Fresh navigation과 history traversal이 왜 다른 client state semantics를 가져야
하는지 설명할 수 있습니다.
