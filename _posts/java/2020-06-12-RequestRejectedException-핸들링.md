---
title: "RequestRejectedException 핸들링"
date: 2020-06-12T16:30:30-17:00
categories:
  - java
tags:
  - java
  - exception
comments: true
---

### RequestRejectedException 핸들링

서버 모니터링, 알럿을 위해 Telegraf를 달아놓고 debug level 이상에서 로그가 찍히고 알럿이 오게 해놨는데,
간헐적으로 계속 error알럿이 울려서 확인해보니 아래와 같은 에러가 발생중이었다.

```java
org.springframework.security.web.firewall.RequestRejectedException: The request was rejected because the URL contained a potentially malicious String "//"
	at org.springframework.security.web.firewall.StrictHttpFirewall.rejectedBlacklistedUrls(StrictHttpFirewall.java:369) ~[spring-security-web-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.security.web.firewall.StrictHttpFirewall.getFirewalledRequest(StrictHttpFirewall.java:336) ~[spring-security-web-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:194) ~[spring-security-web-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:178) ~[spring-security-web-5.2.4.RELEASE.jar:5.2.4.RELEASE]
```

해석해보면 요청URL에 이상한 문자열이 포함되어서 요청이 거부되었다는 소리이다.  
예를들어  URL에

> http://localhost/test//jo

이런식으로 "//" 이런식으로 이상한 문자열이 포함되어 거부된것이었다.

"//" 이외에도 org.springframework.security.web.firewall.StrictHttpFirewall 클래스를 확인해보면 아래처럼 여러 **[블랙리스트 문자열]** 이 있는 것을 확인 할 수 있다.

```java
private static final List<String> FORBIDDEN_ENCODED_PERIOD = Collections.unmodifiableList(Arrays.asList("%2e", "%2E"));
private static final List<String> FORBIDDEN_SEMICOLON = Collections.unmodifiableList(Arrays.asList(";", "%3b", "%3B"));
private static final List<String> FORBIDDEN_FORWARDSLASH = Collections.unmodifiableList(Arrays.asList("%2f", "%2F"));
private static final List<String> FORBIDDEN_DOUBLE_FORWARDSLASH = Collections.unmodifiableList(Arrays.asList("//", "%2f%2f", "%2f%2F", "%2F%2f", "%2F%2F"));
private static final List<String> FORBIDDEN_BACKSLASH = Collections.unmodifiableList(Arrays.asList("\\", "%5c", "%5C"));
private Set<String> encodedUrlBlacklist = new HashSet<>();
private Set<String> decodedUrlBlacklist = new HashSet<>();
private Set<String> allowedHttpMethods = createDefaultAllowedHttpMethods();
private Predicate<String> allowedHostnames = hostname -> true;

public StrictHttpFirewall() {
	urlBlacklistsAddAll(FORBIDDEN_SEMICOLON);
	urlBlacklistsAddAll(FORBIDDEN_FORWARDSLASH);
	urlBlacklistsAddAll(FORBIDDEN_DOUBLE_FORWARDSLASH);
	urlBlacklistsAddAll(FORBIDDEN_BACKSLASH);

	this.encodedUrlBlacklist.add(ENCODED_PERCENT);
	this.encodedUrlBlacklist.addAll(FORBIDDEN_ENCODED_PERIOD);
	this.decodedUrlBlacklist.add(PERCENT);
}
```

---

나는 해당 Exception시에 log가 찍히지 않게하는것이 목적이라
그나마 가장 간단한 방법을 찾다 보니 아래와 같은 방법이 있었다.

```java
@Aspect
@Component
public class FilterChainProxyAdvice {

    @Around("execution(public void org.springframework.security.web.FilterChainProxy.doFilter(..))")
    public void handleRequestRejectedException (ProceedingJoinPoint pjp) throws Throwable {
        try {
            pjp.proceed();
        } catch (RequestRejectedException exception) {
            HttpServletResponse response = (HttpServletResponse) pjp.getArgs()[1];
            response.sendError(HttpServletResponse.SC_BAD_REQUEST);
        }
    }
}
```

AOP 를 이용한 방법인데
대충 설명하자면 Around의 타겟 메소드로 doFIlter를 잡아서 익셉션 처리를 하는것이다.

> pjp.proceed();

에서 doFilter가 실행되면서 "//" 문자열이 URL에 포함되어 들어오면 RequestRejectedException 을 발생시키는데, 해당 Exception을 잡아서 로그가 안찍히게 해주면된다.
