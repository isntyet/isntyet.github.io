---
title: "JPA Custom Repository 클래스 이름 문제"
date: 2023-03-04T15:30:30-16:00
categories:
  - jpa
tags:
  - jpa
  - custom repository
comments: true
---

QueryDSL 사용을 위해서 JPA Custom Repository를 만드는데 아래와 같은 에러가 났다.

```diff
springframework.data.repository.query.QueryCreationException: 
Could not create query for public abstract java.util.List com.isntyet.java.practice.home.domain.HomeRepositoryCustom.findAllHomesByFilter(java.lang.String); 
Reason: Failed to create query for method public abstract java.util.List com.isntyet.java.practice.home.domain.HomeRepositoryCustom.findAllHomesByFilter(java.lang.String)! No property 'filter' found for type 'Home'; 
nested exception is java.lang.IllegalArgumentException: Failed to create query for method public abstract java.util.List com.isntyet.java.practice.home.domain.HomeRepositoryCustom.findAllHomesByFilter(java.lang.String)! No property 'filter' found for type 'Home'
```

---

## 현재 상태 & 원인

내가 구현한 CustomRepository를 아래와 같이 평범했다.

- HomeRepository
    ```java
    public interface HomeRepository extends JpaRepository<Home, Long>, HomeRepositoryCustom {
        Home findByName(String name);
    }
    ```


- HomeRepositoryCustom
    ```java
    public interface HomeRepositoryCustom {
        List<Home> findAllHomesByFilter(String name);
    }
    ```


- HomeRepositoryCustomImpl
    ```java
    @RequiredArgsConstructor
    @Repository
    public class HomeRepositoryCustomImpl implements HomeRepositoryCustom {
        private final JPAQueryFactory queryFactory;
    
        @Override
        public List<Home> findAllHomesByFilter(String name) {
            return null;
        }
    }
    ```

![JPA Custom Repository 클래스 이름 문제/0.png](/assets/images/JPA Custom Repository 클래스 이름 문제/0.png)

위의 [4.6.1 Customizing Individual Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.single-repository-behavior) 에서 말했듯이
Custom Repository 구현체 클래스 뒤에 ~Impl 만 붙여주면 될 줄 알았다.

그래서 **HomeRepositoryCustomImpl** 라고 대충 이름 지어버린게 문제였다.

---

## 해결

구현체의 이름은 Repository Interface 이름 그대로 뒤에 Impl을 붙여야 한다.
HomeRepository**Custom**Impl 처럼 가운데에 Custom이라는 이름이 들어가버려서 생긴 문제였다.

HomeRepositoryCustomImpl → **HomeRepositoryImpl** 이렇게 수정해주니 문제는 해결되었다.

---

## 끝.
