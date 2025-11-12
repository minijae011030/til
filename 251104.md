# 251104

## 1. 상태관리: zustand (persist 미들웨어)

### 정의

React 애플리케이션에서 상태 관리를 위한 작고, 빠르며, 확장 가능한 JavaScript 라이브러리이다.

### 장점

- **가볍고 단순**: create() 한 줄로 바로 상태와 액션 선언 가능하며 보일러플레이트가 거의 없다.
- **필요한 부분만 구독 가능**: `useStore((s) => s.slice)`처럼 필요한 slice만 선택적으로 구독하여 불필요한 리렌더가 줄어든다.
- **간단한 영속화**: persist 미들웨어로 로컬스토리지나 세션스토리지에 상태를 쉽게 저장 가능하다. (부분 저장도 가능)

### 단점 / 주의점

- **너무 자유로워서 구조가 흐트러지기 쉬움**: API 호출, 비즈니스 로직, UI 로직이 섞이기 쉬우므로 역할 분리 규칙을 미리 정해야 함.
- **Next.js(SSR/RSC) 환경 주의**: localStorage는 클라이언트에서만 접근 가능 하이드레이션 이슈가 생기면 hasHydrated 사용 필요하다.
- **타입스크립트 가이드 부족**: 팀 단위에서는 상태/액션 네이밍 규칙, 폴더 구조, 미들웨어 사용 방식 등을 통일해두는 게 안정적이다.

### 기존 프로젝트와 차이점

- 기존: 컴포넌트에서 authRepo를 직접 호출 -> 응답을 컴포넌트가 받아서 상태 세팅한다.
- 현재: 스토어 내부에서 authRepo 호출 -> 컴포넌트는 `useAuthStore()`에서 가져온 `login()` 같은 액션만 호출한다.

컴포넌트가 얇아지고, 재사용 쉬움(여러 화면에서 동일 로직 공유)

### 활용 방식

- **다크모드(테마) 관리에 Zustand 사용**
  - persist 미들웨어를 사용해 다크모드 설정을 로컬스토리지에 저장, 새로고침 후에도 이전 테마 상태를 복원하도록 구현하였다.
  - `useThemeStore()`의 `toggleTheme()` 액션으로 모드 전환을 처리하였다.

## 2. 데이터 페칭: React Query (@tanstack/react-query)

### 정의

서버에서 가져오는 서버 상태(비동기 데이터)의 캐싱·동기화를 자동으로 해주는 데이터 페칭 라이브러리이다.

> > 비동기 데이터 다 됨!

### 장점

- **서버 상태 관리에 특화**: 캐싱, 리페치, 리트라이, 동기화 등을 자동으로 처리해 서버 데이터 일관성을 유지한다.
- **로딩/에러 상태 자동 관리**: `isLoading`, `isError`, `data`, `error` 등 내장 상태를 제공해 별도 로직이 필요 없다.
- **자동 리페치 / 백그라운드 업데이트**: 포커스 복귀, 네트워크 재연결 시 자동으로 최신 데이터 요청.
- **데이터 캐싱 및 공유**: 동일한 쿼리 키로 여러 컴포넌트가 데이터를 공유해 불필요한 API 호출 감소.
- **Devtools 지원**: 쿼리 상태(캐시, 요청, 에러 등)를 시각적으로 확인할 수 있다.

### 단점 / 주의점

- **로컬 상태 관리에는 부적합**: 클라이언트 전용 상태(모달, 토글, 폼 등)는 Zustand 같은 전역 상태 관리가 더 적합하다.
- **쿼리 키 관리 복잡도**: 프로젝트 규모가 커질수록 쿼리 키 이름 규칙을 통일하지 않으면 유지보수가 어려워진다.

### 활용 방식

- **블로그 글 조회에 React Query를 사용**
  - `useQuery` 훅으로 게시글 리스트 혹은 게시글을 데이터를 비동기 요청하고, 캐싱된 데이터를 활용해 페이지 이동 시에도 빠르게 가져왔다.
  - `isLoading`, `isError` 상태를 통해 로딩 스피너 및 에러 메시지 UI를 제어하였다.
  - 동일한 쿼리 키 `(['posts', page])`로 페이지 간 데이터 공유 및 캐싱 재활용을 구현하였다.

## @tanstack/react-query Optimistic Updates

Tanstack Query에서 Optimistic Updates는 서버 응답을 기다리기 전에 UI를 먼저 바꿔서 체감 속도를 올리는 패턴이다.

서버가 실패하면 되돌리고, 성공하면 실제 응답으로 확정한다.

### 사용하는 경우

- 버튼을 누르자마자 리스트/카운트/좋아요 상태가 바뀌어야 할 때
- 삭제/수정/좋아요/핀토글 처럼 결과가 예측 가능한 옵션
- 네트워크 지연이 UX를 해치는 구간

## 절차

1. **onMutate**: 서버 호출 직전, 관련 쿼리 refetch 취소 -> 현재 캐시 스냅샷 저장 -> 캐시를 낙관적으로 업데이트
2. **onError**: 실패 시 스냅샷으로 롤백
3. **onSuccess**: 서버 응답으로 정합성 확정(필요하면 캐시 merge)
4. **onSettled**: 마지막에 관련 쿼리 invalidate/refetch

#### ❗️ refetch를 취소하는 이유

1. 게시글 목록을 보고 있음 -> `useQuery(['posts'])`가 캐시를 들고 있음
2. 좋아요 버튼 클릭 -> `useMutation()` 실행됨
3. React Query는 이 와중에 자동으로 refetch 중일 수도 있음 (예: 창을 다시 클릭해서 포커스 됐을 때)
4. refetch가 서버에서 예전 데이터를 받아오면, 방금 낙관적으로 바꿔둔 캐시(좋아요 눌림 상태)가 다시 덮어써짐

### ❗️ refetch는 자동으로 취소되는가?

refetch 취소는 자동으로 되지 않는다.

개발자가 `onMutate` 안에서 직접 `cancelQueries()`를 호출해야 한다.

## 예제: 좋아요 토글

```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";

type Post = {
  id: number;
  title: string;
  liked: boolean;
  likes: number;
};

function useToggleLike() {
  const qc = useQueryClient();

  return useMutation({
    mutationKey: ["post-like"],
    mutationFn: async ({ id, isLiked }: { id: number; isLiked: boolean }) => {
      // 서버에 토글 요청
      const res = await fetch(`/api/posts/${id}/like`, {
        method: "POST",
        body: JSON.stringify({ liked: isLiked }),
        headers: { "Content-Type": "application/json" },
      });

      if (!res.ok) throw new Error("like failed");

      return (await res.json()) as { liked: boolean; likes: number };
    },

    // 1) 서버 호출 직전에 캐시를 먼저 바꾼다
    onMutate: async ({ id, isLiked }) => {
      // 관련 쿼리 refetch 중단
      await qc.cancelQueries({ queryKey: ["post", id] });

      // 스냅샷(롤백용)
      const prevDetail = qc.getQueryData<Post>(["post", id]);

      // 디테일 갱신 -> 낙관적 업데이트
      if (prevDetail) {
        qc.setQueryData<Post>(["post", id], {
          ...prevDetail,
          liked: isLiked,
          // likes 가드: 0 아래로 내려가지 않도록
          likes: Math.max(0, prevDetail.likes + (isLiked ? 1 : -1)),
        });
      }

      // 컨텍스트로 스냅샷 반환 -> onError에서 롤백할 때 사용
      return { prevDetail };
    },

    // 2) 실패시 롤백
    onError: (_err, { id }, ctx) => {
      if (!ctx) return;
      if (ctx.prevDetail) qc.setQueryData(["post", id], ctx.prevDetail);
    },

    // 3) 성공시 서버 응답으로 확정(정밀 값은 서버 것 사용)
    onSuccess: (data, { id }) => {
      qc.setQueryData<Post>(["post", id], (old) =>
        old ? { ...old, liked: data.liked, likes: data.likes } : old
      );
    },

    // 4) 마지막에 최신화
    onSettled: (_data, _err, { id }) => {
      qc.invalidateQueries({ queryKey: ["post", id] });
    },
  });
}
```

### 사용

```tsx
const toggleLike = useToggleLike();

return (
  <button
    disabled={toggleLike.isPending}
    onClick={() => toggleLike.mutate({ id: post.id, isLiked: !post.liked })}
  >
    {post.liked ? "♡" : "♥"} {post.likes}
  </button>
);
```

## 더 공부할것

좋아요를 연속으로 눌렀을때나
무한스크롤 해서 게시글이 너무 많을때 스냅샷저장에 대해
쿼리키 쿼리팩토리
