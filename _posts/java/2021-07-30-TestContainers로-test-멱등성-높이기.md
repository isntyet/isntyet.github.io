---
title: "TestContainers로 test 멱등성 높이기"
date: 2021-07-30T09:30:30-16:00
categories:
  - java
tags:
  - spring boot
  - testcontainers
  - test
  - mariadb
comments: true
---

h2 in-memory db에서 테스트를 하였는데 뭔가 이상했다.
production환경에서 사용중인 mariadb로 배포 전에 혹시나 해서 테스트해보기 위해 로컬에서 docker로 mariadb를 띄워서 테스트를 했을 때와 결과가 달랐다....

내가 하고있던 테스트는 jpa 관련해서 `Pessimistic lock` 테스트 중이었는데,
동시에 5개가 들어왔을때 하나만 성공 해야하는 테스트였다.

테스트 결과가 어떻게 나왔냐면

- maria db

    ```
    Hibernate: insert into locker (status, target_date, id) values (?, ?, ?)
    Hibernate: insert into locker (status, target_date, id) values (?, ?, ?)
    Hibernate: insert into locker (status, target_date, id) values (?, ?, ?)
    Hibernate: insert into locker (status, target_date, id) values (?, ?, ?)
    Hibernate: insert into locker (status, target_date, id) values (?, ?, ?)
    [pool-1-thread-3] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1213, SQLState: 40001
    [pool-1-thread-3] o.h.engine.jdbc.spi.SqlExceptionHelper   : Deadlock found when trying to get lock; try restarting transaction
    [pool-1-thread-5] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1213, SQLState: 40001
    [pool-1-thread-4] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1213, SQLState: 40001
    [pool-1-thread-2] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1213, SQLState: 40001
    [pool-1-thread-4] o.h.engine.jdbc.spi.SqlExceptionHelper   : Deadlock found when trying to get lock; try restarting transaction
    [pool-1-thread-5] o.h.engine.jdbc.spi.SqlExceptionHelper   : Deadlock found when trying to get lock; try restarting transaction
    [pool-1-thread-2] o.h.engine.jdbc.spi.SqlExceptionHelper   : Deadlock found when trying to get lock; try restarting transaction
    ```

  이렇게 동시에 insert하는 순간 Deadlock이 발생했는데

- h2 db

    ```
    [pool-1-thread-5] o.h.engine.jdbc.spi.SqlExceptionHelper   : Unique index or primary key violation: "PUBLIC.UK_Q7TY4EN85RSD1VAL96JEUFDLA_INDEX_8 ON PUBLIC.LOCKER(TARGET_DATE) VALUES 4"; SQL statement:
    insert into locker (status, target_date, id) values (?, ?, ?) [23505-200]
    [pool-1-thread-2] o.h.engine.jdbc.spi.SqlExceptionHelper   : Unique index or primary key violation: "PUBLIC.UK_Q7TY4EN85RSD1VAL96JEUFDLA_INDEX_8 ON PUBLIC.LOCKER(TARGET_DATE) VALUES 4"; SQL statement:
    insert into locker (status, target_date, id) values (?, ?, ?) [23505-200]
    [pool-1-thread-1] o.h.engine.jdbc.spi.SqlExceptionHelper   : Unique index or primary key violation: "PUBLIC.UK_Q7TY4EN85RSD1VAL96JEUFDLA_INDEX_8 ON PUBLIC.LOCKER(TARGET_DATE) VALUES 4"; SQL statement:
    insert into locker (status, target_date, id) values (?, ?, ?) [23505-200]
    [pool-1-thread-3] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 23505, SQLState: 23505
    [pool-1-thread-3] o.h.engine.jdbc.spi.SqlExceptionHelper   : Unique index or primary key violation: "PUBLIC.UK_Q7TY4EN85RSD1VAL96JEUFDLA_INDEX_8 ON PUBLIC.LOCKER(TARGET_DATE) VALUES 4"; SQL statement:
    ```

  h2는 Deadlock이 발생하지 않고 내가 걸어놓은 Unique index에서 걸려버린다...

뭔가 h2 는 lock이 걸리지 않고 진행된 느낌이다.
조심스럽게 추측하기에는 각 db에서 지원하는 locktimeout의 차이 때문이 아닐까 생각했다. (뇌피셜)

![TestContainers로 test 멱등성 높이기/0.png](/assets/images/TestContainers로 test 멱등성 높이기/0.png)

출처: [https://blog.mimacom.com/handling-pessimistic-locking-jpa-oracle-mysql-postgresql-derbi-h2/](https://blog.mimacom.com/handling-pessimistic-locking-jpa-oracle-mysql-postgresql-derbi-h2/)

local에서 테스트할 때 어차피 docker로 mariadb를 쭉 띄워놓고 테스트하면 상관 없지만
Azure pipeline이나 Github action 같이 CI 중에 테스트 실행 할 때도 테스트에 대한 `멱등성`을 유지하고 싶어서 찾아보니 `TestContainers` 라는 것이 있었다.

[Testcontainers](https://www.testcontainers.org/)

Testcontainers 를 사용하면 테스트 실행시 내가 설정한 db를 `container`에 띄워서 테스트를 진행 할 수 있다.
한번 사용해 보자.

---

### Testcontainers 의존성 추가

dependency 추가를 해주자.
나는 JUnit 5를 사용중이기 때문에 해당 기준으로 진행을 할 것 이다.

[JUnit 5 Quickstart](https://www.testcontainers.org/quickstart/junit_5_quickstart/)

Quickstart를 참고해서 의존성 추가를 해주자.
내가 테스트에 사용할 db는 mariadb이기 때문에 아래 module 정보를 보고 추가하자.  
다른 db를 추가 해줘야 하면 공식 홈페이지 모듈메뉴를 참조해서 추가해 주면 되겠다.

[MariaDB Module](https://www.testcontainers.org/modules/databases/mariadb/)

```groovy
dependencies {
		implementation 'com.h2database:h2'
    implementation 'org.mariadb.jdbc:mariadb-java-client:2.2.1'
		testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    .
    .
    .
		
    testImplementation('org.assertj:assertj-core:3.15.0')
    testImplementation 'org.testcontainers:testcontainers:1.15.3'
    testImplementation 'org.testcontainers:junit-jupiter:1.15.3'
    testImplementation 'org.testcontainers:mariadb:1.15.3'

    .
    .
}
```

---

### property 설정

이제 spring properties에서 db연결정보를 입력하면 된다.
나는 test용 property를 따로 만들었는데 아래와 같이 만들었다.

```yaml
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    url: jdbc:tc:mariadb:10.2:///test
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
    username: test
    password: test
```

datasource.url 에는 mariadb:10.2 라고 하였는데 내가 production 환경에서 사용하는 mariadb가 10.2 버전이라서 저렇게 사용하였다. docker image라 생각하면 된다.

db name, username, password는 의존성 추가한 org.testcontainers:mariadb를 까보면 default정보를 획득 할 수 있다.

![TestContainers로 test 멱등성 높이기/1.png](/assets/images/TestContainers로 test 멱등성 높이기/1.png)

---

### Test 실행

이제 ActiveProfiles을 test로 해서 테스트 실행을 해보자.

![TestContainers로 test 멱등성 높이기/2.png](/assets/images/TestContainers로 test 멱등성 높이기/2.png)

실행되면 이렇게 docker container를 띄우고 있는게 확인된다!!

---

### 또 문제 발생

이렇게 db를 띄우게 되면 문제점이 있는데 default 설정으로 띄워지기 때문에 charset이나 timezone 같은 설정들을 할 수 가 없다...
그래서 한글 insert test를 하였더니 아래와 같이 에러가 났다.

```
Caused by: java.sql.SQLDataException: (conn=9) Incorrect string value: '\xEC\xA1\xB0\xEC\x9E\xAC...' for column `test`.`human`.`name` at row 1
	at org.mariadb.jdbc.internal.util.exceptions.ExceptionMapper.get(ExceptionMapper.java:167)
	at org.mariadb.jdbc.internal.util.exceptions.ExceptionMapper.getException(ExceptionMapper.java:110)
	at org.mariadb.jdbc.MariaDbStatement.executeExceptionEpilogue(MariaDbStatement.java:228)
```

검색해보면 docker-compose를 이용해서 설정 후 testcontainer를 띄우는 방법이 있는데 귀찮기 떄문에 꼼수를 부려봤다.

- resources/sql/schema-test.sql

    ```sql
    ALTER DATABASE test CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
    ```

  이렇게 charset을 설정하는 쿼리를 작성한 후

- resources/application-test.yaml

    ```yaml
    spring:
      datasource:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:tc:mariadb:10.2:///test
        driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
        username: test
        password: test
        initialization-mode: always
        schema: classpath:sql/schema-test.sql
    ```

  property에 initialization-mode, schema를 추가하여 해당 쿼리를 실행해줬다.

이렇게 문제는 넘겼지만 더 세세한 설정이나
db뿐만아니라 다른 환경들(redis, kafka 등)도 컨테이너에 같이 띄워서 테스트해야 하는 상황이 오게되면 docker-compose를 이용해서 testcontainer를 띄우는 방법을 고려하면 좋겠다.

---
### 끝.
