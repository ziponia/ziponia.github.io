---
layout: post
title: "[SPRING] Spring Security 사용하기 #4"
summary: "Spring boot oauth2 에 Database 를 연결 하자"
tags: [spring, spring-boot, spring-security]
---

이번에는 지난 시간에 한 oauth2 로그인에 이어, Database 와 연동을 해보겠다.

이번에 구현 할 것은, 1 계정 - 1소셜 이다

무슨 말이냐 하면, 유저 1 : N 소셜 이 아니라, 소셜 계정이 즉 유저계정이 된다는 것이다.

따라서, User 가 소셜 일 수도있고, 아닐수도 있다는 이야기이다.

먼저, 소셜 타입을 정의하자.

```java
package ziponia.spring.security;

public enum SocialProvider {
    KAKAO,
    FACEBOOK,
    GITHUB
}

```

다음으로 User 테이블에 SocialProvider 를 추가하자

```java
package ziponia.spring.security;

import lombok.Getter;
import lombok.Setter;

import javax.persistence.*;

@Entity
@Table(name = "tbl_users")
@Getter
@Setter
public class UserEntity {

    @Id
    @GeneratedValue
    private Integer idx;

    private String username;

    private String password;

    @Enumerated(EnumType.STRING)
    private SocialProvider sns; // new
}

```

다음으로, 권한 및 인증 추출기를 만들겠다.

프로젝트가 조금 복잡해지니, social 이라는 패키지를 만들고 그 안에 facebook / github / kakao 라는 패키지를 만든다. 그 다음 social 패키지 아래에 BasePrincipalExtractor 라는 추상화 객체를 만들겠다.

쿼리를 중복해서 써야 하는 부분을 제거하기 위함이다.

그럼 최종적으로

BasePrincipalExtractor 라는 추상화 객체가 있고, BasePrincipalExtractor 는 PrincipalExtractor 를 구현한 후

FacebookPrincipalExtractor, GithubPrincipalExtractor, KakaoPrincipalExtractor 클래스들이 BasePrincipalExtractor 를 상속하는 것이다.

```java
public abstract class BasePrincipalExtractor implements PrincipalExtractor {
}

```

```java
@Component
public class FacebookPrincipalExtractor extends BasePrincipalExtractor {

    @Override
    public Object extractPrincipal(Map<String, Object> map) {
        return null;
    }
}

```

```java
@Component
public class KakaoPrincipalExtractor extends BasePrincipalExtractor {

    @Override
    public Object extractPrincipal(Map<String, Object> map) {
        return null;
    }
}

```

```java
@Component
public class GithubPrincipalExtractor extends BasePrincipalExtractor {

    @Override
    public Object extractPrincipal(Map<String, Object> map) {
        return null;
    }
}

```

그리고, Database 로 접근 할 수 있는, JpaRepository를 하나 만들자.

> 다시한번 얘기하지만, JPA 는 지금 언급하지 않겠다.

_UserRepository.java_

```java
public interface UserRepository extends JpaRepository<UserEntity, Integer> {
}
```

이제 BasePrincipalExtractor 에서 UserRepository 를 주입시켜준 후, Social 계정을 받아 처리 할 수 있는, 쿼리를 만들고,

최종적으로 Principal 을 대신해서 생성 해 주는 부분 까지 생성하자.

```java
public abstract class BasePrincipalExtractor implements PrincipalExtractor {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    private static final String[] PRINCIPAL_KEYS = new String[] { "user", "username",
            "userid", "user_id", "login", "id", "name" };

    @Transactional
    public void saveSocialUser(String id, SocialProvider provider) {
        UserEntity userEntity = userRepository.findByUsernameAndSns(id, provider);
        if (userEntity == null) {
            userEntity = new UserEntity();
            userEntity.setUsername(id);
            userEntity.setSns(provider);
            userEntity.setPassword(passwordEncoder.encode(UUID.randomUUID().toString()));
        }

        userRepository.save(userEntity);
    }

    protected Object createPrincipal(Map<String, Object> map) {
        for (String key : PRINCIPAL_KEYS) {
            if (map.containsKey(key)) {
                return map.get(key);
            }
        }
        return null;
    }
}
```

이제 각각, Facebook, github, kakao PrincipalExtractor 에서 saveSocialUser 메서드를 불러와 사용하자.

반복코드니, 카카오 예시 하나만 보겠다.

```java
@Component
public class KakaoPrincipalExtractor extends BasePrincipalExtractor {

    @Override
    public Object extractPrincipal(Map<String, Object> map) {
        String id = map.get("id").toString();
        this.saveSocialUser(id, SocialProvider.KAKAO); // 이부분만 각각 맞는걸로 교체 해 주면 된다.
        return this.createPrincipal(map);
    }
}
```

`extractPrincipal` method 에서 각각 소셜로그인 후 가지고 온 정보를 가지고, DB 로 등록 해주면 된다.

마지막으로, WebSecurityConfig 를 살짝 건드려 주면 된다.

_WebSecurityConfig.java_

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private FacebookPrincipalExtractor facebookPrincipalExtractor;

    @Autowired
    private KakaoPrincipalExtractor kakaoPrincipalExtractor;

    @Autowired
    private GithubPrincipalExtractor githubPrincipalExtractor;

    // ...
    private Filter ssoFilter() {
        CompositeFilter filter = new CompositeFilter();
        List<Filter> filters = new ArrayList<>();
        filters.add(ssoFilter(facebook(), "/login/facebook", SocialProvider.FACEBOOK));
        filters.add(ssoFilter(github(), "/login/github", SocialProvider.GITHUB));
        filters.add(ssoFilter(kakao(), "/login/kakao", SocialProvider.KAKAO));
        filter.setFilters(filters);
        return filter;
    }

    private Filter ssoFilter(ClientResources client, String path, SocialProvider provider) {
        OAuth2ClientAuthenticationProcessingFilter filter = new OAuth2ClientAuthenticationProcessingFilter(path);
        OAuth2RestTemplate template = new OAuth2RestTemplate(client.getClient(), auth2ClientContext);
        filter.setRestTemplate(template);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(
                client.getResource().getUserInfoUri(), client.getClient().getClientId());
        tokenServices.setRestTemplate(template);
        if (provider.equals(SocialProvider.FACEBOOK)) {
            tokenServices.setPrincipalExtractor(facebookPrincipalExtractor);
        }

        if (provider.equals(SocialProvider.GITHUB)) {
            tokenServices.setPrincipalExtractor(githubPrincipalExtractor);
        }

        if (provider.equals(SocialProvider.KAKAO)) {
            tokenServices.setPrincipalExtractor(kakaoPrincipalExtractor);
        }
        filter.setTokenServices(tokenServices);
        return filter;
    }

    // ...
}
```

이제 각각 로그인을 하면서, 확인 해 보면 새로운 레코드가 추가 되는걸 볼 수 있다.

_최초_

![최초이미지](/images/2019-5-17/security_oauth_default_record.png)

_페이스북 로그인 후_

![페이스북추가](/images/2019-5-17/security_oauth_default_facebook.png)

_깃허브 로그인 후_

![깃허브추가](/images/2019-5-17/security_oauth_default_github.png)

카카오 로그인은 직접 해보자

소스는

[https://github.com/ziponia/spring-security-example](https://github.com/ziponia/spring-security-example)

tag v3 로 가자
