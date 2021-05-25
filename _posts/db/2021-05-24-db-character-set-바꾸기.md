---
title: "db character set 바꾸기"
date: 2021-05-24T13:30:30-14:00
categories:
  - db 
tags:
  - mariadb
  - char-set
  - aws-rds
comments: true
---

Spring 어플리케이션에서 maria db의 varchar type의 컬럼에 이모티콘(🍯)을 insert하려 했더니
```
SQLDataException Incorrect string value: '\xF0\x9F\x8D\xAF'....
```
요런 에러가 발생했다.  
저 이상한 형식의 string(\xF0\x9F\x8D\xAF)을 검색해보니 내가 db에 insert하려했던 값인 🍯 이모티콘으로 확인 되었다.   
뭔가 character set이 안맞아서 insert에 실패하는 것 같다.

insert 하려는 테이블을 아래 쿼리로 살펴보면
```sql
show full columns from '테이블 이름';
```
컬럼별 type, collation 등을 확인 할 수 있는데

![db character set 바꾸기/0.png](/assets/images/db character set 바꾸기/0.png)
요렇게 collation이 latin... 로 들어가 있는 것 을 확인 할 수 있다.

아래 쿼리로 db의 캐릭터셋을 **_utf8mb4_**로 바꿔주고 collate도 **_utf8mb4_unicode_ci_**로 바꿔주자.
```sql
ALTER DATABASE 'db이름' CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```  


---


그래도 이모티콘이 insert가 안된다면 아래 쿼리로 global 변수 값을 확인 해 보자.
```sql
show global variables where Variable_name like 'character\_set\_%' or variable_name like 'collation%';
```

![db character set 바꾸기/1.png](/assets/images/db character set 바꾸기/1.png)
DB의 char-set과 collation을 바꿨지만 데이터베이스 기본설정은 바뀌지 않은 것을 확인 할 수 있다.

내가 사용하는 db는 aws rds의 mariadb 10.4 인데  
해당 글로벌 변수를 바꾸기 위해서는 파라미터 그룹을 바꿔 줘야 한다.

aws rds에 들어가서 해당 database의 구성을 확인해 보면
![db character set 바꾸기/2.png](/assets/images/db character set 바꾸기/2.png)

파라미터 그룹이 default.mariadb10.4 로 설정되어 있는데, 해당 파라미터 그룹에서 이전에 조회했던 글로벌 변수들을 확인해보면...
![db character set 바꾸기/3.png](/assets/images/db character set 바꾸기/3.png)
`값` 이 비워져 있는 것을 확인 할 수 있다.

해당 파라미터 그룹은 default라서 수정이 불가능 하니 mariadb 10.4 기반으로 새로운 파라미터 그룹을 만들어 주자.
![db character set 바꾸기/4.png](/assets/images/db character set 바꾸기/4.png)


파라미터 그룹을 만들고 아래처럼 값을 utf8mb4로 바꿔주자.
![db character set 바꾸기/5.png](/assets/images/db character set 바꾸기/5.png)

collation_connection 와 collation_server 값은 **_utf8mb4_unicode_ci_** 로 바꿔주자.
![db character set 바꾸기/6.png](/assets/images/db character set 바꾸기/6.png)

이제 데이터베이스에서 해당 파라미터 그룹으로 바꿔줘야 한다.  
해당 데이터베이스의 수정 → 데이터베이스 옵션 항목에서   
DB 파라미터 그룹을 위에서 만든 파라미터 그룹으로 변경하고 저장한다.
![db character set 바꾸기/7.png](/assets/images/db character set 바꾸기/7.png)

![db character set 바꾸기/8.png](/assets/images/db character set 바꾸기/8.png)

나는 test db라 즉시 적용을 선택했다.

**DB 인스턴스 수정**을 누르고 해당 db 상태값을 보면 **수정 중** 이라고 표시되는데 기다리다가  
**사용 가능**으로 변경되고  
구성 항목에서 파라미터 그룹을 확인 해 보면 **재시작 보류중** 이라고 표시되어 있는데....  
db 재부팅을 해야 적용이 된다...
![db character set 바꾸기/9.png](/assets/images/db character set 바꾸기/9.png)

딱히 다운타임이 문제가 되지 않는다면 db 재부팅을 해주자.

재부팅 후 아래 쿼리로 다시 글로벌 변수를 확인 해 보면 값이 바뀐 것을 확인 할 수 있다.

```sql
show global variables where Variable_name like 'character\_set\_%' or variable_name like 'collation%';
```

![db character set 바꾸기/10.png](/assets/images/db character set 바꾸기/10.png)
