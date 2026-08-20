---
layout: post
title: "SelectBox로 고른 값인데 서버가 부분일치로 검색하던 문제"
date: 2026-08-20 17:30:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, Spring, 그리드필터, DevExtreme, 검색조건]
---

그리드 필터행에서 `영업1`을 골랐는데 `영업1지원` 소속까지 함께 조회됐다.
필터행은 목록에서 값을 **선택**하는 SelectBox라 정확일치여야 하는데 서버가 부분일치로 판정하고 있었다.
프론트가 값에 실어 보낸 연산자를 서버 일부 구현이 해석하지 않은 것이 원인이었다.

## 문제 상황

유닛 컬럼 필터행에서 `영업1`을 선택했다.

```
필터행 선택: 영업1
조회 결과 : 영업1 소속 + 영업1지원 소속   ← 후자는 나오면 안 된다
```

같은 값을 **헤더필터**(체크박스)로 고르면 정상이었다. 같은 컬럼인데 필터행과 헤더필터의 결과가 달랐다.

그리고 필터행에는 연산자 선택 메뉴 자체가 없다. 공통 그리드가 선택형 컬럼의 연산자를 고정해두기 때문이다.

```js
if (column.lookup && column.allowFiltering !== false) {
  column.filterOperations = [];
  column.selectedFilterOperation = '=';
}
```

즉 이 컬럼에는 `contains` 같은 연산자가 붙을 수 없다. **부분일치가 나올 여지가 애초에 없어야 한다.**

## 원인

### 프론트는 연산자를 값에 실어 보낸다

이 그리드는 필터행 연산자를 값 문자열에 인코딩해 전달하는 계약을 쓴다.

| 사용자 조작 | 전송 값 |
|---|---|
| SelectBox 선택 (`=`) | `영업1` |
| 텍스트 입력 + "다음을 포함함" | `%영업1%` |
| "다음으로 시작함" | `영업1%` |
| "다름" | `<>영업1` |

공통 조건 빌더는 이 인코딩을 해석해 연산자를 판별한다. 와일드카드가 없으면 `EQUAL`이다.

```java
public enum SearchOperator {
    LIKE("like '%%%s%%'",            List.of(compile("^%[^%]+%$"))),
    LIKE_START_WITH("like '%s%%'",   List.of(compile("^[^%]+%$"))),
    LIKE_END_WITH("like '%%%s'",     List.of(compile("^%[^%]+$"))),
    NOT_EQUAL("<> '%s'",             List.of(compile("^<>[^%,]"))),
    IS_NULL("is null",               List.of(compile("^null$", CASE_INSENSITIVE))),
    EQUAL("= '%s'",                  List.of());   // ← 어디에도 안 걸리면 정확일치
}
```

여기까지는 문제가 없다. 선택형 컬럼은 `%` 없는 값을 보내므로 `EQUAL`로 판정된다.

### 서버 구현이 두 갈래였다

문제는 조직명 컬럼이 **공통 조건 빌더를 타지 못한다**는 점이었다.

공통 빌더는 리플렉션으로 **"DTO 필드명 == 엔티티 필드명"**일 때만 조건을 만든다. 그런데 조직 3단계 이름은 테이블에 없는 **계산 필드**다.

```java
// 엔티티에 있는 컬럼
memberId, memberName, extensionNo, positionCode, orgId   ✅ 매칭됨

// 엔티티에 없음 — 3중 self-join 으로 계산
divisionName, teamName, unitName                          ❌ 매칭 실패
```

부서 계층은 `orgId` 하나에서 부모를 타고 올라가 만든다.

```sql
left join org o1 on o1.id = m.org_id      -- 본인 조직
left join org o2 on o2.id = o1.parent_id  -- 부모
left join org o3 on o3.id = o2.parent_id  -- 조부모

case when o3.id is not null then o2.org_name
     when o2.id is not null then o1.org_name
     else '' end                           -- 팀 이름
```

그래서 조직명 필터는 화면별 전용 로직으로 빠지는데, 그중 일부가 인코딩을 보지 않고 **무조건 contains**로 비교하고 있었다.

```java
// 문제 구현 — % 를 값의 일부로 취급한다
if (hasText(expected)
    && (actual == null || !actual.toLowerCase().contains(expected.trim().toLowerCase()))) {
    return false;
}
```

같은 성격의 코드를 전수로 확인하니 두 부류로 갈려 있었다.

| 판정 로직 | 인코딩 해석 |
|---|---|
| 조건 표현식 방식 A (여러 화면) | ✅ `%`/`<>` 해석 |
| 네이티브 쿼리 LIKE 바인딩 | ✅ 값 그대로 전달 |
| **화면 전용 매칭 3곳** | ❌ **무조건 contains** |

정상 구현 쪽은 이렇게 되어 있었다.

```java
String value = scalar.trim();
boolean negated = value.startsWith("<>");
if (negated) value = value.substring(2);

boolean startsWildcard = value.startsWith("%");
boolean endsWildcard = value.endsWith("%");
String keyword = value.replaceAll("^%|%$", "");

BooleanExpression predicate = startsWildcard && endsWildcard
    ? expression.containsIgnoreCase(keyword)
    : startsWildcard ? expression.endsWithIgnoreCase(keyword)
    : endsWildcard   ? expression.startsWithIgnoreCase(keyword)
    : expression.equalsIgnoreCase(keyword);   // 와일드카드 없으면 정확일치

condition.and(negated ? predicate.not() : predicate);
```

### 같은 컬럼인데 판정이 갈렸던 이유

문제 구현을 다시 보면 **값의 형태에 따라 비교 방식이 달랐다**.

```java
// 필터행 단일값 → 부분일치
if (hasText(expected) && !actual.toLowerCase().contains(expected.toLowerCase())) return false;

// 헤더필터 목록 → 정확일치
if (!includes.isEmpty() && !includes.contains(actual)) return false;
```

필터행으로 고르면 `영업1지원`이 걸리고, 헤더필터로 고르면 안 걸린다. 증상이 조작 방식마다 달랐던 이유다.

## 해결 방법

### 조직 전용 판정은 정확일치로 고정

호출부를 확인하니 두 곳은 조직 3레벨에서만 쓰이는 전용 메서드였다. 부분일치가 들어올 여지가 없으므로 정확일치로 못박았다.

```java
// 조직 필터행은 목록에서 값을 고르는 SelectBox 라 부분일치가 나올 여지가 없다.
// 부분일치로 두면 '영업1' 선택이 '영업1지원' 까지 걸리고,
// 같은 값을 헤더필터로 고를 때(정확일치)와 결과가 달라진다.
if (hasText(expected)
    && (actual == null || !actual.trim().equalsIgnoreCase(expected.trim()))) {
    return false;
}
```

### 공용 메서드는 인코딩을 해석하도록

나머지 한 곳은 조직명과 **이름·사번·직급명이 함께 쓰는** 메서드였다.

```java
addTextFilter(conditions, divisionName, ...);   // 조직 — 선택형
addTextFilter(conditions, member.memberId, ...); // 사번 — 텍스트 입력
addTextFilter(conditions, member.memberName, ...); // 이름 — 텍스트 입력
```

텍스트 입력 컬럼까지 정확일치로 바꾸면 이름 부분검색이 깨진다. 그래서 **호출을 나눴다**.

- 조직 3개 → 정확일치 전용 메서드
- 나머지 → 인코딩 해석 (규칙은 이미 정상 동작하던 구현에서 그대로 옮겨 공용 유틸로 분리)

```java
public final class TextFilter {
    /** 필터행이 값에 실어 보낸 연산자(<>, %)를 해석한다. 와일드카드가 없으면 정확일치. */
    public static BooleanExpression toPredicate(StringExpression expression, String scalar) {
        if (scalar == null || scalar.isBlank()) return null;
        // ... 위의 정상 구현과 동일한 판정
    }
}
```

이 교체로 **덤이 하나 생겼다**. 기존 구현은 화면이 보낸 `%kim%`을 문자 그대로 찾고 있었다.

```java
conditions.and(expression.containsIgnoreCase(keyword.trim()));
//  keyword = "%kim%" → '%kim%' 이라는 문자열을 포함하는 이름을 검색 → 0건
```

즉 그 화면의 이름·사번 부분검색은 **아예 동작하지 않고 있었다**. 인코딩을 해석하면서 함께 정상화됐다.

## 결과

| 조작 | 이전 | 이후 |
|---|---|---|
| 필터행 `영업1` | `영업1` + `영업1지원` | `영업1`만 |
| 헤더필터 `영업1` | `영업1`만 | 동일 |
| 텍스트 입력 "kim 포함" (해당 화면) | 0건 | 정상 검색 |

필터행과 헤더필터의 결과 건수가 같아졌다.

## 정리

- 프론트가 연산자를 값에 인코딩해 보내는 계약이라면, **그 값을 받는 모든 경로가 같은 규칙으로 해석**해야 한다
- 계산 필드(조인·CASE)는 공통 조건 빌더가 매칭하지 못해 화면별 전용 로직으로 빠진다. 그 지점마다 규칙이 재구현되면서 어긋나기 쉽다
- 전용 메서드인지 공용 메서드인지 **호출부를 먼저 확인**해야 한다. 조직 전용이면 정확일치로 고정하는 게 의도가 명확하고, 텍스트 입력과 공유한다면 인코딩을 해석해야 한다
- 규칙이 여러 곳에 흩어져 있으면 하나로 모아두는 편이 낫다. 정상 동작하던 구현이 이미 있었는데 다른 곳에서 다시 짜다가 빠뜨린 경우였다
