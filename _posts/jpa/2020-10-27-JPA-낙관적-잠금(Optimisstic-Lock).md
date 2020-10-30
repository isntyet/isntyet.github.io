---
title: "JPA 낙관적 잠금(Optimisstic Lock)"
date: 2020-10-27T15:30:30-16:00
categories:
  - jpa
tags:
  - jpa
  - optimisstic lock
comments: true
---

###  낙관적 잠금(Optimisstic Lock) 이란?
 - 비선점 잠금이라고 불리기도 함
 - 현실적으로 대부분의 트랜잭션이 충돌이 발생하지 않는다고 낙관적으로 가정하고 잠금
 - 트랜잭션을 커밋하기 전까지는 트랜잭션의 충돌을 알 수 없음
 - Application Level에서의 잠금
 - JPA가 제공하는 버전 관리 기능 사용

-----

### 참고  

* Repository 참고
  - [java-practice](https://github.com/isntyet/java-practice)
  - Human domain 참고
  - inmemory db는 h2사용 (쿼리는 schema.sql, data.sql 참고)
  - db console은 http://localhost:8080/h2 로 접속

------

### Lock 걸지 않고 시도해보기

* Human (Entity)
```java
@Entity
@Data
@NoArgsConstructor
public class Human {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long idx;
    private String name;
    private Integer money;
    private LocalDate birth;

    public Human(String name, Integer money, LocalDate birth) {
        this.name = name;
        this.money = money;
        this.birth = birth;
    }

    public int decreaseMoney(int money) {
        if (this.money - money < 0) {
            throw new IllegalArgumentException("돈이 부족해");
        }
        return this.money -= money;
    }
}
```  

* HumanService.class
```java
@Service
@RequiredArgsConstructor
public class HumanService {
    private final HumanRepository humanRepository;

    public int currentMoney(String name) {
        Human human = humanRepository.findByName(name);
        return human.getMoney();
    }

    @Transactional
    public int decreaseMoney(String name, int money) {
        Human human = humanRepository.findByName(name);
        human.decreaseMoney(money);
        return human.getMoney();
    }
}
```  
이름과 돈을 입력하면 해당하는 사람의 돈을 차감하는 기능을 만들어주자.

* HumanController.class
```java
@RestController
@RequestMapping("/human")
@RequiredArgsConstructor
@Slf4j
public class HumanController {
    private final HumanService humanService;

    @GetMapping("/decrease")
    public String decreaseMoney(@RequestParam(value = "name") String name, @RequestParam(value = "money") int money) {
        String result;
        try{
            humanService.decreaseMoney(name, money);
            result =  "현재 남은돈 : " + humanService.currentMoney(name);
        } catch (Exception e) {
            log.info(e.toString());
            result = "에러났어";
        }
        log.info(result);
        return result;
    }
}
```  
여러번 call을 해보기위한 컨트롤러도 만들어주자.

* 실행, 테스트 해보기  
  ***'조재영'이라는 유저에서 1000원을 동시에! 여러번! 차감 테스트해보자.***

  해당 어플리케이션을 실행하고 터미널에 curl을 이용해서 동시에 여러번 호출을 해보자.  
  터미널 창을 열고  
  `curl url & curl url & curl url & ....` 이런식으로 입력해주면 간단하게 테스트가 가능하다.
```bash
curl 'http://localhost:8080/human/decrease?name=%EC%A1%B0%EC%9E%AC%EC%98%81&money=1000' & curl 'http://localhost:8080/human/decrease?name=%EC%A1%B0%EC%9E%AC%EC%98%81&money=1000' & curl 'http://localhost:8080/human/decrease?name=%EC%A1%B0%EC%9E%AC%EC%98%81&money=1000' & curl 'http://localhost:8080/human/decrease?name=%EC%A1%B0%EC%9E%AC%EC%98%81&money=1000' & curl 'http://localhost:8080/human/decrease?name=%EC%A1%B0%EC%9E%AC%EC%98%81&money=1000'
```
 ![터미널에서 호출해보기](https://drive.google.com/uc?id=1Ccg6KJ2lFj1krdMuvZgTbTv-mY2qz5yC)  

 * 실행 결과
 ![콘솔 로그 보기](https://drive.google.com/uc?id=1emfKLBKnlRVfpcHUhVY3qTnUUxzYa1Y5)  
 ![디비 보기](https://drive.google.com/uc?id=1LbvWloFi3EmSVNZiYFtIs3E3RfdGCSW2)  
 처음 `조재영` 의 값은 `10000원`을 가지고 있었다.  
 다섯번을 호출했으니 5천원이 남아있어야 되지만 남은돈은 `9000원`이다.  
 모든 트랜잭션이 동시에 10000원을 읽어서 1000을 뺐기때문에,  
 다 9000원으로 업데이트 된것이다.



------

### 낙관적 락 구현해보기  
이제 위의 소스를 수정해서 낙관적 락을 구현해보자.

* Entity에 `@Version` 사용
  ```java
  @Entity
  @Getter
  @NoArgsConstructor
  public class Human {
      @Id
      @GeneratedValue(strategy = GenerationType.AUTO)
      private Long idx;
      private String name;
      private Integer money;
      private LocalDate birth;

      @Version
      private Integer version; //여기 추가

      public Human(String name, Integer money, LocalDate birth) {
          this.name = name;
          this.money = money;
          this.birth = birth;
      }
  }
```
낙관적 잠금을 사용하기위해서는 `@Version` 어노테이션을 이용해야하니 추가.  
***해당 테이블에 version 필드를 생성해주자***

_@Version을 사용하면 수정이 될 때 자동으로 버전을 상승시키며,  
조회시점과 버전이 다른경우 __OptimisticLockException__ 예외를 발생시킨다.  
어찌보면 Lock을 건다기보다는 충돌감지에 가깝다_

* 위에 했던 curl테스트 다시 진행 후의 콘솔로그
![콘솔 로그 보기](https://drive.google.com/uc?id=1C8HUnZM1Tk93e18ivWyByQmJGgTRCHjq)  
결과를 보면 5번을 시도하였지만 한번만 성공하고 나머지는 `ObjectOptimisticLockingFailureException`을 발생켰다.

* 하나만 성공한 이유  
위에 설명했다시피 version으로 인해서
변경사항이 생길 때마다 해당 version값을 증가시키고
version 값을 이용해 변경감지를 한다.  
그러므로 최초의 커밋만 성공하고 나머지는 실패된 것이다.
  ```sql
  Hibernate:
    update
        human
    set
        birth=?,
        money=?,
        name=?,
        version=?
    where
        idx=?
        and version=?
  ```
  위 쿼리는 실행될 때 찍어본 쿼리인데,
  where절에서 version 체크를 하여 select할때의 version값과
  같은지 체크 하는 행위를 확인할 수 있다.

* 재시도
```java
@GetMapping("/decrease")
 public String decreaseMoney(@RequestParam(value = "name") String name, @RequestParam(value = "money") int money) {
     String result;
     try {
         humanService.decreaseMoney(name, money);
         result = "현재 남은돈 : " + humanService.currentMoney(name);
     } catch (ObjectOptimisticLockingFailureException oe) {
         log.info("재시도");
         return decreaseMoney(name, money);
     } catch (Exception e) {
         result = e.getMessage();
     }
     log.info(result);
     return result;
 }
```
  이렇게 ObjectOptimisticLockingFailureException 예외를 잡아서  
  재시도를 하게 해주면 요청건들 모두 결국에는 성공하게 되겠지만  
  동시요청이 많을 경우 예외처리에 주의해야할 듯 하다.  
  (이름그대로 대부분 트랜잭션이 충돌이 발생하지 않는다는 낙관적인 경우에만 사용하자)

-----

### LockMode 종류

* 적용  
  ```java
  public interface HumanRepository extends JpaRepository<Human, Integer> {

      Human findByName(String name);

      @Lock(LockModeType.PESSIMISTIC_WRITE) //여기
      @Query("select h from Human h where h.name = :name")
      Human findWithNameForUpdate(@Param("name") String name);
  }
```

* LockModeType.NONE  
  기본, @Version 이 적용되어있으면 낙관적 락 적용 됨  

* LockModeType.OPTIMISTIC  
  Entity를 조회만 해도 버전을 체크함

* LockModeType.OPTIMISTIC_FORCEJNCREMENT  
  Entity를 수정하지 않아도 버전을 강제로 증가시킴.
  연관관계에 있는 다른곳이 수정되었어도 해당 Entity가 버전업 되어야 한다면 사용.

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
10000원에서 1000원씩 열번만 성공하고,  
충돌감지된것은 '충돌감지'를 출력하고 실패처리되며,  
이미 10번 성공한 후의 시도에서는 남은돈이 없다고 출력된다.  
이렇게해서 성공 카운트는 딱 10번이 되게된다.  



-----
