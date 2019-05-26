---
layout: post
title: "[SPRING] Spring Security 사용하기 #11"
summary: "AuthenticationManager 를 이용하여 로그인 프로세스를 커스터마이징 해보자."
tags: [spring, spring-boot, spring-security]
---

AuthenticationManager 를 사용하여, 원하는 시점에 로그인을 해보자.

지금은 항상 어떤 로그인이든지, `POST /login` 경로에서만 로그인이 되었었다.

AuthenticationManager 를 이용하여, 원하는 시점에 로그인이 될 수 있도록 바꿔보자.

먼저, AuthenticationManager 를 외부에서 사용 하기 위해, AuthenticationManagerBean 을 이용하여 Sprint Securtiy 밖으로 AuthenticationManager 빼 내야 한다.

_WebSecurityConfig.java_

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ...
        http
            .requestMatchers()
            .mvcMatchers("/login/**", "/logout/**", "/private/**", "/admin/**", "/", "/profile/**", "/my-login") // /my-login 을 시큐리티 관리포인트에 추가하였다.
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

이제 AuthenticationManager 를 밖에서 사용 할 수 있다.

> 이렇게 하지 않으면 AuthenticationManager 를 injection 할 수 없다. 위 메서드를 주석 처리하고, 컨트롤러에서 AutehtnciationManager 를 @Autowired 하면 컴파일 에러가 난다.

이제 SecurityController 에서 AuthenticationManager 를 inject 하고 /my-login 이라는 경로를 만들자.

_SecurityController.java_

```java
@RequiredArgsConstructor // add lombok inject
public class SecurityController {

    private final AuthenticationManager authenticationManager; // @Autowired

    // ...

    @PostMapping(value = "/my-login")
    public String customLoginProcess(
            @RequestParam String username,
            @RequestParam String password
    ) {

        // 아이디와 패스워드로, Security 가 알아 볼 수 있는 token 객체로 변경한다.
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(username, password);

        // AuthenticationManager 에 token 을 넘기면 UserDetailsService 가 받아 처리하도록 한다.
        Authentication authentication = authenticationManager.authenticate(token);

        // 실제 SecurityContext 에 authentication 정보를 등록한다.
        SecurityContextHolder.getContext().setAuthentication(authentication);

        // 그외
        return "redirect:/";
    }
}
```

이제 로그인 페이지에서, /login 으로 form 전송을 보내었던 url 을 /my-login 으로 바꿔보자.

_templates/login.html_

```html
<form th:action="@{/my-login}" th:method="post">
  <!-- form.... -->
</form>
```

이제 서버를 부팅 후 loginfrom 에서 로그인 을 한 후, 개인페이지로 들어가면 접근 이 잘 되는 것을 알 수 있다.

다음으로, 정상적으로, 로그인 처리가 되지 않았을 때, Exception 처리를 해보자.

AuthenticationManager 의 authenticate 은 UserDetails 컨텍스트가 내부적으로 잘못 되었다고 판단 되었을 경우,

AuthenticationException 을 throw 하도록 되어있고, AuthenticationException 은 다음 주요 Exception 구현체를 가지고 있다.

- DisabledException
- LockedException
- BadCredentialsException
- UsernameNotFoundException

그리고 이중에 UserDetailsServiceImpl throw 했던 UsernameNotFoundException 은 BadCredentialsException 을 호출 하도록 되어있다.

```
- UsernameNotFoundException
    - DaoAuthenticationProvider
        - AbstractUserDetailsAuthenticationProvider
            - BadCredentialsException
```

순으로 들어가게 되어 기본적으로 BadCredentialsException 을 호출 하게 되는것이다.

이제 코딩을 해보자.

_SecurityController.java_

```java
@RequiredArgsConstructor // add lombok inject
public class SecurityController {

    private final AuthenticationManager authenticationManager; // @Autowired

    // ...

    @PostMapping(value = "/my-login")
    public String customLoginProcess(
            @RequestParam String username,
            @RequestParam String password
    ) {
       // 아이디와 패스워드로, Security 가 알아 볼 수 있는 token 객체로 변경한다.
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(username, password);
        try {
            // AuthenticationManager 에 token 을 넘기면 UserDetailsService 가 받아 처리하도록 한다.
            Authentication authentication = authenticationManager.authenticate(token);
            // 실제 SecurityContext 에 authentication 정보를 등록한다.
            SecurityContextHolder.getContext().setAuthentication(authentication);
        } catch (DisabledException | LockedException | BadCredentialsException e) {
            String status;
            if (e.getClass().equals(BadCredentialsException.class)) {
                status = "invalid-password";
            } else if (e.getClass().equals(DisabledException.class)) {
                status = "locked";
            } else if (e.getClass().equals(LockedException.class)) {
                status = "disable";
            } else {
                status = "unknown";
            }
            return "redirect:/login?flag=" + status;
        }
        return "redirect:/";
    }
}
```

각각 throw 된 Exceptino 을 분류 해서, status 파라미터를 나누도록 하였다.

이렇게, authenticationManager 를 이용하여 로그인 프로세스를 커스터마이징 해 보았다.
