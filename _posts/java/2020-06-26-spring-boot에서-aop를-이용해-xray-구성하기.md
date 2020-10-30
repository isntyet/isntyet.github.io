---
title: "spring boot에서 aop를 이용해 xray 구성하기"
date: 2020-06-26T14:30:30-16:00
categories:
  - java
tags:
  - spring boot
  - xray
  - aop
comments: true
---

* Repository = [xray](https://github.com/isntyet/xray)

---

## 시작하기

API 호출을 많이 당하고? 있는 서버가 있는데,
문득 어떤 API를 많이 호출하고 있는지 확인 해보고 싶어져서 AWS의 x-ray를 사용해서 tracing 해보고 싶어졌다  

Spring boot, aop, aws x-ray 를 사용해서 api를 tracing 해보자.  
(해당 포스팅은 로컬 mac환경에서 테스트해 본 것을 기준으로 할 꺼다)

---

## X-RAY 데몬 설치하기

x-ray sdk가 전송하는 데이터가 x-ray 서비스에 도달하려면 데몬이 실행 중 이어야 한다.  
xray 데몬을 설치하기 전에 aws x-ray 서비스에 데이터를 전송할 권한이 있어야하는데, 나는 로컬에서 할 것이기 때문에 로컬에다가 자격증명(credentials) 하면된다.  

aws iam에서 xray 권한을 가진 user를 만들고, cli로 credentials 해준다.  

```bash
$ aws configure
AWS Access Key ID [None]: 엑세스키
AWS Secret Access Key [None]: 시크릿키
Default region name [None]: ap-northeast-2
Default output format [None]: text
```

다음 아래 xray 데몬을 다운 받아야하는데

[AWS X-Ray 데몬](https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-daemon.html#xray-daemon-downloading)  

위 가이드에서 참고하여 다운로드 후 압축해제 해준다.  
(나의 로컬 환경은 OS X라서 aws-xray-daemon-macos-3.x.zip(sig) 를 다운)

이제 다운받은 데몬을 각 환경에 맞게 실행해주면 된다.  

[로컬에서 X-Ray 데몬 실행](http://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-daemon-local.html)

```bash
./xray_mac -o -n ap-northeast-2
```

이렇게 실행하면 아래와 같이 2000 port로 실행 성공한 것을 확인 할 수 있다.  

```bash
[Info] Initializing AWS X-Ray daemon 3.2.0
[Info] Using buffer memory limit of 81 MB
[Info] 1296 segment buffers allocated
[Info] Using region: ap-northeast-2
[Info] HTTP Proxy server using X-Ray Endpoint : https://xray.ap-northeast-2.amazonaws.com
[Info] Starting proxy http server on 127.0.0.1:2000
```

---

## Spring boot에서 dependencies 추가

이제  x-ray 트레이싱을 위해 gradle에 dependency를 추가해 주자.  

[Java용 AWS X-Ray SDK](https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-sdk-java.html#xray-sdk-java-dependencies)

```java
// xray 관련
dependencyManagement {
	imports {
		mavenBom('com.amazonaws:aws-xray-recorder-sdk-bom:2.4.0')
	}
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-aop'
	annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
	annotationProcessor 'org.projectlombok:lombok'

	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	compile 'com.h2database:h2'

	testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}

  // xray 관련
	compile("com.amazonaws:aws-xray-recorder-sdk-core")
	compile("com.amazonaws:aws-xray-recorder-sdk-aws-sdk")
	compile("com.amazonaws:aws-xray-recorder-sdk-aws-sdk-instrumentor")
	compile("com.amazonaws:aws-xray-recorder-sdk-apache-http")
	compile("com.amazonaws:aws-xray-recorder-sdk-spring")
}
```

디펜던시 추가 시 datasource 가 필수인듯(추후에 다시 설명) 해서 jpa도 추가하고, 프로퍼티에서 datasource도 설정해준다.  

aop를 사용해서 트레이싱할 예정이니 aop도 추가해주자.  

---

## 전역 레코더 구성, TracingFilter 설정

아래 코드를 생성하여 AWSXRayRecorderBuilder를 전역에 설정해주고,  
샘플링 룰을 설정할 수 있는 sampling-rules.json 파일도 생성해준다.

이 레코더가 json형태의 데이터(세그먼트)를 생성하여 데몬에 전달하는 듯 하다.  
세그먼트를 생성할 때의 규칙을 해당 설정 json파일(sampling-rules.json)에 정의하면  
허용 url, 허용 method, rate 등 세그먼트에 대한 규칙을 설정할 수 있다.  

```java
package com.isntyet.test.aws.xray.config;

import com.amazonaws.xray.AWSXRay;
import com.amazonaws.xray.AWSXRayRecorderBuilder;
import com.amazonaws.xray.javax.servlet.AWSXRayServletFilter;
import com.amazonaws.xray.plugins.EC2Plugin;
import com.amazonaws.xray.strategy.sampling.CentralizedSamplingStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.servlet.Filter;
import java.net.URL;

@Configuration
public class WebConfig {
    static {
        AWSXRayRecorderBuilder builder = AWSXRayRecorderBuilder.standard().withPlugin(new EC2Plugin());

        URL ruleFile = WebConfig.class.getResource("/sampling-rules.json");
        builder.withSamplingStrategy(new CentralizedSamplingStrategy(ruleFile));
        AWSXRay.setGlobalRecorder(builder.build());
    }

    @Bean
    public Filter TracingFilter() {
        return new AWSXRayServletFilter("JoTest");
    }
}
```

- /src/main/resources/sampling-rules.json

```json
{
  "version": 1,
  "default": {
    "fixed_target": 1,
    "rate": 1
  }
}
```

[X-Ray 콘솔에서 샘플링 규칙 구성](https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-console-sampling.html)

TracingFilter 에서는 AWSXRayServletFilter를 통해서 세그먼트 이름을 fix 할 수 있다.

---

## XrayInspector 생성 (aop)

aop를 활용하면 특별한 로직을 추가하지 않아도 쉽게 설정할 수 있다.  

* [Spring의 AOP 및 X-Ray SDK for Java](https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-sdk-java-aop-spring.html)

* aop 활성화
  ```java
  @SpringBootApplication
  @EnableAspectJAutoProxy
  public class XrayApplication {

  	public static void main(String[] args) {
  		SpringApplication.run(XrayApplication.class, args);
  	}
  }
  ```
  aop를 활성화 해주고

* XrayInspector
  ```java
  package com.isntyet.test.aws.xray.interceptor;

  import com.amazonaws.xray.entities.Subsegment;
  import com.amazonaws.xray.spring.aop.AbstractXRayInterceptor;
  import org.aspectj.lang.ProceedingJoinPoint;
  import org.aspectj.lang.annotation.Aspect;
  import org.aspectj.lang.annotation.Pointcut;
  import org.springframework.stereotype.Component;

  import java.util.Map;

  @Aspect
  @Component
  public class XrayInspector extends AbstractXRayInterceptor {
      @Override
      protected Map<String, Map<String, Object>> generateMetadata(ProceedingJoinPoint proceedingJoinPoint, Subsegment subsegment) {
          return super.generateMetadata(proceedingJoinPoint, subsegment);
      }

      @Override
      @Pointcut("@within(com.amazonaws.xray.spring.aop.XRayEnabled) && bean(*Controller)")
      public void xrayEnabledClasses() {
      }
  }
  ```

  AbstractXRayInterceptor 를 상속받은 XrayInspector를 구현해준다.

  xrayEnabledClasses 포인트컷에서 설정한 것 처럼 XRayEnabled 설정된 Controller bean이 래핑되어 해당 컨트롤러에서 추적이 될것이다.

  이전에 datasource 설정이 필요하다고 했는데 이유는
  AbstractXRayInterceptor 를 들어가보면

  ```java
  @Pointcut("execution(public !void org.springframework.data.repository.Repository+.*(..))")
  protected void springRepositories() {
  }
  ```

  부분에서 data repository 가 포인트컷에 들어가 있기 때문에 datasource 설정을 해주지 않으면 실행이 되지 않는다ㅡㅡ;;

---

## 테스트 controller 만들기

이제 테스트로 호출해 볼 api가 필요하니 controller를 구성해보자.

* XrayController
  ```java
@RestController
@XRayEnabled
public class XrayController {

    private XrayService xrayService;

    public XrayController(XrayService xrayService){
        this.xrayService = xrayService;
    }

    @GetMapping("/test")
    public String test(){
        return "test";
    }

    @GetMapping("/real")
    public String real(){
        String testStr = xrayService.testStr();
        System.out.println(testStr);

        return testStr;
    }
}
```

대충 스트링이 반환되게 구성하고 @XRayEnabled 를 꼭 기입하여 xray 활성화를 시켜준다.

---

## API 테스트

이제 준비가 끝났으니 만들어진 api를 호출해보면 이전에 실행해놓은 데몬에 로그가 찍힐것이다.

* 데몬 로그
  ```bash
[Info] 1296 segment buffers allocated
[Info] Using region: ap-northeast-2
[Info] HTTP Proxy server using X-Ray Endpoint : https://xray.ap-northeast-2.amazonaws.com
[Info] Starting proxy http server on 127.0.0.1:2000
[Info] Successfully sent batch of 1 segments (0.632 seconds)
[Info] Successfully sent batch of 1 segments (0.027 seconds)
```

  이렇게 'Successfully' 라고 뜨면 성공한것이다.

* aws의 x-ray 서비스 콘솔의 트레이스 메뉴
  ![콘솔 확인](https://drive.google.com/uc?id=1eKLTkB0FfVRHwK1APisfdtQVGMvxeDb5)

  위와 같이 트레이싱 되어 url, 평균응답시간, 트레이스 비율 등을 볼 수 있다.  

#### 콘솔 세부 설정은 다음에...  

-----
## 끝.
