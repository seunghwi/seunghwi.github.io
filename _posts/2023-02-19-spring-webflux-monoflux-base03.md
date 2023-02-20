---
layout: single
title:  "Mono, Flux 이해"
categories: "Spring"
tag: ["Spring", "mono", "flux"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

Mono와 Flux는 리액티브 스트림 처리를 지원하기 위해 SpringFramework5부터 도입된 비동기적 데이터를 처리하기 위한 리액티브 타입 입니다.

데이터를 처리하는 방법에 대한 공통 인터페이스를 제공하는데, **비동기적으로 데이터를 처리하며, 이를 통해 데이터 소스에서 데이터를 가져오는 동안 I/O 스레드를 블로킹하지 않고 다른 작업을 수행할 수 있습니다.** 

또한 여러 Operators 데이터를 처리할 수 있습니다. 예를 들어, map(), flatMap(), filter() 등의 Operators를 사용하여 데이터를 변환하고, 필터링하고, 다른 Mono나 Flux로 변환할 수 있습니다. 이러한 Operators를 통해 데이터 처리를 효율적으로 수행할 수 있습니다.

## Mono
<span style="color:blue">**0 또는 1개의 데이터**</span>를 처리하는 리액티브 타입입니다. 예를 들어, 데이터베이스에서 단일 객체를 가져올 때 사용될 수 있습니다. Mono는 **subscribe()** 메서드를 호출할 때 Mono에 대한 비동기적인 데이터 처리를 시작합니다.

아래는 Mono를 생성하고, 이를 subscribe() 메서드를 통해 데이터 스트림을 시작하고, 데이터와 에러를 처리하는 예제입니다.  
```java
Mono.just("Hello, world!")
  .subscribe(
    data -> System.out.println(data), // 데이터 처리
    error -> System.err.println(error) // 에러 처리
  );
```
"Hello, world!" 데이터를 생성하고, subscribe() 메서드를 호출하여 데이터 스트림을 시작합니다. 이벤트를 처리하는 메서드는 람다식으로 구현되어 있으며, 데이터 처리와 에러 처리를 각각 수행합니다.

## Flux
<span style="color:blue">**0개 이상의 데이터**</span>를 처리하는 리액티브 타입입니다. 예를 들어, 데이터베이스에서 여러 개의 객체를 가져올 때 사용될 수 있습니다. Flux는 **subscribe()** 메서드를 호출할 때 Flux에 대한 비동기적인 데이터 처리를 시작합니다.

Flux의 경우도, subscribe() 메서드는 데이터 스트림의 각 데이터와 에러, 완료 이벤트를 처리하는 Consumer를 인자로 받습니다.  
아래 코드는 Flux로부터 데이터 스트림을 생성하고, 이를 구독하여 각 이벤트를 출력합니다.
```java
Flux.just("Hello", "World")
  .subscribe(
    data -> System.out.println(data), // 데이터 처리
    error -> System.err.println(error), // 에러 처리
    () -> System.out.println("Completed") // 완료 처리
  );
```
데이터 처리와 에러 처리는 Mono 예제와 같지만, 완료 이벤트 처리가 추가되어 있습니다. 이벤트 처리 메서드는 각각 람다식으로 구현되어 있으며, 데이터 처리와 에러 처리는 이전 예제와 동일하며, 완료 처리는 "Completed"를 출력합니다.


## flatMapMany()와 flatMap()
flatMapMany()와 flatMap() Operators는 리액티브 프로그래밍에서 사용되는 Operators 중 두 가지입니다.


### flatMapMany()
<span style="color:blue">**Flux와 Mono를 다른 Flux로 변환합니다.**</span> 즉, Flux를 각각의 데이터 항목에 대해 여러 개의 데이터 항목으로 확장합니다. 이러한 동작은 데이터를 병렬로 처리할 때 유용합니다.

두개의 String을 각각의 배열로 변환 후 출력하는 예제입니다.
```java
Flux<String> words = Flux.just("Hello", "World");

Flux<String> letters = words
        .flatMapMany(word -> Flux.fromArray(word.split("")))
        .log();

letters.subscribe();
```
결과
```
20:03:36.992 [main] DEBUG reactor.util.Loggers$LoggerFactory - Using Slf4j logging framework
20:03:37.078 [main] INFO reactor.Flux.FlatMap.1 - onSubscribe(FluxFlatMap.FlatMapMain)
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - request(unbounded)
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - onNext(H)
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - onNext(e)
...
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - onNext(l)
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - onNext(d)
20:03:37.090 [main] INFO reactor.Flux.FlatMap.1 - onComplete()
```

### flatMap()
<span style="color:blue">**Mono와 Flux를 다른 Mono 또는 Flux로 변환합니다.**</span> 즉, 각각의 데이터 항목에 대해 하나의 Mono나 Flux를 반환합니다. 이러한 동작은 데이터를 처리하고 결과를 반환해야 할 때 유용합니다.

```java
Mono<String> username = Mono.just("john.doe");
Mono<User> userMono = username.flatMap(name -> userService.getUser(name));
userMono.subscribe();
```
이 예제는 "john.doe"라는 문자열이 있는 Mono를 만들고, flatMap()을 사용하여 userService.getUser()를 호출합니다. 그 결과로 User 객체를 발행하는 Mono를 반환합니다.

<span style="color:blue">**flatMapMany()는 데이터를 병렬로 처리하고 각 데이터를 다룰 때 사용하고, flatMap()은 데이터를 처리하고 결과를 반환할 때 사용합니다.**</span>

### 복합 예제
예를 들어, 사용자 ID 목록이 있는 Flux를 가져와 각 사용자 ID에 대한 주문을 가져오는 경우, flatMapMany()를 사용하여 각 사용자 ID에 대한 주문 Flux를 생성하고, 모든 주문 Flux를 병렬로 결합할 수 있습니다

```java
public Mono<List<UserWithOrders>> findAllOrdersEachUser() {
        return userRepository.findAllUser()
                .flatMapMany(Flux::fromIterable)
                .flatMap(user -> orderRepository.getUserOrders(user.getId())
                        .collectList()
                        .map(orders -> new UserWithOrders(user, order)))
                .collectList();
    }
```
이 소스 코드는 userRepository를 통해 모든 사용자 정보를 검색한 후, flatMapMany()를 사용하여 각 사용자 정보를 처리합니다. 그런 다음, orderRepository를 사용하여 사용자의 주문 목록을 가져온 후, collectList() 를 사용하여 주문 목록을 리스트로 반환되고, UserWithOrders 객체를 생성하여 사용자 정보와 주문 목록을 하나의 객체로 생성합니다. 이것은 Mono<List<UserWithOrders>> 유형의 Mono 객체로 반환됩니다.
