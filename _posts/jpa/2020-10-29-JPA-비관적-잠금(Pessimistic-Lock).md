---
title: "JPA 비관적 잠금(Pessimistic Lock)"
date: 2020-10-29T15:30:30-16:00
categories:
  - jpa
tags:
  - jpa
  - pessimistic lock
comments: true
---

###  비관적 잠금(Pessimistic Lock) 이란?
 - 선점 잠금이라고 불리기도 함
 - 트랜잭션끼리의 충돌이 발생한다고 가정하고 우선 락을 거는 방법
 - DB에서 제공하는 락기능을 사용

-----

### 참고  

* Repository 참고
  - [java-practice](https://github.com/isntyet/java-practice)
  - Home domain 참고
  - inmemory db는 h2사용 (쿼리는 schema.sql, data.sql 참고)
  - db console은 http://localhost:8080/h2 로 접속

------

### Lock 걸지 않고 시도해보기

* Home (Entity)
```java
@Entity
@Getter
@NoArgsConstructor
public class Home {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long idx;
    private String name;
    private String address;
    private int price;

    public Home(String name, String address, int price) {
        this.name = name;
        this.address = address;
        this.price = price;
    }

    public int decreasePrice(int price) {
        if (this.price - price < 0) {
            throw new IllegalArgumentException("가격이 부족해");
        }
        return this.price -= price;
    }
}
```  

* HomeRepository
  ```java
  public interface HomeRepository extends JpaRepository<Home, Long> {

      Home findByName(String name);
  }
```

* HomeService.class
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class HomeService {
    private final HomeRepository homeRepository;

    @Transactional
    public int currentPrice(String name) {
        Home home = homeRepository.findByName(name);
        return home.getPrice();
    }

    @Transactional
    public int decreasePrice(String name, int price) {
        Home home = homeRepository.findWithNameForUpdate(name);
        home.decreasePrice(price);
        return home.getPrice();
    }
}
```  
이름과 가격을 입력하면 해당하는 집의 가격이 깎이는 기능을 만들어주자.

* HomeController.class
```java
@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/home")
public class HomeController {
    private final HomeService homeService;

    @GetMapping("/decrease")
    public String decreasePrice(@RequestParam(value = "name") String name, @RequestParam(value = "price") int price) {
        String result;
        try {
            homeService.decreasePrice(name, price);
            result = "현재 가격 : " + homeService.currentPrice(name);
        } catch (Exception e) {
            result = e.getMessage();
        }
        log.info(result);
        return result;
    }
}
```  
여러번 call을 해보기위한 컨트롤러도 만들어주자.

* 실행, 테스트 해보기  
  ***'한옥'이라는 집에 1000원을 동시에! 여러번! 차감 테스트해보자.***

  해당 어플리케이션을 실행하고 터미널에 curl을 이용해서 동시에 여러번 호출을 해보자.  
  터미널 창을 열고  
  `curl url & curl url & curl url & ....` 이런식으로 입력해주면 간단하게 테스트가 가능하다.
```bash
curl 'http://localhost:8080/home/decrease?name=%ED%95%9C%EC%98%A5&price=1000' & curl 'http://localhost:8080/home/decrease?name=%ED%95%9C%EC%98%A5&price=1000' & curl 'http://localhost:8080/home/decrease?name=%ED%95%9C%EC%98%A5&price=1000' & curl 'http://localhost:8080/home/decrease?name=%ED%95%9C%EC%98%A5&price=1000' & curl 'http://localhost:8080/home/decrease?name=%ED%95%9C%EC%98%A5&price=1000'
```
 ![터미널에서 호출해보기](https://drive.google.com/uc?id=1pEe395tMsPm8nJA1aH_HhYBuE92-WPAZ)  

 * 실행 결과
 ![콘솔 로그 보기](https://drive.google.com/uc?id=199JPs8WkRlD8hkDvMGkPP_h7mvP_Dmua)  
 ![디비 보기](https://drive.google.com/uc?id=1pgBCvpPjNEOpKux_tA0qExk-UT3iUpkk)  
 처음 `한옥`의 값은 `20000원`을 가지고 있었다.  
 다섯번을 호출했으니 15000천원이 남아있어야 되지만 남은돈은 `19000원`이다.  
 모든 트랜잭션이 동시에 20000원을 읽어서 1000을 뺐기때문에,  
 다 19000원으로 업데이트 된것이다.



------

### 비관적 락 구현해보기  
이제 위의 소스를 수정해서 비관적 락을 구현해보자.

* HomeRepository
  ```java
  public interface HomeRepository extends JpaRepository<Home, Long> {

      Home findByName(String name);

      @Lock(LockModeType.PESSIMISTIC_WRITE)
      @Query("select h from Home h where h.name = :name")
      Home findWithNameForUpdate(@Param("name") String name);
  }
```
비관적 잠금을 하기 위해 업데이트용 find method를 구현하고  
해당 메소드에 @Lock 어노테이션과 모드를 설정해주자.  
LockModeType은 아래에서 다시 설명.  

* HomeService  
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class HomeService {
    private final HomeRepository homeRepository;

    @Transactional
    public int currentPrice(String name) {
        Home home = homeRepository.findByName(name);
        return home.getPrice();
    }

    @Transactional
    public int decreasePrice(String name, int price) {
        Home home = homeRepository.findWithNameForUpdate(name); //수정
        home.decreasePrice(price);
        return home.getPrice();
    }
}
```

* 위에 했던 curl테스트 다시 진행 후의 콘솔로그
![콘솔 로그 보기](https://drive.google.com/uc?id=1bFX_CS9MgjCNDDEscsCMpyKtcKlsEpro)  
결과를 보면 5번을 시도하여 낙관적 락 일때와는 다르게 전부 순차적으로 가격이 차감된것을 확인 할 수 있다.

* 이유  
  ```sql
  Hibernate:
      select
          home0_.idx as idx1_0_,
          home0_.address as address2_0_,
          home0_.name as name3_0_,
          home0_.price as price4_0_
      from
          home home0_
      where
          home0_.name=? for update
  ```
  위 쿼리는 find 실행될 때 찍어본 쿼리인데,
  ***SELECT FOR ~ UPDATE*** 쿼리가 나가는것을 확인할 수 있다.  
  ***SELECT FOR UPDATE***=_동시성 제어를 위해 특정row에 배타적 LOCK을 거는 행위_  
  ***"데이터 수정하려고 찾은 것이니, 다른분들은 건드리지 마세요!"***  

-----

### LockMode 종류

* 적용방법  
  ```java
  @Transactional
  @Lock(value = LockModeType.PESSIMISTIC_WRITE) //여기
  public int decreasePrice(String name, int price)
```

* LockModeType.PESSIMISTIC_WRITE  
  일반적인 옵션. 데이터베이스에 쓰기 락  
  다른 트랜잭션에서 읽기도 쓰기도 못함. (배타적 잠금)

* LockModeType.PESSIMISTIC_READ  
  반복 읽기만하고 수정하지 않는 용도로 락을 걸 때 사용  
  다른 트랜잭션에서 읽기는 가능함. (공유 잠금)

* LockModeType.PESSINISTIC_FORCE_INCREMENT  
  Version 정보를 사용하는 비관적 락

-----


### 테스트 코드 작성  

* HumanServiceTest
  ```java
@SpringBootTest
class HumanServiceTest {
    @Autowired
    HumanService humanService;

    @Test
    @DisplayName("돈 줄여보기(멀티 스레드) 테스트")
    void decreaseMoneyForMultiThreadTest() throws InterruptedException {
        AtomicInteger successCount = new AtomicInteger();
        int numberOfExcute = 100;
        ExecutorService service = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(numberOfExcute);

        for (int i = 0; i < numberOfExcute; i++) {
            service.execute(() -> {
                try {
                    humanService.decreaseMoney("조재영", 1000);
                    successCount.getAndIncrement();
                    System.out.println("성공");
                } catch (ObjectOptimisticLockingFailureException oe) {
                    System.out.println("충돌감지");
                } catch (Exception e) {
                    System.out.println(e.getMessage());
                }
                latch.countDown();
            });
        }
        latch.await();

        assertThat(successCount.get()).isEqualTo(10);
    }
}
```
이렇게 스레드풀을 생성하고 비동기적으로 여러번 실행시켜보는것으로 테스트가 가능할것 같다.  
총 100번을 시도하는데, 20000원에서 1000원씩 20번만 성공하고  
이미 20번 성공한 후의 시도에서는 가격이 부족하다고 출력된다.  
이렇게해서 성공 카운트는 딱 20번이 되게된다.  



-----
