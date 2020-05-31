---
title: "java stream 정리(filter)"
date: 2020-05-29T15:30:30-16:00
categories:
  - java
tags:
  - java
  - stream
comments: true
---

* Repository = [java-practice](https://github.com/isntyet/java-practice)

-----

### Filter

스트림내 요소에 대해서 필터링하는 작업

-----

### 준비하기  

`java stream`을 사용하는데 아직 미숙한것 같아서 여러가지 예제를 사용해보며 연습해보자.

* sample data (Human)  

| 번호 | 이름 | 가진돈 | 생일 |
|:---:|:------|:------|:------:|
| 1 | jojae | 2900 | 1991-02-26 |
| 2 | haha | 1000 | 2003-03-02 |
| 3 | arabia | 30000 | 2001-04-06 |
| 4 | cici | 150 | 1982-05-16 |
| 5 | zzang | 40000 | 1910-06-26 |
| 6 | ssu | 200000 | 2012-07-11 |
| 7 | kuku | 150 | 1991-02-27 |


```java
public class Human {
    private Long idx;
    private String name;
    private Integer money;
    private LocalDate birth;
}
```  

-----

### 기본 사용법  
```java
@DisplayName("이름이 zzang인 사람")
void filterTest1() {
    Human human = humans.stream()
        .filter(h -> h.getName().equals("zzang"))
        .findFirst()
        .orElseThrow(() -> new IllegalArgumentException());

    System.out.println(human.getIdx());
}
```

* 결과
```diff
5
```  


-----


### 메소드 참조 사용법  
이름이 zz로 시작하는 사람을 찾을때  
`h -> h.getName().startsWith("zz")`  
대신  
`Human::isNameStartwithZz`  
이렇게 해당 메소드가 정의되어있다면 `메소드 참조`를 사용해서 간략하게 필터링 가능

```java
@DisplayName("이름이 zz로 시작하는 사람(메소드 참조 사용)")
void filterTest2() {
    Human human = humans.stream()
        .filter(Human::isNameStartwithZz)
        .findFirst()
        .orElseThrow(() -> new IllegalArgumentException());

    System.out.println(human.getIdx());
}
```

* 결과
```diff
5
```  


-----  


### 조건에 맞는 여러요소 가져오기  

`findFirst()` 는 조건에맞는 첫번째 요소를 가져오지만  
전부를 가져오고 싶으면 사용하지 않으면됨.

`collect(Collectors.toList());`  
를 이용해서 List로 반환 가능.

```java
@DisplayName("돈이 2000원 이상인 사람 전부")
void filterTest3() {
    List<Human> tmpHumans = humans.stream()
            .filter(h -> h.getMoney() > 2000)
            .collect(Collectors.toList());

    for (Human human : tmpHumans) {
        System.out.println(human.getIdx());
    }
}
```

* 결과
```diff
1
3
5
6
```

-----


### 여러조건에 맞는 여러요소 가져오기  

```java
@DisplayName("이름이 zzang이고 돈이 2000원 이상인 사람")
void filterTest4() {
    List<Human> tmpHumans = humans.stream()
            .filter(h -> "zzang".equals(h.getName()) && h.getMoney() > 2000)
            .collect(Collectors.toList());

    for (Human human : tmpHumans) {
        System.out.println(human.getIdx());
    }
}
```

* 결과
```diff
5
```

-----
