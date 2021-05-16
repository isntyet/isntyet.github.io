---
title: "spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기"
date: 2021-05-15T14:30:30-16:00
categories:
  - java
tags:
  - spring boot
  - kms
  - encrypt
comments: true
---

프로퍼티에 db 연결정보나 api key같은 정보가 그대로 들어갔다가 git 계정이 털리는 등 소스를 탈취 당하게 됬을 경우 아주 곤란해 질 수 있다.  
보안이 필요한 값들은 애초에 암호화 해주면 그런 걱정을 덜 수 있겠다.  
AWS KMS를 이용해서 spring boot 프로젝트의 프로퍼티 값들을 암호화 해보자.  

---

### 시작하기

아래 repo를 보고 진행 해보자.  
[zalando/spring-cloud-config-aws-kms](https://github.com/zalando/spring-cloud-config-aws-kms)  
아래 프로퍼티의 db연결정보를 암호화 하는것이 목표다.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/0.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/0.png)

---

### AWS에서 IAM 생성하기

kms는 aws iam user나 role을 통해서 권한을 가질 수 있으니 우선 local에서 사용할 iam을 만들어 보자.  
AWS → IAM 에서 사용자를 만들어주면 된다. (만드는 방법은 알아서 찾아보자)  
나는 jo-mac이라는 이름으로 user를 만들었다.  

---

### AWS에서 KMS(Key Management Service) 생성하기

aws에 접속해서 KMS를 생성 해 주자.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/1.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/1.png)

키 사양에 대해선 따로 알아보자. 여기선 RSA_4096 사양으로 선택한다.
다음 별칭을 알아서 정하고

`키 관리 권한 정의` 에서 만들었던 iam user를 선택 해 주자.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/2.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/2.png)

`키 사용 권한 정의` 에서도 만들었던 iam user를 선택 해 주자.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/3.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/3.png)

생성을 완료 한 후 `키 ID` 를 기억해 주자.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/4.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/4.png)

팀 단위로 사용해야 할 때는  IAM User 하나를 직접 등록 하지 말고, 사용자 그룹에 Role을 부여해서 해당 Role을  등록해 주는것이 편할 것 같다.

---

### AWS CLI로 암호화 되는지 확인 해 보기

Spring boot 프로젝트에 적용해보기 전에 local에서 aws cli로 암호화, 복호화를 해보자.  
먼저 aws configure을 이용해 credentials 설정을 해주자. (설정법을 모른다면 따로 알아보자.)

```bash
aws configure
```

아래 커맨드를 이용해서 'Hello World'를 `암호화` 해보자.  
key-id는 아까 kms 생성 후 해당 kms에 대한 키 id 이고, 암호화 알고리즘은 `RSAES_OAEP_SHA_1` 와 `RSAES_OAEP_SHA_256` 을 지원한다.  

```bash
aws kms encrypt --key-id 키아이디 --plaintext fileb://<(echo -n '암호화할 내용') --encryption-algorithm 암호화 알고리즘
```

ex)  key-id 가 '123456'이고 암호화할 내용은 'Hello World'이고 알고리즘은 RSAES_OAEP_SHA_256을 사용한다면?

```bash
aws kms encrypt --key-id 123456 --plaintext fileb://<(echo -n 'Hello World') --encryption-algorithm RSAES_OAEP_SHA_256
```

위 커맨드를 사용하면 아래 같이 `CiphertextBlob`가 나오는것을 확인 할 수 있다.

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/5.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/5.png)

`CiphertextBlob` 값이 암호문이라 생각하면 된다.

---

### AWS CLI로 복호화 해보기

위와 반대로 복호화를 해보자. 위에 암호화된 내용을 넣어서 실행하면...

```bash
aws kms decrypt --key-id {{key_id}} --ciphertext-blob fileb://<(echo -n '복호화할 내용' | base64 --decode) --output text --encryption-algorithm 암호화 알고리즘 --query Plaintext | base64 --decode
```

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/6.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/6.png)

요렇게 'Hello World' 로 복호화 되는 것을 확인 할 수 있다.

---

### Spring boot 프로젝트에 dependency 추가하기

이제 프로젝트에 적용해보자.
아래 디펜던시를 추가해주자. (버전은 다 최신버전으로 했다.)

```groovy
implementation("org.zalando:spring-cloud-config-aws-kms:5.1.2")
implementation("com.amazonaws:aws-java-sdk-core:1.11.1019")
implementation("com.amazonaws:aws-java-sdk-kms:1.11.1019")
implementation("com.amazonaws:jmespath-java:1.11.1019")
```

---

### bootstrap.yml 추가하기

project resources경로에 bootstrap.yml 파일을 추가해서 key-id와 사용할 알고리즘을 넣어준다.

```yaml
aws:
  kms:
    keyId: 키 아이디
    encryptionAlgorithm: "RSAES_OAEP_SHA_256"
```

![spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/7.png](/assets/images/spring boot에서 aws kms를 이용해 프로퍼티값 암호화 하기/7.png)

---

### 암복호화 Test code 작성

cli로 암호화 하면 되지만 매번 그러면 귀찮으니까 테스트 코드를 작성해보자.

```java
import com.amazonaws.regions.Regions;
import com.amazonaws.services.kms.AWSKMS;
import com.amazonaws.services.kms.AWSKMSClientBuilder;
import com.amazonaws.services.kms.model.DecryptRequest;
import com.amazonaws.services.kms.model.EncryptRequest;
import com.amazonaws.services.kms.model.EncryptResult;
import com.amazonaws.services.kms.model.EncryptionAlgorithmSpec;
import org.apache.commons.codec.binary.Base64;
import org.junit.jupiter.api.Test;
import org.springframework.test.context.ActiveProfiles;

import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;

@ActiveProfiles("test")
public class KmsTest {
    private static final String KEY_ID = "Key-ID";

    @Test
    void encrypt() {
        final String plaintext = "암호화할 값";

        try {
            AWSKMS kmsClient = AWSKMSClientBuilder.standard()
                    .withRegion(Regions.AP_NORTHEAST_2)
                    .build();

            EncryptRequest request = new EncryptRequest();
            request.withKeyId(KEY_ID);
            request.withPlaintext(ByteBuffer.wrap(plaintext.getBytes(StandardCharsets.UTF_8)));
            request.withEncryptionAlgorithm(EncryptionAlgorithmSpec.RSAES_OAEP_SHA_256);

            EncryptResult result = kmsClient.encrypt(request);
            ByteBuffer ciphertextBlob = result.getCiphertextBlob();

            System.out.println("ciphertextBlob: " + new String(Base64.encodeBase64(ciphertextBlob.array())));
        } catch (Exception e) {
            System.out.println("encrypt fail: " + e.getMessage());
        }
    }

    @Test
    void decrypt() {
        final String encriptedText = "복호화할 값 (암호화된 값)";

        try {
            AWSKMS kmsClient = AWSKMSClientBuilder.standard()
                    .withRegion(Regions.AP_NORTHEAST_2)
                    .build();

            DecryptRequest request = new DecryptRequest();
            request.withCiphertextBlob(ByteBuffer.wrap(Base64.decodeBase64(encriptedText)));
            request.withKeyId(KEY_ID);
            request.withEncryptionAlgorithm(EncryptionAlgorithmSpec.RSAES_OAEP_SHA_256);
            ByteBuffer plainText = kmsClient.decrypt(request).getPlaintext();

            System.out.println("plainText: " + new String(plainText.array()));
        } catch (Exception e) {
            System.out.println("decrypt fail: " + e.getMessage());
        }
    }
}
```

이제 테스트 코드를 이용해 암호화, 복호화 할 수 있다.

---

### 프로퍼티 값 암호화 하기

위 테스트 코드를 통해 프로퍼티 값을 암호화 하자.

프로퍼티의 아래 값을 암호화 하면

```yaml
url: jdbc:mysql://localhost:3306/jojae?serverTimezone=UTC&characterEncoding=UTF-8
```

대충 이런 형태의 값이 나오는데

```
lE2vSpNUiPPVWGb/F3p1zvK9vWIE8s.....길어서 줄임......SOht20JJMP9sGtBXwntdux7mA=
```

아래와 같이 암호화된 값을 넣어주면 되는데 앞에 `{cipher}` 를 포함해서 값을 넣어주면 된다.

```yaml
url: '{cipher}lE2vSpNUiPPVWGb/F3p1zvK9vWIE8s.....길어서 줄임......SOht20JJMP9sGtBXwntdux7mA='
```

런타임시에 모든 프로퍼티를 읽은 후 위의 라이브러리에서 제공되는 파서가 {cipher}가 포함된 프로퍼티 값을 이용하여 복호화 한다.

실행해보면 db가 정상적으로 연결되는 것을 확인 할 수 있다.

---

### Local 환경이 아닌 곳에서 사용하기

위에는 로컬환경 기준으로 하였는데 로컬환경에서 띄울게 아니라면 Role을 만들어 연결해 주자.  
위에서 iam user를 만들어서 해당 user에게 KMS 사용권한을 줬던 것 처럼  
Iam Role을 만들고 해당 Role을 연결 해 주면 된다.

---
## 끝.
