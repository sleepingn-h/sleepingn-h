## 👋

# 1. [Todo](https://github.com/sleepingn-h/todo)

# 1-1. 프로젝트의 주요 기능과 사용된 기술 스택 설명:

## 1-1-1. 사용된 기술 스택

React Router, React Query, react-hook-form, zod

## 1-1-2. 하이라이트 코드

### 의존성 주입 기반 설계 (IHttpClient 인터페이스 구현)

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

### 커스텀 훅을 통한 Todo 비즈니스 로직 추상화

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

# 2. 클라이언트 구현 안내

## Assignment 1 - Login / SignUp

- /auth 경로에 로그인 / 회원가입 기능을 개발합니다
  - 로그인, 회원가입을 별도의 경로로 분리해도 무방합니다
  - [ ] 최소한 이메일, 비밀번호 input, 제출 button을 갖도록 구성해주세요
- 이메일과 비밀번호의 유효성을 확인합니다
  - [ ] 이메일 조건 : 최소 `@`, `.` 포함
  - [ ] 비밀번호 조건 : 8자 이상 입력
  - [ ] 이메일과 비밀번호가 모두 입력되어 있고, 조건을 만족해야 제출 버튼이 활성화 되도록 해주세요
- 로그인 API를 호출하고, 올바른 응답을 받았을 때 루트 경로로 이동시켜주세요
  - [ ] 응답으로 받은 토큰은 로컬 스토리지에 저장해주세요
  - [ ] 다음 번에 로그인 시 토큰이 존재한다면 루트 경로로 리다이렉트 시켜주세요
  - [ ] 어떤 경우든 토큰이 유효하지 않다면 사용자에게 알리고 로그인 페이지로 리다이렉트 시켜주세요

## Assignment 2 - Todo List

- Todo List API를 호출하여 Todo List CRUD 기능을 구현해주세요
  - [ ] 목록 / 상세 영역으로 나누어 구현해주세요
  - [ ] Todo 목록을 볼 수 있습니다.
  - [ ] Todo 추가 버튼을 클릭하면 할 일이 추가 됩니다.
  - [ ] Todo 수정 버튼을 클릭하면 수정 모드를 활성화하고, 수정 내용을 제출하거나 취소할 수 있습니다.
  - [ ] Todo 삭제 버튼을 클릭하면 해당 Todo를 삭제할 수 있습니다.
- 한 화면 내에서 Todo List와 개별 Todo의 상세를 확인할 수 있도록 해주세요.
  - [ ] 새로고침을 했을 때 현재 상태가 유지되어야 합니다.
  - [ ] 개별 Todo를 조회 순서에 따라 페이지 뒤로가기를 통하여 조회할 수 있도록 해주세요.
- 한 페이지 내에서 새로고침 없이 데이터가 정합성을 갖추도록 구현해주세요

  - [ ] 수정되는 Todo의 내용이 목록에서도 실시간으로 반영되어야 합니다

# 3. API 스펙

## [Todos](#todos)

- [getTodos](#getTodos)
- [getTodoById](#getTodoById)
- [createTodo](#createTodo)
- [updateTodo](#updateTodo)
- [deleteTodo](#deleteTodo)

## [Auth](#auth)

- [login](#login)
- [signUp](#signUp)

# <span id="todos">1) Todos</span>

## <span id="getTodos">1-2) getTodos</span>

### URL

- GET `/todos`
- Headers
  - Authorization: login token

### Query Parameters

- **sort** (string, optional): 정렬 기준 (`createdAt`, `updatedAt`, `priority`)
- **order** (string, optional): 정렬 순서 (`asc` 또는 `desc`)
- **priorityFilter** (string, optional): 우선순위 필터링 (`urgent`, `normal`, `low`)
- **keyword** (string, optional): 제목 또는 내용에서 검색할 키워드
- **countOnly** (boolean, optional): `true`로 설정하면 할 일의 개수만 반환

### 응답 예시

```json
{
  "data": [
    {
      "title": "hi",
      "content": "hello",
      "id": "z3FGrcRL55qDCFnP4KRtn",
      "createdAt": "2022-07-24T14:15:55.537Z",
      "updatedAt": "2022-07-24T14:15:55.537Z",
      "priority": "urgent"
    },
    {
      "title": "hi",
      "content": "hello",
      "id": "z3FGrcRL55qDCFnP4KRtn",
      "createdAt": "2022-07-24T14:15:55.537Z",
      "updatedAt": "2022-07-24T14:15:55.537Z",
      "priority": "normal"
    }
  ]
}
```

예시 사용법:

- **우선순위 필터링**: `/todos?priorityFilter=urgent`
- **정렬 및 순서 지정**: `/todos?sort=createdAt&order=desc`
- **조합된 조건**: `/todos?priorityFilter=normal&sort=updatedAt&order=asc&keyword=project`

---

## <span id="todos">1-2) getTodoById</span>

### URL

- GET `/todos/:id`
- Headers
  - Authorization: login token

### 응답 예시

```json
{
  "data": {
    "title": "hi",
    "content": "hello",
    "id": "z3FGrcRL55qDCFnP4KRtn",
    "createdAt": "2022-07-24T14:15:55.537Z",
    "updatedAt": "2022-07-24T14:15:55.537Z",
    "priority": "urgent"
  }
}
```

---

## <span id="createTodo">1-3) createTodo</span>

### URL

- POST `/todos`
- Parameters
  - title: string
  - content: string
  - priority: "urgent" | "normal" | "low"
- Headers
  - Authorization: login token

### 응답 예시

```json
{
  "data": {
    "title": "hi",
    "content": "hello",
    "id": "z3FGrcRL55qDCFnP4KRtn",
    "createdAt": "2022-07-24T14:15:55.537Z",
    "updatedAt": "2022-07-24T14:15:55.537Z",
    "priority": "normal"
  }
}
```

---

## <span id="updateTodo">1-4) updateTodo</span>

### URL

- PUT `/todos/:id`
- Parameters
  - title: string
  - content: string
  - priority: "urgent" | "normal" | "low"
- Headers
  - Authorization: login token

### 응답 예시

```json
{
  "data": {
    "title": "제목 변경",
    "content": "내용 변경",
    "id": "RMfi3XyOKoI5zd0A_bsPL",
    "createdAt": "2022-07-24T14:25:48.627Z",
    "updatedAt": "2022-07-24T14:25:48.627Z",
    "priority": "urgent"
  }
}
```

---

## <span id="deleteTodo">1-5) deleteTodo</span>

### URL

- DELETE `/todos/:id`
- Headers
  - Authorization: login token

### 응답 예시

```json
{
  "data": null
}
```

---

# <span id="auth">2) Auth</span>

## <span id="login">2-1) login</span>

### URL

- POST `/users/login`
- Parameters
  - email: string
  - password: string

### 응답 예시

```json
{
  "message": "성공적으로 로그인 했습니다",
  "token": "eyJhbGciOiJIUzI1NiJ9.YXNkZkBhc2RmYXNkZi5jb20.h-oLZnV0pCeNKa_AM3ilQzerD2Uj7bKUn1xDft5DzOk"
}
```

---

## signUp

## <span id="signUp">2-2) signUp</span>

### URL

- POST `/users/create`
- Parameters
  - email: string
  - password: string

### 응답 예시

```json
{
  "message": "계정이 성공적으로 생성되었습니다",
  "token": "eyJhbGciOiJIUzI1NiJ9.YXNkZkBhc2RmYXNkZi5jb20.h-oLZnV0pCeNKa_AM3ilQzerD2Uj7bKUn1xDft5DzOk"
}
```

```

```

