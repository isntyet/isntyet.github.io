---
title: "github action과 aws code deploy를 이용하여 spring boot 배포하기(3)"
date: 2020-03-10T22:44:30-04:00
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

이제 EC2 설정이 끝났으니 AWS CodeDeploy로 넘어가서 설정해 보자.

-----

### CodeDeploy IAM 설정  

CodeDeploy 설정 전에 EC2에서 IAM만든것처럼  
CodeDeploy IAM이 필요하다.

IAM 설정에 가서 만들어보자.

  * `IAM`->`CodeDeploy` 선택
   ![CodeDeploy iam 설정1](https://drive.google.com/uc?id=1qmSB7-iQi5ZDgf2esio4HPFKNRu5F2Y5)  

  * 선택되어 있는 `CodeDeployRole`그대로 두고 진행
   ![CodeDeploy iam 설정2](https://drive.google.com/uc?id=163rQxde67l-gLznDXNbgFO4pmq4Kh-lg)  

  * 역할만들기
   ![CodeDeploy iam 설정3](https://drive.google.com/uc?id=1wavUQv8Ca9p6NLDpjBZbpTbhYRzj_VdK)  

  * 생성확인
   ![CodeDeploy iam 설정4](https://drive.google.com/uc?id=1bk7Rz2S7Lo9D3MnYIqCRrZ8fnzCgC9GC)  

여기까지 하면 IAM 설정이 끝난다.

-----

### CodeDeploy 생성, 설정  

이제 배포를 위한 CodeDeploy 애플리케이션을 생성해보자.


* `CodeDeploy`->`어플리케이션 생성` (여기선 'testapp' 이름으로 생성)
 ![CodeDeploy 설정1](https://drive.google.com/uc?id=13N9FjbqTOO8IEsvndKCY1chvkSjjXI8O)  

* `배포 그룹 생성` 선택 (예를들어 stage, product 처럼 배포 목적 에따라 구분이 가능하다)
 ![CodeDeploy 설정2](https://drive.google.com/uc?id=1fNg7cIyrE4KgQAxbxolJB3x9B2QFPU5D)  

* 배포그룹 이름과 서비스 역할을 설정 (이전 단계에서 만들어놓은 IAM으로 설정)
 ![CodeDeploy 설정3](https://drive.google.com/uc?id=1O976Zx13kHUDFQuKtEgG0J8SuUSrJjFM)  

* 어플리케이션 배포방법, 환경 설정  
 ![CodeDeploy 설정4](https://drive.google.com/uc?id=1Mc2-gj76MAUCvBeKLbmeKYe6cD0D-Gpe)
`배포방법`은 현재위치방식과 블루그린 방식이 있는데 `무중단배포`를 원하면 `블루그린`으로 해야한다.  
`환경구성`은 어느곳에 배포할 것인지, 배포대상 그룹을 선택하는 것인데  
미리 만들어놓은 EC2가 있으니 Name태그를 이용해 대상을 설정.  

* 배포, 로드밸런서 설정
 ![CodeDeploy 설정5](https://drive.google.com/uc?id=1zJKgfanYvwhtmVabEvPOpe-3C5IZ6uTl)  
 `배포설정`은 해당 링크를 참조하자 [배포설정들](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/deployment-configurations.html)
 `로드밸런서`는 현재 EC2에는 설정안했으니 스킵한다.  
 (배포대상이 Autoscaling 그룹이고 로드밸런서가 물려있을때 설정하면
   배포중인 서버에대한 트래픽차단에 용이하다.)  


여기까지하면 CodeDeploy 생성, 설정은 끝난다.

-----

### 배포용 S3 생성  

CodeDeploy를 이용해 배포를 하려면 S3를 통하게된다.  
프로젝트를 zip으로 압축 후 S3에 업로드하고  
CodeDeploy 해당 파일경로를 이용해 배포대상에 소스를 옮긴다.  
그렇기 떄문에 배포용 S3가 필요하다.  

* S3 생성
  ![S3 생성](https://drive.google.com/uc?id=1E2CqYb3R4w1-EnH_i07oPrFs54C7XBSl)  


-----

### AWS CLI용 IAM user 생성  

다음 작업에서 workflow의 step을 추가할건데  
거기서 필요한 cli에서 사용할 iam user가 필요하기떄문에  
미리 생성해보자.

* `IAM`-> `사용자 추가` 선택
 ![cli iam 설정1](https://drive.google.com/uc?id=1K7jfBbCccWu0gYh3gn_7ta4FJh4fRXKv)  

* 이름, access 유형 설정
 ![cli iam 설정2](https://drive.google.com/uc?id=1rSoVQHe2p78IpIpPCHJBEfjvOUuO81bv)  

* 기존정책 직접연결에서 S3FullAccess 선택
 ![cli iam 설정3](https://drive.google.com/uc?id=1wK2dIH1zpGtLD7csI2huM2XRcaU0UhWR)  

* 기존정책 직접연결에서 CodeDeployFullAccess 선택
 ![cli iam 설정4](https://drive.google.com/uc?id=1Rd9_6f8u2Uo4Jj2n_q42j0aDKtd5y7U-)  

* 확인
 ![cli iam 설정5](https://drive.google.com/uc?id=1noSElOp-XRL95rO9pLtBmMsSXtsihfIV)  

* 만들어지면 `accees-key`와 `secret-access-key`가 생기는데 이걸 꼭 저장해놓자!!!
 ![cli iam 설정5](https://drive.google.com/uc?id=10kxHllF9fiW4ea3TJnQGa3Eh39XxwhUM)   



-----

이제 AWS상에서 설정할 내용이 모두 끝났다.
다음은 workflow에 step을 추가하여 배포를 해보자.


계속...

-----
#### 다음 [github action과 aws code deploy를 이용하여 spring boot 배포하기(4)](/deploy/github-action%EA%B3%BC-aws-code-deploy%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-spring-boot-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0(4)/)
