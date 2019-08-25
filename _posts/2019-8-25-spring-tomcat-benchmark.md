---
layout: post
title: (Benchmark) Spring boot tomcat 부하 테스트
category: benchmark
tags: [benchmark]
---

## 스프링 부트 + 톰캣 벤치마크 테스트
- - -

### 개요

  API 서버는 하나의 thread가 얼마나 많은 요청을 처리할 수 있는지가 퍼포먼스에 영향을 끼친다.
우리가 자주 사용하는 스프링 부트 프레임워크에서 톰캣의 스레드 수를 하나로 고정하고 얼마나 많은
요청을 처리할 수 있는지 부하 테스트를 하고, 이 부하테스트를 통해 병목지점을 고민하고 어떤 식으로
개선할 수 있을지 느껴보는 것은 중요한 공부가 될 것이다.

### 동기방식

 동기방식으로 서버 코드를 작성하면 프로그래밍 코드가 순차적으로 진행되기 때문에, 흐름을 이해하기가 쉽다.
그러나 하나의 요청이 하나의 스레드를 차지하게 되면, 해당 스레드는 더 많은 일을 할 수 가 없어진다.
 스레드가 Block 함수를 만나게 되면 스레드는 `Runnable` 상태로 `Ready` Status를 유지하게 된다.
그때 바로 `스레드 컨텍스트 스위칭`이 발생하게 된다. 그리고 블락 상태에 대기중인 스레드는 다른 일을 하지 못하고 해당 함수가 종료 될 때까지, `가만히 기다리게 된다.`
이러한 현상이 지속되면, 일을 할 수 있는 스레드가 병목 지점에서 대기하게 되고, 결국 스레드 풀 헬 현상까지 이어지게 된다.

![thread-pool-hell](https://github.com/heesuk-ahn/server-benchmark/blob/master/spring-boot-with-tomcat/images/thread-pool-hell.png?raw=true)

### 동기 방식 테스트 시나리오

  기본적으로 2가지 스프링 부트 서비스를 띄운다. 하나는 요청을 받는 서비스고 하나는 원격 서비스로써
1초동안 작업 시간이 소요되는 Remote 서비스이다.
 여기에서 RestTemplate와 AsyncRestTemplate를 통해 부하 테스트를 진행한다.

```
TestCase 1) 톰캣 스레드 풀 200개 + Remote Blocking API Call

    - 톰캣의 스레드 풀을 200개로 고정한다.
    - RemoteService에 RestTemplate로 http call을 한다.
    - wrk를 통해 Request Context를 생성 후 부하를 준다.
    - 부하시에 Virtual VM을 통해 스레드 상태를 확인한다.
```

```
TestCase 2) 톰캣 스레드 풀 1개 + Remote Blocking API Call

    - 톰캣의 스레드 풀을 1개로 고정한다.
    - RemoteService에 RestTemplate로 http call을 한다.
    - wrk를 통해 Request Context를 생성 후 부하를 준다.
    - 부하시에 Virtual VM을 통해 스레드 상태를 확인한다.
```

```
TestCase 3) 톰캣 스레드 풀 1개 + Async API Call

    - 톰캣의 스레드 풀을 1개로 고정한다.
    - RemoteService에 AsyncRestTemplate로 http call을 한다.
    - wrk를 통해 Request Context를 생성 후 부하를 준다.
    - 부하시에 Virtual VM을 통해 스레드 상태를 확인한다.
```

### 실행 결과

Test Case 1)

- 톰캣 스레드 수 : 200개
- Task : 1초 Blocking이 걸리는 Remote Service Call
- 요청 수 : 100개의 요청을 10개의 스레드가 동시 요청
- 10초동안 처리 량 : 900개 요청 처리 완료
- RTT 평균 Latency : 1.03s

```
wrk -c100 -t10 -d10s http://localhost:8080/v1/hello

Running 10s test @ http://localhost:8080/v1/hello
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.03s    49.58ms   1.20s    88.89%
    Req/Sec    25.15     29.32    90.00     74.65%
  900 requests in 10.07s, 110.74KB read
  Socket errors: connect 0, read 1, write 0, timeout 0

```

Test Case 2)

- 톰캣 스레드 수 : 1개
- Task : 1초 Blocking이 걸리는 Remote Service Call
- 요청 수 : 100개 요청을 10개의 스레드가 동시 요청
- 10초동안 처리 량 : 9개 요청 처리 완료
- RTT 평균 Latency : 1.02s

```
wrk -c100 -t10 -d10s http://localhost:8080/v1/hello

Running 10s test @ http://localhost:8080/v1/hello
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.02s    35.30ms   1.11s    88.89%
    Req/Sec     0.11      0.33     1.00     88.89%
  9 requests in 10.01s, 1.11KB read
  Socket errors: connect 0, read 1, write 0, timeout 0
Requests/sec:      0.90
Transfer/sec:     113.29B
```

Test Case 3)

- 톰캣 스레드 수 : 1개
- Task : 1초 Non-Blocking Remote Service Call
- 요청 수 : 100개 요청을 10개의 스레드가 동시 요청
- 10초동안 처리 량 : 902개 요청 처리 완료
- RTT 평균 Latency : 1.04s

```
wrk -c100 -t10 -d10s http://localhost:8082/v1/helloAsyncRestTemplate

Running 10s test @ http://localhost:8082/v1/helloAsyncRestTemplate
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.04s   111.02ms   1.51s    89.58%
    Req/Sec    25.50     24.44    90.00     70.80%
  902 requests in 10.10s, 123.32KB read
Requests/sec:     89.29
Transfer/sec:     12.21KB
```
![ASYNC-RestTemplate](https://github.com/heesuk-ahn/server-benchmark/blob/master/spring-boot-with-tomcat/images/ASYNC-RestTemplate.png?raw=true)

 Test Case 3번의 경우에는 톰캣 스레드가 1개이지만 좋은 퍼포먼스를 보인다.
하지만 이미지를 보면 알수 잇듯이 실제 워커 스레드가 100개가 넘게 생성되어 처리를 한다.
AsyncRestTemplate의 내부에서 워커 스레드를 생성하여 처리하기 때문이다.
이렇게 되면 사실상 스레드는 많이 사용하기 때문에 성능상에 큰 이점이 있다고 보기는 어렵다.

 이를 개선하기 위해서 Netty의 Netty4ClientHttpRequestFactory 를 이용하면 해결될 수 있다.

```
AsyncRestTemplate asyncRestTemplate = new AsyncRestTemplate(new Netty4ClientHttpRequestFactory(new NioEventLoopGroup(1)));
```

이렇게 사용하면 tomcat의 스레드 1개와 netty에 사용되는 몇개의 스레드만으로 좋은 퍼포먼스를 낼 수 있다.

### 결론

 Remote Service에 Network IO 상황을 RestTemplate와 AsyncRestTemplate로 테스트를 하였다.
이를 통해서 스레드 1개로도 충분한 퍼포먼스를 보일 수 있다는 것을 알 수 있었다.
 위에 테스트 처럼 실제로 개발시에 Remote Call을 할 때는 blocking이 걸리는 RestTemplate 대신에 Non-Blocking
client를 사용하는 것이 성능에 이점이 있다. 특히 스프링 5부터는  AsyncRestTemplate는 Deprecated되고 Webflux의
`WebClient`를 사용하면 간단히 논블락/어싱크 콜을 구현할 수 있다.
 다음번에는 NetworkIO가 아니라 DatabaseIO 관점에서 테스트 후 글을 작성해보고자 한다.



 ### 내용 보강 (TODO)

 - thread가 block method에 만나면 어떤 Status에 빠지게 될까?
 - Tomcat에서 BIO와 NIO 퍼포먼스 차이는?
