---
layout: post
title: "[리액트 #8] List 에 데이터 추가하기"
---

List 에 데이터를 추가하자

지난시간에 만들어 둔 리스트에 데이터를 추가 해보자.

데이터를 추가하려면 input 박스가 있어야 데이터를 넣을 수 있다.

먼저 데이터를 추가 할 form 컴포넌트를 만들자

```jsx
// src/AddForm.js
import React from "react";

const AddForm = props => {
  return (
    <div>
      <fieldset>
        <legend>개인정보</legend>
        <label for="name">이름:</label>
        <input type="text" id="name" />
        <br />
        <label for="name">나이:</label>
        <input type="number" id="age" />
        <br />
      </fieldset>
    </div>
  );
};

export default AddForm;
```

그 다음 App.js 에다 AddForm 컴포넌트를 랜더링 해주자

```jsx
// src/App.js
import React, { Component } from "react";
import List from "./List";
import AddForm from "./AddForm";

class App extends Component {
  render() {
    return (
      <div className="App">
        <AddForm />
        <List />
      </div>
    );
  }
}

export default App;
```

이 다음엔, AddForm 컴포넌트의 엘리먼트들에 이벤트 들을 연결 해 주도록 하겠다.

```jsx
import React from "react";

const AddForm = props => {
  return (
    <div>
      <fieldset>
        <legend>개인정보</legend>
        <label htmlFor="name">이름:</label>
        <input type="text" id="name" onChange={props.onNameChange} />{" "}
        {/* 여기와 */}
        <br />
        <label htmlFor="name">나이:</label>
        <input type="text" id="age" onChange={props.onAgeChange} /> {/* 여기 */}
        <br />
      </fieldset>
    </div>
  );
};

export default AddForm;
```

input 필드 두곳에다 onChange 이벤트를 연결 하였다.

이전에 이야기 했다시피 props 로 부모 컴포넌트에다 이벤트를 전달 해 주면 된다.

그 다음, App.js 에다 해당 onChange 함수를 넘겨 받은 함수를 만들자.

```jsx
import React, { Component } from "react";
import List from "./List";
import AddForm from "./AddForm";

class App extends Component {
  nameChangeHandler = () => {
    // name 필드가 변경 될때 호출 된다.
  };

  ageChangeHandler = () => {
    // age 필드가 변경 될때 호출 된다.
  };

  render() {
    return (
      <div className="App">
        <AddForm
          onNameChange={this.nameChangeHandler}
          onAgeChange={this.ageChangeHandler}
        />
        <List />
      </div>
    );
  }
}

export default App;
```

다음으로, props 로 전달 받은 value 가 App.js 의 state 를 변경 할 수 있도록 하자

```jsx
class App extends Component {
  constructor(props) {
    super(props);

    // state 를 추가 하였다.
    this.state = {
      age: "",
      name: ""
    };
  }

  nameChangeHandler = e => {
    // name 필드가 변경 될때 호출 된다.
    this.setState({
      ...this.state,
      name: e.target.value
    });
  };

  ageChangeHandler = e => {
    // age 필드가 변경 될때 호출 된다.
    this.setState({
      ...this.state,
      age: e.target.value
    });
  };

  render() {
    return (
      <div className="App">
        <AddForm
          onNameChange={this.nameChangeHandler}
          onAgeChange={this.ageChangeHandler}
        />
        <List />
      </div>
    );
  }
}

export default App;
```

이 다음엔, 변경 된 state 가 AddForm 컴포넌트의 props 로 전달되어, AddForm 컴포넌트 안의 input value 가 변경 될 수 있도록 하자.

```jsx
import React, { Component } from "react";
import List from "./List";
import AddForm from "./AddForm";

class App extends Component {
  constructor(props) {
    super(props);

    // state 를 추가 하였다.
    this.state = {
      age: "",
      name: ""
    };
  }

  nameChangeHandler = e => {
    // name 필드가 변경 될때 호출 된다.
    this.setState({
      ...this.state,
      name: e.target.value
    });
  };

  ageChangeHandler = e => {
    // age 필드가 변경 될때 호출 된다.
    this.setState({
      ...this.state,
      age: e.target.value
    });
  };

  render() {
    return (
      <div className="App">
        <AddForm
          onNameChange={this.nameChangeHandler}
          onAgeChange={this.ageChangeHandler}
          newAge={this.state.age} // newAge 라는 이름으로 state 를 전달하였다.
          newName={this.state.name} // newName 이라는 이름으로 state 를 전달하였다.
        />
        <List />
      </div>
    );
  }
}

export default App;
```

AddForm 컴포넌트에서 props 를 input 필드로 전달 해주자.

아! 액션을 할 수 있는 버튼을 만들어야 한다.

```jsx
import React from "react";

const AddForm = props => {
  return (
    <div>
      <fieldset>
        <legend>개인정보</legend>
        <label htmlFor="name">이름:</label>
        <input
          type="text"
          id="name"
          onChange={props.onNameChange}
          value={props.newName}
        />
        <br />
        <label htmlFor="name">나이:</label>
        <input
          type="text"
          id="age"
          onChange={props.onAgeChange}
          value={props.newAge}
        />
        <br />
      </fieldset>
      <button onClick={props.onSubmit}>저장</button>
    </div>
  );
};

export default AddForm;
```

저장 버튼도 만들어서 props 의 submit 이벤트로 전달 해 주었다.

```jsx
// src/App.js
class App extends Component {
  // ...some code

  onSubmitHandler = () => {
    // 저장 버튼을 클릭 했을 때 호출된다.
  };

  render() {
    // ... some jsx
  }
}
```

이제 List 컴포넌트 안의 data 를 App.js 에다 옮기고 그 데이터는 List 컴포넌트에 Props 로 주어 랜더링 할 수 있도록 하겠다.

```jsx
import React from "react";
// import { users } from "./data.json"; 삭제한다.

const List = props => {
  return (
    <ul>
      {/* @start loop */}
      {/* users.map => props.users.map 으로 바꾼다. */}
      {props.users.map(user => (
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

```jsx
// App.js
import { users } from "./data.json";  // 상단에 data 를 임포트 한다.

class App extends Component {

    constructor(props) {
    super(props);

    // state 를 추가 하였다.
    this.state = {
      age: "",
      name: "",
      users: users // state 로 관리하기 위해 불러 온 데이터를 state 로 집어 넣는다.
    };
  }

  ...

  render() {
    return (
      <div className="App">
        <AddForm
          onNameChange={this.nameChangeHandler}
          onAgeChange={this.ageChangeHandler}
          newAge={this.state.age}
          newName={this.state.name}
          onSubmit={this.onSubmitHandler}
        />
        <List users={this.state.users}/> {/* state 를 List 컴포넌트에 Props 로 전달 해준다. */}
      </div>
    );
  }
}
```

이제 App.js 파일에서 onSubmitHandler method 를 구현한다.

```jsx
class App extends Component {

    ...some...

    onSubmitHandler = () => {
    // 저장 버튼을 클릭 했을 때 호출된다.
        const newArr = this.state.users.concat({
        id: this.state.users.length + 1,
        age: this.state.age,
        name: this.state.name
        });
        this.setState({ users: newArr });
    };

  ...
}

```

이제 form 에서 이름과 나이를 입력하고 저장 버튼을 누르면 리스트가 추가 되는것을 볼 수 있다.
