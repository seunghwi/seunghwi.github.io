---
layout: single
title:  "Spring에서 Redis 사용 그리고 HyperLogLog"
categories: "Spring"
tag: ["Spring", "Redis", "HyperLogLog"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

## 소개
Redis는 NoSQL 데이터베이스로서 메모리 기반 데이터 저장소입니다. 메모리에서 데이터를 읽고 쓰기 때문에 빠른 속도로 데이터를 처리할 수 있습니다. 다양한 데이터 유형을 지원하며, 주로 캐싱, 세션 관리, 메시징 등에 사용됩니다.

Spring Redis는 Redis를 사용하는 Spring 프레임워크의 모듈로, 데이터 액세스를 위한 객체-관계 매핑을 제공합니다.  

**RedisTemplate, StringRedisTemplate 및 Redis Repository를 제공.**
1. RedisTemplate은 Redis와 통신하기 위한 메서드를 제공하는 Bean입니다.
2. StringRedisTemplate은 RedisTemplate과 유사하지만 키와 값이 모두 문자열인 경우 사용됩니다. 
3. Redis Repository는 Redis에서 데이터를 저장하고 검색하기 위한 인터페이스입니다.

Spring Redis를 사용하려면 Redis와의 연결을 설정해야 합니다. Redis 연결 정보는 RedisConnectionFactory를 통해 설정됩니다. 기본적으로 Spring Boot는 로컬 Redis 인스턴스에 대한 기본 연결을 설정하므로 별도의 구성 없이 Redis와 통신할 수 있습니다.

Redis와 연결되면 RedisTemplate 빈을 주입받아 Redis 데이터베이스에서 데이터를 저장, 검색, 업데이트, 삭제할 수 있습니다. RedisTemplate의 opsForValue() 메서드를 사용하면 Redis에 문자열 값을 저장하고 검색할 수 있습니다. opsForHash() 메서드를 사용하면 Redis에 해시 값을 저장하고 검색할 수 있습니다. opsForList() 메서드를 사용하면 Redis에 리스트 값을 저장하고 검색할 수 있습니다.

Spring Redis는 Redis 클러스터/Sentinel을 지원하며, Redis 클러스터에 대한 연결을 자동으로 관리할 수 있습니다.

## 설정 및 예제
아래 내용은 Spring에서 Redis관련 설정과 예제소스입니다.

### 설정
Gradle을 사용하는 경우 build.gradle 파일에 다음 의존성을 추가합니다.
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

다음으로, Redis 설정을 application.properties 파일에 추가합니다.
```
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=
```

### 예제


RedisRepository 인터페이스를 생성하고, String 값을 저장, 추출, 갱신, 삭제 메소드를 추가할수 있습니다.
```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Repository;

@Repository
public class RedisRepository {

    private RedisTemplate<String, String> redisTemplate;

    public RedisRepository(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void setValue(String key, String value) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        ops.set(key, value);
    }
    public String getValue(String key) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        return ops.get(key);
    }
    public void updateValue(String key, String value) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        ops.getAndSet(key, value);
    }
    public void deleteValue(String key) {
        redisTemplate.delete(key);
    }
}
```
위와같이 Redis에 접근하는 레파지토리를 작성할수 있습니다.

### Redis Standalone Bean 등록하기
Spring에서 Redis를 사용하기 위해서는 Redis와의 연결 정보를 설정해야 합니다. Redis 연결 정보는 RedisConnectionFactory를 통해 설정되며, RedisConnectionFactory는 Redis 인스턴스에 연결하고 RedisTemplate 빈을 생성하는 데 사용됩니다.

Standalone으로 구성된 Redis는 아래와 같이 Bean으로 등록 할 수 있습니다.
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration("localhost", 6379);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        return template;
    }
}
```
위 코드에서는 RedisStandaloneConfiguration을 사용하여 Redis 호스트 및 포트 정보를 설정하고, LettuceConnectionFactory를 사용하여 RedisConnectionFactory를 생성합니다.

그리고 RedisTemplate 빈을 생성하여 RedisConnectionFactory를 주입하고, Redis 데이터베이스에서 사용할 직렬화 및 역직렬화 클래스를 설정합니다. 위 코드에서는 Jackson2JsonRedisSerializer를 사용하여 객체를 JSON으로 직렬화하고 역직렬화합니다.

이제 RedisConfig 클래스를 Spring 구성 클래스로 등록하면 RedisConnectionFactory 및 RedisTemplate 빈이 생성되고 Redis와의 연결이 설정됩니다.

### Redis Cluster Bean 등록하기

Redis 클러스터와의 연결을 설정하려면, RedisConnectionFactory 대신 RedisClusterConfiguration을 사용하여 Redis 클러스터 노드 정보를 설정해야 합니다. 아래는 Redis 클러스터 연결을 설정하는 예제 코드입니다.
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration config = new RedisClusterConfiguration(Arrays.asList(
                new RedisNode("redis-node-1", 6379),
                new RedisNode("redis-node-2", 6379),
                new RedisNode("redis-node-3", 6379)
        ));
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        return template;
    }
}
```
위 코드에서는 RedisClusterConfiguration을 사용하여 Redis 클러스터 노드 정보를 설정하고, LettuceConnectionFactory를 사용하여 RedisConnectionFactory를 생성합니다.

RedisTemplate Bend은 RedisConnectionFactory를 주입받아 생성하며, Redis 데이터베이스에서 사용할 직렬화 및 역직렬화 클래스를 설정합니다.

위 코드에서는 Redis 클러스터 노드가 3개이며, 각 노드의 호스트 및 포트 정보를 RedisNode 객체로 전달합니다. 따라서 이 예제에서는 3개의 Redis 클러스터 노드가 동일한 호스트에서 실행되고, 포트가 6379인 경우에 해당됩니다.

이제 RedisConfig 클래스를 Spring 구성 클래스로 등록하면 RedisConnectionFactory 및 RedisTemplate 빈이 생성되고 Redis 클러스터와의 연결이 설정됩니다.

### HyperLogLog란
HyperLogLog는 대량의 고유한 요소를 다루는 데 유용한 확률적 데이터 구조입니다. 예를 들어, 인터넷에서 방문자 수를 추적하거나, 중복을 제거하거나, 세션 수를 계산하는 데 사용됩니다. HyperLogLog는 중복 요소를 제거하고 이러한 요소의 개수를 추정하는 데 사용됩니다

HyperLogLog는 매우 작은 메모리 크기로 대규모 요소 집합에 대한 추정치를 제공하며, 정확도는 기본 오차 상한이 log(log(n))이기 때문에 n이 증가함에 따라 더 정확해집니다.

HyperLogLog를 사용하기 위해서는 Redis에서 제공하는 PFADD, PFCOUNT 등의 명령어를 사용해야 합니다
**Java에서 Redis HyperLogLog를 사용하는 예제 코드입니다.**
```java
// HyperLogLog 추가
String key = "unique_visitors";
String[] visitors = {"user1", "user2", "user3", "user1", "user2", "user4", "user5"};
Long count = redisTemplate.opsForHyperLogLog().add(key, visitors);

// HyperLogLog 개수 조회
Long uniqueCount = redisTemplate.opsForHyperLogLog().size(key);
System.out.println("Unique visitors count: " + uniqueCount);
```
RedisTemplate을 사용하여 HyperLogLog에 방문자 정보를 추가하고, Redis의 PFADD 명령을 사용하여 HyperLogLog에 고유한 방문자의 수를 측정합니다.

마지막으로 Redis의 PFCOUNT 명령어를 사용하여 HyperLogLog에 추가된 고유 방문자의 수를 조회합니다.

