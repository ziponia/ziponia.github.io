---
layout: post
title: "[SPRING] Spring Security 사용하기 #8"
summary: "jwt token 으로 변경 해보자."
tags: [spring, spring-boot, spring-security]
---

# 토큰 type 을 jwt 로 변경 해보자.

기존에 인증은, Resource server 가 요청을 받을 때마다, Authorization Server 에게 확인 하는 방식이다.

쉽게 대화형식으로 따지자면...

```
User - U
ResourceServer - R
AuthorizationServer - A

U - 토큰 발급 부탁드립니다.
A - 여기 토큰나왔습니다.
U - R 님 /api/private 에 대한 리소스를 주세요. (토큰을 건냄)
R - 기달려주세요
R - A 님 여기 토큰(유저가 건넨) 이 사용 가능 한 토큰인가요?
A - 네 사용가능합니다.
R - U 님 A 에게 확인 받았고, 여기 리소스가 있습니다. /api/private 을 건넴
```

식이다.

매번 요청 할때마다, 지속적으로 검시하므로, AuthorizationServer 에게 부담 일수밖에 없다.

jwt 토큰은 이러한 점을 좀 개선시켰다. token 자체를 인증정보로 나타냈기 때문이다.

이걸 대화형으로 한다면

```
User - U
ResourceServer - R
AuthorizationServer - A

U - 토큰 발급 부탁드립니다.
A - 여기 토큰나왔습니다.
U - R 님 /api/private 에 대한 리소스를 주세요. (토큰을 건냄)
R - (사전에 A와 규칙을 정한) 토큰이 맞군요! 여기 리소스가 있습니다.
```

라는 식이다.

조금 웃기긴 하지만, 얼추 비슷하니 재밌게 참조하자.

이제 실제로 코드로 구현 해보겠다.

먼저, AuthorizationServer Config 에 기본 토큰을 담아낼 store 를 만들자.

_AuthorizationServerConfig.java_

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    // ...

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
        endpoints.accessTokenConverter(jwtAccessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        return new JwtAccessTokenConverter();
    }
}
```

JwtAccessTokenConverter 가 token 을 jwt 형식으로 교환해주고, AuthorizationServerEndpointsConfigurer 에 totkenConverter 를 주입 해주었다.

그럼 토큰 발급시 다음과 같이 변경 되었을 것이다.

다음으로, AuthorizeEndpointController 의 `/oauth/confirm_access` 를 다음과 같이 바꿔주자.

_AuthorizeEndpointController.java_

```java
public class AuthorizeEndpointController {

    @SuppressWarnings("unchecked")
    @GetMapping(value = "/oauth/confirm_access")
    public String authorizeConfirmPage(HttpServletRequest request, Model model) {
//        Map<String, Boolean> scopes = (HashMap<String, Boolean>) request.getAttribute("scopes"); remove
        String[] scopes = request.getAttribute("scope").toString().split(" "); // <- here
        model.addAttribute("scopes", scopes);
        return "authorize_confirm";
    }
}

```

jwt 로 변경하면, 가지고 오는 scope 이름이 scopes 가 scope 로 바뀐다. (사실 이부분은 왜 틀린지 모르겠다..)

view 파일도 form 태그 쪽을 살짝 변경 해주자.

_authorize_confirm.html_

```html
<!-- before... -->
<form action="#" th:action="@{/oauth/authorize}" th:object="${scopes}" th:method="post">
  <input type="hidden" name="user_oauth_approval" value="true" />
  <h4
    th:text="|${#request.getParameter('client_id')} 에서 회원님에 대한 아래와 같은 정보를 요청합니다|"
  ></h4>
  <div th:each="scope : ${scopes}">
    <label th:for="${scope}" th:text="${scope}">scope.read</label>
    <input type="checkbox" th:id="${scope}" th:name="${scope}" th:value="true" />
  </div>
  <div>
    <button type="submit">동의하기</button>
  </div>
</form>

<!-- after... -->
```

이제 토큰을 받아보자. 아마 조금 더 길어진 토큰을 볼 수 있을것이다.

토큰을 복사해서 [https://jwt.io/](https://jwt.io/) 에 붙혀넣으면 토큰에 대한 정보를 볼 수 있다.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-23/oauth-jwt-io.png)

# jwt 토큰 키 설정 하기

하나 문제가 있다. 서버를 재부팅 하고, 기존 토큰을 그대로 요청을 해보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-23/oauth-jwt-retoken.gif)

정상적으로 요청이 안되는 것을 볼 수 있다.

따라서, 이 상태로 서비스를 하려면, 서버가 재부팅 될 때, 우리의 서비스를 사용 하고 있는 사람들 에게 토큰을 다시 받으라고 공지 해야한다.

이게 얼마나 비효율적인 일인지 알 것이다.

왜 이런일이 일어나는가?

토큰을 받게 되면, 우리가 가져 온 토큰정보는, Security context 안에 저장이 된다.

따라서, memory 저장 방식인 것이다.

이는, 서버가 재부팅 될 때, 저장 공간이 갱신 되기 떄문이다.

다음 코드를 추가 해 보자. `/oauth/token_key` 로 들어 가 보자.

_AuthorizationServerConfig.java_

```java
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    //...
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        super.configure(security);
        security
                .tokenKeyAccess("isAuthenticated()") // <- here
                .allowFormAuthenticationForClients();
    }
    //...
}
```

이제 다음 URL 로 들어 가 보자.

[http://localhost:8080/oauth/token_key](http://localhost:8080/oauth/token_key)

confirm 창이 나오면 client id 와 client secret 를 넣으면 된다.

그럼 화면상에 다음 과 비슷한 데이터가 나올 것이다.

```json
{ "alg": "HMACSHA256", "value": "Sp3P55" }
```

서버를 재부팅 하고 다시 들어 가보자.

아마도, value 값이 변할 것이다.

```json
{ "alg": "HMACSHA256", "value": "AM1juH" }
```

서버를 재부팅 하게 되면, 토큰의 알고리즘 자체가 바뀌게 되어, 기존 토큰으로는 인증이 안된다.

따라서, 우리는 서버가 용납(?) 할 수 있는 키를 만들어야 한다.

적당한 폴더에 다음과 같이 입력 해보자

> keytool 이 설치 되어있다는 가정에 따른다. java 가 설치되어있으면, 왼만하면, 동작한다.

```cmd
PS C:\keys> keytool -genkey -alias jwt -keyalg RSA -keystore jwtkey.jks -keysize 2048
키 저장소 비밀번호 입력:
새 비밀번호 다시 입력:
이름과 성을 입력하십시오.
  [Unknown]:  zef
조직 단위 이름을 입력하십시오.
  [Unknown]:  zef
조직 이름을 입력하십시오.
  [Unknown]:  zef
구/군/시 이름을 입력하십시오?
  [Unknown]:  Seoul
시/도 이름을 입력하십시오.
  [Unknown]:  Seoul
이 조직의 두 자리 국가 코드를 입력하십시오.
  [Unknown]:  KR
CN=zef, OU=zef, O=zef, L=Seoul, ST=Seoul, C=KR이(가) 맞습니까?
  [아니오]:  예

<jwt>에 대한 키 비밀번호를 입력하십시오.
        (키 저장소 비밀번호와 동일한 경우 Enter 키를 누름):

Warning:
JKS 키 저장소는 고유 형식을 사용합니다. "keytool -importkeystore -srckeystore jwtkey.jks -destkeystore jwtkey.jks -deststoretype pkcs12"를 사용하는 산업 표준 형식인 PKCS12로 이전하는 것이 좋습니다.
```

난 window 환경이라, powershell 로 실행하였다. 명령 행에 따라, 맞춰 쓰면 된다.

위처럼 하였다면, 폴더 안에 jwtkey.jks 라는 파일이 생성 되었을 것이다. 해당 파일을 프로젝트의 classpath (resource) 로 이동 시키자.

> 이 행동은, 실제 서비스에서는 하지 않는다. key 는 보안규칙에 따라 관리 되어야 한다. (환경변수 등) 여기서는 포스트 용도로 공개한다.

```
{projectRoot}/src/main/resource/jwtkey.jks
```

이제 AuthorizationServerConfig 에 jwtAccessTokenConverter 를 다음처럼 변경 해주자.

```java
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    //...
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        String keyPassword = "123456", alias = "jwt", keyFile = "jwtkey.jks";
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        KeyPair keyPair = new KeyStoreKeyFactory(new ClassPathResource(keyFile), keyPassword.toCharArray())
                .getKeyPair(alias, keyPassword.toCharArray());
        converter.setKeyPair(keyPair);
        return converter;
    }
    //...
}
```

포스트 내용과 똑같이 만들었다면, 위와 같이 설정하면 되고, 아니라면, 각자 환경에 맞게 셋팅하면 된다.

> 패스워드는 포스팅 용도로 123456 으로 만들었다 실제서비스에서는 당연히 이렇게 하면 안된다.

이제 다시 [http://localhost:8080/oauth/token_key](http://localhost:8080/oauth/token_key) 로 들어가 보자.

아까와 조금 다른 값들이 나올것이다.

```json
{
  "alg": "SHA256withRSA",
  "value": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAg14zVmL/2S8xn23iW88vC0l+4PQyWEbYYLgx5phjMbvy8ay/VWyl8SPbZf759t6uG330IWe69gJNs+UNXPnhC8MZ5OxEYzpiUd7IiHVKS+oJRvZPS+F82fS9pWXX0dHtxo1dG/K+IApvumzUFczpmpXHw81RFaZak9hu4u+IkOdYwc5EKPBbN3OHZfKmWJhUtklsncHA8Xm+59P9lokYBBwLgD34RBAASAyNBl02l/kPuWXVtYNktgOiFeARf7hsMV890MG7SSGNjDltbdP51ET0ASGEXX9xYnNBdxfwlLoQYPBVJgdfc18dEuBjPXFpvYIpGKkAuxamPq/q6sFBrQIDAQAB\n-----END PUBLIC KEY-----"
}
```

이제 다시 토큰을 발급 받은 후, 서버를 재부팅하고 api 요청을 해보자.

![이미지](https://s3.ap-northeast-2.amazonaws.com/ziponia.github.io/2019-5-23/oauth-jwt-reboot.gif)

성공적으로 재부팅 후에도 토큰이 유효성을 가질 수 있게 되었다.

source [https://github.com/ziponia/spring-security-example](https://github.com/ziponia/spring-security-example)
