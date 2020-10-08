---
title: "github action과 aws code deploy를 이용하여 spring boot 배포하기(1)"
date: 2020-03-10T21:34:30-04:00
categories:
  - deploy
tags:
  - spring boot
  - deploy
  - aws codedeploy
  - github action
comments: true
---

* Repository = [action_codedeploy](https://github.com/isntyet/action_codedeploy)

[github action과 aws code deploy를 이용하여 spring boot 배포하기(1)](https://isntyet.github.io/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(1))    
[github action과 aws code deploy를 이용하여 spring boot 배포하기(2)](https://isntyet.github.io/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(2))  
[github action과 aws code deploy를 이용하여 spring boot 배포하기(3)](https://isntyet.github.io/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(3))  
[github action과 aws code deploy를 이용하여 spring boot 배포하기(4)](https://isntyet.github.io/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(4))  

-----


### 시작하기  

aws ec2에 spring boot 프로젝트를 배포하고싶은데 너무 귀찮다...  
ec2에 접속해서 git에서 프로젝트를 내려받은뒤 빌드를 하고 실행을 해야하는데  
짧게 말했지만 매우 번거롭다. (이걸 또 소스 변경할때마다 매번 배포해야한다;;)  
너무너무 귀찮은니까 git에 push를 했을때 `자동으로` 배포되게 해보자.

전체적인 순서는 아래와 같다.  
`git push` -> `github action` -> `code deploy` -> `ec2 배포` -> `실행`

-----
### 프로젝트에 Workflow YML파일 만들기

* 프로젝트의 github repository의 Actions에서 `set up a workflow yourself` 를 클릭  
 ![workflow yml 만들기](https://drive.google.com/uc?id=1vFltY65YuZcm2kt60Q5VjD23NGTlwxc5)  


* yml 이름을 정하고 파일을 작성한 후 commit, push 하기  
 ![workflow yml 만들기2](https://drive.google.com/uc?id=1b3aUUt8pNxDW99ACsFzKGNCFhsT4PNI4)  


해당 yml 파일은 action에서 실행할 job들을 정의하는 것인데 일단은 build 하는것 까지만 정의를 해보자.  
```yaml
on:
  push:
    branches:
      - master

name: Deploy String boot to Amazon EC2
env:
  PROJECT_NAME: action_codedeploy

jobs:
  deploy:
    name: DEPLOY
    runs-on: ubuntu-18.04

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
        shell: bash

      - name: Build with Gradle
        run: ./gradlew build
        shell: bash
```

대충 내용은 master 브랜치에 push가 되면 pushubuntu-18.04 에서 해당 소스를 내려받고 java 1.8 환경에서 gradlew build를 해주는 내용이다.  
이렇게 yml을 채워주고 (추후 내용을 더 추가) 푸시를 해보자.

* push후 Actions에서 아래와 같이 workflow가 생성되며 step을 진행하는 것을 확인 가능   
  ![workflow 돌려보기](https://drive.google.com/uc?id=1kVSyaENo-2xEfB7NWFgObE8gf6-GNhU1)

* 왠만하면 아래와 같이 해당 workflow는 성공
  ![workflow 돌려보기](https://drive.google.com/uc?id=1t8ON0LY2avof6lBjP317DWckMltbvSwP)  


대략 이런 방식으로 step들을 추가해 가며 진행해보자.


계속...

-----
#### 다음 [github action과 aws code deploy를 이용하여 spring boot 배포하기(2)](https://isntyet.github.io/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(2))  
