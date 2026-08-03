---
layout: post
title: "설정 한 줄이 프로필 yml에만 빠져서 애플리케이션이 죽었다 — @ConfigurationProperties와 배포 구조"
date: 2026-08-03 14:10:00 +0900
categories: [오류해결, Spring Boot]
tags: [Spring Boot, ConfigurationProperties, yaml, RestTemplate, 배포]
---

## 문제 상황

로컬에서는 멀쩡히 뜨던 API 서버가 개발 서버에 배포되자마자 기동에 실패했다. 로그는 이렇게 남았다.

```
ERROR o.s.boot.SpringApplication : Application run failed

org.springframework.beans.factory.BeanCreationException:
  Error creating bean with name 'recordingApi' defined in class path resource
  [.../config/RestTemplateConfig.class]:
  Factory method 'recordingApi' threw exception with message:
  Cannot invoke "java.lang.Integer.intValue()" because the return value of
  "ExternalApiProperties.connectTimeoutSec()" is null
```

문제의 빈은 외부 시스템 연동용 `RestTemplate`을 만드는 평범한 `@Bean`이다.

```java
@Bean
public RestTemplate recordingApi() {
    final ExternalApiProperties properties = appProperties.getRecordingApi();
    return restTemplateBuilder
        .connectTimeout(Duration.ofSeconds(properties.connectTimeoutSec()))  // ← 여기서 NPE
        .readTimeout(Duration.ofSeconds(properties.readTimeoutSec()))
        .build();
}
```

`connectTimeoutSec`는 `Integer`고, `Duration.ofSeconds(long)`에 넘기는 순간 언박싱된다. 값이 `null`이면 NPE다. 게다가 `@Bean`이라 **기동 시점에 즉시 생성**되므로 이 연동 기능을 아무도 쓰지 않았는데도 애플리케이션 전체가 뜨지 못한다.

에러 메시지에서 눈여겨볼 부분은 `properties`가 null이 아니라 **`connectTimeoutSec()`의 반환값이 null**이라는 점이다. 즉 설정 객체 자체는 만들어졌고 필드만 비어 있다.

## 원인

### 1. 새 설정을 기본 `application.yml`에만 추가했다

연동 기능을 추가하면서 설정은 여기에만 넣었다.

```yaml
# application.yml
app:
  external:
    recordingApi:
      # 실제 연동 전까지 빈 값으로 유지
      base-url: ""
      authorization: ""
      connect-timeout-sec: 5
      read-timeout-sec: 10
```

로컬에서는 이 파일이 그대로 읽히니 문제가 없었다.

### 2. 배포는 프로필 yml **한 개만** 서버로 올린다

정작 원인은 애플리케이션이 아니라 배포 파이프라인에 있었다. 배포 스크립트가 이런 식이었다.

```groovy
// 프로필별 yml 한 개를 서버의 application.yml 자리에 그대로 덮어쓴다
sshTransfer(
    sourceFiles: "src/main/resources/application-${profile}.yml",
    execCommand: """
        cp -rf ${HOME_PATH}/application-${profile}.yml ${CFG_PATH}/application.yml
    """
)
```

즉 배포 환경에서 실제로 읽히는 설정은 **프로필 yml 하나뿐**이고, 클래스패스의 `application.yml`은 설정 소스로 합류하지 않는다. 로컬에서 당연하게 기대하던 "기본 yml + 프로필 yml 병합"이 배포 환경에서는 성립하지 않았던 것이다.

`application-dev.yml`에는 `recordingApi` 블록이 없으니 하위 값이 전부 비었고, `Integer` 필드는 그대로 `null`이 됐다.

### 3. record 값 객체는 필드가 비어도 조용히 만들어진다

설정 클래스는 생성자 바인딩을 쓰는 record다.

```java
public record ExternalApiProperties(
    String baseUrl,
    String authorization,
    Integer connectTimeoutSec,
    Integer readTimeoutSec
) {}
```

`Integer`는 참조 타입이라 값이 없어도 바인딩 실패가 나지 않는다. **설정 누락이 예외가 아니라 null로 전파되고**, 실제 폭발은 한참 뒤 `Duration.ofSeconds()` 언박싱 지점에서 일어난다. 원인 지점과 증상 지점이 떨어져 있어 로그만 보고는 "왜 dev만?"이 바로 안 보였다.

## 해결 방법

두 층에서 같이 막았다. 설정을 채우는 것만으로는 다음에 프로필이 하나 늘면 똑같이 재발하기 때문이다.

### 1. 프로필 yml 전체에 설정 블록 추가

운영/QA/개발/로컬 등 존재하는 프로필 yml **전부**에 같은 블록을 넣었다. 배포되는 파일이 곧 유일한 설정 소스이므로, 어느 프로필로 배포하든 값이 있어야 한다.

### 2. 바인딩 기본값 지정

`@DefaultValue`로 설정이 없을 때의 값을 선언한다. Spring Boot 표준 방식이고, "이 값이 없으면 5초"라는 의도가 코드에 남는다는 점이 좋다.

```java
import org.springframework.boot.context.properties.bind.DefaultValue;

public record ExternalApiProperties(
    String baseUrl,
    String authorization,
    @DefaultValue("5") Integer connectTimeoutSec,
    @DefaultValue("10") Integer readTimeoutSec
) {}
```

### 3. 값 객체 자체가 null인 경우까지 방어

`@DefaultValue`는 **값 객체가 바인딩될 때** 각 파라미터에 적용된다. 상위 프로퍼티(`app.external.recordingApi`)가 통째로 없어서 객체 자체가 `null`이면 여전히 NPE가 난다. 빈 생성 지점에서 한 번 더 막았다.

```java
private static final int DEFAULT_CONNECT_TIMEOUT_SEC = 5;
private static final int DEFAULT_READ_TIMEOUT_SEC = 10;

@Bean
public RestTemplate recordingApi() {
    final ExternalApiProperties properties = appProperties.getRecordingApi();
    if (properties == null) {
        // 설정이 없어도 기동은 되도록 기본값으로 생성한다.
        log.warn("Recording API configuration is missing. RestTemplate is created with default timeout.");
    }

    final int connectTimeoutSec = (properties == null || properties.connectTimeoutSec() == null)
        ? DEFAULT_CONNECT_TIMEOUT_SEC : properties.connectTimeoutSec();
    final int readTimeoutSec = (properties == null || properties.readTimeoutSec() == null)
        ? DEFAULT_READ_TIMEOUT_SEC : properties.readTimeoutSec();

    return restTemplateBuilder
        .connectTimeout(Duration.ofSeconds(connectTimeoutSec))
        .readTimeout(Duration.ofSeconds(readTimeoutSec))
        .build();
}
```

연동 값(`baseUrl`, `authorization`)이 비어 있는 상태는 그대로 둬도 된다. 실제 호출 시점에 검증해서 막는 로직이 이미 있기 때문에, **기동은 성공하고 기능만 비활성**인 상태가 된다. 설정 누락으로 서버 전체가 못 뜨는 것보다 훨씬 낫다.

## 결과

- 개발 서버 기동 복구
- 어느 프로필로 배포해도 같은 이유로 죽지 않음
- 설정이 빠지면 죽는 대신 `WARN` 로그를 남기고 기본값으로 동작

재현 조건을 로컬에서 확인하려면 배포 서버처럼 프로필 yml 하나만 설정 소스로 주면 된다.

```bash
./gradlew bootRun --args="--spring.config.location=file:src/main/resources/application-dev.yml"
```

`spring.config.location`은 기본 설정 위치를 **대체**한다(추가하려면 `spring.config.additional-location`). 배포 환경과 로컬의 설정 병합 방식이 다를 때 이 차이를 모르면 "로컬에서는 되는데요"가 반복된다.

정리하면 배운 것은 두 가지다.

1. **배포되는 설정 파일이 무엇인지 알고 설정을 추가할 것.** 기본 yml에만 넣은 값은 배포 구조에 따라 없는 값일 수 있다.
2. **설정 누락은 기동 실패가 아니라 기본값으로 흡수시킬 것.** 특히 `@Bean`에서 언박싱되는 `Integer` 설정값은 누락 시 애플리케이션 전체를 멈춘다.
