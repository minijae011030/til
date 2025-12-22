# @tanstack/react-query Optimistic Updates

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

#### ❗️ refetch는 자동으로 취소되는가?

❌ refetch 취소는 자동으로 되지 않는다.

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
