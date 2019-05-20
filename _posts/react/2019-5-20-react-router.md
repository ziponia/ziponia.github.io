---
layout: post
title: "[리액트] React router 사용하기"
summary: "react router 를 사용 해 보자."
tags: [react]
---

source: [https://github.com/ziponia/react-kakao-search-api](https://github.com/ziponia/react-kakao-search-api)

[지난 포스트](https://ziponia.github.io/2019/05/19/react-http-kakao-search/) 에 이어 이번엔 React router 를 사용 해 보도록 하자.

현재 서비스에는 문제점이 있다.

현재 페이지 어떤 URL 로 접속 하든간에 결과물을 볼 수 없다.

예를들어, A 가 "아메리카노" 라는 키워드를 검색하고, 그 결과물을 B에게 공유 하려고 URL 을 B 에게 보내도

B 는 그냥 빈 화면만 보게 될것이다.

이 문제를 해소하려면, 상태를 저장 해야하는데, 리액트에서 state 는 어떠한 환경에서든 (따로 구현을 하지 않는다면) 가장 초기에 설정 한 state 를 반환 할 수밖에 없는것이다.

그래서 이 부분은 react router 를 이용하여, 해소 해 보도록 하자.

먼저 react router 를 설치하자

```
$ yarn add react-router dom
```

다음으로, Route 처리를 할 컴포넌트를 만들자.

_src/Client.js_

```jsx
import React from "react";
import { BrowserRouter } from "react-router-dom";
import App from "./App";

const Client = props => {
  return (
    <BrowserRouter>
      <App />
    </BrowserRouter>
  );
};

export default Client;
```

그 다음, index.js 를 가서 원래 App.js 를 랜더링 하던것을, Client 컴포넌트를 랜더링 하도록 변경하자.

```jsx
import React from "react";
import ReactDOM from "react-dom";
import "./index.css";
import Client from "./Client";
import * as serviceWorker from "./serviceWorker";

ReactDOM.render(<Client />, document.getElementById("root"));

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: https://bit.ly/CRA-PWA
serviceWorker.unregister();
```

이제 경로에 따라서, 검색 쿼리를 변경 하도록 해줄것이다.

Client js 의 내용을 아래와 같이 변경 해주자.

```jsx
import React from "react";
import { BrowserRouter, Route, Switch } from "react-router-dom";
import App from "./App";

const Client = props => {
  return (
    <BrowserRouter>
      <Switch>
        <Route path="/search/:keyword" component={App} />
        <Route exact path="/" component={App} />
      </Switch>
    </BrowserRouter>
  );
};

export default Client;
```

- /search/:keyword 로 들어왔을 때 App 컴포넌트를 랜더링 한다.
- / 경로로 들어왔을때 App 컴포넌트를 랜더링 한다.

첫번째와 두번째의 차이점은, `:keyword` 가 있냐 없냐이다. 나중에 `keyword` 값을 컴포넌트 내부에서 받을 것이다.

app js 로 가서 App 컴포넌트를 withRouter 로 감싸준다.

_src/App.js_

```jsx
// ...

import { withRouter } from "react-router-dom";

const App = props => {
  // ...
  // console.log(props);
};

export default withRouter(App);
```

App.js 안에서 콘솔을 보면 router props 들이 추가 된 것을 볼 수 있다.

App.js 상단에 아래와 같이 넣어주자.

_App.js_

```jsx
const App = props => {
  const { params } = props.match; // 1
  const keyword = params.keyword || ""; // 2

  // other..
};
```

1: route props 중 match 의 params 를 변수로 할당.

2: params 의 keyword 를 변수로 할당 및 빈 문자열로 초기화

이제 useEffect 함수의 query 를 keyword 로 변경 해주자.

```jsx
const App = props => {
  // ...
  useEffect(() => {
    if (keyword.length > 0) {
      blogSearchHttpHandler(keyword, true);
    } else {
      setBlogs([]);
    }
  }, [keyword]);
  //...
};
```

URL 에다가 쿼리를 넘겨보자

[http://localhost:3000/search/짱구는못말려](http://localhost:3000/search/%EC%A7%B1%EA%B5%AC%EB%8A%94%EB%AA%BB%EB%A7%90%EB%A0%A4)

이제 검색 필드의 default 값을 키워드로 변경 해 주자. query 는 필요없으니 지워준다.

```jsx
const [text, setText] = useState("");

// to

const [text, setText] = useState(keyword);
// const [query, setQuery] = useState(keyword);
```

마지막으로, 엔터 이벤트가 발생 할때 url 을 교체 해 주는 작업을 해주면된다.

onEnter 함수를 다음과 같이 변경하자.

```jsx
const App = props => {
  //...

  // 엔터를 눌렀을 때 호출 되는 함수
  const onEnter = e => {
    if (e.keyCode === 13) {
      if (text.length === 0) props.history.push(`/`);
      else props.history.push(`/search/${text}`);
    }
  };

  //...
};
```

이제 새로고침을 해도, 상태가 유지 되는것을 볼 수 있다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/react-kakao-search-router.gif)

좀 더 코드에 대해 이해하고 싶다면,

- [React Context API](https://reactjs.org/docs/context.html)
- [React Higher Order Component(HOC)](https://reactjs.org/docs/higher-order-components.html)
