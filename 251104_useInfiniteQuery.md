# @tanstack/react-query useInfiniteQuery

## 기본 개념

- useQuery: 한 번에 하나의 데이터 세트만 가져온다

  ```ts
  useQuery(["posts", page], () => fetchPosts(page));
  ```

- useInfiniteQuery: 여러 페이지를 연속적으로 캐싱하면서, 필요할 때 다음 페이지를 이어붙여 보여준다.

**무한스크롤**이나 **더보기버튼 UI**를 구현할 때 사용한다.

## 기본 사용법

```ts
import { useInfiniteQuery } from "@tanstack/react-query";

async function fetchPosts({ pageParam = 1 }) {
  const res = await fetch(`/api/posts?page=${pageParam}`);
  return res.json(); // { posts: [...], nextPage: number | null }
}

export function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: fetchPosts,
    getNextPageParam: (lastPage) => lastPage.nextPage, // 다음 페이지 번호 리턴
    initialPageParam: 1,
  });
}
```

### 리턴값 구조

```ts
const {
  data, // { pages: [page1, page2, ...], pageParams: [1, 2, ...] }
  fetchNextPage, // 다음 페이지 요청 함수
  hasNextPage, // 다음 페이지가 있는지 여부
  isFetchingNextPage, // 다음 페이지 로딩 상태
  refetch, // 전체 새로고침
} = usePosts();
```

data.pages는 각 페이지의 데이터를 배열 형태로 누적 저장한다.

```ts
data.pages = [
  { posts: [{ id: 1 }, { id: 2 }] }, // page 1
  { posts: [{ id: 3 }, { id: 4 }] }, // page 2
];
```

그래서 렌더링 시엔 이런 식으로 펼쳐 쓰면 된다.

```ts
{
  data?.pages.flatMap((page) =>
    page.posts.map((post) => <PostItem key={post.id} {...post} />)
  );
}
```
