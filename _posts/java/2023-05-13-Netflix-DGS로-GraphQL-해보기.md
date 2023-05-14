---
title: "Netflix DGS로 GraphQL 해보기"
date: 2023-05-13T15:30:30-16:00
categories:
  - java
tags:
  - java
  - graphql
  - dgs
  - spring boot
comments: true
---

DGS는 "Domain Graph Service"의 약어이고, GraphQL 기반 마이크로서비스 아키텍처에서 사용되는 자바 기반 프레임워크인데  회사에서 DGS Federation(여러 개의 GraphQL 서비스를 하나의 GraphQL 엔드포인트로 노출시키는 방식)을 하기 위해 각 MSA 서비스에서 GraphQL 구현을 해야하는 상황이 생겨서 해보게 되었다.

---

## 의존성 추가하기

문서를 보면 Spring Boot 3.0 이상에서는 최신버전의 DGS를 사용하면되는데, 나같은 경우는 Spring Boot 2.7을 사용중이라 DGS 5.5.x 를 사용해야 했다.  DGS 6.x 을 사용하려면 Spring Boot 3.0이상을 쓰면 되겠다.

```groovy
plugins {
	id 'com.netflix.dgs.codegen' version '5.6.0'
}

repositories {
    mavenCentral()
}

dependencies {
	implementation(platform("com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:5.5.1"))
  implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter")
	implementation("com.netflix.graphql.dgs:graphql-dgs-extended-scalars") // graphql 타입이랑 java type을 맞추기 위해 필요함
}
```

---

## IDEA Plugin 추가하기

조금 더 편하게 DGS 개발을 하기 위해 Intellij IDEA를 사용중이라면 아래 두가지 plugin을 설치해주자

- [GraphQL](https://plugins.jetbrains.com/plugin/8097-graphql)
- [DGS](https://plugins.jetbrains.com/plugin/17852-dgs)

---

## schema 추가하기

이제 스키마 파일을 추가해서 스키마들을 정의할건데 **`src/main/resources/schema`** 경로에 .graphqls 확장자로 파일을 생성하고 schema를 정의해주자.

- src/main/resources/schema/human.graphql
    ```graphql
    type Human {
        idx: Int!
        name: String!
        money: Int
    }

    type Query {
        getHumansByName(name: String): [Human]
    }
    ```

  DGS에서는 .graphql 확장자나 .graphqls 확장자가 크게 차이가 없다고 하니 알아서 선택해서 쓰면 된다.
  나는 기존에 Human이라는 Domain이 존재해서 그것을 그대로 type화 하였다.


---

## DataFetcher 만들기

위에 의존성 추가에서 plugins에 `com.netflix.dgs.codegen` 를 추가했다면 `gradle dgs graphql codegen` 항목에 generateJava가 있는 것을 확인 할 수 있다.

![Netflix DGS로 GraphQL 해보기 0](/assets/images/Netflix DGS로 GraphQL 해보기/0.png)

generateJava를 실행하면 build/generated 에 예시 DataFetcher를 제공해주는데

![Netflix DGS로 GraphQL 해보기 1](/assets/images/Netflix DGS로 GraphQL 해보기/1.png)

이렇게 schema파일을 참고하여 codegen된 샘플 코드를 복사해서 작업을 진행해주면 된다.

```java
@DgsComponent
@RequiredArgsConstructor
public class HumanResolver {
    private final HumanService humanService;

    @DgsData(
            parentType = "Query",
            field = "getHumansByName"
    )
    public List<Human> getHumansByName(@InputArgument String name) {
        return this.humanService.getHumansByName(name);
    }
}
```

sample code에 있는 **DgsDataFetchingEnvironment**를 쓰면 머리아파지니 @InputArgument를 이용해서 파라미터를 구현해주자.

그리고 `@DgsData` 는 getGetHumansByName 처럼 method이름을 그대로 쓸 예정이면 `@DgsQuery` 로 대체 가능하다.

```java
package com.isntyet.java.practice.human.controller;

import com.isntyet.java.practice.human.application.HumanService;
import com.isntyet.java.practice.human.domain.Human;
import com.netflix.graphql.dgs.DgsComponent;
import com.netflix.graphql.dgs.DgsQuery;
import com.netflix.graphql.dgs.InputArgument;
import lombok.RequiredArgsConstructor;

import java.util.List;

@DgsComponent
@RequiredArgsConstructor
public class HumanResolver {
    private final HumanService humanService;

    @DgsQuery
    public List<Human> getHumansByName(@InputArgument String name) {
        return this.humanService.getHumansByName(name);
    }
}
```

---

## Query 실행 해보기

이제 sample 코드 작성이 끝났으니 실행을 해보자.

혹시 실행을 했는데 아래와 같은 에러가 뜬다면

```bash
An attempt was made to call a method that does not exist. The attempt was made from the following location:
	com.apollographql.federation.graphqljava.Federation.ensureFederationDirectiveDefinitionsExist(Federation.java:194)
The following method did not exist:
	'graphql.schema.idl.RuntimeWiring graphql.schema.idl.RuntimeWiring.transform(java.util.function.Consumer)'
```

의존성 문제 때문에 그런것이니 gradle.properties에 아래를 추가해보자

```groovy
graphql-java.version=19.2
```

실행이 되었다면 [http://localhost:8080/graphiql](http://localhost:8080/graphiql) 여기로 접속하면 graphql 테스트를 할 수 있는 화면이 뜰 것이다.

![Netflix DGS로 GraphQL 해보기 2](/assets/images/Netflix DGS로 GraphQL 해보기/2.png)

이곳에서 아까 작성했던 getGetHumansByName query를 실행해보자.

```graphql
{
  getHumansByName(name: "jojo") {
    idx
    name
    money    
  }
}
```

![Netflix DGS로 GraphQL 해보기 3](/assets/images/Netflix DGS로 GraphQL 해보기/3.png)

---

## Mutation 해보기

한김에 mutation도 해보자.
Human 생성을 해볼건데 기존에 Human을 생성하기 위해서 요구되는 포맷은 아래와 같은데

```java
public class CreateHumanRequest {
    private final String name;

    private final Integer money;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private final LocalDate birth;
}
```

해당 dto에 해당하는 input type을 schema에 만들어 주면 된다.

```graphql
input CreateHumanInput {
    name: String!
    money: Int
    birth: Date
}

type Mutation {
    createHuman(input: CreateHumanInput): Human
}
```

저기서 birth 필드의 `Date` type은 그냥 쓰면 graphql에 없는 타입이라 에러가 날텐데
위에서 의존성 추가했던 graphql-dgs-extended-scalars 를 이용하여 Long이나 DateTime 등 java와 mapping되는 여러 type들을 확장할 수 있다.  아래처럼 scalar를 추가해주자.

```graphql
scalar Date

input CreateHumanInput {
    name: String!
    money: Int
    birth: Date
}

type Mutation {
    createHuman(input: CreateHumanInput): Human
}
```

추가한 mutation에 대한 DataFetcher를 똑같이 추가해주자.

```java
@DgsComponent
@RequiredArgsConstructor
public class HumanResolver {
    private final HumanService humanService;

    @DgsQuery
    public List<Human> getHumansByName(@InputArgument String name) {
        return this.humanService.getHumansByName(name);
    }

    // 여기
    @DgsMutation
    public Human createHuman(@InputArgument CreateHumanRequest input) {
        return this.humanService.create(input);
    }
}
```

이제 다시 앱을 실행하고 mutation 을 작성하고 실행해보면…

```graphql
mutation createHuman($input: CreateHumanInput) {
  createHuman(input: $input) {
    idx
    name
    money    
  }
}
```

```json
{
  "input": {
    "name": "jojojo",
    "money": 1,
    "birth": "1991-02-26"
  }
}
```

![Netflix DGS로 GraphQL 해보기 4](/assets/images/Netflix DGS로 GraphQL 해보기/4.png)

다음과 같이 에러가 난다.  input data를 CreateHumanRequest로 변환하다가 실패한 것이다.  추측하기에 아마 reflection 하여 생성하는데 내가 만든 CreateHumanRequest DTO에 기본 생성자가 없어서 인듯하니 바꿔주자.

```java
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CreateHumanRequest {
    private String name;

    private Integer money;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birth;
}
```

이제 다시 실행해보면 정상적으로 생성되는 것을 확인 할 수 있다.

![Netflix DGS로 GraphQL 해보기 5](/assets/images/Netflix DGS로 GraphQL 해보기/5.png)

---

## Custom Scalars

graphql type에서는 java의 LinkedHashSet 같은 타입이 없기 때문에 그럴때는 어떻게 해야하는지 알아보자.
위에서 LocalDate를 사용한 것과 비슷한 맥락인데 [이곳](https://github.com/graphql-java/graphql-java-extended-scalars)을 확인해보면 기본적으로 BigDecimal, Long, Url 등과 같은 타입은 만들어진게 있어서 그대로 사용하면 되지만 LinkedHashSet 같은 타입은 따로 구현을 해줘야 한다.

[여기](https://netflix.github.io/dgs/scalars/)
를 보면 Scalar를 @DgsScalar를 이용해서 커스텀하게 만들 수 있는데 한번 구현해보자.

- config/graphql/scalar/LinkedHashSetScalar.java
    ```java
    @DgsScalar(name = "LinkedHashSet")
    public class LinkedHashSetScalar implements Coercing<LinkedHashSet<String>, List<String>> {

        @Override
        public List<String> serialize(Object value) throws CoercingSerializeException {
            if (value instanceof LinkedHashSet) {
                LinkedHashSet<String> set = (LinkedHashSet<String>) value;
                List<String> list = new ArrayList<>();
                for (String element : set) {
                    list.add(element);
                }
                return list;
            }
            return new ArrayList<>();
        }

        @Override
        public LinkedHashSet<String> parseValue(Object input) throws CoercingParseValueException {
            if (input instanceof List) {
                List<?> inputList = (List<?>) input;
                LinkedHashSet<String> result = new LinkedHashSet<>();
                for (Object element : inputList) {
                    if (element instanceof String) {
                        result.add((String) element);
                    } else {
                        throw new CoercingParseValueException("Invalid input value: " + element);
                    }
                }
                return result;
            }
            throw new CoercingParseValueException("Invalid input value: " + input);
        }

        @Override
        public LinkedHashSet<String> parseLiteral(Object input) throws CoercingParseLiteralException {
            if (input instanceof List) {
                List<?> inputList = (List<?>) input;
                LinkedHashSet<String> result = new LinkedHashSet<>();
                for (Object element : inputList) {
                    if (element instanceof String) {
                        result.add((String) element);
                    } else {
                        throw new CoercingParseValueException("Invalid input value: " + element);
                    }
                }
                return result;
            }
            throw new CoercingParseValueException("Invalid input value: " + input);
        }
    }
    ```

  Coercing 을 implement해서 serialize(내보낼때)와 parseValue,parseLiteral(들어올때)를 구현해주면 된다.  (위 코드는 chatGPT을 이용해서 대충 돌아가게만 일단 만들어서 참고만 하자)

- resoures/schema/human.graphql
    ```graphql
    scalar Date
    scalar LinkedHashSet

    type Human {
        "사람 ID"
        idx: Int!
        "사람 이름"
        name: String!
        "가진 돈"
        money: Int
        "태그"
        tags: LinkedHashSet
    }

    input CreateHumanInput {
        name: String!
        money: Int
        birth: Date
        tags: [String!]
    }

    type Query {
        getHumansByName(name: String): [Human]
    }

    type Mutation {
        createHuman(input: CreateHumanInput): Human
    }
    ```

  `scalar LinkedHashSet` 를 선언해주고 테스트할 필드인 tags를 추가해주자. (Human.java엔 이미 tags가 LinkedHashSet 타입으로 추가 되어 있다)

- build.gradle
    ```groovy
    generateJava {
        typeMapping = [
                "LinkedHashSet": "java.util.LinkedHashSet"
        ]
    }
    ```

  codegen plugin의 정상적인 동작을 위해서 추가한 type에 대한 mapping을 위와같이 진행해주자.


이제 query와 mutation을 실행해보면

- createHuman
    ```graphql
    mutation createHuman($input: CreateHumanInput) {
      createHuman(input: $input) {
        idx
        name
        money
        tags    
      }
    }
    ```

    ```json
    {
      "input": {
        "name": "jojojo",
        "money": 1,
        "birth": "1991-02-26",
        "tags": ["backend", "happy"]
      }
    }
    ```

  ![Netflix DGS로 GraphQL 해보기 6](/assets/images/Netflix DGS로 GraphQL 해보기/6.png)

- getHuman
    ```groovy
    {
      getHumansByName(name: "jojojo") {
        idx
        name
        money
        tags
      }
    }
    ```

  ![Netflix DGS로 GraphQL 해보기 7](/assets/images/Netflix DGS로 GraphQL 해보기/7.png)


위와같이 tags필드가 정상적으로 생성도 되고 조회도 되는 것을 확인 할 수 있다.

---

## 참조

- DGS docs - [https://netflix.github.io/dgs/getting-started/](https://netflix.github.io/dgs/getting-started/)
- Example repo - [https://github.com/isntyet/java-practice/commit/69384a2d3eedc743775ead19220e9c247470472c](https://github.com/isntyet/java-practice/commit/69384a2d3eedc743775ead19220e9c247470472c)

---

## 끝.
