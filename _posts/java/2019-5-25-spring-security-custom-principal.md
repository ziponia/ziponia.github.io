---
layout: post
title: "[SPRING] Spring Security 사용하기 #10"
summary: "Spring Security Principal Custom"
tags: [spring, spring-boot, spring-security]
---

이번에는 Security Principal 객체를 커스터마이징 해보자.

기존에 로그인 하게 되면 기본적으로 spring security context

package org.springframework.security.core.userdetails.UserDetails; 를 구현하는것이며,

<a href="/2019/05/15/spring-security-with-boot/" target="_blank">가장 첫 포스트에서 만든</a> security context 객체 는

Security 의 UserDetails 를 사용하기 쉽게 미리 만들어 둔, UserDetails 의 구현 객체인 User `org.springframework.security.core.userdetails.User` 객체이다.

이 User 객체를 살펴보게 되면 다음과 같이 생성자가 두개 미리 생성되어있다.

```java
public class User implements UserDetails, CredentialsContainer {
    public User(String username, String password,
			Collection<? extends GrantedAuthority> authorities) {
		// ...
	}

    public User(String username, String password, boolean enabled,
			boolean accountNonExpired, boolean credentialsNonExpired,
			boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {

		// ...
	}
}
```

하나는 단순하게 아이디, 비밀번호, 권한을 생성하는 컨스트럭터이고,

다른 하나는 아이디, 비밀번호, 사용가능여부, 만료여부, 비밀번호가 만려되었는지, 잠겨있는지, 권한 을 생성하는 객체이다.

아이디, 비밀번호, 권한 을 제외하고는 설정하지 않으면 기본적으로 true 이다.

그런데, 왼지 컨텍스트에 위에 정보들만 가지고 다니기 좀 꺼림칙 하다.

예를들면, 우리의 서비스는 임대형이고, 각 페이지를 돌아다니면서 각각 사용자의 맞는 리소스를 가지고 오려면 위의 컨텍스트 객체의 username 을 가지고 온 다음,

DB 에서 username 에 대한 유저 객체를 조회하고, 조회한 유저 객체가 속한 그룹 정보를 가지고 와야 한다.

> 더 좋은 방법도 있겠지만, 설득을 위해 조금 억지를 부려 보았다. 납득하자.

이걸 security context 에서 바로 해당 유저의 그룹 정보가 '미리' 담겨 있다면, 왼지 더 심플 해 질 것 같다.

예시를 보자.

먼저 사용자가 속한 그룹을 만들자.

_GroupEntity.java_

```java
package ziponia.spring.security;

import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import javax.persistence.*;
import java.util.Collection;
import java.util.Date;

@Entity
@Getter
@Setter
@Table(name = "tbl_group")
public class GroupEntity {

    @Id
    @GeneratedValue
    private Integer idx;

    private String groupName;

    @OneToMany(mappedBy = "group")
    private Collection<UserEntity> users;

    @CreationTimestamp
    private Date createTime;

    @UpdateTimestamp
    private Date updateTime;
}

```

그리고 tbl_user 에 group 을 추가하자.

_UserEntity.java_

```java
public class UserEntity {

    // ...

    @ManyToOne(fetch = FetchType.LAZY)
    private GroupEntity group;
}
```

GroupEntity 와 UserEntity 는 1:n 양방향 관계로 설정하고, LAZY 로딩을 채택 하였다.

이제 import.sql 구문에 기본 group 을 만들고, user 테이블에 해당 그룹을 설정하겠다.

_import.sql_

```java
insert into tbl_group (idx, group_name, create_time) values (1, '제프컴퍼니', current_timestamp());
insert into tbl_users (idx, username, password, nick_name, group_idx) values (1, 'user', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK', '제프', 1);
insert into tbl_users (idx, username, password, nick_name, group_idx) values (2, 'admin', '{bcrypt}$2a$10$Wyc.IrbO.bqraF58565Yde6J6heWdARvbDUKfaQYr9v/IoHcQ1RlK', '블로그 관리자', 1);
insert into tbl_oauth_client (idx, authorities, auto_approve, client_id, client_secret, grant_types, redirect_uri, scope, user_entity_idx) values (1, 'CLIENT', false, 'client', '{bcrypt}$2a$10$cxEU57mmmEm9FfhAJBMW7ec9oG4Y5Uq4Os8CfpxoL6TLzxPCCqzXK', 'client_credentials,authorization_code,refresh_token,password', 'http://localhost:4000/api/callback', 'read,basic,profile', 1);
```

tbl_group 에 1번 레코드에 '제프컴퍼니' 라는 그룹을 추가하고, user 계정 group 에 제프컴퍼니를 추가 해 주었다.

> 스크립트 상으로, tbl_group 이 tbl_user 보다 먼저 올라와야 한다.

이제 UserController 에서 유저 그룹 정보를 가지고 오는 `GET /profile/group` 를 만들고, 그룹정보를 가지고 오자.

_UserController.java_

```java
public class UserController {

    //...

    @PersistenceContext
    private EntityManager em; // JPA

    @GetMapping(value = "/profile/group")
    @PreAuthorize("isAuthenticated()")
    public String myGroup(
            @AuthenticationPrincipal Authentication authentication,
            Model model
    ) {
        // user 정보를 가지고 온다.
        UserEntity user = em.createQuery("select u from UserEntity u where u.username = :username", UserEntity.class)
                .setParameter("username", authentication.getName()).getSingleResult();

        // group 정보를 가지고 온다.
        GroupEntity group = em.createQuery("select g from GroupEntity g where g.idx = :idx", GroupEntity.class)
                .setParameter("idx", user.getGroup().getIdx()).getSingleResult();

        model.addAttribute("group", group);

        return "user_group";
    }
}
```

이제 user_group 페이지를 만들고 네비게이션 메뉴에 추가하자.

_templates/user_group.html_

```html
<template
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  layout:decorator="layout/layout"
  th:remove="tag"
>
  <th:block layout:fragment="contents">
    <h1>제 그룹은 [[ ${group.groupName} ]] 입니다.</h1>
  </th:block>
</template>
```

_templates/layout/navigation.html_

```html
<template
  xmlns:th="http://www.thymeleaf.org"
  xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
  th:remove="tag"
>
  <th:block layout:fragment="navigation">
    <nav class="navbar navbar-default">
      <ul class="nav navbar-nav">
        <li><a th:href="@{/}">홈</a></li>
        <li><a th:href="@{/admin}">관리자 페이지</a></li>
        <li><a th:href="@{/private}">개인 페이지</a></li>
        <li><a th:href="@{/profile}">유저 검색 페이지</a></li>
        <!-- new -->
        <li><a th:href="@{/profile/group}">내 그룹 관리</a></li>
        <li>
          <a href="javascript: void(0);" onclick="document.getElementById('logout-form').submit()">
            로그아웃
          </a>
        </li>
      </ul>
    </nav>
    <form id="logout-form" th:action="@{/logout}" method="post"></form>
  </th:block>
</template>
```

이제 [http://localhost:8080/profile/group](http://localhost:8080/profile/group) 로 들어 가 보면 아래 페이지를 볼 수 있을 것이다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-26/spring-security-principal-default.png)

근데 문제가 있다.

컨트롤러를 살펴 보면

```java
// user 정보를 가지고 온다.
UserEntity user = em.createQuery("select u from UserEntity u where u.username = :username", UserEntity.class)
        .setParameter("username", authentication.getName()).getSingleResult();

// group 정보를 가지고 온다.
GroupEntity group = em.createQuery("select g from GroupEntity g where g.idx = :idx", GroupEntity.class)
        .setParameter("idx", user.getGroup().getIdx()).getSingleResult();
```

user 정보를 가져와, 유저가 가지고 있는 group의 fk 값을 조회하고, 조회 한 fk 값으로 group 정보를 조회하는 쿼리가 2개 인 것이다.

이 부분을 개선하기 위해, SecurityContext 에 저장 될 Context 객체를 수정 해보자.

먼저 Security 의 User 객체를 상속 할 CustomUserDetail 객체를 만들자.

_CustomUserDetail.java_

```java
package ziponia.spring.security;

public class CustomUserDetail {
}

```

그리고 이 객체는 User 객체를 상속한다.

```java
public class CustomUserDetail extends User {

    public CustomUserDetail(String username, String password, Collection<? extends GrantedAuthority> authorities) {
        super(username, password, authorities);
    }
}
```

Security User 의 username 과 password 와 authorities 만 가지고 있는 컨스트럭터를 상속하였다.

그리고 group 아이디 필드를 만들자

```java
public class CustomUserDetail extends User {

    private Integer groupIdx;

    public CustomUserDetail(String username, String password, Collection<? extends GrantedAuthority> authorities, Integer groupIdx) {
        super(username, password, authorities);
        this.groupIdx = groupIdx;
    }

    public Integer getGroupIdx() {
        return groupIdx;
    }
}

```

컨스트럭터의 4번째 인자값으로, groupIdx 를 넣고, 멤버변수에 groupIdx 로 집어 넣어 준 다음, getter 를 만들었다.

이제 UserDetailsServiceImpl 에서 new User(...) 를 우리가 만든 CustomUserDetail 로 변경하자.

_UserDetailsServiceImpl.java_

```java
public class UserDetailsServiceImpl implements UserDetailsService {

    // ... some

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {

        // ... some

        return new CustomUserDetail(userEntity.getUsername(), userEntity.getPassword(), authorities, userEntity.getGroup().getIdx());
    }
}
```

마지막으로 UserController 에서 /profile/group 에 들어있는 Authentication 객체를 CustomUserDetail 로 변경 하고, 기존에 userEntity 를 가지고 오는 객체를 주석 처리 해주면, 쿼리 한번으로 그룹을 조회 할 수 있다.

_UserController.java_

```java
public class UserController {

    // ...

    @GetMapping(value = "/profile/group")
    @PreAuthorize("isAuthenticated()")
    public String myGroup(
            // Authentication -> CustomUserDetail
            @AuthenticationPrincipal CustomUserDetail authentication
            Model model
    ) {
        // user 정보를 가지고 온다.
        /*UserEntity user = em.createQuery("select u from UserEntity u where u.username = :username", UserEntity.class)
                .setParameter("username", authentication.getName()).getSingleResult();*/

        // group 정보를 가지고 온다.
        GroupEntity group = em.createQuery("select g from GroupEntity g where g.idx = :idx", GroupEntity.class)
                .setParameter("idx", authentication.getGroupIdx()).getSingleResult();

        model.addAttribute("group", group);

        return "user_group";
    }
}
```

> 만약에 서버 부팅 후 에러가 난다면, 홈으로 가서 로그아웃 후 다시 접속 하자. SecurityContext 에 groupIdx 가 담기지 안은 상태로, 컨텍스트 객체가 만들어 지지 않아서 생기는 문제이다.

이렇게 SecurityContext 객체를 입맛대로 수정 해 보았다.

이제 입맛대로 다른 정보들도 관리 해보자.
