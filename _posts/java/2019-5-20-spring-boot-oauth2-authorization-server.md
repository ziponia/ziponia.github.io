---
layout: post
title: "[SPRING] Spring Security 사용하기 #6"
summary: "web security 에 api 인증 서비스 만들기"
tags: [spring, spring-boot, spring-security]
---

# 최종 목표

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-7.gif)

지난번포스팅까지 잘 따라왔다면, 사실 조금 더 서비스에 맞는부분을 커스터마이징 해서 사용하는데 문제가 없다.

근데 이런경우라면 생각해보자.

```
N 사가 우리의 리소스를 사용하기 위해 회사랑 계약을 하였다.
우리는 내부 소스코드는 노출 하지 않은채, N 사에게 인터페이스만 제공 하여야 한다.
우리는 N 사에게만, 리소스를 제공 하여야 하며, 외부로 오픈되면 안된다.
사용자가 N 사를 통해서 접속 할 때, 우리가 N 사에게 허용 한 리소스만 N 사는 사용자에게 정보를 제공 할 수 있다.
```

사실 이것을 구현 하는일은, 쉽지 않다.

그래서, 머리 좋은 어떤 사람들이 [oauth2](https://oauth.net/2/) 라는 인증 워크 플로우를 만들어 냈고, 이 기술은 표준으로 등록 되어 질 만큼 핫(?) 하다.

이 플로우를 설계 하는데 앞서, 우리는 일단 용어들을 대충이라도 알고 있어야 한다.

이전 포스팅 까지 진행했던 [소셜 로그인 서비스](https://ziponia.github.io/2019/05/16/spring-security-with-boot-oauth2/) 의 facebook 로그인을 살펴보자.

우리는 facebook 측으로 부터 client id 와 client secret 을 발급 받은적이 있다.

![페이스북 개발자센터](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/facebook_oauth_client_id_secret.png)

이것은 우리가 페이스북 서버에 접속하여, 페이스북 안에 들어있는 사용자 와 관련된 정보를 가져 올 때 사용 하는

일종에 아이디와 패스워드이다.

그리고, facebook 에서 우리 에게 주는 facebook 사용자의 정보들 이메일, 유저 프로필 사진, 아이디 등을 `scope` 라고 한다.

facebook 에서 발급 한, client id 와 client secret 을 가지고 페이스북에 사용자 정보 요청을 대행(?) 해 주는

우리를 `클라이언트` 라고 한다.

페이스북에서 우리에게 리소스 서버에 접근 권한을 주고, `access_token` 이라는 입장권한(?) 을 주는 서버를

`Authorization server (권한 서버)` 라고 한다.

실제 페이스북 안의 사용자의 리소스를 우리에게 제공해 주는 API 서버를 `Resource Server (리소스 서버)` 라고 한다.

우리를 통해 facebook 안에 리소스를 요청하고, 리소스의 주인이 되는 User 를 `Resource Owner (리소스 주인)` 라고 한다.

우리가 Resource Server 에 사용자 정보를 요청하고, 제공 받을 수 있도록 Authorization Server 가 부여 한 입장권을 `access token` 이라고 한다.

access token 이 유통기한(?) 이 지나, 사용 할 수 없도록 되어 진 것을 `access token expired (토큰 만료)` 라고 한다.

access token 을 재발급 할 수 있도록 임시로 유통기한이 긴 토큰을 `refresh token` 이라고 한다.

...

oauth2 에 workflow 에 대한 이미지는 구글에 `oauth2 workflow` 라고만 검색해도, 엄청난 정보들이 쏟아지니 참고하자.

이제 지루한 이론은 뒤로하고, security 로 구현 방법을 보자.

# 시큐리티 관리 분리

사전에 해야 할 일은, security 가 관여 해야 할 url 을 분리하는것이다.

RootSecurityConfig class 를 만들자

_RootSecurityConfig.java_

```java
package ziponia.spring.security;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableOAuth2Client;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;

@Configuration
@EnableOAuth2Client
@EnableWebSecurity
public class RootSecurityConfig {

    @Configuration
    public static class FormSecurityConfigAdapter extends WebSecurityConfig {}

}
```

기존에 WebSecurityConfig 에서 사용하던 어노테이션들을 옮기고 static class를 만들어 기존에 WebSecurityConfig 를 상속 받도록 하였다.

다음으로 WebSecurityConfig 가 관여 해야 할 path 를 지정하자.

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .requestMatchers()
            .mvcMatchers("/login/**", "/logout/**", "/private/**", "/admin/**", "/")
            .and()

            // ... other settings....
        ;
    }
}

```

class 에 어노테이션들을 제거하고, http 바로 아래, requestMatchers 를 추가하였자.

`"/login/**", "/logout/**", "/private/**", "/admin/**", "/"` 패턴의 URL 들만 기존 시큐리티가 관리하도록 설정하였다.

> 반드시, 다른 설정보다 위에 있어야 한다. 그렇지 않으면, 기본으로 시큐리티가 모든 path 를 관리한다.

> 지정하지 않아도 문제가 된다. 한번 `/ 경로` 를 제외 하고 `/` 경로에서 로그 아웃을 해보라!

# Resource Server Config 생성

이제 API 를 요청 할 수 있는 Resource Server 를 생성하자

```java
package ziponia.spring.security;

import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configurers.ResourceServerSecurityConfigurer;

public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        super.configure(resources);
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .antMatcher("/api/**") // /api/** 경로 관리
                .authorizeRequests()
                .antMatchers("/api/**").hasAuthority("CLIENT");
    }
}

```

일단 단순 한 형태의 시큐리티 패턴만 주었다.

이제 RootSecurityConfig 에 아래 항목을 추가하자.

_RootSecurityConfig_

```java
@Configuration
@EnableOAuth2Client
@EnableWebSecurity
public class RootSecurityConfig {

    @Configuration
    public static class FormSecurityConfigAdapter extends WebSecurityConfig {}

    @Configuration
    @EnableResourceServer
    public static class ApiSecurityConfigAdapter extends ResourceServerConfig {}

}
```

# 권한 서버(토큰발급 서버) 생성

이제 권한 서버를 생성하자

_AuthorizationServerConfig.java_

```java
package ziponia.spring.security;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer;

@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        // 기본 AuthorizationServer Security 설정은 유지 한채,
        // client 의 인증정보를 Header 가 아닌, form 으로 받을 수 있도록 한다.
        super.configure(security);
        security
                .allowFormAuthenticationForClients();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients
                .inMemory()
                .withClient("client")
                .secret(passwordEncoder.encode("secret"))
                .authorizedGrantTypes("client_credentials", "authorization_code", "refresh_token", "password")
                .authorities("CLIENT")
                .scopes("read")
                .redirectUris("http://localhost:4000/api/callback")
                .autoApprove(false);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
    }
}

```

모든 설정이 완료되었다.

이제 postman 에서 테스트를 진행 해 보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-2.gif)

정상적으로 소셜 로그인도 되고, 토큰방식의 로그인도 되는것을 볼 수 있다.

# token 제공 용도의 login page 생성하기

먼저 기존에 일반적인 로그인 폼이 아닌, 외부에서 제공 할 login form 을 만들자.

_oauth_login.html_

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0"
    />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <link rel="stylesheet" th:href="@{/css/main.css}" />
    <title>Document</title>
  </head>
  <body>
    <fieldset>
      <legend>제프의 로그인 시스템</legend>
      <form th:action="@{/api/login}" th:method="post">
        <input type="text" name="username" placeholder="아이디를 입력 해 주세요." />
        <br />
        <input type="password" name="password" placeholder="비밀번호를 입력 해 주세요." />
        <br />
        <button type="submit">로그인</button>
      </form>
    </fieldset>
  </body>
</html>
```

기존 폼에서, 문구만 제프의 로그인 이라는 문구로 변경하고 action 을 `/oauth/login` 으로 변경 하였다.

다음으로, AuthorizationServerSecurityConfig class 를 만들자.

_AuthorizationServerSecurityConfig.java_

```java
package ziponia.spring.security;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.UserDetailsService;

public class AuthorizationServerSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .requestMatchers()
                .mvcMatchers("/oauth/authorize/**", "/api/login")
                .and()
                .authorizeRequests()
                .antMatchers("/oauth/authorize/**").authenticated()
            .and()
                .formLogin()
                .loginPage("/api/login");
    }
}
```

이 클래스의 역할은, `grant type : authorization_code` 로 요청하면, `"User must be authenticated with Spring Security before authorization can be completed."`

에러가 뜨게 되는데, 기본적으로 시큐리티 컨텍스트로 인증이 된 후 요청하라는 의미 인 것 같다.

그것을 방지 하기 위한 역할이다. `/oauth/authorize/` 경로로 들어오면 로그인이 되어있지 않은경우, `/api/login` 으로 리다이렉트 시킨다.

로그인 방식은, 기존에 구현 해 둔, `UserDetailsService` 를 주입 해준다.

이제 RootSecurityConfig 에서, AuthorizationServerSecurityConfig 를 등록하자.

_RootSecurityConfig.java_

```java
@Configuration
@EnableOAuth2Client
@EnableWebSecurity
public class RootSecurityConfig {

    @Configuration
    public static class FormSecurityConfigAdapter extends WebSecurityConfig {}

    @Order(2)
    @Configuration
    public static class AuthorizationSecurityConfigAdapter extends AuthorizationServerSecurityConfig {}

    @Configuration
    @EnableResourceServer
    public static class ApiSecurityConfigAdapter extends ResourceServerConfig {}

}
```

이제 아래 URL로 접속 해서, 로그인 해보자.

```
http://localhost:8080/oauth/authorize
?client_id=client
&redirect_uri=http://localhost:4000/api/callback
&response_type=code
```

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-4.gif)

# grant_type code

code 방식의 로그인을 살펴 보자.

우리가 카카오 로그인을 하게 되면, 아마 이런 페이지가 처음에 떳을 것이다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-3.png)

조금씩 내용이 다를 수도있지만, 전체적으로 크게 다르지 않다.

이제 저런 화면을 만들어 볼 것이다.

> 이미 인증을 해서 로그인시에 위에 이미지가 안보인다면, 자신의 카카오톡의 설정 > 카카오계정 > 연결된 서비스 관리 > [그 외 카카오계정으로 로그인] 항목에 자신의 만든 앱 아이콘이 나올 것이다. 클릭 후 모든 정보 삭제 후 다시 로그인 하면된다.

다시 돌아가서, 브라우저에서 카카오 로그인을 누른 후 URL 을 보면 아래와 비슷한 URL 이 나올것이다.

```
https://kauth.kakao.com/oauth/authorize?client_id={CLIENT_ID}&redirect_uri=http://localhost:8080/login/kakao&response_type=code&state=5xDqcR
```

위에서 우리가 요청했던 URL 과 동일하게 생긴것을 볼 수 있다.

인가 페이지를 좀 더 꾸며보자.

인가 페이지를 변경하는 방법은, 그냥 Endpoint 를 오버라이딩 하면 된다고 한다.

[스택 오버플로우 Dave Syer 아저씨의 답변](https://stackoverflow.com/a/29435900)

AuthorizeEndpointController class 를 아래처럼 만들어 주자.

_AuthorizeEndpointController.java_

```java
package ziponia.spring.security;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.SessionAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.HashMap;
import java.util.Map;

@Controller
@SessionAttributes("authorizationRequest")
public class AuthorizeEndpointController {

    @SuppressWarnings("unchecked")
    @GetMapping(value = "/oauth/confirm_access")
    public String authorizeConfirmPage(HttpServletRequest request, Model model) {
        Map<String, Boolean> scopes = (HashMap<String, Boolean>) request.getAttribute("scopes");
        model.addAttribute("scopes", scopes);
        return "authorize_confirm";
    }
}

```

위 처럼 하면 scope 를 가져 올 수 있다.

이제 html 만 랜더링 해 주면 끝이다.

_authorize_confirm.html_

```html
<!DOCTYPE html>
<html lang="ko" xmlns:th="http://www.thymeleaf.org">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0"
    />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Document</title>
    <style>
      * {
        box-sizing: border-box;
      }
      html,
      body {
        margin: 0;
        padding: 0;
      }
      .wrap {
        width: 100vw;
        height: 100vh;
        background-color: #eeeeee;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
      }

      form {
        display: block;
        width: 600px;
        background-color: #ffffff;
        border: 1px solid #dddddd;
        padding: 20px;
      }
    </style>
  </head>
  <body>
    <div class="wrap">
      <h1>제프의 인증시스템</h1>
      <form action="#" th:action="@{/oauth/authorize}" th:object="${scopes}" th:method="post">
        <input type="hidden" name="user_oauth_approval" value="true" />
        <h4
          th:text="|${#request.getParameter('client_id')} 에서 회원님에 대한 아래와 같은 정보를 요청합니다|"
        ></h4>
        <div th:each="scope : ${scopes.keySet()}">
          <label th:for="${scope}" th:text="${scope}">scope.read</label>
          <input type="checkbox" th:id="${scope}" th:name="${scope}" th:value="true" />
        </div>
        <div>
          <button type="submit">동의하기</button>
        </div>
      </form>
    </div>
  </body>
</html>
```

이제 url 요청을 하고, 최종결과, parameter 로 code 가 들어온다면, 성공이다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-5.gif)

이제 받아 온 코드로 토큰 발급 요청을 해 보자.

```http
POST http://localhost:8080/oauth/token
[Header]
Content-Type: application/x-www-form-urlencoded

[Params]
grant_type=authorization_code
&client_id=clinet
&clinet_secret=secret
&redirect_uri=http://localhost:4000/api/callback
&code={코드 받기에서 발급받은 code 값}
```

사실 [카카오 로그인 하기](https://developers.kakao.com/docs/restapi/user-management#%EB%A1%9C%EA%B7%B8%EC%9D%B8)

에서 나오는 인증 flow 와 완전히 동일하다. (같은 oauth2 인증 플로우를 사용하니...)

이제 넘어 온 코드로 토큰을 받아보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-6.gif)

이제 ResourceServerConfig 에 /api/private 경로를 추가하고, 컨트롤러를 만들어 테스트 해보자.

_ResourceServerConfig.java_

```java
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {

        http
                .requestMatchers()
                .mvcMatchers("/api/**")
                .and()
                .authorizeRequests()
                .antMatchers("/api/login").permitAll()
                .antMatchers("/api/private").hasAnyRole("USER") // here
                .antMatchers("/api/**").hasAuthority("CLIENT")
                .and()
                .formLogin()
                .loginPage("/api/login")
                .and()
        ;
    }
}

```

_ApiController.java_

```java
@RestController
public class ApiController {

    // ...

    @GetMapping(value = "/api/private")
    public Map<String, String> privateApi() {
        Map<String, String> model = new HashMap<>();
        model.put("name", "jihoon");
        model.put("nick", "zef");
        return model;
    }

    // ...
```

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-20/oauth-7.gif)

꽤 멋진(?) 인증 시스템이 완료 되었다. 이제 외부로 client id 와 client secret 를 제공하여,

리소스에 접근 권한을 줄 수 있다.
