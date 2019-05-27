---
layout: post
title: "[SPRING] Spring Security 사용하기 #12"
summary: "loadClientByClientId Method Call Stack hit 4"
tags: [spring, spring-boot, spring-security, jpa]
---

[지난 oauth2 client DB 로 관리하기](/2019/05/22/spring-boot-oauth2-client-with-db/) 에서 우리는

ClientDetailsService 인터페이스를 ClientServiceImpl 로 구현하고 loadClientByClientId 를 오버라이드 하여, 제공했었다.

사실 이부분은 매우 큰 결점이 있는데, Security 포스팅과 맞지 않는다고 생각하여, 그냥 넘어가려고 했지만...

다시 생각해 보니 언급하고 넘어가는게 좋을 것 같아서 진행 하려고 한다.

loadByClientByClientId 에서 clientEntity 라인 밑에 콘솔을 찍어보자. (디버깅을 돌려본다면 더 좋다.)

아래 영상을 보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-27/spring-oauth2-client-overhit.gif)

이상한 점이 보이는가?

단한번 토큰 발행 요청에 무료 11번 오버히트가 났다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-27/oauth2-overhit-image1.png)

사실 이건, 같은 행동만 여러번 반복되는것이 아니고,

Securit 내부에서 Scope 를 만들고...인증정보만들고...client 의 각항목을 체크하고.. 권한만들고.. 하면서 나오는 현상인데

사실 우리에게는 매우 불필요하고, SQL 에게도 미안한 행동이다.

이부분을 개선시켜보자.

# @Cacheable

사실 앞에서 거창하게 이야기 했지만 복잡한게 아니고 어노테이션 두개면 끝난다.

먼저 maven 에 `spring-boot-starter-cache` 를 추가 해 주자.

```xml
<dependencies>
    <!-- more... -->

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <!-- more.. -->
</dependencies>
```

다음 메인 클래스에서 `@EnableCaching` 를 선언 해주자.

```java
// ...
@EnableCaching
@SpringBootApplication
public class SecurityApplication {
    //...
}
```

ClientDetailsServiceImpl 의 loadByClientByClientId 에다 캐시 처리를 해주면 된다.

_ClientDetailsServiceImpl.java_

```java
public class ClientDetailsServiceImpl implements ClientDetailsService {

    // ...

    @Cacheable("oauth2-load-client") // new
    public ClientDetails loadClientByClientId(String clientId) throws ClientRegistrationException {
        // ...
    }
}
```

이제 테스트를 해보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-27/oauth2-overhit-final.gif)

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-27/oauth2-ovehit-image-final.png)

이게 끝이다!!! :) happy~
