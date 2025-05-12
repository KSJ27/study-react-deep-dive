# Next.js 13과 리액트 18

## App 디렉터리의 등장

### 라우팅

**`layout`**

- 페이지별로 기본 레이아웃을 구성할 수 있음.
- 이전에는 전역에서 _document와 _app 파일만 사용할 수 있었음.
- ⚠️ layout 내부에서도 API 요청과 같은 비동기 작업을 수행할 수 있다.

**`page`**

- `params`와 `searchParams` prop을 받음.
    - params: 동적 라우트의 파라미터
    - `searchParams`: `URLSearchParams`를 의미함. /?searchquery=asdasd

**`error`**

- 해당 라우팅 영역에서 사용되는 공통 에러 컴포넌트
- 해당 라우트의 `<Layout><Error>{children}</Error></Layout>` 을 씌운 것과 같은 구조이다.

**`not-found`**

- 404 에러 발생 시 보여줄 페이지

**`loading`**

- `Suspense` 기반으로 해당 컴포넌트가 불러오는 중임을 나타낼 때 사용

**`api/route`**

- REST API를 처리할 수 있음.

## 리액트 서버 컴포넌트

### 기존 컴포넌트와 Pages Router의 한계

☹️ **SSR을 해도, JS를 클라이언트에 넘기는 걸 피할 수는 없었다.**

```tsx
// 방법 A
import marked from 'marked'; // 35.9K (11.2K gzipped)
import sanitizeHtml from 'sanitize-html'; // 206K (63.3K gzipped)

function Page({page}) {
  const [content, setContent] = useState('');
  useEffect(() => {
    fetch(`/api/content/${page}`).then((data) => {
      setContent(data.content);
    });
  }, [page]);

  return <div>{sanitizeHtml(marked(content))}</div>;
}

// 방법 B
export async function getServerSideProps() {
  const raw = await fs.readFile('page.md');
  const html = sanitizeHtml(marked(raw.toString()));
  return { props: { html } };
}
```

A나 B나 클라이언트에서 하이드레이션을 하기 위해 `sanitize-html`, `marked`가 번들에 포함됨.

→ SSR을 해도 클라이언트에 무거운 JS를 보내야 한다는 구조적 한계가 있음.

😎 Server component와 App router

```tsx
// app/page.tsx (Server Component)
import marked from 'marked';
import sanitizeHtml from 'sanitize-html';

export default async function Page() {
  const raw = await fs.readFile('page.md');
  const html = sanitizeHtml(marked(raw.toString()));

  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

```

이 컴포넌트는 클라이언트에 전송되지 않음.

`sanitize-html`, `marked`는 서버에서만 사용되고 번들에서 제거됨.

☹️ **백엔드 리소스에 직접 접근하지 못했다.**

React 컴포넌트는 기본적으로 브라우저에서 실행된다고 가정됨. 따라서 `getServerSideProps`를 통해 데이터를 먼저 패칭해서 props로 넘겨야 함.

```tsx
export async function getServerSideProps() {
  const content = fs.readFileSync('some.md', 'utf-8');
  return { props: { content } };
}

export default function Page({ content }) {
  return <div>{content}</div>;
}
```

😎 Server component와 App router

서버 컴포넌트 내부에서 직접 서버 코드를 실행할 수 있음.

```tsx
export default async function Page() {
  const content = fs.readFileSync('some.md', 'utf-8');
  return <div>{content}</div>;
}
```

**☹️ 자동 코드 분할(code splitting)이 어려웠다.**

```tsx
import { lazy } from "react"

const OldPhotoRenderer = lazy(() => import ("./OldPhotoRenderer.js"));
const NewPhotoRenderer = lazy(() => import ("./NewPhotoRenderer.js"));

function Photho(props) {
	return isNew ? <NewPhotoRenderer /> : <OldPhotoRenderer />
```

일일이 `lazy`로 감싸야 함. + 조건문을 판단하기 전까지 지연 로딩한 어떤 컴포넌트를 불러올지 결정할 수 없음.

😎 Server component와 App router

조금 더 정교하게 자동 코드 분할이 이뤄짐. 예) 서버/클라이언트 컴포넌트를 나누면, 자동으로 번들도 분리됨.

**☹️ 렌더링에 필요한 요청이 많았다.**

B 컴포넌트의 렌더링이 A에 의존한다고 가정하면 (A → B 순서), A의 요청과 렌더링이 끝나기 전까지는 B의 요청과 렌더링이 끝나지 않음. ⇒ 그만큼 불필요한 리렌더링과 요청이 발생함.

😎 Server component와 App router

데이터 요청과 렌더링이 서버에서 한 번에 이뤄짐.

### 서버 컴포넌트란?

> 서버 컴포넌트는 번들링 전에 클라이언트 앱이나 SSR 서버와는 분리된 환경에서 미리 렌더링되는 새로운 유형의 컴포넌트입니다. - [React 공식 문서](https://ko.react.dev/reference/rsc/server-components)
> 
- **클라이언트 번들**
    
    – 브라우저에서 실행되는 React 컴포넌트
    
- **SSR 서버**
    
    – `getServerSideProps`나 API route 같은 전통적 SSR
    
- **RSC 서버 번들**
    
    – 오직 서버 컴포넌트만 포함된 런타임
    
    - **분리된 빌드 타겟**
        
        Next.js는 `app/` 디렉토리의 서버 컴포넌트를 컴파일할 때, 이들만 따로 묶어서 “서버 전용 번들”을 만듭니다
        
    - **실행 환경**
        
        이 서버 전용 번들은 웹 서버 안의 SSR 핸들러와는 별도로, 빌드 중 한 번 실행(Static Rendering)할 수도 있고, 요청 시마다 실행(Server Rendering)할 수도 있음.
        
    - **출력 형태**
        
        전통적 SSR처럼 완성된 HTML 문자열을 반환하기보다는, “컴포넌트 트리 + props” 형태의 React Flight payload를 만들어 클라이언트로 스트리밍함.
        

**컴포넌트 트리**

![image.png](image.png)

**서버 컴포넌트**

- 요청 순간 딱 한 번 실행되기 때문에 상태를 가질 수 없음. (useState, …)
- 렌더링 생명 주기를 사용할 수 없음. (useEffect, … )
- 따라서 effect와 state에 의존하는 훅은 사용할 수 없으나, 서버에서 제공할 수 있는 기능만 사용하는 훅은 사용할 수 있음.
- 브라우저에서 실행되지 않기 때문에 브라우저에서 제공하는 api를 사용할 수 없음.
    - **DOM / BOM**: `document`, `window`, `navigator`, `history` 등
    - **Web Storage**: `localStorage`, `sessionStorage`
    - **WebRTC, Service Worker, Canvas, WebGL** 등 각종 클라이언트 기능
- 대신, 서버에서 실행되지 때문에 서버에만 있는 데이터를 비동기로 접근할 수 있음.
    - DB, 내부 서비스, 파일 시스템, …

**클라이언트 컴포넌트**

- 생략

**공용 컴포넌트(shared component)**

- 서버/클라 모두 사용할 수 있음. 두 컴포넌트의 제약을  모두 받음. 아래 예시 확인.

```tsx
export default function ParentServerComponent() {
	return (
		<ClientComponent>
			<ServerComponent/>
		</ClientComponent>
	)
}
```

### 서버 사이드 렌더링과 서버 컴포넌트의 차이

서버 사이드 렌더링 (pages router)

**목적:** 

정적인 HTML을 빠르게 내려줌.

**과정:**

1. 응답 받은 페이지 전체를 HTML로 렌더링하는 과정을 서버에서 수행한 후, 그 결과를 클라이언트에 내려줌.
2. 클라이언트에서 하이드레이션 과정을 거쳐 서버의 결과물을 확인하고 이벤트를 붙이는 등의 작업을 수행함.

**한계:** 

초기 HTML이 로딩된 이후에는 클라이언트에서 JS를 다운로드하고, 파싱하고, 실행해야함.

서버 컴포넌트 (app router)

**목적:**

- 서버 전용 로직(파일 시스템·DB·무거운 라이브러리 사용 등)을 클라이언트 번들에 포함시키지 않고 처리
- JS 번들 크기를 최소화하여 초기 로딩 성능 최적화
- React 18 Streaming SSR을 활용해 사용자에게 가능한 빨리 의미 있는 내용(HTML)을 제공

**과정:**

1. **서버 컴포넌트 실행**
    - `app/` 디렉토리 내의 서버 컴포넌트가 서버(Node.js 런타임 또는 Edge 런타임)에서 실행
    - `async` 로직, `fs`, DB 쿼리, `sanitize-html`·`marked` 같은 무거운 라이브러리 등 모두 서버에서 수행
2. **Flight payload 생성 & 스트리밍**
    - React가 “컴포넌트 트리 + props” 형태의 Flight payload를 생성
    - HTML 조각과 메타데이터를 스트리밍 방식으로 클라이언트에 전송
3. **클라이언트 렌더링**
    - 클라이언트는 받은 HTML을 곧바로 렌더링
    - 이후 **클라이언트 컴포넌트**(useState/useEffect 포함)에 대해서만 JS를 다운로드·파싱·실행하여 인터랙션을 활성화
- **한계**
    - **인터랙티브 로직 불가:** 버튼 클릭, 폼 입력 처리 등은 반드시 클라이언트 컴포넌트로 분리해야 함
    - **캐싱·재검증 세팅 필요:** `fetch()`의 `cache`/`revalidate` 설정을 통해 효율적 캐싱을 구성해야 하고, 그렇지 않으면 매 요청마다 불필요한 재생성 발생

### 서버 컴포넌트의 작동 과정

1. **렌더링 요청 발생과 서버 렌더링 실행**
2. **컴포넌트를 JSON 형식으로 전송** 
    
    ```json
    M1: ("id":"'/src/SearchField.client.js", "chunks": ["client5"], "name": ""}
    M2: ("id":"./src/EditButton.client.js"', "chunks": ["client1"], "name": ""}
    S3: "react.suspense"
    J0: ["s", "div", null, {"className": "main", "children" : [["$", "section", null, {"className": "co
    sidebar", "children" : [["$" , "section", null,{"className": "sidebar-header" , "children": [["$", "img", null, {"className": " logo", "src": "logo.svg", "width": "22px", "height": "20px
    , "alt":"","role": "presentation"}],["$", "strong", null, {"children": "React Notes"}]]}],["$"," section", null, 
    {"className": "sidebar-menu", "role": "menubar", "children" : [["$", **"@1"**, null,}], ["$ ,"®2", null, {"noteId": null, "children": "New"}]l}],["$", "nav", null, {"children": ["$", "$3", null, f"
    ```
    
    M: 클라이언트 컴포넌트
    
    S: Suspense
    
    J: 서버에서 렌더링된 컴포넌트. `@1` 는 `M1` 이 렌더링되면 들어갈 자리라는 의미이다.
    
3. **브라우저가 컴포넌트 트리를 구성. JSON 결과물로 트리를 재구성** 

**특징**

- 결과물이 HTML이 아니라 JSON 형태로 전달된다. 따라서 단순 HTML을 그리는 게 아니라, 리액트 컴포넌트 트리의 구성을 돕는다는 걸 알 수 있다.
- 서버에서 클리언트로 스트리밍 형태로 데이터를 보냄으로써, 클라이언트가 줄 단위로 JSON을 읽고 컴포넌트를 렌더링할 수 있어 브라우저에서 되도록 빨리 사용자에게 결과물을 보여줄 수 있다는 장점이 있다.
- 컴포넌트별로 번들링이 되어 있어서, 필요에 따라 지연해서 받거나 따로 받을 수 있다.
