---
title: "java stream 정리(map)"
date: 2020-05-29T17:30:30-18:00
categories:
  - java
tags:
  - java
  - stream
comments: true
---

* Repository = [java-practice](https://github.com/isntyet/java-practice)

-----

### Mapping

스트림내 요소들에 대해 함수가 적용된 결과의 새로운 요소로 매핑해준다.

-----

### 준비하기  

* sample data (Human)  

| 번호 | 이름 | 가진돈 | 생일 | 여행지 |
|:---:|:------|:------|:------:|:------|
| 1 | jojae | 2900 | 1991-02-26 | seoul, hawai |
| 2 | haha | 1000 | 2003-03-02 | busan |
| 3 | arabia | 30000 | 2001-04-06 | seoul, paris |
| 4 | cici | 150 | 1982-05-16 | daegu, hongkong |
| 5 | zzang | 40000 | 1910-06-26 | gwangju |
| 6 | ssu | 200000 | 2012-07-11 | busan |
| 7 | kuku | 150 | 1991-02-27 | seoul, hawai |
| 8 | kuku | 2222 | 1998-07-27 | hawai |


```java
public class Human implements Comparable<Human> {

    private Long idx;
    private String name;
    private Integer money;
    private LocalDate birth;
    private List<String> travelDestinations;
}
```  

-----

### 기본 사용법  

```java
@DisplayName("이름만 가져와서 List 만들기")
void mapTest1() {
    List<String> humanNames = humans.stream()
            .map(h -> h.getName())
            .collect(Collectors.toList());

    for (String humanName : humanNames) {
        System.out.print(humanName + " ");
    }
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku kuku
```  

-----

### 중복제거

`distinct()`를 이용하여 중복제거 가능

```java
@DisplayName("중복제거")
void mapTest2() {
    printHumanNames(humans);

    List<String> names = humans.stream()
            .map(h -> h.getName())
            .distinct()
            .collect(Collectors.toList());

    System.out.println();
    for (String name : names) {
        System.out.print(name + " ");
    }
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku kuku
jojae haha arabia cici zzang ssu kuku
```  

-----

### 내부요소에 대한 평면화 (flatMap)

`flatMap` 을 이용하면 스트림이 아니라 스트림의 콘텐츠로 매핑이 가능함.  
`map`으로 `travelDestinations`으로 변환하였지만 해당값 자체가 `List<String>` 이기때문에  
여행지 하나하나에 대한 중복제거를 할수 없었지만  

`flatMap`을 이용해서 다시 스트림내의 컨텐츠를 가져와 매핑하였기에  
중복제거를 가능하게 함.

```java
@DisplayName("다녀온 여행지 종합")
void mapTest3() {
    printHumanTravelDestination(humans);

    List<String> travelDestinations = humans.stream()
            .map(h -> h.getTravelDestinations())
            .flatMap(Collection::stream)
            .distinct()
            .collect(Collectors.toList());

    for (String travelDestination : travelDestinations) {
        System.out.print(travelDestination + " ");
    }
}
```

* 결과

```vim
[seoul, hawai]
[busan]
[seoul, paris]
[daegu, hongkong]
[gwangju]
[busan]
[seoul, hawai]
[hawai]

seoul hawai busan paris daegu hongkong gwangju
```  

-----

### 내부요소에 대한 평면화2 (flatMap)


```java
@DisplayName("이름에 쓰인 문자 가져오기 (중복제거하여)")
void mapTest4() {
    printHumanNames(humans);

    List<String> humanNameWords = humans.stream()
            .map(h -> h.getName().split(""))
            .flatMap(Arrays::stream)
            .distinct()
            .collect(Collectors.toList());

    System.out.println();
    for (String humanNameWord : humanNameWords) {
        System.out.print(humanNameWord + " ");
    }
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku kuku
j o a e h r b i c z n g s u k
```

-----
