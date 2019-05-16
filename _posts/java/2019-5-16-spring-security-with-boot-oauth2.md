---
layout: post
title: "[SPRING] Spring Security 사용하기 #3"
---

social login 서비스를 구현해보자

이번에는 좀 재미있는것을 해보자.

카카오로그인, 페이스북 로그인 같은 소셜 로그인을 구현 할것이다.

# facebook 로그인 / github 로그인

사전작업으로 https://developers.facebook.com/ 에 가서 새로운 앱을 만든 후, client id 와 client secret 를 받아두자.

로그인을 할것이니, 로그인 설정을 하는것도 잊지말자.

먼저 의존성을 다운받자

_pom.xml_

```xml
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.1.4.RELEASE</version>
</dependency>
```

그 다음 Application 전역에 @EnableOAuth2Sso 어노테이션을 달아주자.

다음으로 application.yml 파일을 resources 아래 만들어서 다음과 같은 내용을 넣어주자

```
facebook:
  client:
    clientId: {my client id}
    clientSecret: {my client secret}
    accessTokenUri: https://graph.facebook.com/oauth/access_token
    userAuthorizationUri: https://www.facebook.com/dialog/oauth
    tokenName: oauth_token
    authenticationScheme: query
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://graph.facebook.com/me
```

my client id 와 my client secret 항목에서 facebook 에서 발급받은 client id 와 client secret 를 넣어주면 된다.

그리고 WebSecurityConfig 를 다음과 같이 수정한다.

```java
@Configuration
@EnableWebSecurity
@EnableOAuth2Client // new
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    //...

    @Autowired
    private OAuth2ClientContext auth2ClientContext;

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
            .logout()
            .deleteCookies("JSESSIONID")
            .clearAuthentication(true)
            .invalidateHttpSession(true)
        .and()
            .exceptionHandling()
                // .accessDeniedPage("/access_denied")
                .accessDeniedHandler(customAccessDeniedHandler)
        .and() // new
            .addFilterBefore(ssoFilter(), BasicAuthenticationFilter.class) // new
        ;
    }

    // new
    private Filter ssoFilter() {
        OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
        OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), auth2ClientContext);
        facebookFilter.setRestTemplate(facebookTemplate);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
        tokenServices.setRestTemplate(facebookTemplate);
        facebookFilter.setTokenServices(tokenServices);
        return facebookFilter;
    }

    // new
    @Bean
    @ConfigurationProperties("facebook.client")
    public AuthorizationCodeResourceDetails facebook() {
        return new AuthorizationCodeResourceDetails();
    }

    // new
    @Bean
    @ConfigurationProperties("facebook.resource")
    public ResourceServerProperties facebookResource() {
        return new ResourceServerProperties();
    }

    // new
    @Bean
    public OAuth2ClientContext auth2ClientContext() {
        return new DefaultOAuth2ClientContext();
    }

    // new
    @Bean
    public FilterRegistrationBean<OAuth2ClientContextFilter> oauth2ClientFilterRegistration(OAuth2ClientContextFilter filter) {
        FilterRegistrationBean<OAuth2ClientContextFilter> registration = new FilterRegistrationBean<OAuth2ClientContextFilter>();
        registration.setFilter(filter);
        registration.setOrder(-100);
        return registration;
    }
}
```

new 라고 주석처리 한 항목을 수정 하면 된다.

이제 login.html 에 다음과 같이 넣어주겠다.

_login.html_

```html
<fieldset>
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
    <!-- 이부분을 추가하자. -->
  </div>
</fieldset>
```

그리고 현재 유저 정보를 볼 수 있도록, /private 에 현재 유저 id 를 가지고 올 수 있도록 하자

_SecurityController.java_

```java
@Controller
public class SecurityController {

    @GetMapping(value = "/private")
    public String privatePage(Principal principal, Model model) {
        model.addAttribute("user", principal);
        return "private";
    }
}
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
    <p th:text="${user.name}"></p>
  </th:block>
</html>
```

그럼 비공개 페이지 밑에 자기 자신의 아이디가 뜨는 것을 알 수 있다.

# Github 로그인

이제 깃헙 로그인을 추가 해보자. 당연히 이번에도 github 에서 oauth2 어플리케이션을 추가해야한다.

_login.html_

```html
<fieldset>
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
</fieldset>
```

_application.yml_

```
github:
  client:
    clientId: {my client id}
    clientSecret: {my client secret}
    accessTokenUri: https://github.com/login/oauth/access_token
    userAuthorizationUri: https://github.com/login/oauth/authorize
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://api.github.com/user
```

_WebSecurityConfig.java_

```java
@Configuration
@EnableWebSecurity
@EnableOAuth2Client
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...
    private Filter ssoFilter() {
        CompositeFilter filter = new CompositeFilter();
        List<Filter> filters = new ArrayList<>();

        OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
        OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), auth2ClientContext);
        facebookFilter.setRestTemplate(facebookTemplate);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
        tokenServices.setRestTemplate(facebookTemplate);
        facebookFilter.setTokenServices(tokenServices);
        filters.add(facebookFilter);

        OAuth2ClientAuthenticationProcessingFilter githubFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/github");
        OAuth2RestTemplate githubTemplate = new OAuth2RestTemplate(github(), auth2ClientContext);
        githubFilter.setRestTemplate(githubTemplate);
        tokenServices = new UserInfoTokenServices(githubResource().getUserInfoUri(), github().getClientId());
        tokenServices.setRestTemplate(githubTemplate);
        githubFilter.setTokenServices(tokenServices);
        filters.add(githubFilter);

        filter.setFilters(filters);
        return filter;
    }

    @Bean
    @ConfigurationProperties("github.client")
    public AuthorizationCodeResourceDetails github() {
        return new AuthorizationCodeResourceDetails();
    }

    @Bean
    @ConfigurationProperties("github.resource")
    public ResourceServerProperties githubResource() {
        return new ResourceServerProperties();
    }

    //...
}

```

ssoFilter 쪽만 변경 하면 된다.

이전에 facebook 로그인을 구현해놔서 복잡하진 않을것이다.

facebook 과 github 쪽에 filter 가 비슷하게 생겼다. 메뉴얼에 따라 리펙토링을 해보자.

먼저 ClientResources 를 만든다.

```java
class ClientResources {

    @NestedConfigurationProperty
    private AuthorizationCodeResourceDetails client = new AuthorizationCodeResourceDetails();

    @NestedConfigurationProperty
    private ResourceServerProperties resource = new ResourceServerProperties();

    public AuthorizationCodeResourceDetails getClient() {
        return client;
    }

    public ResourceServerProperties getResource() {
        return resource;
    }
}
```

다음으로, WebSecurityConfig 쪽을 수정하자

_WebsecurityConfig.java_

```java
@Configuration
@EnableWebSecurity
@EnableOAuth2Client
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private Filter ssoFilter() {
        CompositeFilter filter = new CompositeFilter();
        List<Filter> filters = new ArrayList<>();
        filters.add(ssoFilter(facebook(), "/login/facebook"));
        filters.add(ssoFilter(github(), "/login/github"));
        filter.setFilters(filters);
        return filter;
    }

    private Filter ssoFilter(ClientResources client, String path) {
        OAuth2ClientAuthenticationProcessingFilter filter = new OAuth2ClientAuthenticationProcessingFilter(path);
        OAuth2RestTemplate template = new OAuth2RestTemplate(client.getClient(), auth2ClientContext);
        filter.setRestTemplate(template);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(
                client.getResource().getUserInfoUri(), client.getClient().getClientId());
        tokenServices.setRestTemplate(template);
        filter.setTokenServices(tokenServices);
        return filter;
    }

    @Bean
    @ConfigurationProperties("github")
    public ClientResources github() {
        return new ClientResources();
    }

    @Bean
    @ConfigurationProperties("facebook")
    public ClientResources facebook() {
        return new ClientResources();
    }
}

```

이 예제는 [https://spring.io/guides/tutorials/spring-boot-oauth2/#\_social_login_github](https://spring.io/guides/tutorials/spring-boot-oauth2/#_social_login_github) 예제 이다.

이제 페이스북 로그인 / 깃허브 로그인을 번갈아가며 해보자.

# 카카오 로그인 추가하기

예제만 따라해서는 의미가 없다. 응용해서 카카오 로그인을 구현해보자.

[카카오 개발자센터](https://developers.kakao.com/) 에서 키 발급받는거 있지 말자

이제 차근차근 응용해보자.

먼저 WebSecurityConfig 를 바꿔보자

```java
@Configuration
@EnableWebSecurity
@EnableOAuth2Client
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private Filter ssoFilter() {
        CompositeFilter filter = new CompositeFilter();
        List<Filter> filters = new ArrayList<>();
        filters.add(ssoFilter(facebook(), "/login/facebook"));
        filters.add(ssoFilter(github(), "/login/github"));
        filters.add(ssoFilter(kakao(), "/login/kakao")); // new
        filter.setFilters(filters);
        return filter;
    }

    // new
    @Bean
    @ConfigurationProperties("kakao")
    public ClientResources kakao() {
        return new ClientResources();
    }
}
```

다음으로 application.yml 에 카카오 정보를 입력 해 보자.

```
kakao:
  client:
    clientId: {kakao rest api key}
    clientSecret: {kakao secret key}
    accessTokenUri: https://kauth.kakao.com/oauth/token
    userAuthorizationUri: https://kauth.kakao.com/oauth/authorize
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://kapi.kakao.com/v2/user/me
```

각프로퍼티들은 문서에 있는 내용들이다.

이제 로그인 페이지에 카카오 로그인을 추가하자

_login.html_

```html
<fieldset>
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

너무 쉽게 끝났다. security 는 정말 놀랍지 않은가. 오늘은 여기까지

현재까지의 소스는 [https://github.com/ziponia/spring-security-example](https://github.com/ziponia/spring-security-example) tag v2 로 따라가자
