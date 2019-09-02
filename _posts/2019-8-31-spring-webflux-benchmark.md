---
layout: post
title: (Benchmark) 스프링 웹플럭스 벤치마크
category: benchmark
tags: [benchmark]
---

# 스프링 부트 + 웹 플럭스 테스트
- - -

## 개요

 스프링 5 부터 리액티브 API를 기반으로 한 웹플럭스가 나왔다. 이 웹플럭스는 이벤트 드리븐 아키텍처 구조로
이벤트 루프를 도는 스레드들이 빠르게 요청을 받고 워커 스레드로 넘겨 처리를 하는 방식을 사용한다.

![event-driven](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/event-driven.png?raw=true)

 위의 이미지를 살펴보면, 어떤 작업이 완료되면 이벤트가 발생하고 그 이벤트를 다시 이벤트 루프 스레드가 처리하여 반환하는 것을
알 수 있다. 오늘은 이 웹플럭스가 무엇인지와 그리고 이 웹플럭스의 성능 벤치마킹을 통해서 스레드들이 어떤 식으로 사용되는지에 대해서
이해해보고자 한다.

## 스프링 웹플럭스는 왜 만들어졌을까?

```
***첫번째 이유***,

However, using it leads away from the rest of the Servlet API, where contracts are synchronous (Filter, Servlet) or
blocking (getParameter, getPart). This was the motivation for a new common API to serve as a foundation across
any non-blocking runtime. That is important because of servers (such as Netty) that are well-established in the async,
non-blocking space.

***두번째 이유***,
The other part of the answer is functional programming. Much as the addition of annotations in Java 5 created opportunities
(such as annotated REST controllers or unit tests), the addition of lambda expressions in Java 8 created opportunities
for functional APIs in Java. This is a boon for non-blocking applications and continuation-style APIs
(as popularized by CompletableFuture and ReactiveX) that allow declarative composition of asynchronous logic.
At the programming-model level, Java 8 enabled Spring WebFlux to offer functional web endpoints alongside annotated controllers.
- spring webflux document 발췌
```

  위의 웹플럭스 문서에 따르면, 웹플럭스를 만든 `첫번째 이유` 는 기존 서블릿 컨테이너 3.1에서는 non-blocking IO를 제공했지만, 내부적으로는 아직
동기적이거나 블락킹 되는 메서드들이 남아있기 때문이라고 한다. 이를테면 동기적인 Filter, Servlet 블락킹 메서드인 getParameter, getPart등이 그러하다
이러한 모티베이션으로로 인해서 새로운 non-blocking api를 만들게 된 것이라고 한다. 이런 non-blocking api를 위해 서블릿 컨테이너 대신
내부적으로 async, non-blocking 하게 동작하는 netty와 같은 것을 사용하게 된 것이다.

```
(참고)

netty는 비동기식 이벤트 기반 네트워크 애플리케이션 프레임워크다. TCP/IP 통신을 위해서는 네트워크 OSI
7계층에서 TCP/IP 계층과 어플리케이션 계층이 소켓 연결을 해야한다. 기존 java.net에서 제공해주던
소켓 연결은 함수 자체가 blocking 이였기 때문에, 소켓 연결 하나당 스레드 하나가 필요하였다. 이는
시스템 리소스의 과도한 소비로 이어지게 되었다.
 이러한 블락킹 이슈로 인한 성능 저하를 개선하기 위해 java new io가 나오게 되었는데, 여기서 소켓 연결을
할 때, 채널과 셀렉터라는 컨셉을 사용하여 non-blocking 네트워크 통신을 지원하게 되었다. 하지만 이것을
row 레벨부터 직접 사용할 경우, 잘못 사용하면 버그가 발생될 수 있기 때문에, 이를 기반으로 한 프레임워크인
Netty가 만들어지게 된 것이다.
 Java NEW IO에 대한 간단한 개념과 소켓 연결하는 예제와 Netty 예제는 다음번에 한번 글로 다루어 보고자
한다.
```

`두번째 이유` 는 함수적인 프로그래밍 스타일 때문이다. java 8부터 람다 표현식을 이용하여 자바에서도 함수적인 스타일 코드
를 작성할 수 있게 되었고, 이를 웹플럭스에서 함수적인 웹 엔드 포인트를 제공 하기 위함이라고 한다.

그렇다면 두번째 이유처럼 웹플럭스는 함수형 스타일 기반일까? 그렇지는 않다. 웹플럭스는 Mono와 Flux라는
데이터 타입을 이용하는데, 이는 "Reactive" 스타일이다.

Reactive 스타일은 이벤트 드리븐 스타일로 호출한 쪽에서 호출을 확인하기 위해 pull하는 것이 아니라,
역으로 호출을 받은 쪽에서 호출한 쪽으로 push하여 알려주는 것이다.

간단한 예를 생각해보면 자바의 옵저버 패턴을 생각해보면 된다. 옵저버 패턴에서는 호출한 쪽에서 이벤트를
등록하고, 이벤트가 발생하면 `notifyObservers`를 이용하여 이벤트를 등록된 구독자들에게 push하여
알려주기 때문이다.

이런 옵저버 패턴의 단점은 이벤트가 언제 종료가 되었는지에 대해서 알 수 없고, 에러를 받아서
처리하는 것이 없기 때문에 이를 개선하여 reactive-streams 표준을 만들었다.

현재는 넷플릭스, typesafe 등에서 지원을 하고 있는데 이 reactive-stream의 표준 API는
4가지가 있다.

| Processer |
| Subscriber |
| Publisher |
| Subscription |

웹 플럭스에서 Mono와 Flux는 이런 리액티브 프로그래밍 메소드들을 지원해준다. 리액티브 프로그래밍에
대해서는 다음번에 좀 더 자세하게 정리해보고자 한다.


## 벤치마크 테스트 시나리오

 아래와 같은 3가지의 벤치마크를 실시한다.

 ```
 Test Case 1) "HELLO"를 리턴하는 함수 콜

 Test Case 2) Database IO 테스트

 Test Case 3) 원격지 Call Test with WebClient
 ```

## 테스트 결과

### Test Case 1) "HELLO"를 리턴하는 함수 콜

[동기 with 톰캣]
  동기 서버 케이스에서 톰캣에 스레드 수를 1000으로 주고 테스트를 할 경우,
10초 동안 약 `124011 requests in 10.05s`를 처리하는 결과를 얻을 수 있었다.
하지만, 생성된 스레드 수는 아래 이미지와 같다. 스레드가 120개까지 올라가는 것을 알 수 있다.

![tomcat-case](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/tomcat.png?raw=true)

[웹 플럭스]

 웹플럭스의 경우에는 10초동안 ` 800003 requests in 10.10s`를 처리했다. 처리량으로만 봤을 때는
톰캣 with 동기에 비해서  `8배` 정도 차이가 나는 것을 알 수 있다.
 스레드는 어떨까? 아래 이미지를 살펴보면 약 20개정도로 전체 스레드의 수는 톰캣에 비해 `6배 적게 사용` 한것을 알 수 있다.

![webflux-hello-case](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/webflux-hello-case.png?raw=true)

 단순하게 Hello만 리턴하는 경우로 비교했을 때, 큰 성능차이가 나는 것을 알 수 있다. 이런차이는 왜 발생할까?
여기서만 봤을 때는 추측할 수 있는 것이 `이벤트 루프`가 빠르게 요청을 받고 워커 스레드에 넘기고 다시 요청을 받기 위한 스레드 풀로
반환이 되었기에 속도의 차이가 난다고 생각해 볼 수 있다.


### Test Case 2) Database IO 테스트

이번 테스트는 H2 인메모리 데이터베이스를 이용하여 DB IO 테스트를 해보도록 하자.
컨트롤러 코드는 아래와 같고, 테스트를 위해 Get 메소드로 설정되어있으나 실제로 생성관련에서는 POST를 쓰는 것이
일반적이다.

```
[톰캣 코드]  

@GetMapping("/v1/books")
public Book getHelloBlockRoute() {
  Book newBook = new Book();
  newBook.setName("TEST");
  return bookRepository.save(newBook);
}
```

```
[웹플럭스 코드]

@GetMapping("/v1/books")
  public Mono<Book> getHelloBlockRoute() {
    Book newBook = new Book();
    newBook.setName("TEST");
    return Mono.just(bookRepository.save(newBook));
}
```


[동기 with Tomcat]

 톰캣-동기 방식에 경우에는 역시나 스레드 수가 120개까지 올라가고, 스레드들을 보면 아래 처럼 러닝 상태와 Park 상태를
 왔다갔다 하는 것을 볼 수 있다.
  처리량은  `262381 requests in 10.03s` 이였다.

![tomcat-db-running](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/tomcat-db-running.png?raw=true)


[웹 플럭스 with Netty]

 이번에는 웹플럭스의 케이스에서 테스트를 해보자.
웹플럭스의 처리량은 `418479 requests in 10.03s`으로 톰캣과 비교하여 약 4배의 처리량을 보였다.
하지만 스레드 수는 event loop의 default size인 CPU Core 갯수인데, 8개로 이정도의 처리량을 보였다.

![webflux-db-running](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/webflux-db-running.png?raw=true)

event loop의 디폴트 사이즈 스레드로만도 이정도 성능을 낸다면, 만약 워커 스레드까지 동원한다면 어느정도까지
성능을 올릴 수 있을까?
 코드를 좀 더 변화시켜서 스레드 컨텍스트 스위칭이 일어나도록 수정하였다.

```
@GetMapping("/v1/books")
  public Mono<Book> getHelloBookRoute() {
    Book newBook = new Book();
    newBook.setName("TEST");
    return Mono.just(bookRepository.save(newBook)).subscribeOn(Schedulers.elastic());
  }
```

위의 코드를 보면 구독시에 스레드를 `Schedulers.elastic()`을 이용한 것을 볼 수 있다.
워커 스레드를 사용할 경우 스레드는 아래와 같이 컨텍스트 스위칭을 함을 알 수 있다.

```
2019-09-01 02:54:36.925  INFO 19588 --- [ctor-http-nio-3] reactor.Mono.SubscribeOnValue.42453      : onSubscribe([Fuseable] FluxSubscribeOnValue.ScheduledScalar)
2019-09-01 02:54:36.926  INFO 19588 --- [ctor-http-nio-3] reactor.Mono.SubscribeOnValue.42453      : request(unbounded)
2019-09-01 02:54:36.926  INFO 19588 --- [     elastic-11] reactor.Mono.SubscribeOnValue.42453      : onNext(com.heesuk.springbootwithwebflux.springbootwithwebflux.repository.Book@463ab731)
2019-09-01 02:54:36.926  INFO 19588 --- [     elastic-11] reactor.Mono.SubscribeOnValue.42453      : onComplete()
```

 위의 로깅을 보면 요청은 webflux netty의 event loop 스레드가 처리하고 IO 작업은 elastic thread pool에서 처리한다.
그렇다면 성능은 어떨까?
 처리량은 오히려 떨어졌다.  `347829 requests in 10.02s` 동안 처리를 했고, 레이턴시 평균은 `3.82ms` 였다.


![webflux-elastic](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/webflux-elastic.png?raw=true)

![webflux-worker](https://github.com/heesuk-ahn/server-benchmark/blob/master/springboot-with-webflux/images/webflux-worker.png?raw=true)

 왜 처리량이 떨어졌을까? 아마 예측하기로 스레드 컨텍스트 스위칭이 발생하면서 해당 비용으로 인해 성능 저하가 발생하지 않았을까 싶다.
하지만 일반적으로 event loop thread는 blocking이 안걸리게 하는 것이 당연히 성능에 이점이 있다.
현재 테스트 케이스에서는 database IO가 빠르게 처리되므로 오히려 내부 워커 스레드를 이용하는 것이 더 처리량이 낮지만, 실제 프러덕션에서는
이처럼 간단한 비즈니스 로직이 아닐 가능성이 크다. 그렇게 되면 event loop내 스레드가 blocking method를 만나서 다음 요청
처리를 위해 빠르게 복귀를 하지 못한다면 처리량은 확 떨어지게 될것이다.
 그렇기 때문에 blocking io 이슈에 있어서는 워커 스레드를 사용해야 한다. 하지만 이 경우 결국에는 기존 동기 서버 모델과 크게 성능의 이점 차이는
얻을 수는 없을 것이다.


## 결론

 Blocking IO가 코드 내에 존재하지 않는다면 Webflux는 적은 스레드 수로 최대의 성능을 낼 수 있다.
하지만 현재 JDBC는 Blocking 코드로 이루어져 있으므로, JDBC가 아닌 Non-block이 보장되는 mongo 디비를 사용하는 것을
추천한다. 기대하기로는 JDBC Non block API가 나온다면, (R2DBC) RDBMS에서도 좋은 성능을 기대해 볼 듯하다.


## 깃허브

벤치마크 예제는 깃허브에 업데이트 해놓았습니다! 문서에서 잘못된 부분이 있다면 언제든지 heesuk.dev@gmail.com
으로 연락주시면 감사하겠습니다. :D

```
https://github.com/heesuk-ahn/server-benchmark/tree/master/springboot-with-webflux
```

## TODO
- webClient 테스트 케이스 추가하기  
- webflux 내부에서 이벤트 드리븐 아키텍처에서 스레드가 동작하는 방법 더 자세하게 알아보기
