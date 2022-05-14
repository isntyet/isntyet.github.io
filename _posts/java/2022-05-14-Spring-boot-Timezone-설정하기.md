---
title: "Spring boot Timezone 설정하기"
date: 2022-05-13T09:30:30-16:00
categories:
  - java
tags:
  - spring boot
  - timezone
  - test
comments: true
---

로컬에서는 분명히 적상적으로 작동을 했는데 서버 올라갔을 때  
비정상으로 작동하여 원인을 확인해 보니 타임존 문제였다.

application에 따로 타임존 설정을 따로 안해줬더니  
로컬(내 맥북)의 시간을 가져와서 KST로 설정 되었던 것이다.

Production 환경은 모두 UTC라 동일한 환경을 만들기 위해  
Application **기본 Timezone을 UTC로 맞추기**로 하자.

---

### Application timezone 설정

Spring boot 에 기본 timezone 설정을 하기는 쉽다.  
SpringBootApplication에 **@PostConstruct** 를 이용하여 타임존 설정을 해주면 된다.

```java
@SpringBootApplication
public class PracticeApplication {

    public static void main(String[] args) {
        SpringApplication.run(PracticeApplication.class, args);

        LocalDateTime now = LocalDateTime.now();
        System.out.println("현재시간 " + now);
    }

    @PostConstruct
    public void init() {
        // timezone 설정
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
    }
}
```

![Spring boot Timezone 설정하기/0.png](/assets/images/Spring boot Timezone 설정하기/0.png)

> **_@PostConstruct_**:  빈이 생성되고, 빈의 의존관계 주입이 완료된 후 호출
>

해당 설정이 안먹을 때는 _SpringApplication.run_ 되기 전에 설정해보자.

```java
@SpringBootApplication
public class PracticeApplication {

    public static void main(String[] args) {
        // timezone 설정
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        SpringApplication.run(PracticeApplication.class, args);

        LocalDateTime now = LocalDateTime.now();
        System.out.println("현재시간 " + now);
    }
}
```  


---

### Junit timezone 설정

하다 보니 또 문제가 발생됬다.  로컬에서 실행해서 하는건 이제 해결되었는데 **Test Code**를 돌릴 때 똑같은 문제가 있었다.

**@SpringBootTest** 시에는 상관없었지만  
(SpringBootTest시 Bean을 등록하게 되는데 Bean이 등록되면 위에서 설정했던 **@PostConstruct** 이 실행되기 때문에)

**@SpringBootTest**  없이 **@Test** 단위 테스트시에는 Bean 등록을 따로 하지 않기 때문에 **@PostConstruct** 를 따로 호출하지 않아서 timezone이 그대로 KST였던 것이다.

Test시에도 TImezone을 설정할 수 있는 방법 중 하나는
각 테스트 class마다 default timezone을 설정해주는 것 이다.

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class HumanTest {

    @BeforeAll
    void setup() {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
    }

    @Test
    void test() {
        LocalDateTime now = LocalDateTime.now();
        System.out.println("시간 " + now);
    }
}
```

---

### Gradle test 에서 timezone 설정

또 다른 방법은 **Gradle** 에 설정하는 방법이 있다.

- build.gradle
    ```java
    dependencies {
        .
        .
        .
    }

    test {
        // timezone 설정
        systemProperty 'user.timezone', 'UTC'
    }
    ```


위 설정을 했다면 아래처럼 Preferences에서 test실행을 intellij가 아니라 Gradle로 하게 바꿔야 한다

![Spring boot Timezone 설정하기/1.png](/assets/images/Spring boot Timezone 설정하기/1.png)

---

### IntelliJ IDEA 에서 timezone 설정

Run tests using 설정을 **IntelliJ IDEA**로 유지하고 테스트를 진행하고 싶다면  
**Edit Configurations...** 에서
![Spring boot Timezone 설정하기/2.png](/assets/images/Spring boot Timezone 설정하기/2.png)

**Edit configuration templates...** 로 들어가면
![Spring boot Timezone 설정하기/3.png](/assets/images/Spring boot Timezone 설정하기/3.png)

JUnit 항목의 **vm option** 에
```
-Duser.timezone=UTC
```
를 추가하면 된다
![Spring boot Timezone 설정하기/4.png](/assets/images/Spring boot Timezone 설정하기/4.png)

---
### 끝.

repo = [https://github.com/isntyet/java-practice/commit/0583e93ac8af8f03c0574646bdc7cdd3c8c312d6](https://github.com/isntyet/java-practice/commit/0583e93ac8af8f03c0574646bdc7cdd3c8c312d6)

---
