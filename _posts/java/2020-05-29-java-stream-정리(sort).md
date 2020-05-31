---
title: "java stream 정리(sort)"
date: 2020-05-29T16:30:30-17:00
categories:
  - java
tags:
  - java
  - stream
comments: true
---

* Repository = [java-practice](https://github.com/isntyet/java-practice)

-----

### Sorted

스트림내 요소들에 대해서 정렬하는 작업

-----

### 준비하기  

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

기본적으로 sorted를 사용하려면 정렬하려는 객체에 `Comparable 인터페이스`가 구현되어 있어야함.

* Comparable 구현
```java
public class Human implements Comparable<Human> {

    ...

    @Override
    public int compareTo(Human o) {
        return this.name.compareTo(o.name);
    }
}
```  

```java
@DisplayName("이름 사전순 정렬")
void sortedTest1() {
    printHumans(humans);

    List<Human> sortedHumans = humans.stream()
            .sorted()
            .collect(Collectors.toList());

    printHumans(sortedHumans);
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku
arabia cici haha jojae kuku ssu zzang
```  


-----

### 역순으로 정렬하기  

sorted의 파라미터로 `Comparator.reverseOrder()`를 넘겨주면 됨.

```java
@DisplayName("이름 사전 역순 정렬")
void sortedTest2() {
    printHumans(humans);

    List<Human> sortedHumans = humans.stream()
            .sorted(Comparator.reverseOrder())
            .collect(Collectors.toList());

    printHumans(sortedHumans);
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku
zzang ssu kuku jojae haha cici arabia
```  


-----

### Comparator 파라미터로 넘겨서 정렬하기  

객체에 정의된 compareTo가 아닌 정렬기준을 직접 넘기고 싶으면
직접 Comparator 를 구현해서 넘겨주면 됨.

```java
@DisplayName("돈 순으로 정렬")
void sortedTest3() {
    printHumans(humans);

    List<Human> sortedHumans = humans.stream()
            .sorted(Comparator.comparingInt(Human::getMoney))
            .collect(Collectors.toList());

    printHumans(sortedHumans);
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku
cici kuku haha jojae arabia zzang ssu
```  


-----


### Comparator 파라미터로 넘겨서 정렬하기 (역순)

마찬가지로 `.reverseOrder` 를 사용해주면 됨.

```java
@DisplayName("돈 순으로 정렬 (역순)")
void sortedTest4() {
    printHumans(humans);

    List<Human> sortedHumans = humans.stream()
            .sorted(Comparator.comparingInt(Human::getMoney).reversed())
            .collect(Collectors.toList());

    printHumans(sortedHumans);
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku
ssu zzang arabia jojae haha cici kuku
```  


-----

### 람다 표현식을 사용하여 정렬  

람다 표현식을 사용해서 정렬하는것도 가능하다.  
(하지만 idea에서 아래처럼 replace 권고함.)  
`Comparator.comparingInt(Human::getMoney)`

```java
@DisplayName("람다 표현식을 사용하여 정렬")
void sortedTest5() {
    printHumans(humans);

    List<Human> sortedHumans = humans.stream()
            .sorted((h1, h2) -> h1.getMoney() - h2.getMoney())
            .collect(Collectors.toList());

    printHumans(sortedHumans);
}
```

* 결과
```diff
jojae haha arabia cici zzang ssu kuku
cici kuku haha jojae arabia zzang ssu
```  


-----
