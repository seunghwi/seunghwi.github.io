---
layout: single
title:  "Java Collection Stream 예제"
categories: "Java"
tag: ["Java", "Stream"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

스트림(Stream)은 컬렉션(Collection) 처리를 위해 Java 8에서 추가된 API입니다. 스트림은 컬렉션의 요소를 다루는 함수형 스타일의 연산을 지원하며, 병렬 처리(parallel processing)도 쉽게 구현할 수 있습니다.

스트림은 크게 다음과 같은 특징을 가집니다.

1. 연속된 요소: 스트림은 연속된 요소들을 처리합니다. 이때 요소들은 컬렉션, 배열, I/O 자원 등에서 가져올 수 있습니다.

2. 내부 반복: 스트림은 내부적으로 반복을 처리합니다. 즉, 개발자가 요소를 반복하는 코드를 작성할 필요가 없습니다.

3. 중간 연산: 스트림은 중간 연산과 최종 연산으로 구분됩니다. 중간 연산은 다른 스트림을 반환할 수 있으며, 최종 연산은 최종 결과를 반환합니다.

4. 지연 연산: 스트림은 지연 연산(lazy evaluation)을 지원합니다. 이는 최종 연산이 호출되기 전까지 중간 연산을 실행하지 않고 대기한다는 의미입니다.

5. 병렬 처리: 스트림은 병렬 처리를 쉽게 구현할 수 있습니다. parallel() 메서드를 사용하여 스트림을 병렬 스트림으로 변환할 수 있으며, parallelStream() 메서드를 사용하여 직접 병렬 스트림을 생성할 수도 있습니다.

## 사용 예제
**연속된 두 요소 중 값이 큰 요소를 선택하는 코드입니다.**
```java
List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6, 5, 3);

List<Integer> result = IntStream.range(1, numbers.size())
    .filter(i -> numbers.get(i) > numbers.get(i - 1))
    .mapToObj(i -> numbers.get(i))
    .collect(Collectors.toList());

System.out.println(result);  // [3, 4, 5, 9, 6]
```
IntStream.range(1, numbers.size())는 인덱스 1부터 리스트의 끝까지의 범위를 생성하며, filter() 메서드는 인접한 두 요소를 비교하여 현재 요소가 이전 요소보다 크다면 선택하도록 필터링합니다. 마지막으로 mapToObj() 메서드와 collect() 메서드를 이용하여 선택된 요소들을 리스트로 변환합니다.

이와 같은 방식으로 인접한 요소들을 비교하고, 필터링하여 원하는 요소를 선택하는 작업을 수행할 수 있습니다.

**문자열 리스트에서 길이가 5 이하인 단어만 골라내는 예제입니다.**
```java
List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");

List<String> shortWords = words.stream()
        .filter(word -> word.length() <= 5)
        .collect(Collectors.toList());

System.out.println(shortWords); // [apple, date]
```
위 예제에서는 문자열 리스트에서 filter() 메서드를 이용하여 길이가 5 이하인 단어만 골라내고, collect() 메서드를 이용하여 결과를 리스트로 변환하고 있습니다.

**병렬처리 예시입니다..**
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

Stream<Integer> stream = numbers.stream()
        .filter(n -> {
            System.out.println("Filtering " + n);
            return n % 2 == 0;
        })
        .map(n -> {
            System.out.println("Mapping " + n);
            return n * 2;
        });

// 최종 연산이 호출되기 전까지는 중간 연산이 실행되지 않음
stream.forEach(System.out::println);
``` 
위 코드에서는 리스트(numbers)를 스트림으로 변환한 후 filter() 메서드와 map() 메서드에서는 각각 짝수인 요소와 요소를 2배로 변환하는 처리를 하고 있습니다.

그러나 최종 연산(forEach())이 호출되기 전까지는 중간 연산이 실행되지 않습니다. 따라서 위 코드를 실행하면 다음과 같은 출력 결과가 나옵니다.

## 주요 메서드
스트림의 주요메소드를 설명합니다. [Java Doc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html)

1. filter(Predicate<T> predicate) : 주어진 Predicate에 해당하는 요소만 필터링하여 새로운 스트림을 생성합니다.
2. map(Function<T, R> mapper) : 주어진 Function을 이용하여 요소를 변환하고, 변환된 요소로 이루어진 새로운 스트림을 생성합니다.
3. flatMap(Function<T, Stream<R>> mapper) : 주어진 Function을 이용하여 요소를 변환하고, 변환된 요소를 포함하는 하나의 스트림으로 이루어진 새로운 스트림을 생성합니다.
4. distinct() : 스트림에서 중복된 요소를 제거한 새로운 스트림을 생성합니다.
5. sorted() : 스트림에서 요소를 기본 정렬 순서 또는 주어진 Comparator를 이용하여 정렬한 새로운 스트림을 생성합니다.
6. limit(long maxSize) : 스트림에서 주어진 크기(maxSize) 이하의 요소를 포함하는 새로운 스트림을 생성합니다.
7. skip(long n) : 스트림에서 처음 n개의 요소를 제외한 나머지 요소로 이루어진 새로운 스트림을 생성합니다.
8. forEach(Consumer<T> action) : 스트림의 각 요소에 대해 주어진 동작(action)을 실행합니다.
9. toArray(IntFunction<A[]> generator) : 스트림의 요소를 배열로 변환합니다.
10. reduce(T identity, BinaryOperator<T> accumulator) : 주어진 초기값(identity)과 BinaryOperator를 이용하여 스트림의 모든 요소를 처리하고 결과를 반환합니다.
11. collect(Collector<? super T, A, R> collector) : 주어진 Collector를 이용하여 스트림의 요소를 수집(collect)하여 결과를 반환합니다.
12. min(Comparator<? super T> comparator) : 스트림에서 주어진 Comparator에 따라 최소값을 가지는 요소를 반환합니다.
13. max(Comparator<? super T> comparator) : 스트림에서 주어진 Comparator에 따라 최대값을 가지는 요소를 반환합니다.
14. count() : 스트림의 요소 개수를 반환합니다.
15. anyMatch(Predicate<? super T> predicate) : 스트림의 요소 중 주어진 Predicate에 해당하는 요소가 하나라도 있는지 검사합니다.
16. allMatch(Predicate<? super T> predicate) : 스트림의 모든 요소가 주어진 Predicate에 해당하는지 검사합니다.
17. noneMatch(Predicate<? super T> predicate) : 스트림의 모든 요소가 주어진 Predicate에 해당하지 않는지 검사합니다.
18. findFirst() : 스트림의첫 번째 요소를 반환합니다.
19. findAny() : 스트림의 임의의 요소를 반환합니다.
20. concat(Stream<? extends T> a, Stream<? extends T> b) : 두 개의 스트림을 이어붙여 새로운 스트림을 생성합니다.
21. of(T... values) : 주어진 요소들로 이루어진 새로운 스트림을 생성합니다.
22. iterate(T seed, UnaryOperator<T> f) : 주어진 초깃값(seed)과 UnaryOperator를 이용하여 새로운 스트림을 생성합니다.
23. generate(Supplier<T> s) : 주어진 Supplier를 이용하여 무한한 스트림을 생성합니다.
24. empty() : 요소가 없는 빈 스트림을 생성합니다.

