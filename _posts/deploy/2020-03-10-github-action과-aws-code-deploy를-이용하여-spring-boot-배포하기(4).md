---
title: "github action과 aws code deploy를 이용하여 spring boot 배포하기(4)"
date: 2020-03-10T22:55:30-04:00
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

이제 AWS에 해야하는 설정이 모두 끝났으니  
workflow 스텝을 추가하여 CodeDeploy배포를 해보자.

-----

### Test code 작성  

Web을 띄워 테스트해보기위해 Controller에 테스트용 코드를 작성해보자.

```java
@RestController
public class TestController {
    @GetMapping("/test")
    public String test(){
        String testStr = "Hi~~";
        System.out.println(testStr);
        return testStr;
    }
}
```  

* 실행 후 확인
 ![web확인](https://drive.google.com/uc?id=1Dm2Sn-nyBQuInbcnVea0KNP0LIOMRDNd){: .custom-img}  

-----

### Workflow step 추가  

이제 프로젝트의 .github/workflows 경로에 가보면  
이전에 만들어놓은 workflow가 있는데,  
이곳에 step을 추가해서 codedeploy를 동작시켜보자.



이제 AWS상에서 설정할 내용이 모두 끝났다.
다음은 workflow에 step을 추가하여 배포를 해보자.


deploy.yaml 수정  

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

      - name: Make zip file
        run: zip -qq -r ./$GITHUB_SHA.zip .
        shell: bash

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Upload to S3
        run: aws s3 cp --region ap-northeast-2 ./$GITHUB_SHA.zip s3://isntyet-deploy/$PROJECT_NAME/$GITHUB_SHA.zip

      - name: Code Deploy
        run: aws deploy create-deployment --application-name testapp --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name dev --s3-location bucket=isntyet-deploy,bundleType=zip,key=$PROJECT_NAME/$GITHUB_SHA.zip

```  


추가한 step에 대해 자세히 설명해보자면

```yaml
- name: Make zip file
  run: zip -qq -r ./$GITHUB_SHA.zip .
  shell: bash
```  
위 step은  
전에 언급했듯이 CodeDeploy 배포를 하기 위해  
S3를 통해야하는데 압축된 파일로 옮겨져야 하기 때문이다.  
(.zip 또는 .tar .tar.gz 등의 방식도 가능)  

$GITHUB_SHA 는 github action에서 제공하는 기본 환경변수인데  
여기서 확인 가능하다. [기본 환경변수](https://help.github.com/en/actions/configuring-and-managing-workflows/using-environment-variables)
(매번 배포될때 zip파일 이름을 다르게 하기 위해 가져다썼다.)


```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}

```  
위 step은  
S3에 이전 스텝에서 압축한 파일을 S3에 업로드하기 위해  
aws cli credentials 하는 과정인데  
한마디로 aws 서비스를 사용하기위해 인증을 받는다고 생각하면 된다.

여기서 ${{ secrets.AWS_ACCESS_KEY_ID }} 와 같이  
치환코드? 같은것을 볼수 있는데,  
오픈되면 안되는 정보들은 이렇게 다른곳에 저장한후  
deploy할때 불러올 수 있다.

* github 해당 `repository`->`Setting`->`Secrets` 에서 key, value를 추가하면 사용가능
 ![secret var 추가](https://drive.google.com/uc?id=1p0oF4quWHEusKBU0obqCv8a5FnhjY54m){: .custom-img}  

이전 포스팅에서 만들어서 저장해놓았던  
cli용 IAM user의 `access-key`와 `secret-access-key`를
```bash
AWS_ACCESS_KEY_ID 에는 access-key
AWS_SECRET_ACCESS_KEY 에는 secret-access-key
AWS_REGION 에는 ap-northeast-2
```
각 항목에 맞게 저장해놓으면 된다.

* 저장 후 확인
![secret var 추가](https://drive.google.com/uc?id=1jwrPOOEPzT3xv5b8ymnO9VLgYBpamgIA){: .custom-img}  


```yaml
- name: Upload to S3
  run: aws s3 cp --region ap-northeast-2 ./$GITHUB_SHA.zip s3://isntyet-deploy/$PROJECT_NAME/$GITHUB_SHA.zip

- name: Code Deploy
  run: aws deploy create-deployment --application-name testapp --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name dev --s3-location bucket=isntyet-deploy,bundleType=zip,key=$PROJECT_NAME/$GITHUB_SHA.zip
```

위 step은  
이전 step에서 설정한 aws cli credential을 이용해 s3에 업로드 후 배포를 생성하는 작업이다.  
aws s3 cp 명령어를 통해 지정한 `s3:버킷이름/파일path`에 저장하는 내용이고  
aws deploy 명령어를 통해 `testapp` 어플리케이션의 `dev` 배포그룹에  
배포를 생성하는 작업이다.

-----

### appspec.yml 생성  

위 내용까지는 프로젝트를 빌드하고 EC2에 옮기는 작업까지 한다.
이제 EC2배포된 후 의 작업을 정의할 단계인데  
프로젝트 root경로에 `appspec.yml` 이름의 파일이 필요하다.

배포가 실행되면 ec2에 설치된 CodeDeploy agent에서
해당 파일을보고, 받아온 프로젝트를 어디에 저장할지   
그리고 무엇을 실행할지를 정하기 때문에

아래내용으로 해당 파일을 만들어주자.  

```yaml
version: 0.0
os: linux

files:
  - source: /
    destination: /opt/testapp
permissions:
  - object: /opt/testapp/
    owner: ubuntu
    group: ubuntu
    mode: 755
hooks:
  AfterInstall:
    # location은 프로젝트의 root경로를 기준
    - location: deploy.sh
      timeout: 60
      runas: root

```

-----

### deploy.sh 생성  

`appspec.yml`에서 사용된  
location: deploy.sh 구문으로 인해 deploy.sh 파일을 실행해줄텐데
deploy.sh 파일을 만들어보자.

해당파일에서 jar를 실행해준다고 생각하면된다.


```bash
#!/usr/bin/env bash

REPOSITORY=/opt/testapp
cd $REPOSITORY

APP_NAME=action_codedeploy
JAR_NAME=$(ls $REPOSITORY/build/libs/ | grep '.jar' | tail -n 1)
JAR_PATH=$REPOSITORY/build/libs/$JAR_NAME

CURRENT_PID=$(pgrep -f $APP_NAME)

if [ -z $CURRENT_PID ]
then
  echo "> 종료할것 없음."
else
  echo "> kill -9 $CURRENT_PID"
  kill -15 $CURRENT_PID
  sleep 5
fi

echo "> $JAR_PATH 배포"
nohup java -jar $JAR_PATH > /dev/null 2> /dev/null < /dev/null &


```

-----

### git Push  

이제 git push해서 gtihub action과 CodeDeploy 배포를 지켜보자.


* github action 성공
![github action 성공](https://drive.google.com/uc?id=11mcPcaSgemRvsmSX3HOizmlpVewKuQ2T){: .custom-img}  

이렇게 통과되면 s3까지 파일이 올라갔고 codedeploy에 배포하나가 생긴 상태이다.
(실패가 발생했다면 이전 설정중 빠진것이 없는지 되돌아보자)  

다음 AWS CodeDeploy에 가서 해당 애플리케이션에서 확인해보면  
배포가 진행중인것을 확인 할 수 있다.


* CodeDeploy 배포 확인
![CodeDeploy 배포 확인](https://drive.google.com/uc?id=1QSIfa9kXmq32Dr9W_F9SonuSFLiV7QzU){: .custom-img}  

* 해당 배포의 `View Events` 클릭
![CodeDeploy 배포 확인](https://drive.google.com/uc?id=1QJWpKkZBeG_-cR74JsBnjt8kfiyF8Lo6){: .custom-img}  

* 이벤트들 확인
![CodeDeploy 배포 확인2](https://drive.google.com/uc?id=10d4Bk3Be74xb_K3cxeASBrmvc07AIrkO){: .custom-img}

마찬가지로 CodeDeploy 에서 실패가 있다면
CodeDeploy agent에 문제가 있거나 (sudo service codedeploy-agent restart 해보자)  

AfterInstall에서 에러가 났다면 deploy.sh 또는 appspec.yml에 문제가 있는 것 일수도 있으니 다시 확인해보자.  

EC2에 접속해서 jar가 정상적으로 실행되었는지도 확인해보자.  
(/var/log/aws/codedeploy-agent/codedeploy-agent.log 에서 로그를 확인할 수도 있다.)

-----

### Web 들어가보기  

이제 띄워진 web에 접속해보자.
http://ip:8080/test

* 이벤트들 확인
![접속 확인](https://drive.google.com/uc?id=1sZyHX8CWBOBdPdOaL6NXBREbNPV37wJZ){: .custom-img}


만약 접속안된다면 8080 보안그룹을 열려있는지 확인해보자.

-----

### 마치며..  

이제 java프로젝트를 수정하고 push만 해도 서버에 반영할 수 있다.  
엄청 손이 많이가는 작업이지만 길게보면 개이득이니 세팅해놓으면 좋을듯하다.  



-----
#### 끝.
