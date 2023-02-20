---
layout: single
title:  "빠르고 간단한 Webclient 이해"
categories: "Spring"
tag: ["Spring", "Webflux", "Webclient"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

## WebClient
WebClient는 Spring Framework 5부터 도입된 non-blocking HTTP 클라이언트입니다.  
WebClient는 기존의 RestTemplate과 달리 non-blocking 방식으로 HTTP 처리합니다. I/O 스레드를 블로킹하지 않아, 효율적인 자원관리가 가능합니다.

**GET, POST, PUT, DELETE, HEAD와 같은 다양한 HTTP 메서드를 지원하며, 각 메서드에 대한 요청을 생성하는 데 사용될 수 있는 다양한 빌더 메서드를 제공합니다.** 또한, HTTP 요청에 대한 응답을 비동기적으로 처리하므로, Flux나 Mono와 같은 리액티브 타입을 반환합니다.

WebClient는 요청에 대한 인증 정보, 헤더, 쿠키 등을 설정할 수 있는 등, 요청 및 응답 데이터의 표현 방식을 지정할 수도 있습니다. 또한, WebClient는 HTTP 요청 및 응답에 대한 로깅, 타임아웃, 리다이렉트 처리, 백 프레셔 등 다양한 기능을 제공합니다.

이러한 특징으로 외부 API 호출, 마이크로서비스(MSA) 간 통신 등 다양한 상황에서 비동기적으로 HTTP 요청을 처리할 수 있습니다. 

## 샘플

### 의존성 추가 
Gradle 기준으로 build.gradle 파일에 다음 내용을 추가합니다.
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webclient'
}
```

### WebClient 구성
WebClient는 비동기 HTTP 클라이언트를 다음과 같이 간단하게 구성 할 수 있습니다.
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder().build();
    }
}
```

#### Build Option
WebClient를 사용할 때, build() 메소드를 호출하여 WebClient 인스턴스를 생성하는데, 이 때 다양한 옵션을 설정할 수 있습니다.
1. baseUrl(String baseUrl): 요청을 보낼 기본 URL을 설정합니다.

2. defaultHeader(String headerName, String... headerValues): 기본 HTTP 헤더를 설정합니다.

3. defaultCookie(String cookieName, String cookieValue): 기본 쿠키를 설정합니다.

4. defaultRequest(RequestHeadersSpec<?> headersConsumer): 기본 요청 설정을 적용합니다.

5. filter(ExchangeFilterFunction filter): HTTP 요청 및 응답에 대한 필터링 로직을 적용합니다.

6. exchangeStrategies(ExchangeStrategies strategies): 요청 및 응답에 대한 serialization 및 deserialization을 설정합니다.

7. clientConnector(ReactorClientHttpConnector connector): HTTP 클라이언트 연결을 설정합니다.

8. codecs(Consumer<ClientCodecConfigurer> configurerConsumer): HTTP 메시지 변환을 위한 코덱을 설정합니다.

9. responseTimeout(Duration responseTimeout): HTTP 응답 제한 시간을 설정합니다.

```java
WebClient webClient = WebClient.builder()
        .baseUrl("http://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .responseTimeout(Duration.ofSeconds(10))
        .build();

```

### REST API 호출 및 데이터 변환 예제
다음과 같이 WebClient를 사용하여 내부 REST API를 호출하고, 데이터를 변환하여 Mono로 반환합니다.
```java
@RestController
@RequestMapping("/api/v1")
public class UserController {

    @Autowired
    private WebClient webClient;

    @GetMapping("/users/{id}")
    public Mono<UserDetails> getUserDetails(@PathVariable("id") String id) {
        Mono<User> userMono = webClient.get().uri("http://user-service/api/v1/users/{id}", id)
                .retrieve()
                .bodyToMono(User.class);

        Mono<List<Order>> ordersMono = webClient.get().uri("http://order-service/api/v1/orders?userId={id}", id)
                .retrieve()
                .bodyToFlux(Order.class)
                .collectList();

        // User과 Order을 받아 UserDetails를 생성합니다.
        return Mono.zip(userMono, ordersMono, UserDetails::new);
    }
}
```
위와 같이 WebClient를 사용하여 내부 REST API를 호출하고, Mono로 반환합니다. 이 예제에서는 두 개의 내부 REST API를 호출하여 User와 Order 목록 데이터를 가져옵니다. 이후 Mono.zip() 메소드를 사용하여 User와 Order 목록을 합쳐 UserDetails 객체로 변환하여 Mono로 반환합니다. 이 예제에서 UserDetails 객체는 User와 Order 목록을 담고 있습니다.


