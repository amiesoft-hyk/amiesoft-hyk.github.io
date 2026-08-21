---
layout: post
title: "NOT IN 이 NULL 행을 함께 지운다 - 헤더필터 제외 조건의 함정"
date: 2026-08-21 15:40:00 +0900
categories: [오류해결, SQL]
tags: [SQL, QueryDSL, DevExtreme, 3값논리]
---

## 문제 상황

그리드 헤더필터에서 값 목록 중 **하나만 체크 해제**했더니, 해제한 값뿐 아니라 **빈 값 행까지 함께 사라졌다.**

예를 들어 `등록`, `변경`, 그리고 값이 없는 행이 섞여 있는 컬럼에서 `변경`만 체크 해제하면 기대는 이렇다.

| | 기대 | 실제 |
|---|---|---|
| 등록 | 남음 | 남음 |
| 변경 | 사라짐 | 사라짐 |
| (빈 값) | **남음** | **사라짐** |

## 원인 — SQL의 3값 논리

서버는 제외 조건을 이렇게 만들고 있었다.

```java
if (!excludes.isEmpty()) {
    builder.and(path.notIn(excludes));
}
```

생성되는 SQL은 다음과 같다.

```sql
where change_type_cd NOT IN ('변경')
```

여기가 함정이다. SQL에서 `NULL`과의 비교는 `TRUE`도 `FALSE`도 아닌 **`UNKNOWN`**이다.

```sql
NULL NOT IN ('변경')
-- → NULL <> '변경'
-- → UNKNOWN
```

`WHERE` 절은 `TRUE`인 행만 남기므로, `UNKNOWN`인 행은 **조용히 탈락한다.** 값이 `NULL`인 행은 "제외 대상이 아닌데도" 결과에서 빠진다.

빈 문자열(`''`)은 `NULL`이 아니라서 정상적으로 살아남는다. 그래서 데이터에 따라 재현이 되기도 하고 안 되기도 해서 더 헷갈렸다.

### `IN` 은 왜 문제가 안 되나

포함 조건은 반대로 동작한다.

```sql
NULL IN ('등록')   -- → UNKNOWN → 탈락
```

이건 **의도한 결과**다. 값이 없는 행은 `등록`을 고른 결과에 나오면 안 되니까. 같은 3값 논리인데 포함에서는 맞고 제외에서는 틀린 셈이다.

## 해결 방법

제외 조건에 "값이 없는 행"을 명시적으로 살려준다.

```java
// Before
builder.and(path.notIn(excludes));

// After
builder.and(path.notIn(excludes).or(emptyExpression(path)));
```

```sql
-- 생성되는 SQL
where (change_type_cd NOT IN ('변경')
       OR change_type_cd IS NULL
       OR TRIM(change_type_cd) = '')
```

빈 값 판정은 컬럼 타입에 따라 나뉜다. 문자열은 공백만 있는 값도 빈 값으로 봐야 한다.

```java
private static <T> BooleanExpression emptyExpression(SimpleExpression<T> expression) {
    if (expression instanceof StringExpression stringExpression) {
        StringExpression trimmed = Expressions.stringTemplate("TRIM({0})", stringExpression);
        return stringExpression.isNull().or(trimmed.eq(""));
    }
    return expression.isNull();
}
```

### 날짜 컬럼도 같다

날짜를 범위로 제외할 때도 동일한 함정이 있다.

```java
// Before — 값이 없는 행이 사라진다
excludes.forEach(date -> builder.and(
    path.lt(date.atStartOfDay()).or(path.goe(date.plusDays(1).atStartOfDay()))));

// After
excludes.forEach(date -> builder.and(
    path.lt(date.atStartOfDay())
        .or(path.goe(date.plusDays(1).atStartOfDay()))
        .or(path.isNull())));
```

## 결과

수정 후에는 제외 조건을 걸어도 빈 값 행이 남는다.

```
change_type_cd != '변경' || change_type_cd is null || TRIM(change_type_cd) = ''
```

**주의할 점은 이게 겉보기에 "회귀"처럼 보인다는 것이다.** 이전에는 안 나오던 행이 나오기 시작하므로, 검증하는 사람에게 "제외 시 빈 값 행이 남는 것이 정상"이라고 미리 알려둬야 한다.

## 정리하며

`NOT IN`, `<>`, `NOT LIKE` 처럼 **부정형 조건은 NULL을 만나면 행을 조용히 떨어뜨린다.** 에러도 경고도 없다.

체크리스트로 삼을 만한 것:

- 부정 조건을 쓸 때 그 컬럼이 `NULL` 가능한지 먼저 확인한다
- `NULL` 가능하면 `OR col IS NULL`을 붙일지 의식적으로 결정한다
- 문자열 컬럼은 `NULL`과 빈 문자열이 **다르게 동작**한다는 점을 함께 고려한다

특히 "제외" UI를 서버 필터로 옮길 때 자주 걸린다. 사용자가 생각하는 제외는 "고른 값만 빼기"인데, SQL의 `NOT IN`은 "고른 값과 비교 불가능한 것도 빼기"이기 때문이다.
