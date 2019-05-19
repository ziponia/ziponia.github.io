---
layout: post
title: "[SPRING] Spring Security 사용하기 #5"
summary: "security 부가 옵션 알아보기"
---

source: [https://github.com/ziponia/spring-security-example](https://github.com/ziponia/spring-security-example)

security 에 대해서 좀 더 알아보자.

# 로그인 세션 제한하기

시큐리티에서 로그인 세션 제한을 둘 수 있다.

만약, 사용자가 서울에서 로그인 한 후, 인천에서 로그인하면 서울에서 되어있는 로그인 세션이 풀리는것이다.

코드를 보자

_WebSecurityConfig.java_

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // other....
        http
                .authorizeRequests()
                .antMatchers("/private/**").hasAnyRole("USER")
                .antMatchers("/admin/**").hasAnyRole("ADMIN")
                .anyRequest().permitAll()
        .and()
                .sessionManagement()
                .maximumSessions(1) // 최대 접속수를 1개로 제한한다.
                .expiredUrl("/login?expire=true") // 세션이 제한 되었을 경우 리다이렉트 할 URL
        // other....
        ;
    }

    // ...
}
```

_login.html_

```html
<fieldset>
  <div th:if="${#request.getParameter('expire')}">
    새로운 곳에서 로그인 시도되었습니다.
  </div>
  <legend>로그인</legend>
  <form th:action="@{/login}" th:method="post">
    <input type="text" name="username" placeholder="아이디를 입력 해 주세요." />
    <br />
    <input type="password" name="password" placeholder="비밀번호를 입력 해 주세요." />
    <br />
    <button type="submit">로그인</button>
  </form>
  <div>
    <a th:href="@{/login/facebook}">페이스북 로그인</a>
  </div>
  <div>
    <a th:href="@{/login/github}">깃허브 로그인</a>
  </div>
  <div>
    <a th:href="@{/login/kakao}">카카오 로그인</a>
  </div>
</fieldset>
```

이제 창 두개를 열고, 각각 로그인 해보자

![one session login](/images/2019-5-19/1.gif)

한쪽이 로그인 하면, 다른 한쪽이 로그인 해제 되는것을 볼 수 있다.

# remember-me , 나를 기억해줘 시큐리티!

로그인 기억하기 기능을 알아보자. 이것도 매우 단순하다.

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // other....
        http
                .authorizeRequests()
                .antMatchers("/private/**").hasAnyRole("USER")
                .antMatchers("/admin/**").hasAnyRole("ADMIN")
                .anyRequest().permitAll()
        .and()
                .sessionManagement()
                .maximumSessions(1)
                .expiredUrl("/login?expire=true")
        .and()
        .and()
                .rememberMe()
                .alwaysRemember(false) // 항상 로그인을 기억한다.
                .rememberMeParameter("remember-me") // 기본 파라메터
        // other....
        ;
    }

    // ...
}
```

.rememberMeParameter() 에서는 form 에서 전달 받을, name 을 넣어주면 된다. default 는 remember-me 이다.

```html
<fieldset>
  <div th:if="${#request.getParameter('expire')}">
    새로운 곳에서 로그인 시도되었습니다.
  </div>
  <legend>로그인</legend>
  <form th:action="@{/login}" th:method="post">
    <input type="text" name="username" placeholder="아이디를 입력 해 주세요." />
    <br />
    <input type="password" name="password" placeholder="비밀번호를 입력 해 주세요." />
    <br />
    <button type="submit">로그인</button>

    <!-- 자동 로그인 설정 -->
    <label for="remember-me">
      자동 로그인
      <input type="checkbox" name="remember-me" id="remember-me" />
    </label>
  </form>
  <div>
    <a th:href="@{/login/facebook}">페이스북 로그인</a>
  </div>
  <div>
    <a th:href="@{/login/github}">깃허브 로그인</a>
  </div>
  <div>
    <a th:href="@{/login/kakao}">카카오 로그인</a>
  </div>
</fieldset>
```

이제 체크박스에 체크 후 로그인 하면 remember-me 라는 쿠키가 생성 되는것을 볼 수 있다.

![remember me cookie](/images/2019-5-19/2.png)

브라우저를 닫고 다시 접속 후 개인페이지로 다시 들어 가 보자.

물론, 만료시간, remember-me 기록 방식 등도 지정 할 수 있다.

# 최종 접속일 기록하기

security 의 AuthenticationSuccessHandler 를 이용하여 최종 로그인을 기록 해보자.

먼저 UserEntity 에 필드를 만들어 주자.

```java
public class UserEntity {

    @Id
    @GeneratedValue
    private Integer idx;

    private String username;

    private String password;

    @Enumerated(EnumType.STRING)
    private SocialProvider sns;

    private Date lastLogin; // 최종 로그인 기록
}
```

그리고 userRepository 에서 username 으로 entity 를 찾을 수 있는 쿼리를 만들자.

```java
@Repository
public interface UserRepository extends JpaRepository<UserEntity, Integer> {

    UserEntity findByUsernameAndSns(String id, SocialProvider sns);
    UserEntity findByUsername(String username); // here
}

```

다음으로 LoginSuccessHandler class 를 만들고, SimpleUrlAuthenticationHandler 를 상속 받아 쿼리를 끼워 넣자.

```java
package ziponia.spring.security;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Date;

@Component
public class LoginSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    @Autowired
    private UserRepository userRepository;

    @Override
    @Transactional
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        UserEntity userEntity = userRepository.findByUsername(authentication.getName());
        userEntity.setLastLogin(new Date());
        userRepository.save(userEntity);
        super.onAuthenticationSuccess(request, response, authentication);
    }
}

```

마지막으로 WebSecurityConfig 에서 handler 를 주입 해 주면 된다.

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...
    @Autowired
    private LoginSuccessHandler loginSuccessHandler;

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http

        // ...

        .and()
            .formLogin()
            .loginPage("/login")
            .usernameParameter("username")
            .passwordParameter("password")
            .failureUrl("/login?error=true")
            .successHandler(loginSuccessHandler) // <-- 이곳이다.

        // ...
        ;
    }
    // ...
}
```

이제 로그인 후 h2 콘솔을 열어보면 last_login 필드에 기록 되는걸 볼 수 있다.

![마지막 로그인 기록](/images/2019-5-19/3.png)

오늘은 여기까지

다음엔... 우리가 직접 Social Provider 가 되어 oauth2 인증 시스템을 만들어 보자

## To Be continue...
