---
layout: post
title: "빈 응답이 \"API 통신 오류\"가 된 이유 — 광범위한 catch(Exception)가 NPE를 삼킨 사례"
date: 2026-07-31 18:00:00 +0900
categories: [오류해결, Jackson]
tags: [Jackson, Spring Boot, RestTemplate, 예외처리]
---

## 문제 상황

공통코드 서버에 "이 코드가 실제로 존재하고 사용 중인가"를 물어보는 검증 로직이 있었다. 특정 코드값으로 저장을 시도하면 이런 메시지가 떴다.

```
API 서버와 통신 중 오류가 발생했습니다. (http://.../codes)
```

그런데 이상한 점이 있었다.

- 공통코드 서버는 **정상 기동 중**이고, 다른 코드값으로는 저장이 잘 된다
- 네트워크 문제도 아니고, 타임아웃도 아니다
- 같은 URL을 직접 호출해보면 **HTTP 200**이 떨어진다

직접 호출해본 응답은 이랬다.

```json
{
  "header": { "resCode": "success" },
  "data": null
}
```

즉 서버는 정상이고, 단지 **조건에 맞는 코드가 하나도 없어서 빈 결과**를 내려준 것이었다. 기대 동작은 "코드 없음 → `false` 반환 → 검증 실패 메시지"인데, 실제로는 "통신 오류"라는 전혀 다른 메시지가 나왔다.

## 원인

문제의 코드는 대략 이런 모양이었다.

```java
public boolean existsActiveCode(String codeKey, String codeValue) {
    try {
        String url = buildCodeSearchUrl(codeKey, codeValue);
        ResponseEntity<String> response = restTemplate
            .exchange(url, HttpMethod.GET, defaultRequestEntity(), String.class);

        ObjectMapper mapper = new ObjectMapper();
        Map<String, Object> result = mapper
            .readValue(response.getBody(), new TypeReference<>() {});

        List<Map<String, Object>> codes = mapper
            .convertValue(result.get("data"), new TypeReference<List<Map<String, Object>>>() {});

        return codes.stream()                       // ← 여기서 NPE
            .anyMatch(code -> codeKey.equals(code.get("codeKey"))
                && codeValue.equals(code.get("codeValue"))
                && "Y".equalsIgnoreCase(String.valueOf(code.get("useFl"))));
    } catch (Exception e) {                          // ← 모든 예외를 여기서 흡수
        handleRemoteError(e);                        // ← 내부에서 통신 오류 예외를 던짐
    }
    return false;
}
```

원인은 **두 가지가 겹친 것**이다.

### 1. `convertValue(null, ...)`은 빈 리스트가 아니라 `null`을 돌려준다

`data`가 `null`인 응답에서 `result.get("data")`는 당연히 `null`이다. 이걸 `ObjectMapper.convertValue()`에 넘기면 어떻게 될까?

```java
List<Map<String, Object>> codes =
    mapper.convertValue(null, new TypeReference<List<Map<String, Object>>>() {});
// codes == null  (빈 List 가 아니다)
```

`convertValue`는 입력이 `null`이면 **타입과 무관하게 `null`을 그대로 반환**한다. 컬렉션 타입으로 요청했으니 빈 리스트가 나올 것 같지만 그렇지 않다. 여기서 바로 `.stream()`을 호출하면 `NullPointerException`이 난다.

`Collections.emptyList()`를 기대하고 방어 코드를 생략하기 쉬운 지점이다. 참고로 JSON에 `"data": []`로 내려오면 정상적으로 빈 리스트가 된다. **`null`과 `[]`가 다르게 동작**한다는 게 핵심이다.

### 2. `catch (Exception e)`가 로직 버그를 통신 오류로 둔갑시켰다

`try` 블록이 **HTTP 호출부터 응답 파싱, 비즈니스 판정까지 전부**를 감싸고 있었다. 그리고 catch는 그 안에서 무슨 일이 나든 "원격 API 통신 오류"로 변환했다.

```java
private void handleRemoteError(Exception e) {
    log.error("", e);
    throw new BusinessException(API_COMMUNICATION_ERROR, apiUrl());
}
```

그 결과 이런 경로가 만들어진다.

```
조회 결과 없음 → data: null → convertValue → null → stream() NPE
   → catch(Exception) → "API 통신 오류" 예외
```

정상 응답을 받은 코드에서, **정상적인 "결과 없음" 상황이 인프라 장애로 보고**된 것이다. 로그에는 NPE 스택트레이스가 남아 있었지만, 예외 메시지가 "통신 오류"라 서버/네트워크부터 확인하느라 원인 파악이 늦어졌다.

넓은 `catch (Exception e)`의 전형적인 대가다. 통신 예외(`RestClientException`)와 파싱 예외(`JsonProcessingException`), 그리고 **우리 로직의 버그(`NullPointerException`)** 가 전부 같은 바구니에 담긴다.

## 해결 방법

### 1. `null` / 빈 목록을 명시적으로 처리

가장 직접적인 수정. `stream()` 호출 전에 결과 없음을 판정해서 빠져나온다.

```java
List<Map<String, Object>> codes = mapper
    .convertValue(result.get("data"), new TypeReference<List<Map<String, Object>>>() {});

// 조회 결과가 없으면(data 가 null 이거나 빈 목록) "코드 없음"으로 처리한다.
// null 인 채로 stream 을 호출하면 NPE 가 catch 로 잡혀 통신 오류로 둔갑한다.
if (codes == null || codes.isEmpty()) {
    return false;
}

return codes.stream().anyMatch(...);
```

`Objects.requireNonNullElseGet(codes, List::of)`로 정규화하거나, `convertValue` 자체를 감싸는 헬퍼를 두는 방법도 있다.

```java
private <T> List<T> toList(ObjectMapper mapper, Object raw, TypeReference<List<T>> type) {
    List<T> converted = mapper.convertValue(raw, type);
    return converted == null ? List.of() : converted;
}
```

응답 파싱 지점이 여러 곳이라면 헬퍼 쪽이 낫다. 같은 클래스 안의 다른 조회 메서드들도 `convertValue` 결과를 그대로 쓰고 있었기 때문에, 잠재적으로 같은 함정을 공유하고 있었다.

### 2. `try` 범위를 통신 구간으로 좁힌다 (권장)

근본적으로는 catch가 감싸는 범위를 줄이는 게 맞다.

```java
public boolean existsActiveCode(String codeKey, String codeValue) {
    String body = fetchCodeSearchResponse(codeKey, codeValue);  // 통신 실패만 여기서 처리
    return parseAndMatch(body, codeKey, codeValue);             // 파싱/판정 오류는 그대로 노출
}

private String fetchCodeSearchResponse(String codeKey, String codeValue) {
    try {
        return restTemplate
            .exchange(buildCodeSearchUrl(codeKey, codeValue),
                HttpMethod.GET, defaultRequestEntity(), String.class)
            .getBody();
    } catch (RestClientException e) {   // Exception 이 아니라 통신 예외만 잡는다
        log.error("공통코드 조회 통신 실패", e);
        throw new BusinessException(API_COMMUNICATION_ERROR, apiUrl());
    }
}
```

이렇게 두면 NPE 같은 우리 쪽 버그는 "통신 오류"로 위장되지 않고 그대로 드러난다. 장애 대응 시 **어디를 봐야 하는지**가 예외 메시지만으로 결정되기 때문에, 이 구분은 생각보다 비용이 크다.

### 3. `return false`가 죽은 코드였다는 점도 확인

원래 코드의 마지막 `return false`는 사실상 도달하지 않는다. `handleRemoteError()`가 항상 예외를 던지기 때문이다. 컴파일은 통과하지만 "예외를 잡아서 false를 돌려주는구나"라고 읽히기 쉬운 구조라, 리뷰 때 오해를 부른다. 예외를 던지는 헬퍼라면 반환 타입을 활용하거나(`throw handleRemoteError(e);`) 호출부에서 흐름이 끊긴다는 걸 드러내는 편이 낫다.

## 결과

- 코드가 존재하지 않는 정상 케이스에서 "통신 오류" 대신 **의도한 검증 실패 메시지**가 노출됨
- 원격 서버 장애와 응답 파싱 버그가 로그·예외 메시지 상에서 구분됨
- 같은 클래스의 다른 조회 메서드도 동일한 `null` 함정을 점검

## 정리

> `ObjectMapper.convertValue(null, ...)`는 컬렉션 타입이어도 **빈 리스트가 아니라 `null`** 을 반환한다. 여기에 통신·파싱·비즈니스 로직을 통째로 감싼 `catch (Exception e)`가 겹치면, 단순한 NPE가 "API 통신 오류"로 둔갑해 엉뚱한 곳을 뒤지게 된다. **catch 범위는 실패 원인을 구분할 수 있는 만큼만** 잡는 것이 원칙이다.
