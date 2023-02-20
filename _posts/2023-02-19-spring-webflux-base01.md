---
layout: single
title:  "빠르고 간단한 WebFlux 이해"
categories: "Spring"
tag: ["Spring", "Webflux"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

WebFlux는 Spring Framework 5.0부터 추가된 <span style="color:blue">**비동기/논블로킹 처리 웹 프레임워크**</span>입니다. 
기존의 Spring MVC는 서블릿 스레드를 사용하여 요청 처리했지만, WebFlux는 Netty, Undertow, Tomcat 등의 서버를 사용하여 비동기 처리합니다.

함수형 프로그래밍을 기반으로 하며, 리액티브 스트림(Reactive Streams) 프로그래밍 모델을 사용합니다. Reactive Streams은 비동기적으로 데이터를 처리하는 스트림 처리 모델입니다.

WebFlux에서는 [Mono와 Flux](../spring-webflux-monoflux-base03)라는 리액티브 타입을 사용합니다. 

## Spring Webflux VS Spring MVC
모두 Spring Framework의 일부입니다. 둘 다 스프링에서 RESTful 웹 애플리케이션을 빌드하기위한 도구입니다. 그러나 WebFlux는 Spring MVC와는 조금 다른 방식으로 작동합니다.

### Webflux
서블릿 API에 의존하지 않고, 비동기 논블로킹 I/O 모델을 사용하여 요청 스레드를 블로킹하지 않고 동시에 많은 요청을 처리할 수 있습니다.
#### 장점
1. Non-blocking I/O를 사용하여 동시성을 높여 더 적은 리소스로 더 높은 처리를 할 수 있습니다.
2. Reactor라는 강력한 리액티브 라이브러리를 사용하여 비동기 프로그래밍이 더 쉽고 간결하며 유연해집니다

#### 단점
1. Spring MVC와 비교하여 최신 기술이라 Spring MVC대비 레퍼런스가 적습니다.
2. Reactor와 WebFlux는 스레드 블로킹 API를 대체하기 위한 것이기 때문에, 모든 애플리케이션에 적합하지 않을 수 있습니다.

### Spring MVC
Spring MVC는 전통적인 Servlet API를 사용하여 스레드 당 하나의 요청을 처리하고 블로킹 I/O를 사용합니다
#### 장점
1. 많은 개발자들이 Spring MVC를 사용하여 웹 애플리케이션을 구축하는 방법에 대해 많은 경험이 있습니다.
2. 상대적으로 구성이 간단하고 쉽습니다.

#### 단점
1. Spring MVC는 기본적으로 동기적인 프로그래밍 모델을 사용하기 때문에 비동기 처리를 위해 추가 구성이 필요합니다.
2. Blocking I/O 방식을 사용하기 때문에 더 많은 자원이 필요 할 수 있습니다.

## Webflux 샘플

### 의존성 추가
Gradle 기준으로 build.gradle 파일에 다음 내용을 추가합니다.
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

### 모델 생성
REST API의 데이터 모델을 정의하는 모델 클래스를 생성합니다.
```java
@Document(collection = "users")
public class User {

    @Id
    private String id;
    private String name;
    private int age;

    // getters and setters
}
```

### Reactive Repository 생성
데이터베이스를 사용하는 경우 리액티브 리포지토리를 구현해야 합니다.
```java
@Repository
public interface UserRepository extends ReactiveCrudRepository<User, String> {
}
```

### Service 생성
컨트롤러에서 호출되는 비즈니스 로직을 구현하는 서비스를 생성합니다.
```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }

    public Mono<User> getUserById(String id) {
        return userRepository.findById(id);
    }

    public Mono<User> createUser(User user) {
        return userRepository.save(user);
    }

    public Mono<User> updateUser(String id, User user) {
        return userRepository.findById(id)
                .flatMap(existingUser -> {
                    existingUser.setName(user.getName());
                    existingUser.setAge(user.getAge());
                    return userRepository.save(existingUser);
                });
    }

    public Mono<Void> deleteUser(String id) {
        return userRepository.deleteById(id);
    }
}
```

### RestController 생성
다음과 같이 간단한 REST API 컨트롤러를 생성합니다.
```java
@RestController
@RequestMapping("/api/v1")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public Flux<User> getAllUsers() {
        return userService.getAllUsers();
    }

    @GetMapping("/users/{id}")
    public Mono<User> getUserById(@PathVariable("id") String id) {
        return userService.getUserById(id);
    }

    @PostMapping("/users")
    public Mono<User> createUser(@RequestBody User user) {
        return userService.createUser(user);
    }

    @PutMapping("/users/{id}")
    public Mono<User> updateUser(@PathVariable("id") String id, @RequestBody User user) {
        return userService.updateUser(id, user);
    }

    @DeleteMapping("/users/{id}")
    public Mono<Void> deleteUser(@PathVariable("id") String id) {
        return userService.deleteUser(id);
    }
}
```



