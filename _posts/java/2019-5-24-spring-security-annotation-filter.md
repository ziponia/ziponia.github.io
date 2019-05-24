---
layout: post
title: "[SPRING] Spring Security 사용하기 #9"
summary: "@PreAuthorize @PostAutorize"
tags: [spring, spring-boot, spring-security, jpa]
---

@PreAuthorize 와 @PostAuthorize 에 대해서 알아보자.

오늘 할일
![https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-24/spring-security-annotation-final.gif](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-24/spring-security-annotation-final.gif)

현재까지 우리 프로젝트는 접근관리를, configuration 으로 관리했다.

WebSecurityConfig 파일을 보자

_WebSecurityConfig.java_

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
            .requestMatchers()
            .mvcMatchers("/login/**", "/logout/**", "/private/**", "/admin/**", "/")
            .and()
            .authorizeRequests()
                .antMatchers("/private/**").hasAnyRole("USER")
                .antMatchers("/admin/**").hasAnyRole("ADMIN")
        // ...
    }
}
```

private/ 경로에응 ROLE_USER 를 가지고 있는 user 만 접근가능하고, /admin 경로는 ROLE_ADMIN 경로만 가지고 있는 사람만 접근 가능하다.

하지만, 항상 url 패턴으로 상세하게 관리하기는 조금 버거운 면이 있다.

이런 시나리오를 생각 해 보자.

```
zef 라는 유저가 글을 썻다.
zipo 라는 유저가 zef 의 글을 수정한다.
```

일반 게시판의 위에 시나리오대로 한다면, 누구든지 말이 안되는 것을 인지 하면서도, 시큐리티 입장에서는 허용가능하다.

zef 라는 user 도 USER 역할을 하고있고, zipo 라는 유저도 수정 할 수 있다는 것이다.

코드로 한번 알아보자.

먼저 tbl_user 테이블에 nickname 이라고 만들고, sql 에 nickname 을 추가해보자.

_UserEntity.java_

```java
public class UserEntity {
    // ...

    private String nickName;

    // ...
}
```

```sql
insert into tbl_users (idx, username, password, nick_name) values (1, 'user', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK', '제프');
insert into tbl_users (idx, username, password, nick_name) values (2, 'admin', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK', '블로그 관리자');
insert into tbl_oauth_client (idx, authorities, auto_approve, client_id, client_secret, grant_types, redirect_uri, scope, user_entity_idx) values (1, 'CLIENT', false, 'client', '{bcrypt}$2a$10$cxEU57mmmEm9FfhAJBMW7ec9oG4Y5Uq4Os8CfpxoL6TLzxPCCqzXK', 'client_credentials,authorization_code,refresh_token,password', 'http://localhost:4000/api/callback', 'read,basic,profile', 1);
```

sql 에 nick_name 부분을 추가하고, value 에다 user 는 제프, admin 은 블로그 관리자 라는 닉네임을 주었다.

다음으로, UserController 를 만들자.

```java
package ziponia.spring.security;

import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;

    @GetMapping(value = "/profile")
    public String userProfilePage(
            @RequestParam(required = false) String username,
            Model model
    ) {
        UserEntity user = null;

        if (username != null) {
            user = userRepository.findByUsername(username);
        }

        model.addAttribute("user", user);
        return "profile";
    }

    @PostMapping(value = "/profile/update")
    public String userProfileUpdate(UserEntity entity) {
        UserEntity findUser = userRepository.findByUsername(entity.getUsername());
        findUser.setNickName(entity.getNickName());
        userRepository.save(findUser);
        return "redirect:/profile?username=" + findUser.getUsername();
    }
}

```

username 을 받아, 유저를 검색하고, /profile/update 로 nickname 을 수정 하는 컨트롤러를 만들었다.

이제 profile 화면을 만들자.

_profiile.html_

```html
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorator="layout/layout">

<th:block layout:fragment="contents">
    <div class="row">
        <div class="col-md-6">
            <form action="#" th:action="@{/profile}" method="get" class="form-group block">
                <input type="search" name="username" class="form-control" placeholder="찾으시려는 유저를 검색 해주세요." th:value="${#request.getParameter('username')}">
            </form>
        </div>
    </div>

    <h1 th:if="${#request.getParameter('username') == '' and user == null}">
        유저를 찾을 수 없습니다.
    </h1>

    <template th:if="${user != null}" th:remove="tag">
        <h1 th:inline="text">
            [[ ${user.nickName} ]] 님 의 프로필 입니다
        </h1>

        <form action="#" th:object="${user}" th:action="@{/profile/update}" method="post" class="form-group">
            <input type="hidden" th:field="*{username}"/>
            <label th:for="${user.nickName}" class="control-label">
                닉네임
                <input type="text" th:field="*{nickName}" class="form-control"/>
            </label>
            <button type="submit" class="btn btn-primary">수정하기</button>
        </form>
    </template>
</th:block>

</html>
```

마지막으로, 네비게이션 메뉴에 /profile 경로를 추가하자.

_navigation.html_

```html
<html xmlns:th="http://www.thymeleaf.org" xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
  <th:block layout:fragment="navigation">
    <nav class="navbar navbar-default">
      <ul class="nav navbar-nav">
        <li><a th:href="@{/}">홈</a></li>
        <li><a th:href="@{/admin}">관리자 페이지</a></li>
        <li><a th:href="@{/private}">개인 페이지</a></li>
        <!-- 여기 -->
        <li><a th:href="@{/profile}">유저 검색 페이지</a></li>
        <li>
          <a href="javascript: void(0);" onclick="document.getElementById('logout-form').submit()">
            로그아웃
          </a>
        </li>
      </ul>
    </nav>
    <form id="logout-form" th:action="@{/logout}" method="post"></form>
  </th:block>
</html>
```

이제 서버를 부팅하고 [http://localhost:8080/profile](http://localhost:8080/profile) 로가서 user 를 검색한 후, 닉네임을 수정해보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-24/spring-security-24-1.gif)

잘된다..

하지만 누구나 알것이다 잘되면 안된다.

로그인도 하지않았고, user 로 로그인 하더라도, admin 닉네임은 수정해서는 안된다.

자, 이제 configuration 은 건드리지 않고, 메서드 별로 처리 해보자.

main application 에서 다음과 같은 어노테이션을 추가하자.

_SecurityApplication.java_

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityApplication {
    // ... more..
}
```

다음으로, userController 에 /profile/update 부분에 다음과 같이 추가하자.

_UserController.java_

```java
public class UserController {

    // ...

    @PostMapping(value = "/profile/update")
    @PreAuthorize("isAuthenticated() and #entity.username == authentication.principal.username") // <-- here
    public String userProfileUpdate(UserEntity entity) {
        UserEntity findUser = userRepository.findByUsername(entity.getUsername());
        findUser.setNickName(entity.getNickName());
        userRepository.save(findUser);
        return "redirect:/profile?username=" + findUser.getUsername();
    }
}
```

마지막으로, 접근 거부 창을 띄우기 위해 SecurityController 에 /access_denied 경로를 다음과 같이 변경하자.

_SecurityController.java_

```java
public class SecurityController {
    // ... more

    //@GetMapping(value = "/access_denied")
    @RequestMapping(value = "/access_denied") // 모든 http method 를 허용했다.
    public String access_denied_page() {
        return "access_denied";
    }
}
```

이제 다시 요청 해보자.

최종 결과
![https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-24/spring-security-annotation-final.gif](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-24/spring-security-annotation-final.gif)

@PostAuthorize 와 @PreAuthorize 의 차이점은,

메서드를 실행 한 후, 와 메서드 실행 전 의 차이이다.

@PreAuthorize 를 @PostAuthorize 로 변경 한 다음, 메서드 은에 콘솔을 찍어 보자.
