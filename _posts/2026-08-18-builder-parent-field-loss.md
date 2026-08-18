---
layout: post
title: "@Builder로 검색 DTO를 재조립하면 상속 필드가 사라진다"
date: 2026-08-18 15:41:00 +0900
categories: [오류해결, Lombok]
tags: [Lombok, Spring, QueryDSL, DTO, 그리드필터]
---

그리드 헤더필터에 "(필드 값 없음)" 선택지를 추가했는데, 선택해도 필터가 걸리지 않고 전체가 조회됐다.
요청 파라미터는 정상이었고 컨트롤러까지도 값이 도달했는데 SQL에는 조건이 없었다.
원인은 서비스 계층에서 `@Builder`로 검색 객체를 새로 만들면서 **상위 클래스 필드가 통째로 유실**된 것이었다.

## 문제 상황

빈 값(NULL 또는 공백)인 행만 조회하는 필터를 만들었다. 계약은 이렇다.

- 프론트는 빈 값으로 거를 컬럼명을 CSV로 보낸다 → `blankFields=positionCode`
- 서버는 그 컬럼에 `IS NULL OR TRIM(col) = ''` 조건을 추가한다

프론트에서 요청은 정상적으로 나갔다.

```
GET /api/v1/agents?pagesize=100&blankFields=positionCode&userState=EMPLOYED
```

그런데 응답은 필터가 적용되지 않은 전체 건수(814건)였다. 실행된 SQL을 보니 조건 자체가 없었다.

```sql
WHERE user_state = 'EMPLOYED'
  AND center_dept = 1
  AND del_flag = 'N'
  AND true = true      -- 헤더필터 조건이 비어 있음
```

`AND true = true`는 QueryDSL에서 조건이 하나도 담기지 않은 `BooleanBuilder`가 남긴 흔적이다.
오류도 예외도 없이 **조용히 전체가 조회되는** 형태라 더 찾기 어려웠다.

## 원인

### 계층별로 값이 살아있는지 확인

세 지점을 각각 확인했더니 중간에서 끊긴다는 게 드러났다.

| 지점 | 확인 방법 | 결과 |
|---|---|---|
| 브라우저 → 서버 | 요청 쿼리스트링 | `blankFields=positionCode` **있음** |
| 컨트롤러 | 파라미터 로그 | **도달함** |
| Repository | 실행 SQL | 조건 **없음** |

컨트롤러까지 왔는데 SQL에 없다면 그 사이 서비스 계층이 범인이다.

### @Builder는 상속 필드를 모른다

공통 검색 조건은 상위 클래스에 있었다.

```java
@Getter
@Setter
public class GridSearchBase {
    private String blankFields;        // 빈 값으로 거를 컬럼 목록(CSV)
    private String blankExcludeFields;

    public boolean includesBlank(String field) {
        return parseFields(blankFields).contains(field);
    }
}
```

화면별 검색 조건은 이를 상속한다.

```java
@Builder
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public static class Search extends GridSearchBase {
    private String userState;
    private String positionCode;
    private String positionCodeList;
    // ... 수십 개의 화면 전용 필드
}
```

서비스는 조회 직전에 검색 조건을 한 번 가공한다. 여기가 문제였다.

```java
private Search toQueryCondition(Search source) {
    return Search.builder()
        .userState(source.getUserState())
        .positionCode(source.getPositionCode())
        .positionCodeList(source.getPositionCodeList())
        // ... 자식 필드만 하나씩 옮긴다
        .build();
}
```

**Lombok의 `@Builder`는 그 클래스가 직접 선언한 필드로만 빌더를 만든다.**
상위 클래스인 `GridSearchBase`의 `blankFields`는 빌더 메서드 자체가 존재하지 않으므로,
이 변환을 거치는 순간 값이 사라진다.

```
컨트롤러 (blankFields=positionCode)
   ↓
서비스 toQueryCondition() → 새 객체 생성, 상속 필드 미복사
   ↓
Repository (blankFields=null) → 조건 생성 안 함 → 전체 조회
```

조건이 "잘못 걸리는" 게 아니라 "아예 없어지는" 형태라 예외가 나지 않는다.
빈 결과가 아니라 **전체 결과**가 나오기 때문에 눈에도 잘 띄지 않는다.

## 해결 방법

### 1. 변환 후 상속 필드를 명시적으로 옮긴다

```java
private Search toQueryCondition(Search source) {
    Search target = Search.builder()
        .userState(source.getUserState())
        .positionCode(source.getPositionCode())
        .positionCodeList(source.getPositionCodeList())
        .build();

    // @Builder 는 상위 클래스 필드를 만들어주지 않아 이 변환에서 통째로 유실된다.
    // 조건이 사라지면 오류 없이 전체가 조회되므로 반드시 함께 옮긴다.
    target.setBlankFields(source.getBlankFields());
    target.setBlankExcludeFields(source.getBlankExcludeFields());
    return target;
}
```

주석에 "왜 필요한지"를 남기는 게 중요하다. 이 두 줄은 지워도 컴파일되고 테스트도 통과하기 때문이다.

### 2. 근본적으로는 @SuperBuilder

상속 구조에서 빌더를 쓸 거라면 `@SuperBuilder`가 정답이다.
상위·하위 클래스 **양쪽 모두**에 붙여야 하며, 그러면 빌더가 상속 필드까지 포함한다.

```java
@SuperBuilder
@Getter
@Setter
@NoArgsConstructor
public class GridSearchBase {
    private String blankFields;
}

@SuperBuilder
@Getter
@Setter
@NoArgsConstructor
public static class Search extends GridSearchBase {
    private String userState;
}
```

```java
Search.builder()
    .userState("EMPLOYED")
    .blankFields("positionCode")   // 상속 필드도 빌더에 노출된다
    .build();
```

다만 `@SuperBuilder`는 상위 클래스에도 손을 대야 하고, 기존 `@Builder` 호출부의 동작이 미묘하게 달라질 수 있다.
공통 클래스를 여러 도메인이 상속하고 있다면 영향 범위를 먼저 확인하는 편이 안전하다.
이번에는 변경 범위를 좁히려고 1번을 택했다.

### 3. 진단이 쉽도록 toString에 상위 필드를 포함

이 문제를 찾는 데 가장 오래 걸린 이유는, 파라미터 로그에 상속 필드가 **찍히지 않았기 때문**이다.
Lombok의 `@ToString`은 기본값이 `callSuper = false`라 상위 클래스 필드를 출력하지 않는다.

```java
@ToString(callSuper = true)
public static class Search extends GridSearchBase { ... }
```

상위 클래스에도 `@ToString`을 붙여야 필드가 나온다. 안 붙이면 `super=GridSearchBase@1a2b3c`처럼 해시만 찍힌다.

```java
@Getter
@Setter
@ToString
public class GridSearchBase { ... }
```

적용 후 로그는 이렇게 바뀐다.

```
Search(super=GridSearchBase(blankFields=positionCode, blankExcludeFields=null),
       userState=EMPLOYED, positionCode=null, ...)
```

이 한 줄 덕분에 "컨트롤러까지는 도달했고 그 이후에 사라진다"가 확정됐다.

## 결과

수정 후 SQL에 조건이 정상적으로 붙었다.

```sql
-- 이전
WHERE user_state = 'EMPLOYED' AND true = true

-- 이후
WHERE user_state = 'EMPLOYED' AND position_code IS NULL
```

## 정리

- **`@Builder`는 상속 필드를 빌더에 포함하지 않는다.** 상속 구조에서 빌더로 객체를 재조립하면 상위 필드는 조용히 사라진다.
- 상속 + 빌더 조합이 필요하면 **`@SuperBuilder`** 를 쓴다. 상위·하위 양쪽에 붙여야 한다.
- **검색 조건이 유실되는 버그는 예외가 나지 않는다.** 조건이 사라지면 결과가 0건이 아니라 전체가 되기 때문에 "필터가 안 먹네" 정도로만 보인다. 조건이 확실히 걸려야 하는 로직이라면 조건 없이 조회되는 상황을 로그로 남겨두는 편이 좋다.
- 계층을 넘나드는 값이 사라질 때는 **요청 파라미터 / 컨트롤러 진입 / 실행 SQL** 세 지점을 각각 확인하면 구간이 좁혀진다. 이때 DTO의 `toString`이 상위 필드를 빠뜨리고 있으면 진단이 어려워지므로 `callSuper = true`를 먼저 확인하자.
