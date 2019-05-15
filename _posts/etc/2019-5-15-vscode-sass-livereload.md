---
layout: post
title: "[vscode] live server 및 sass 사용하기"
---

vscode 에서 liver server 및 sass 를 사용하는 방법에 대해 알아보자.

vscode plugin 으로 live server 와 live sass 가 있다.

- [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer)
- [Live Sass Compiler](https://marketplace.visualstudio.com/items?itemName=ritwickdey.live-sass)

두개 플러그인을 설치하자

### sass 사용하기

project 폴더를 만들고 `index.html` 을 만든다.

그 다음 css 폴더를 만들고 `main.scss` 파일을 만든다.

```
/index.html
    /css/
        /main.scss
```

그 다음 플러그인이 정상적으로 설치 되었다면 vscode 하단에 아래와 같은 아이콘이 보일것이다.

![vscode sass image](/images/2019-5-15/sass-vscode-toolbar.png)

아이콘을 클릭하면 `Watching...` 상태로 변경 된다.

그 다음 scss 파일을 다음과 같이 수정 해 보자

```scss
@for $i from 0 through 10 {
  .mb-#{$i * 10} {
    margin-bottom: #{$i * 10}px;
  }
  .mt-#{$i * 10} {
    margin-top: #{$i * 10}px;
  }
  .ml-#{$i * 10} {
    margin-left: #{$i * 10}px;
  }
  .mr-#{$i * 10} {
    margin-right: #{$i * 10}px;
  }
  .m-#{$i * 10} {
    margin-bottom: #{$i * 10}px;
  }
}
```

margin bottom, top, left, right 를 한꺼번에 0px 부터 100px 까지 만들었다.

그럼 css 폴더에 css 파일과 css.map 파일이 생긴다.

결과적으로는

```
/index.html
    /css/
        /main.scss
        /main.css
        /main.css.map
```

형태가 된다.

### Live Server 사용하기

index.html 파일에서 우클릭 후 Open with Live server 를 클릭하자.

브라우저가 열리면서 html 파일을 수정하면 실시간으로 수정 할 수 있게 된다.

상단에 우리가 sass 로 컴파일 한 css 파일을 링크 하면 sass 에서 변경 한 스타일이 바로 적용되어 볼수 있게 된다.

```html
<link rel="stylesheet" href="css/main.css" />
```
