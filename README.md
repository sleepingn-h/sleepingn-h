## 👋

# 1. [Todo](https://github.com/sleepingn-h/todo)
## 프로젝트의 주요 기능과 사용된 기술 스택 설명:
### 사용된 기술 스택
React Router, React Query, react-hook-form, zod

### 하이라이트 코드
#### 의존성 주입 기반 설계 (IHttpClient 인터페이스 구현)
Http클래스와 FakeHttp클래스를 구현하여 실제 API 또는 Mock API를 쉽게 교체할 수 있도록 설계되었습니다.
실제 백엔드 서버 없이도 클라이언트 기능을 모두 개발하고 테스트할 수 있도록 FakeHttp 클래스를 설계했습니다.
이 클래스는 IHttpClient 인터페이스를 구현하여, HttpClient와 FakeHttp 간의 유연한 교체를 가능하게 했습니다.
내부적으로는 localStorage를 활용한 CRUD 구현 및 초기 mock 데이터를 동적으로 로드함으로써, 의존성 주입 기반의 확장성과 테스트 가능성을 갖춘 아키텍처를 구현했습니다.

```
// main.tsx
const fakeClient = new FakeHttp();
const baseURL = import.meta.env.VITE_SERVER_URL;
const httpClient = new HttpClient(baseURL);
const tokenStorage = new TokenStorage('token');
const authErrorEventBus = new AuthErrorEventBus();
const authService = new AuthService(httpClient, tokenStorage);

// /service/fakeHttp.ts
export default class FakeHttp implements IHttpClient { ... }

// /src/service/http.ts
export default class Http implements IHttpClient { ... }
```

#### 커스텀 훅을 통한 Todo 비즈니스 로직 추상화
useTodos 훅을 작성하여 Todo 관련 API 호출, 필터링, URL 파라미터 제어 로직을 컴포넌트에서 완전히 분리하였습니다.
내부적으로는 React Query의 useQuery 및 useMutation을 통해 서버 상태를 관리하고,
useSearchParams를 이용해 URL 기반 필터링 및 정렬 기능을 구현하여 UX와 유지보수성을 높였습니다.
- React Query 기반의 커스텀 훅 구조화 : useQuery와 useMutation을 내부에서 캡슐화해 호출 컴포넌트는 API 호출 로직을 몰라도 되며, createTodoItem, updateTodoItem, removeTodoItem 같은 기능들이 훅 안에 모두 포함됨.
- 검색 및 필터 파라미터 처리 : useSearchParams를 사용하여 URL 쿼리 스트링으로 필터/정렬 기능을 관리하여, updateQueryParams, getQueryParams 등 유틸성 함수들을 포함하여 라우팅 기반 필터링이 가능.
- 데이터 필터링을 위한 getFilteredTodo 함수 사용
- 모든 기능을 단일 훅으로 추상화

```
// /src/hooks/useTodos.tsx
const useTodos = (id?: string) => {
  ...
  const todoQuery = useQuery<FetchTodo[] | FetchTodo, Error>({
    queryKey: ['todos', id],
    queryFn: () => todoService.fetchTodo(id),
    staleTime: 1000 * 6 * 5,
  });

  const createTodoItem = useMutation({
    mutationFn: (todo: FetchTodo) => todoService.createTodo(todo),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const updateTodoItem = useMutation({
    mutationFn: (todo: FetchTodo) => todoService.updateTodo(todo, id as string),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
      queryClient.invalidateQueries({ queryKey: ['todos', id] });
    },
  });

  const removeTodoItem = useMutation({
    mutationFn: () => todoService.deleteTodo(id as string),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
      queryClient.removeQueries({ queryKey: ['todo', id] });
    },
  });
  ...

  return {
    todoQuery,
    filteredTodos,
    createTodoItem,
    updateTodoItem,
    removeTodoItem,
    updateQueryParams,
    getQueryParams,
  };
```

# 2. [Instangram clone coding](https://github.com/sleepingn-h/nextjs-instagram)
## 프로젝트의 주요 기능과 사용된 기술 스택 설명:
### 사용된 기술 스택
![Next.js](https://img.shields.io/badge/-Nextjs-000000?style=for-the-badge&logo=Next.js&logoColor=ffffff)
![React](https://img.shields.io/badge/-React-222222?style=for-the-badge&logo=react)
![PostCSS](https://img.shields.io/badge/-PostCSS-DD3A0A?style=for-the-badge&logo=PostCSS&logoColor=ffffff)
![Tailwind CSS](https://img.shields.io/badge/-Tailwindcss-06B6D4?style=for-the-badge&logo=Tailwindcss&logoColor=ffffff)
![TypeScript](https://img.shields.io/badge/-TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)
![NextAuth.js](https://img.shields.io/badge/-NextAuth-9421cf?style=for-the-badge)
![SWR](https://img.shields.io/badge/-Swr-000000?style=for-the-badge&logo=swr&logoColor=white)
![Sanity](https://img.shields.io/badge/-Sanity-F03E2F?style=for-the-badge&logo=Sanity&logoColor=ffffff)
![Git](https://img.shields.io/badge/-Git-F05032?style=for-the-badge&logo=git&logoColor=ffffff)

### 하이라이트 코드
#### 댓글 Optimistic UI 구현 (SWR + useCallback)
useFullPost 훅은 SWR을 활용하여 포스트 데이터의 상태를 관리하며, 댓글 작성 시 mutate를 통해 Optimistic UI를 구현하였습니다.
댓글이 작성되면 서버 응답 전에 UI에 즉시 반영되며, 실패할 경우 자동으로 롤백되도록 구성했습니다.
또한 globalMutate를 활용하여 전체 포스트 목록의 캐시도 최신화하여 데이터 일관성을 유지했습니다.

```
// src/hooks/post.ts
const postComment = useCallback(
  (comment: Comment) => {
    if (!post) return;

    const newPost = {
      ...post,
      comments: [...post.comments, comment],
    };

    return mutate(addComment(post.id, comment.comment), {
      optimisticData: newPost,
      populateCache: false,
      revalidate: false,
      rollbackOnError: true,
    }).then(() => globalMutate('/api/posts'));
  },
  [post, mutate, globalMutate]
);
```

#### Next.js App Router 기반 인증 API 구현
/api/posts API는 withSessionUser 헬퍼를 사용하여 로그인된 사용자만 접근할 수 있도록 구성했습니다.
GET 요청 시 로그인 사용자의 팔로잉 기반 피드를 제공하며, POST 요청은 FormData를 처리하여 텍스트와 이미지 업로드를 동시에 지원합니다.
Next.js 13의 App Router 환경에서 인증, 파일 처리, 에러 핸들링을 통합적으로 구현한 예시입니다.

```
// src/app/api/posts/route.ts
export async function POST(req: NextRequest) {
  return withSessionUser(async (user) => {
    const form = await req.formData();
    const text = form.get('text')?.toString();
    const file = form.get('file') as Blob;

    if (!text || !file) {
      return new Response('Bad Request', { status: 400 });
    }

    return createPost(user.id, text, file) //
      .then((data) => NextResponse.json(data));
  });
}

```
