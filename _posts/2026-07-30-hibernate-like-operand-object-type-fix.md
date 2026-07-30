---
layout: post
title: "Hibernate 6에서 like 검색이 Operand of 'like' is of type 'java.lang.Object' 오류로 실패할 때"
date: 2026-07-30 17:10:00 +0900
categories: [오류해결, Hibernate]
tags: [Hibernate, QueryDSL, JPA, MariaDB]
---

## 문제 상황

그리드 목록 화면의 필터 행에 "변경일시"로 텍스트를 입력해 조회하면 500 오류가 났다. 서버 로그에는 SQL이 실행되기도 전에 예외가 찍혔다.

```
org.springframework.dao.InvalidDataAccessApiUsageException:
org.hibernate.query.SemanticException: Operand of 'like' is of type 'java.lang.Object'
which is not a string (its JDBC type code is not string-like)
```

```
Caused by: org.hibernate.query.SemanticException: ...
    at org.hibernate.query.sqm.internal.TypecheckUtil.assertString(TypecheckUtil.java:561)
    at org.hibernate.query.sqm.tree.predicate.SqmLikePredicate.<init>(SqmLikePredicate.java:61)
```

특이한 점은 **같은 컬럼인데 기간 검색(from ~ to)은 정상**이고, 텍스트 부분 일치 검색만 실패했다는 것이다.

해당 조건을 만드는 코드는 이런 형태였다.

```java
// 화면 표시값과 동일한 포맷으로 비교하려고 DB 함수로 문자열을 만든 뒤 like 검색
StringExpression formattedUpdatedAt = Expressions.stringTemplate(
    "DATE_FORMAT({0}, '%Y-%m-%d %H:%i:%s')", record.updatedAt
);

if (hasText(search.getUpdatedAt())) {
    builder.and(formattedUpdatedAt.contains(search.getUpdatedAt().trim()));  // ❌ 여기서 터짐
}
```

## 원인

`DATE_FORMAT`은 **Hibernate에 등록된 HQL 함수가 아니다.**

QueryDSL 쪽에서는 `Expressions.stringTemplate(...)`이 `StringExpression` 타입이므로 자바 컴파일도 통과하고 `.contains()` 호출도 자연스럽다. 하지만 QueryDSL은 이 템플릿을 **HQL 문자열로 그대로 렌더링**할 뿐이고, 최종적으로 타입을 판단하는 주체는 Hibernate의 HQL 파서다.

Hibernate 6은 등록되지 않은 함수의 반환 타입을 알 수 없어 `java.lang.Object`로 추론한다. 그 상태로 `like`를 만들면 `TypecheckUtil.assertString()`에서 "문자열이 아니다"라며 쿼리 생성 단계에서 예외를 던진다.

정리하면 이렇다.

| 단계 | 타입 인식 |
|---|---|
| QueryDSL (자바) | `StringExpression` — 문제 없음 |
| HQL 파서 (Hibernate) | `java.lang.Object` — **like 피연산자로 거부** |

기간 검색이 멀쩡했던 이유도 여기서 설명된다. 기간 검색은 포맷팅된 문자열이 아니라 원본 날짜 컬럼에 `goe`/`loe`를 걸기 때문에 타입 추론이 필요 없었다.

### 왜 SELECT 절에서는 문제가 없었나

같은 `DATE_FORMAT` 템플릿이 SELECT 절 프로젝션에도 쓰이고 있었는데 그쪽은 멀쩡했다. SELECT 절은 결과를 DTO에 담을 뿐 **연산자 타입 검사를 거치지 않기 때문**이다. 타입 검사는 `like`, `in` 같은 비교 연산에서만 작동한다.

### 문제가 되는 함수와 아닌 함수

같은 코드베이스에서 `TRIM()`을 감싼 템플릿도 쓰고 있었지만 그쪽은 정상이었다.

```java
// 정상 — trim() 은 Hibernate 표준 HQL 함수라 반환 타입이 String 으로 잡힌다
StringExpression trimmed = Expressions.stringTemplate("TRIM({0})", record.name);
builder.and(trimmed.eq(""));
```

즉 **Hibernate가 아는 함수인가**가 갈림길이다. `TRIM`, `LOWER`, `CONCAT` 같은 표준 함수는 타입이 잡히고, `DATE_FORMAT`, `SUBSTRING_INDEX` 같은 DB 벤더 전용 함수는 잡히지 않는다.

## 해결 방법

`cast(... as String)`으로 문자열 타입을 명시했다. HQL의 `cast`는 대상 타입이 명확하므로 파서가 더 이상 추론할 필요가 없다.

```java
// ❌ Before
StringExpression formattedUpdatedAt = Expressions.stringTemplate(
    "DATE_FORMAT({0}, '%Y-%m-%d %H:%i:%s')", record.updatedAt
);

// ✅ After
StringExpression formattedUpdatedAt = Expressions.stringTemplate(
    "cast(DATE_FORMAT({0}, '%Y-%m-%d %H:%i:%s') as String)", record.updatedAt
);
```

이 방법의 장점은 **최종 SQL이 그대로라는 점**이다. `cast`는 HQL 레벨에서만 타입을 알려주는 역할이고, 실행되는 SQL은 여전히 `DATE_FORMAT(...)`이라 화면 표시값과 필터 기준이 어긋나지 않는다.

### 대안: 등록된 함수로 바꾸기

Hibernate 6에는 날짜 포맷팅용 표준 함수 `format()`이 있다. 이쪽을 쓰면 애초에 타입이 String으로 잡힌다.

```java
Expressions.stringTemplate("format({0} as 'yyyy-MM-dd HH:mm:ss')", record.updatedAt)
```

MariaDB/MySQL 방언에서는 결국 `date_format`으로 번역되므로 결과는 같다. 다만 패턴 문법이 Java 스타일(`yyyy-MM-dd`)로 바뀌므로, 이미 여러 곳에서 DB 함수 패턴(`%Y-%m-%d`)을 쓰고 있다면 `cast` 쪽이 변경 범위가 작다.

## 같은 패턴 전수 조사

한 곳을 고친 뒤 같은 함정이 더 있는지 찾아봤다.

```bash
grep -rn "stringTemplate" --include=*.java src/main/java
```

`SUBSTRING_INDEX`로 문자열을 잘라 만든 표현식을 `containsIgnoreCase()`에 넘기는 코드가 한 곳 더 있었다. 아직 아무도 그 화면에서 텍스트 검색을 하지 않아 드러나지 않았을 뿐, 검색하는 순간 동일하게 터질 잠복 버그였다. 같은 방식으로 `cast`를 적용했다.

이런 오류는 **해당 조건으로 검색을 시도해야만 재현**되기 때문에, 하나를 만나면 같은 패턴을 전부 훑어보는 편이 좋다.

## 결과

- 필터 행 텍스트 검색과 헤더 필터(`in`) 모두 정상 동작
- 실행 SQL이 변하지 않아 화면 표시값과 검색 기준이 계속 일치
- 잠복해 있던 동일 유형 버그 1건을 미리 제거

## 정리

> QueryDSL의 `Expressions.stringTemplate`은 **자바 타입만 String으로 만들어줄 뿐**, Hibernate에게 타입을 알려주지 않는다. DB 벤더 전용 함수를 감싼 표현식으로 `like`/`in` 비교를 한다면 `cast(... as String)`으로 타입을 명시하거나, Hibernate 표준 함수로 대체해야 한다.
