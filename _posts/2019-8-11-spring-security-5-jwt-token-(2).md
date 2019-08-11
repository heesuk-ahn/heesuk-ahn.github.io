---
layout: post
title: (스프링) 2. 스프링 시큐리티 5 Reactive + JWT 인증 구현
category: spring
tags: [spring]
---

## 스프링 시큐리티 5 Reactive + JWT 인증 구현(2)

지난번 포스팅에 이어서 webflux JWT 인증 구현을 마무리 하려고 한다.
오늘은 JWT 인증을 수행해 줄 ServerHttpBearerAuthenticationConverter 를 만들고, 이를
이용해서 컨트롤러에서 token을 추출하여 응답하는 예제를 진행할 것이다.

전체 코드는 깃허브에 업로드 되어있다.

```
깃허브 jwt-auth-example : https://github.com/heesuk-ahn/spring-webflux-jwt-auth-example
```

### 전체적인 프로세스

![process](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/process.png?raw=true)

전체적인 프로세스는 위와 같이 우리가 구현하게될 Jwt Filter로 부터 인증을 완료하면, Authentication 객체를
ReactiveSecurityContextHolder에 저장하게 되고, Controller에서는 이 ReactiveSecurityContextHolder에
접근하여 인증의 주체 정보를 추출할 수 있게 된다.

### ServerHttpBearerAuthenticationConverter 구현

![converter](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/ServerHttpBearerAuthenticationConverter.png?raw=true)

위에 코드는 Request Context를 받은 후, Authorization Header에서 Bearer 토큰을 추출한 뒤에
JwtVerifyHandler를 통해서 이 토큰이 유효한 토큰인지 검사하는 로직이다. 이 토큰이 유효하다면,
Authentication 객체를 생성해야 하는데, 이 Authentication는 ReactiveSecurityContextHolder에
저장되게 될 것이다. 이 Authentication AbstractAuthenticationToken을 Spring Security에서
제공해주는데 이를 상속받아서 구현할 수 있다.
 Converter를 구현하기 위해서는 AuthorizationHeaderPayload, JwtVerifyHandler, CurrentUserAuthenticationBearer이
필요하다. 하나씩 구현을 해보자.

### 헤더에서 Authorization을 추출하자 AuthorizationHeaderPayload.

![headerExtract](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/authorizationHeaderPayload.png?raw=true)

이 클래스는 주어진 Request Context에서 헤더에 Authorization을 추출하여 반환한다.
특별할만한 코드는 없고, ServerWebExchange에서 제공되는 함수들을 이용하여 쉽게 헤더에서 원하는 값을
추출할 수 있다.

### 토큰 인증을 위한 JwtVerifyHandler

JwtVerifyHandler를 생성하기 전에, java-jwt 디펜던시를 추가하자.

```
<dependency>
  <groupId>com.auth0</groupId>
  <artifactId>java-jwt</artifactId>
  <version>3.8.0</version>
</dependency>
```

위 라이브러리를 통해서 토큰을 생성하고 인증할 것이다.

![verify](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/jwt-verify.png?raw=true)

auth0 jwt는 다양한 암호화 알고리즘을 제공해준다. 선택한 알고리즘과 secret으로 이 token이 우리가
발행한 토큰이 맞는지 확인한다. 토큰이 정상적으로 인증된다면, `DecodedJWT` 객체를 반환한다.

### 커스텀 Authorization 객체를 만들자.

 spring security에서는 Authorization 객체를 위한 추상 클래스 AbstractAuthenticationToken를
제공해준다. 우리는 이 클래스에서 getCredentials과 getPrincipal 메서드를 구현하면 된다.
 물론 이 부분에서 본인이 JWT 토큰 페이로드에 따라서 구현하면 된다. 나의 경우에는 jwt payload를
아래와 같이 선언하였다.

 ```
 jwt payload
 {
   "tokenType": "accessToken",
   "accessToken": "Bearer AccessToken"
 }
 ```

![credentials](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/credentials.png?raw=true)

그리고 이번 예제에서는 요청을 한 주체에 대한 userId만 얻으면 되기 때문에, getPrincipal 메서드에
userId를 반환하도록 했다.
 이 메서드는 추후 SecurityContextHolder에서 Authorization 객체를 얻게되면 사용하게 될 것이다.

### Converter에 필요한 클래스 생성이 모두 완료되었다. 다시한번 확인해보자.

```
@Override
 public Mono<Authentication> apply(ServerWebExchange serverWebExchange) {
   return Mono.justOrEmpty(serverWebExchange)
     .flatMap(AuthorizationHeaderPayload::extract)
     .filter(matchBearerLength)
     .flatMap(isolateBearerValue)
     .flatMap(jwtVerifier::check)
     .flatMap(CurrentUserAuthenticationBearer::create).log();
 }
```

Converter 코드를 살펴보면, Bearer 토큰을 추출 한 후, 추출한 토큰을 jwtVerifier에 check
메서드로 전달하고, 반환된 token을 이용하여 Authorization 객체를 생성한다.

### 생성된 converter를 AuthorizationWebFilter에 셋팅하자.

![converter](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/settingConverter.png?raw=true)

이제 지난번 블로그에서 설명했던 AuthorizationWebFilter에 우리가 만든 converter를 설정하면 된다.

### filter chain을 설정하자.

![filter-chain](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/filter-chain.png?raw=true)

이제 필터체인에 우리가 생성한 AuthorizationWebFilter를 설정한다. 위와 같이 설정하면,
모든 요청에 우리가 만든 필터가 적용되게 된다.

### 컨트롤러에서 토큰을 extract해보자.

이제 마지막 단계로 컨트롤러에서 토큰을 추출하는 단계이다.

![response](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/response.png?raw=true)

위에 컨트롤러에서 ReactiveSecurityContextHolder로 부터 Authorization 객체를 추출한다.
이후, 우리가 이전에 만들었던 getPrincipal을 통해 userId를 얻을 수 있다.

그 후, 컨트롤러에서 HelloUser 객체를 반환하여 최종 출력 화면을 확인할 수 있다.

![success](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/spring-jwt-success.png?raw=true)

포스트맨으로 실제 동작을 확인해보자. 유효한 토큰을 Authorization 헤더에 넣어서 보내면 위와 같이
해당 토큰에서 유저아이디를 추출하고, 반환한다. 그리고 잘못된 토큰을 넣어 보낼 경우에는 권한이 없다는
에러를 반환하게 될 것이다.

### 결론

 spring security 5의 reactive를 이용하여 webflux에서 JWT 토큰을 하는 법을 살펴보았습니다.
spring security 5는 이 외에도 다양한 기본 필터들을 제공합니다. (Basic, login 등)
 아직 spring security 5의 reactive에 대한 시큐리티 관련한 정보가 부족하여 처음 흐름을 이해하는데
어려움이 있었는데, 혹시 궁금한 점이나 잘못된 부분이 있다면 언제든지 heesuk.dev@gmail.com 또는

```
깃허브 jwt-auth-example : https://github.com/heesuk-ahn/spring-webflux-jwt-auth-example
```

깃허브에 이슈를 등록해주면 언제든지 확인해보도록 하겠습니다. 감사합니다.
