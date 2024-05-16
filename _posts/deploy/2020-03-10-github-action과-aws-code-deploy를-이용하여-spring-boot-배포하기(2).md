---
title: "github action과 aws code deploy를 이용하여 spring boot 배포하기(2)"
date: 2020-03-10T22:34:30-04:00
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

[github action과 aws code deploy를 이용하여 spring boot 배포하기(1)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(1)/)    
[github action과 aws code deploy를 이용하여 spring boot 배포하기(2)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(2)/)  
[github action과 aws code deploy를 이용하여 spring boot 배포하기(3)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(3)/)  
[github action과 aws code deploy를 이용하여 spring boot 배포하기(4)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(4)/)  

-----

이제 배포할 곳인 EC2를 생성하고 설정해보자.
EC2를 생성하는 것은 자세히 다루지 않을거다.  
(deploy 관련한 설정만 잘 확인하자)

-----

### EC2 생성  

내가 생성한 EC2 스펙은 아래와 같다.
 * ubuntu 18.04
 * t2.micro

 * TEST라는 이름으로 EC2 생성  
  ![EC2 생성](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/ec2생성하기.png)

-----

### EC2 설정  


서버에 접속해서 jdk 설치를 해주자.  
 * jdk 설치
```bash
sudo apt install openjdk-8-jdk
java -version
```  



그리고 codedeploy를 사용하기 위해 `codedeploy agent를 설치해줘야 한다.  
 * 참고하기 [CodeDeploy agent install](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/codedeploy-agent-operations-install-ubuntu.html)  
```bash
sudo apt-get update
sudo apt-get install ruby
sudo apt-get install wget
cd /home/ubuntu
wget https://bucket-name.s3.region-identifier.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
```  

설치 후 아래 명령어를 입력해서 codedeploy agent가 실행중인지 확인하자.
![codedeploy agent 확인](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/codedeploy agent 설치확인.png)  


-----

### EC2 IAM 설정  


EC2에서 Codedeploy를 사용하기위해서는 IAM Role이 필요하다.
EC2에서 Codedeploy 서비스를 사용할 수 있는 권한을 준다고 생각하면 된다.

 * 서비스 -> IAM -> 역할 -> 역할 만들기  
 ![역할 만들기1](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-1.png)  

 * EC2 클릭
 ![역할 만들기2](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-2.png)  

 * codedeploy 검색 후 CodeDeployFullAccess, S3FullAccess 선택 후 생성 (2개다 꼭!!!)
 ![역할 만들기3](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-3.png)  

 * 생성 확인 (이름은 EC2-Deploy 로 생성하였다)
 ![역할 만들기4](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-4.png)  

 * 다시 EC2로 가서 해당 인스턴스를 선택 후 `작업`->`인스턴스 설정`->`IAM 역할 연결/바꾸기` 클릭
 ![역할 만들기5](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-5.png)  

 * 좀전에 만들었던 IAM Role을 선택
 ![역할 만들기6](/assets/images/github action과 aws code deploy를 이용하여 spring boot 배포하기/2/역할 만들기-6.png)  

이렇게 순서대로 진행하면 EC2에 IAM롤이 부여되어 스스로 Codedeploy서비스를 사용할 수 있게된다.

계속...

-----

#### 다음 [github action과 aws code deploy를 이용하여 spring boot 배포하기(3)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(3))  
