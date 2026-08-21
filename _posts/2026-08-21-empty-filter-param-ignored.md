---
layout: post
title: "파라미터는 서버까지 왔는데 필터가 안 걸린 이유"
date: 2026-08-21 15:20:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, Spring, DevExtreme, Lombok]
---

## 문제 상황

그리드 헤더필터에서 **"(필드 값 없음)"을 체크해도 필터가 걸리지 않고 전체가 조회**되는 문제가 있었다.

프론트는 정상이었다. 공통 그리드 컴포넌트가 공백 선택을 별도 파라미터로 변환해 보내고 있었고, 서버 로그에도 값이 도착해 있었다.

```
GridSearch(emptyFields=changeTypeCd, emptyExcludeFields=null)
```

빈 값은 값 목록(`IN` 절)에 담을 수단이 없다. `null`이나 `''`를 그대로 실으면 파라미터를 거치며 사라지거나 문자열 `"null"`로 뭉개진다. 그래서 프론트는 **"어느 컬럼에서 공백을 골랐는지"를 필드명 목록으로 따로 전달**하는 방식을 쓴다.

```
changeTypeCdList        = 01,02          # 선택한 값
changeTypeCdExcludeList = 03             # 제외한 값
emptyFields             = changeTypeCd   # (필드 값 없음) 선택
```

파라미터는 도착했는데 조건이 안 걸린다. 원인은 서버에 두 층으로 있었다.

## 원인 1 — 검색 DTO가 부모 클래스를 상속하지 않음

공백 필드 목록은 공통 부모 클래스가 담당하고 있었다.

```java
@Getter
@Setter
public class GridSearch {

    private String emptyFields;
    private String emptyExcludeFields;

    public boolean includesEmpty(String field) {
        return parseFields(emptyFields).contains(field);
    }
    // ...
}
```

그런데 일부 화면의 검색 DTO가 이 클래스를 상속하지 않고 있었다.

```java
// Before — 상속이 없다
public static class Search {
    private List<String> changeTypeCdList;
    private List<String> changeTypeCdExcludeList;
    // ...
}
```

필드가 없으니 Spring이 `emptyFields` 파라미터를 **바인딩할 곳 자체가 없고**, `includesEmpty()` 메서드도 존재하지 않는다. 이 화면들은 컬럼 하나가 아니라 **전체 컬럼이 무조건 미동작**이었다.

```java
// After
public static class Search extends GridSearch {
```

`@ToString`도 함께 손봐야 로그에서 확인할 수 있다. Lombok의 `@ToString`은 기본적으로 부모 필드를 찍지 않는다.

```java
@ToString(callSuper = true)   // 이걸 빼면 emptyFields 가 로그에 안 보인다
public static class Search extends GridSearch {
```

## 원인 2 — 필터 헬퍼가 검색 조건 객체를 받지 않음

상속을 고쳐도 안 되는 화면이 남았다. 필터 조건을 만드는 헬퍼가 **값 목록만 받고 있었다.**

```java
// Before
private void addListFilter(BooleanBuilder builder, StringPath path,
    Collection<String> includes, Collection<String> excludes) {
    if (!includes.isEmpty()) builder.and(path.in(includes));
    if (!excludes.isEmpty()) builder.and(path.notIn(excludes));
}
```

공백은 값 목록에 실리지 않으므로, "(필드 값 없음)"만 고르면 `includes`가 비어 있다. 그러면 `if` 조건이 거짓이 되어 **조건이 통째로 사라지고 전체가 조회된다.** 헬퍼는 `search`를 받지 않으니 `includesEmpty()`를 확인할 방법도 없다.

호출부는 컬럼마다 한 줄씩 나열되어 있었다.

```java
addListFilter(builder, hist.userId,      search.getUserIdList(),      search.getUserIdExcludeList());
addListFilter(builder, hist.changeTypeCd, search.getChangeTypeCdList(), search.getChangeTypeCdExcludeList());
addDeptCondition(builder, centerExpression(), search, ..., "centerNm");   // ← 이 헬퍼만 search 를 받는다
```

같은 파일 안에서도 **어떤 컬럼은 되고 어떤 컬럼은 안 되는** 상태였다. 부서명 컬럼만 다른 헬퍼를 쓰고 있었고, 그 헬퍼에는 `search`가 넘어가고 있었기 때문이다.

## 해결 방법

헬퍼가 `search`와 필드명을 받아, 값 목록과 공백 선택을 함께 결합하도록 바꿨다.

```java
public static <T> void applyIncludeExclude(
    BooleanBuilder condition, SimpleExpression<T> expression,
    Collection<T> includeValues, Collection<T> excludeValues,
    GridSearch search, String field
) {
    List<T> includes = normalize(includeValues);
    List<T> excludes = normalize(excludeValues);
    boolean includeEmpty = search != null && search.includesEmpty(field);
    boolean excludeEmpty = search != null && search.excludesEmpty(field);
    BooleanExpression empty = emptyExpression(expression);

    if (!includes.isEmpty() || includeEmpty) {          // ← 값이 없어도 공백만 있으면 진입
        BooleanExpression included = includes.isEmpty() ? null : expression.in(includes);
        if (includeEmpty) {
            included = included == null ? empty : included.or(empty);
        }
        condition.and(included);
    }

    if (!excludes.isEmpty() || excludeEmpty) {
        BooleanExpression excluded = excludes.isEmpty()
            ? null
            : expression.notIn(excludes).or(empty);
        if (excludeEmpty) {
            excluded = excluded == null ? empty.not() : excluded.and(empty.not());
        }
        condition.and(excluded);
    }
}

private static <T> BooleanExpression emptyExpression(SimpleExpression<T> expression) {
    if (expression instanceof StringExpression stringExpression) {
        StringExpression trimmed = Expressions.stringTemplate("TRIM({0})", stringExpression);
        return stringExpression.isNull().or(trimmed.eq(""));
    }
    return expression.isNull();
}
```

핵심은 `if (!includes.isEmpty() || includeEmpty)` 한 줄이다. **값 목록이 비어도 공백 선택이 있으면 조건을 만든다.** 깨져 있던 경로가 정확히 여기였다.

호출부는 인자 두 개만 붙였다.

```java
addListFilter(builder, hist.changeTypeCd,
    search.getChangeTypeCdList(), search.getChangeTypeCdExcludeList(),
    search, "changeTypeCd");
//  ~~~~~~~~~~~~~~~~~~~~~~ 이것만 추가
```

**필드명 문자열은 화면 컬럼의 `dataField`와 반드시 같아야 한다.** 공백 필드 목록이 그 이름으로 오기 때문에, 어긋나면 에러 없이 조용히 무시된다.

## 결과

수정 후 실제 생성되는 SQL이다.

```sql
where del_fl = 'N'
  and (change_type_cd is null or trim(change_type_cd) = '')
  and last_chng_time between '2026-07-01' and '2026-08-21'
```

## 정리하며

이 버그의 본질은 **"컬럼마다 필터 코드를 한 줄씩 나열하는 구조"**였다.

- 컴파일 에러가 안 난다 — 인자 개수가 다른 별개의 메서드니까
- 화면도 멀쩡해 보인다 — 일반 값 필터는 정상 동작하고, 공백만 조용히 무시된다
- 코드를 봐도 열 줄 넘는 호출이 다 비슷하게 생겨서 눈으로 구분이 안 된다

세 가지가 겹치면 누락이 쌓여도 아무도 모른다. 실제로 여러 화면에서 "어떤 컬럼은 되고 어떤 컬럼은 안 되는" 상태로 남아 있었다.

같은 정보를 두 곳에 적는 구조(필터 쪽 나열 + 정렬 쪽 매핑)를 발견하면, 한쪽만 빠져도 드러나지 않는다는 신호로 봐야 한다.
