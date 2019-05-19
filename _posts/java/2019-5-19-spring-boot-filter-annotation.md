---
layout: post
title: "[SPRING] @Filter"
summary: "@Filter"
tags: [spring, spring-boot]
---

spring boot 에서 어노테이션으로 filter 를 지정할 수 있다.

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletComponentScan;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.web.bind.annotation.CrossOrigin;

@SpringBootApplication

/**
 * @WebFilter 를 사용하기 위한 어노테이션
 */
@ServletComponentScan

@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
@CrossOrigin
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

```java
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebFilter(urlPatterns = {"/api/*"}, description = "인증 필터")
@Slf4j
public class ApiFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        log.info("filter => API Token Filter");
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;
        log.info("req header => {}", request.getHeader("x-auth-token"));
        if ( request.getHeader("x-auth-token") == null ) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "인증오류");
        }
        chain.doFilter(req, res);
    }

    @Override
    public void destroy() {

    }
}

```

이렇게 하면 URL 패턴 별로 Filter 가 들어간다.
