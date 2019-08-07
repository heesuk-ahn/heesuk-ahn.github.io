---
layout: post
title: (스프링) 1. 스프링 시큐리티 5 + JWT 인증 구현
category: spring
tags: [spring]
---

### 스프링 시큐리티 5를 이용한 JWT 구현

 토이 프로젝트를 하면서 스프링 시큐리티 5를 이용하여 JWT 인증을 구현하였는데, 이 부분을 공유하고자 한다.
스프링 시큐리티 5에서는 아래와 같은 3가지 feature가 추가되었다.

```
  1) OAuth 2.0 Login
  2) Reactive 지원
  3) 현대화된 비밀번호 인코딩
```

 나는 이중에서 Reactive 지원을 이용하여 JWT 인증에 대해 글을 작성하려 한다.
처음 스프링 시큐리티 5 Reactive를 적용하는 분들에게 작은 도움이 되었으면 한다.

### 스프링 시큐리티란?

 스프링 시큐리티는 인증(Authenticate)와 인가(Authorize) 과정을 쉽게 도와 주기 위한 라이브러리이다.
이 스프링 시큐리티는 필터 기반으로 동작하는데, 필터는 스프링의 디스패처에 오기 전 웹 서버 컨테이너(WAS)에서 관리하는
'서블릿 필터'에서 동작하기 때문에, 스프링은 서블릿 필터들을 관리하기 위해서 'DelegatingFilterProxy'를
web.xml에 설정하여 스프링에서 설정한 서블릿 필터가 동작하도록 한다.
 이 필터는 우리가 커스텀으로 만들 수 도 있고, 이미 선언되어있는 기본적인 필터들을 이용할 수 도 있다.

 ![filter-archi](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/filter-archi.png?raw=true)

이 필터는 체이닝 방식으로 이루어지게 되는데, request context가 path matcher가 되는 필터가 있다면,
필터가 동작해서 인증(Authenticate)을 수행하는 방식이다.
 spring security 5에서 어떤식으로 인증을 하고, 토큰을 얻을 수 있는지 아래에서 좀 더 자세하게 살펴보자.

## 주요 컴포넌트

먼저 주요 컴포넌트에 대해서 간단하게 이해를 하고 넘어가자. 그래야 이해하기가 편하다.

### AuthenticationWebFilter

 웹 필터는 매칭되는 URI에 따라서 인증 과정을 수행하기 위한 기능을 제공한다. 우리는 여러 웹 필터를
AuthenticationWebFilter를 통해서 커스텀하게 만들어서 '필터 체인'에 추가할 수 있다.

### SecurityContextHolder

```
This is where we store details of the present security context of the application, which includes details of the principal currently using the application. By default the SecurityContextHolder uses a ThreadLocal to store these details, which means that the security context is always available to methods in the same thread of execution, even if the security context is not explicitly passed around as an argument to those methods. Using a ThreadLocal in this way is quite safe if care is taken to clear the thread after the present principal’s request is processed. Of course, Spring Security takes care of this for you automatically so there is no need to worry about it.

- spring security javadoc 발췌
```

  스프링 시큐리티의 문서를 살펴보면, SecurityContextHolder에 대해 자세하게 설명이 되어있다.
간단하게 생각하면, 현재 어플리케이션에 접근하는 인증의 주체(Principal)의 정보를 저장하는 것이 SecurityContextHolder라고 한다.
기본적으로 SecurityContextHolder는 ThreadLocal에 주체 정보를 저장해서 같은 스레드의 실행흐름 안에서는
항상 Security context에 접근하는게 가능하다. 그렇기에 요청 context가 어떤 주체가 요청했는지 우리는 알 수
있는 것이다.

### 스프링 시큐리티 5에서 리액티브를 위해 제공하는 ReactiveSecurityContextHolder

 위에 SecurityContextHolder를 Reactive 스타일로 제공하는 것이 ReactiveSecurityContextHolder
이다.

```
Allows getting and setting the Spring SecurityContext into a Context.

- spring security javadoc 발췌
```

ReactiveSecurityContextHolder는 SecurityContext를 얻거나 셋팅을 할 수 있게 도와준다.
아래 API 문서를 살펴보면 static으로 선언된 getContext를 불러올 수 있다는 것을 알 수 있다.
이 getContext 안에는 Mono 타입으로 (Mono는 데이터가 0개 또는 1개인 경우다) SecurityContext가
선언되어있다.
 즉 우리는 ReactiveSecurityContextHolder.getContext를 통해 인증의 주체(Principal) 정보에
접근할 수 있을 것이다.

![getContext](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/getContext.png?raw=true)

### ReactiveAuthenticationManager

AuthenticationManager는 Spring security의 핵심 컴포넌트중 하나이다. AuthenticationManager는
인터페이스 인데 이 인터페이스는 인증을 하는 함수인 `authenticate` 함수를 가지고 있다.

![authenticate-func](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/auth-func.png?raw=true)

이 authenticate 함수는 필터를 통해 넘어온 Authenticate 객체가 존재한다면 성공적으로 넘어왔다면,
onAuthenticationSuccess 를 이용하여 성공한 인증에 대한 처리 로직을 실행하고, 실패한다면,
authenticationFailureHandler 를 이용해서 실패에 대한 로직을 처리한다.

## 코드를 통해 직접 동작 원리를 확인하자.

 이제 웹 필터 체인에 추가하기 위한 JWT 인증 필터를 만들어서 직접 주입해보는 과정이 어떤식으로 이루어지는지
확인해보자.

### 웹 필터 체인에 추가하기 위한 커스텀 필터를 만들자.

커스텀 필터를 만들기 위해서는 위에서 언급한 것 처럼 AuthenticationWebFilter 를 생성해야 한다.
AuthenticationWebFilter는 필터 체이닝에서 하나의 필터 부분을 책임져줄 수 있다.
내부적으로는 어떤 URI에 대해서 패스가 매치되면, ServerAuthenticationConverter 를 통해서
인증을 수행하고, 인증의 성공 / 실패에 따라서 처리를 하게 된다.

![bearerAuthenticationFilter](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/bearerAuthenticationFilter.png?raw=true)

위의 코드를 살펴보자.

먼저 웹 필터를 생성하기 위해서 'AuthenticationWebFilter' 를 생성하는데 생성 인자로 ReactiveAuthenticationManager가
필요하므로, 함께 생성하여 넣어준다.

![bearerAuthenticationFilter](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/bearerTokenManager.png?raw=true)

ReactiveAuthenticationManager는 위에 코드처럼 authenticate라는 인터페이스를 구현한
구현체를 넣어주면 되는데, Authentication 주체를 Mono 타입으로 반환만 하면 된다.

그 아래 코드들을 살펴보면

```
bearerConverter = new ServerHttpBearerAuthenticationConverter();
bearerAuthenticationFilter.setAuthenticationConverter(bearerConverter);
```

위와 같은 코드들을 볼 수 있는데, 실제로 필터 안에서 인증을 위해 사용될 Converter 를 구현해야 한다.
이 Converter는 AuthenticationWebFilter 안에서 사용되며, 이 Converter를 통해서 요청으로 부터
Authorization을 추출하여 Bearer token을 인증하고, 인증에 성공한다면 Authorization을 ReactiveAuthenticationManager로
넘기게 된다.

```
bearerAuthenticationFilter.setRequiresAuthenticationMatcher(ServerWebExchangeMatchers.pathMatchers("/**"));
```

이 부분은 꽤나 중요한 부분인데, 실제로 리퀘스트가 들어오면 이 필터가 어떤 URI에서 동작할지 설정하는 것이다.
이 코드에서는 모든 URI에 대해서 필터가 동작하도록 설정하였다.
만약, /v1/users 에서만 동작하기를 원한다면 ServerWebExchangeMatchers.pathMatchers("/v1/users")
라고 선언하면 된다.

### AuthenticationWebFilter 가 어떤 식으로 동작하는지 살펴보자.

그렇다면, 위에서 선언한 코드가 AuthenticationWebFilter 내부에서는 어떤식으로 동작할까?
spring security 5에서 제공하는 AuthenticationWebFilter 내부 코드를 직접 살펴보자.

## filter 메소드 코드 분석

![bearerAuthenticationFilter](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/filter-code.png?raw=true)

이 filter 메서드는 `ServerWebExchange`와 `WebFilterChain`을 받는다. 각각의 파라미터에 대한 설명은 아래와 같다.

```
[ServerWebExchange]

Contract for an HTTP request-response interaction. Provides access to the HTTP request and response and also exposes additional server-side processing related properties and features such as request attributes.

[WebFilterChain]

Contract to allow a WebFilter to delegate to the next in the chain.

```

쉽게 생각하면 ServerWebExchange는 요청과 응답에 대한 컨텍스트 이고, WebFilterChain은 필터들의 체인
으로, WebFilter가 다음 WebFilter로 전파 될 수 있도록 도와주는 것이다.

```
[URI 매치가 설정한 것과 매칭되는지 확인]

this.requiresAuthenticationMatcher.matches(exchange).filter((matchResult) -> {
      return matchResult.isMatch();
    })
```

위 코드에서 `requiresAuthenticationMatcher`는 request context의 URI가 매치되는지 확인하는
부분이다.

```
bearerAuthenticationFilter.setRequiresAuthenticationMatcher(ServerWebExchangeMatchers.pathMatchers("/**"));
```

이 코드가 requiresAuthenticationMatcher를 set 하는 코드이다. 우리는 현재 /** URI에 대해
모두 매칭되도록 해놓았다.

```
[URI 매칭시, Converter 실행]

return this.authenticationConverter.convert(exchange);
```

이 코드는 우리가 만들게될 JWT 인증 로직을 주입받은 Converter로 여기에 요청 컨텍스트를 넘겨서
JWT 토큰을 헤더로 부터 추출하고 verify한 뒤에 Mono<Authorization>을 반환하게 된다.

```
switchIfEmpty(chain.filter(exchange).then(Mono.empty())).flatMap((token) -> {
      return this.authenticate(exchange, chain, token);
    });
```

Converter로 부터 정상적으로 token이 돌아오게 되면, AuthenticationManager로 인해 최종적으로
authenticate를 하게 되고, 만약 매칭이 안되면, 다음 필터 체인으로 넘기는 것이다.

## authenticate를 메소드 코드 분석

위에 filter 메서드로 부터 Mono<Authorization>을 받게 되면, authenticate는 이에 대해서
성공인지 실패인지에 대한 처리를 실행하게 된다.

![auth-success](https://github.com/heesuk-ahn/heesuk-ahn.github.io/blob/master/assets/images/spring-security-5-jwt/auth-success.png?raw=true)

성공적으로 onAuthenticationSuccess이 호출이 되면, 이제 인증은 성공적으로 이루어졌다는 것이다.
하지만, 인증의 주체(principal)를 저장을 해야 Controller에서 해당 정보를 불러다 사용할 수 있을 것이다.

 ```
[onAuthenticationSuccess 코드 일부]

securityContext.setAuthentication(authentication);
ReactiveSecurityContextHolder.withSecurityContext(Mono.just(securityContext))

 ```

onAuthenticationSuccess 코드의 일부를 살펴보면, 위와 같이 ReactiveSecurityContextHolder에
securityContext를 넣는 것을 알 수 있다. 이 securityContext는 authentication 정보를 지니고 있다.
이 authentication에 관련해서는 다음 글에 커스텀 Converter를 구현하면서 직접 우리가 Authentication
인터페이스를 상속받아서 구현할 것인데, 쉽게 생각하면 JWT 토큰 안에 페이로드를 저장할 수 있도록 만들 수 있다.
인터페이스는 다음에 좀 더 자세히 살펴보겠지만 아래와 같이 생겼다.

```
public interface Authentication extends Principal, Serializable {
  Collection<? extends GrantedAuthority> getAuthorities();

  Object getCredentials();

  Object getDetails();

  Object getPrincipal();

  boolean isAuthenticated();

  void setAuthenticated(boolean var1) throws IllegalArgumentException;
}
```

 결국에 ReactiveSecurityContextHolder로 인해서 위에서 봤던 getContext API를 통해
Controller에서도 Principal 정보 (토큰 페이로드)를 얻을 수 있는 것이다.


## 결론

이번 문서에서는 글이 너무 길어지기 때문에 Spring security 5 reactive에서 주요 컴포넌트에
대해서 정리하고, 어떤 식으로 filter가 동작하는지, 그리고 이 filter에 의해서 SecurityContextHolder에
정보가 저장되는지 코드 내부를 트래킹 하며 정리하였다.
 이어지는 글에서는 이제 토큰을 verify하는 Custom Converter에 대해서 작성하고, 실제로 API call을
통해 어떤식으로 인증이 되는지 확인해보도록 한다.

참고:

[스프링 패키지 요약 문서] https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/server/package-summary.html
[스프링 security code 참고]
