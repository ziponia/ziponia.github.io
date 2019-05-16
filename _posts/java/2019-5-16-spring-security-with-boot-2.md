---
layout: post
title: "[SPRING] Spring Security 사용하기 #2"
---

Security 예외처리를 설정 해보자.

[지난 포스트](/2019/05/15/spring-security-with-boot/) 에서 시큐리티에 대한 기본설정을 해보았다.

이번엔, 좀 더 나아가 예외상황을 직접 커스터마이징 해보자.

# Access Denied (접근 거부) 핸들링

먼저 ROLE_ADMIN 권한 만 접근 할 수 있는 페이지와, 접근이 거부 되었을때 나오는 페이지를 만들어 주자.

_templates/admin.html_

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0"
    />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Document</title>
  </head>
  <body>
    admin 만 들어 올 수 있는 페이지
  </body>
</html>
```

_templates/access-denied.html_

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0"
    />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Document</title>
  </head>
  <body>
    접근이 거부되었습니다.
  </body>
</html>
```

그리고 admin 페이지는 /admin 으로, 접근 거부는 /access-denied 경로로 랜더링 할 수 있는 컨트롤러를 설정하자.

_SecurityController.java_

```java
@Controller
public class SecurityController {

    //...

    @GetMapping(value = "/admin")
    public String adminPage() {
        return "admin";
    }

    @GetMapping(value = "/access_denied")
    public String adminPage() {
        return "access_denied";
    }


    //...
}

```

이제 WebSecurityConfig 에서 다음과 같이 변경 해보자.

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/private/**").hasAnyRole("USER")
                .antMatchers("/admin/**").hasAnyRole("ADMIN") // admin 경로 추가
                .anyRequest().permitAll()
        .and()
            .formLogin()
            .loginPage("/login")
            .usernameParameter("username")
            .passwordParameter("password")
            .failureUrl("/login?error=true")
        .and()
            .exceptionHandling()
                .accessDeniedPage("/access_denied")// access denied page
        ;
    }

    // ...
}

```

> GET /access-denied 는 Security 가 이미 Bean 으로 잡고있어서 못쓴다. 주의하자

이제 서버를 부팅 후, /admin 페이지로 들어가보자. 로그인 창이 뜨고, user / 1234 로 로그인 하게되면, user 는 ADMIN 권한을 가지고 있지 않기 때문에, 접근 거부 페이지가 뜰 것이다.

원활한 개발을 위해 간단하게 네비게이션 메뉴를 추가 해 보자. thymeleaf fragment 를 추가 할껀데, 보여주기만 할거니 자세하게는 몰라도 된다.

_pom.xml_

```xml
<dependency>
    <groupId>nz.net.ultraq.thymeleaf</groupId>
    <artifactId>thymeleaf-layout-dialect</artifactId>
    <version>2.4.1</version>
</dependency>
```

_templates/layout/layout.html_

```html
<!DOCTYPE html>
<html
  lang="ko"
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
>
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0"
    />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <link
      rel="stylesheet"
      href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css"
    />
    <link
      rel="stylesheet"
      href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap-theme.min.css"
    />
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/js/bootstrap.min.js"></script>
    <title>Document</title>
  </head>
  <body>
    <div class="container-fluid">
      <th:block th:replace="layout/navigation" />
      <th:block layout:fragment="contents" />
    </div>
  </body>
</html>
```

_templates/layout/navigation.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
>
  <th:block layout:fragment="navigation">
    <nav class="navbar navbar-default">
      <ul class="nav navbar-nav">
        <li><a th:href="@{/}">홈</a></li>
        <li><a th:href="@{/admin}">관리자 페이지</a></li>
        <li><a th:href="@{/private}">개인 페이지</a></li>
        <li><a th:href="@{/login(logout)}">로그아웃</a></li>
      </ul>
    </nav>
  </th:block>
</html>
```

_access_denied.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
>
  <th:block layout:fragment="contents">
    <h1>접근이 거부되었습니다.</h1>
  </th:block>
</html>
```

_admin.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
>
  <th:block layout:fragment="contents">
    <h1>접근이 거부되었습니다.</h1>
  </th:block>
</html>
```

_index.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
>
  <th:block layout:fragment="contents">
    <h1>안녕하세요 시큐리티 홈 입니다..</h1>
  </th:block>
</html>
```

_private.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
>
  <th:block layout:fragment="contents">
    <h1>비공개 페이지</h1>
  </th:block>
</html>
```

_public.html_

```html
<html
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
>
  <th:block layout:fragment="contents">
    <h1>public 페이지</h1>
  </th:block>
</html>
```

이제 ADMIN 유저를 추가 해보자. import.sql 에 추가하면 된다.

_import.sql_

```sql
insert into tbl_users (idx, username, password) values (1, 'user', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK');
insert into tbl_users (idx, username, password) values (2, 'admin', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK');
```

그리고 admin 유저가 ADMIN 역할을 소유 할 수 있도록 UserDetailsServiceImpl 에 설정 해주자.

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @PersistenceContext
    private EntityManager em; // JPA

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {

        // ...

        if (s.equals("admin")) {
            GrantedAuthority adminRole = new SimpleGrantedAuthority("ROLE_USER");
            authorities.add(adminRole);
        }

        return new User(userEntity.getUsername(), userEntity.getPassword(), authorities);
    }
}

```

이제 각각 user / 1234 와 admin / 1234 로 로그인 - 로그아웃을 반복하면서, admin 페이지에 접근 해 보면 admin 만 접근 할 수있는것을 확인 할 수 있을것이다.

이제 해당 페이지에 접근 하려고 하였을때, 로그를 기록하자

먼저 CustomAccessDeniedHandler class 를 만들고 AccessDeniedHandlerImpl 를 상속받아서 구현하자.

```java
package ziponia.spring.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandlerImpl;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Slf4j
@Configuration
public class CustomAccessDeniedHandler extends AccessDeniedHandlerImpl {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        this.setErrorPage("/access_denied");
        log.info("Access Denied Request: {}, {}", request.getRemoteHost(), request.getRemoteUser());
        super.handle(request, response, accessDeniedException);
    }
}
```

그 다음 WebSecurityConfig 파일로 가서 HttpSecurity 를 교체 해 주면 된다.

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private CustomAccessDeniedHandler customAccessDeniedHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/private/**").hasAnyRole("USER")
                .antMatchers("/admin/**").hasAnyRole("ADMIN")
                .anyRequest().permitAll()
        .and()
            .formLogin()
            .loginPage("/login")
            .usernameParameter("username")
            .passwordParameter("password")
            .failureUrl("/login?error=true")
        .and()
            .exceptionHandling()
                // .accessDeniedPage("/access_denied")
                .accessDeniedHandler(customAccessDeniedHandler)

        ;
    }
}

```

이제 user 로 로그인 해서 관리자 페이지에 들어가면, 위 로그가 뜨는 것을 확인 할 수 있다.
