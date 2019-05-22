---
layout: post
title: "[SPRING] Spring Security 사용하기 #7"
summary: "Client DB 로 관리하기"
tags: [spring, spring-boot, spring-security]
---

[지난포스팅](https://ziponia.github.io/2019/05/20/spring-boot-oauth2-authorization-server/) 에서는 Client 를 메모리에 생성하고, access token 을 받아, api 로 접근하는 부분까지 해보았다.

이렇게 클라이언트를 memory 로 관리 할 수도있지만, 사실 이렇게 하면, 새로운 client 가 추가 될 때 마다, 서버를 다시 배포 해 주어야 한다.

오늘은 client 와 DB 를 연결하여, 관리자가 직접 client 를 추가하고, 관리하는 부분까지 해보자.

# client db 생성

먼저 클라이언트 DB 를 생성 하도록 해보자.

이전에 WebSecurity 의 UserDetails 를 보면,

`UserDetails` 가 Security 에 등록 될 Context 객체가 되고, `UserDetailsService` 가 `UserDetails` 를 생성하기 위한 부분을 처리 하였다.

client 는 `User` 를 `Client` 로 변경 해주기만 하면된다.

먼저 ClientDetails 를 구현 해 보자.

특정 user 가 다수의 client 를 가질 수 있다는 것을 염두하고 하자.

한마디로, user : client 는 1 : N 관계이다.

먼저 테이블을 만들자.

_ClientEntity.java_

```java
package ziponia.spring.security;

import lombok.Getter;
import lombok.Setter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.AuthorityUtils;
import org.springframework.security.oauth2.provider.ClientDetails;

import javax.persistence.*;
import java.util.*;

@Entity
@Table(name = "tbl_oauth_client")
@Getter
@Setter
public class ClientEntity implements ClientDetails {

    @Id
    @GeneratedValue
    private Integer idx;

    @ManyToOne
    private UserEntity userEntity;

    private String clientId;
    private String resourceIds;
    private String clientSecret;
    private String scope;
    private String grantTypes;
    private String redirectUri;
    private String authorities;
    private Integer accessTokenValiditySeconds;
    private Integer refreshTokenValiditySeconds;
    private Boolean autoApprove;

    @Override
    public String getClientId() {
        return clientId;
    }

    @Override
    public Set<String> getResourceIds() {
        if (resourceIds == null) return null;
        String[] s = resourceIds.split(",");
        return new HashSet<>(Arrays.asList(s));
    }

    @Override
    public boolean isSecretRequired() {
        return clientSecret != null;
    }

    @Override
    public String getClientSecret() {
        return clientSecret;
    }

    @Override
    public boolean isScoped() {
        return scope != null;
    }

    @Override
    public Set<String> getScope() {
        if (scope == null) return null;
        String[] s = scope.split(",");
        return new HashSet<>(Arrays.asList(s));
    }

    @Override
    public Set<String> getAuthorizedGrantTypes() {
        if (grantTypes == null) return null;
        String[] s = grantTypes.split(",");
        return new HashSet<>(Arrays.asList(s));
    }

    @Override
    public Set<String> getRegisteredRedirectUri() {
        if (redirectUri == null) return null;
        String[] s = redirectUri.split(",");
        return new HashSet<>(Arrays.asList(s));
    }

    @Override
    public Collection<GrantedAuthority> getAuthorities() {
        if (authorities == null) return new ArrayList<>();
        return AuthorityUtils.createAuthorityList(authorities.split(","));
    }

    @Override
    public Integer getAccessTokenValiditySeconds() {
        return accessTokenValiditySeconds;
    }

    @Override
    public Integer getRefreshTokenValiditySeconds() {
        return refreshTokenValiditySeconds;
    }

    @Override
    public boolean isAutoApprove(String scope) {
        return autoApprove;
    }

    @Override
    public Map<String, Object> getAdditionalInformation() {
        return null;
    }
}

```

`ClientDetails` 구현체로 만들고, userEntity 와 하이버네이트 등록을 위해 `@Id` 까지 넣어주었다.

다음으로 client 를 DB 에서 조회 하기 위해, JPA Repository 를 만든다.

_OAuth2ClientRepository.java_

```java
package ziponia.spring.security;

import org.springframework.data.jpa.repository.JpaRepository;

public interface OAuth2ClientRepository extends JpaRepository<ClientEntity, Integer> {

    ClientEntity findByClientId(String clientId);
}
```

이어서, ClientDetailsService 를 구현하자, UserDetailsService 와 비슷한 개념으로 생각하면 된다.

_ClientDetailsServiceImpl.java_

```java
package ziponia.spring.security;

import lombok.extern.java.Log;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.oauth2.provider.ClientDetails;
import org.springframework.security.oauth2.provider.ClientDetailsService;
import org.springframework.security.oauth2.provider.ClientRegistrationException;
import org.springframework.security.oauth2.provider.client.BaseClientDetails;
import org.springframework.transaction.annotation.Transactional;

public class ClientDetailsServiceImpl implements ClientDetailsService {

    @Autowired
    private OAuth2ClientRepository oAuth2ClientRepository;

    @Override
    @Transactional
    public ClientDetails loadClientByClientId(String clientId) throws ClientRegistrationException {
        ClientEntity clientEntity = oAuth2ClientRepository.findByClientId(clientId);
        return new BaseClientDetails(clientEntity);
    }
}
```

마지막으로, `AuthorizationServerConfig` 에서 기존에 clients.inMemory()..... 부분을 다음처럼 변경 해주자.

_AuthorizationServerConfig.java_

```java
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    // ...other

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        /*clients
                .inMemory()
                .withClient("client")
                .secret(passwordEncoder.encode("secret"))
                .authorizedGrantTypes("client_credentials", "authorization_code", "refresh_token", "password")
                .authorities("CLIENT")
                .scopes("read", "basic", "profile")
                .redirectUris("http://localhost:4000/api/callback")
                .autoApprove(false);*/

        clients
                .withClientDetails(clientDetailsService());
    }
    // ...other
}
```

모든 설정이 끝났다.

이제 import.sql 에다 임의로, 레코드를 추가하여, 애플리케이션을 부팅 후 테스트 해보자.

_import.sql_

```java
insert into tbl_users (idx, username, password) values (1, 'user', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK');
insert into tbl_users (idx, username, password) values (2, 'admin', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK');
insert into tbl_oauth_client (idx, authorities, auto_approve, client_id, client_secret, grant_types, redirect_uri, scope, user_entity_idx) values (1, 'CLIENT', false, 'client', '{bcrypt}$2a$10$cxEU57mmmEm9FfhAJBMW7ec9oG4Y5Uq4Os8CfpxoL6TLzxPCCqzXK', 'client_credentials,authorization_code,refresh_token,password', 'http://localhost:4000/api/callback', 'read,basic,profile', 1);
```

정상적으로 토큰을 받아오는것을 볼 수 있다.

화면은 이전 포스팅 한 것과 동일하니, 참조하자.
