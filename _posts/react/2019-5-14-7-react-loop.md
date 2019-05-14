---
layout: post
title: "[리액트 #7] List 랜더링 하기"
---

list 를 랜더링 하는 방법에 대해서 알아보자

react 에서 리스트를 랜더링 하려면 먼저 아래 문법을 이해해야한다.

```js
const arr = ["사과", "포도", "토마토", "오랜지", "참외"];
const arr_close = arr.map((fruit, index) => {
  if (index === 0) {
    return "먹다 남은 사과";
  }
  return fruit;
});

console.log(arr_close); // ["먹다 남은 사과", "포도", "토마토", "오랜지", "참외"]
```

보통 리스트를 랜더링 할때 `.map` 을 사용하게 되는데, `.map` 은 return 한 객체를 _반환_ 한다.

이 속성을 이용해서 랜더링을 해보자.

먼저 List 라는 컴포넌트를 만들자

```jsx
// src/List.js
import React from "react";

const List = () => {
  return (
    <ul>
      {/* @start loop */}
      <li>
        <p>나이: 32</p>
        <p>이름: 홍길동</p>
      </li>
      {/* @end loop */}
    </ul>
  );
};

export default List;
```

@start loop ~ @end loop 까지가 하나의 아이템이 된다.

그리고 App.js 에서 List 를 랜더링 해주자

```jsx
import React, { Component } from "react";
import List from "./List";

class App extends Component {
  render() {
    return (
      <div className="App">
        <List />
      </div>
    );
  }
}

export default App;
```

자, 이제 데이터를 만들자.
data 는 json 파일로 만들어 보도록 하겠다.

```json
// src/data.json
{
  "users": [
    {
      "id": 1,
      "name": "홍길동",
      "age": 32
    },
    {
      "id": 2,
      "name": "홍길순",
      "age": 28
    },
    {
      "id": 3,
      "name": "김제프",
      "age": 33
    },
    {
      "id": 4,
      "name": "이제프",
      "age": 37
    },
    {
      "id": 5,
      "name": "제프온라인",
      "age": 45
    },
    {
      "id": 6,
      "name": "소녀시대",
      "age": 28
    },
    {
      "id": 7,
      "name": "방탄소년단",
      "age": 27
    }
  ]
}
```

그 다음 json 데이터를 불러오자

```jsx
import React from "react";
import { users } from "./data.json";

const List = () => {
  return (
    <ul>
      {/* @start loop */}
      <li>
        <p>나이: 32</p>
        <p>이름: 홍길동</p>
      </li>
      {/* @end loop */}
    </ul>
  );
};

export default List;
```

그 다음 불러온 데이터를 아까 설명 한 .map 을 이용해서 루프를 돌리자.

```jsx
import React from "react";
import { users } from "./data.json";

const List = () => {
  return (
    <ul>
      {/* @start loop */}
      {users.map(user => (
        <li>
          <p>나이: 32</p>
          <p>이름: 홍길동</p>
        </li>
      ))}
      {/* @end loop */}
    </ul>
  );
};

export default List;
```

```jsx
// 이 두개는 서로 같은 코드다
users.map(user => (...));

// and

users.map(user => {
    return user;
});
```

그 다음 `<p></p>` 값을 교체 해 주면 된다.

```jsx
import React from "react";
import { users } from "./data.json";

const List = () => {
  return (
    <ul>
      {/* @start loop */}
      {users.map(user => (
        <li key={user.id}>
          <p>나이: {user.age}</p>
          <p>이름: {user.name}</p>
        </li>
      ))}
      {/* @end loop */}
    </ul>
  );
};

export default List;
```

그럼 올바르게 랜더링 된 것을 확인 할 수 있다

> 주의! react 에서는 loop 를 돌릴때, 루프 안의 최상위 컴포넌트에 key props 를 전달 해 주어야 한다. 위 코드에서는 `<li key={user.id}>` 가 그것이다.
