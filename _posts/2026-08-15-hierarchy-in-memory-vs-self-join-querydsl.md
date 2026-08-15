---
layout: post
title: "부서 수만큼 WHEN이 늘어나는 ORDER BY — 계층 데이터를 메모리에서 계산해 SQL로 펼친 대가"
date: 2026-08-15 18:30:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, JPA, SpringBoot, MariaDB]
---

## 문제 상황

이력 조회 화면에서 그리드의 **부서 컬럼(본부/팀/파트)으로 정렬하거나 헤더 필터를 걸면**
p6spy 로그에 이상한 SQL이 찍혔다.

```sql
select ...
from record_history h
left join user_master u on h.user_id = u.user_id
where h.del_fl = 'N'
  and u.dept_id in (4, 10, 11, 12, 13, 14, 7, 8, 9)   -- ① 부서 ID 전량 나열
  and h.chng_time >= '2026-07-16T00:00:00.000'
order by
  case
    when (u.dept_id is null) then ''
    when (u.dept_id = 1)  then '본사'
    when (u.dept_id = 2)  then '본사'
    when (u.dept_id = 3)  then '본사'
    ...                                                -- ② WHEN 62개
    when (u.dept_id = 62) then '본사'
    else ''
  end,
  h.id desc
limit 0, 100
```

두 가지가 눈에 걸린다.

- **①** 팀 하나를 선택했을 뿐인데 `IN` 절에 부서 ID 9개가 나열된다 (선택한 팀 + 하위 파트들)
- **②** `ORDER BY`의 `CASE`에 **부서 개수만큼 WHEN이 생긴다.** 게다가 조직 루트가 하나뿐이라
  62개 `THEN` 값이 전부 `'본사'`로 같다 → 정렬 키가 사실상 상수라 **`h.id desc`만 동작한다**

즉 "본부 컬럼 정렬"은 동작하지 않으면서 쿼리 길이만 부서 수에 비례해 늘어나고 있었다.

## 원인

두 증상의 뿌리는 하나다. 이 Repository가 **부서 계층을 SQL이 아니라 자바 메모리에서 계산**하고
있었다. 부서 트리 전체를 캐시에서 꺼내 순회한 뒤, 그 결과를 QueryDSL 표현식으로 "펼쳐서"
쿼리에 밀어 넣는 구조다.

### 원인 1 — 정렬 표현식이 부서 1건당 WHEN 1개

```java
// 부서 트리를 순회하며 dept_id → 부서명 매핑을 CASE로 전개한다
private StringExpression deptNameSortExpression(int level) {
    CaseBuilder.Cases<String, StringExpression> cases =
        new CaseBuilder().when(user.deptId.isNull()).then("");

    for (Dept dept : deptTreeService.getDeptTree().getValues()) {   // ← 부서 전체 순회
        String deptName = deptNameAt(dept, level);
        if (dept.getId() != null && StringUtils.hasText(deptName)) {
            cases = cases.when(user.deptId.eq(dept.getId())).then(deptName);
        }
    }
    return cases.otherwise("");
}
```

부서가 62개면 WHEN 62개, 500개면 WHEN 500개가 된다. **쿼리 크기가 데이터 건수에 비례**하는
구조 자체가 잘못됐다.

`deptNameAt()`은 부서의 전체 경로 문자열(`본사 > 영업2팀 > 1파트`)을 `>`로 잘라 레벨별
이름을 꺼내는 헬퍼였다.

```java
// 경로 문자열을 잘라 레벨별 이름을 얻는다
public static DeptPathNames of(String pathNmFull) {
    List<String> names = split(pathNmFull);
    int offset = names.size() >= 4 ? 1 : 0;   // 4단계 이상이면 루트를 건너뛴다
    return new DeptPathNames(get(names, offset), get(names, offset + 1), get(names, offset + 2));
}
```

조직 루트가 하나(`본사`)이고 부서 깊이가 3단계 이하이므로, 모든 부서의 1레벨 이름이
`본사`로 같아진다. **데이터상 정상**이지만, 그 결과 62개 분기가 전부 같은 상수를 반환하는
쓸모없는 `ORDER BY`가 만들어졌다.

### 원인 2 — 이름 필터가 부서 ID를 나열

```java
// "영업2팀"으로 필터 → 해당 레벨 이름이 일치하는 부서 ID를 전부 모아 IN 절에 넣는다
private void addDeptNameFilter(BooleanBuilder builder, String includeCsv, int level) {
    Set<String> includes = csvValues(includeCsv);
    List<Long> deptIds = deptTreeService.getDeptTree().getValues().stream()
        .filter(dept -> includes.contains(deptNameAt(dept, level)))
        .map(Dept::getId)
        .collect(Collectors.toList());

    builder.and(deptIds.isEmpty()
        ? user.deptId.isNull().and(user.deptId.isNotNull())   // 항상 false인 억지 조건
        : user.deptId.in(deptIds));
}
```

`영업2팀`을 고르면 팀 자신과 하위 파트 8개가 모두 매칭되어 ID 9개가 `IN`에 들어간다.
결과는 맞지만 **부서가 늘어나면 `IN` 목록도 그대로 커진다.**

### 진짜 문제 — 계층 계산이 두 곳에 흩어져 있었다

화면에 뿌릴 부서명은 Service에서 경로 문자열을 잘라 채우고, 정렬·필터는 Repository에서
같은 로직을 QueryDSL 표현식으로 다시 펼친다. **같은 계층 규칙이 자바 두 군데에 존재**하고,
그 중 하나가 SQL로 새어 나온 것이다.

한편 같은 프로젝트의 다른 화면들은 이미 **자기참조 테이블을 self-join해서 SQL 안에서**
계층을 계산하고 있었다. 이 화면만 패턴을 벗어나 있었다.

## 해결 방법

### 1. 부서 테이블을 3단 self-join

`dept` 테이블은 `parent_id`로 자기 자신을 참조하는 인접 리스트 구조다.
조회 대상 부서 → 상위 → 차상위를 각각 별칭으로 조인한다.

```java
private static final QDept dept        = new QDept("targetDept");
private static final QDept parentDept  = new QDept("parentDept");
private static final QDept grandParent = new QDept("grandParentDept");

/** 목록·카운트 쿼리 양쪽에 반드시 같이 적용해야 한다 (조건 빌더가 이 별칭을 참조하므로) */
private <T> JPAQuery<T> joinDept(JPAQuery<T> query) {
    return query
        .leftJoin(dept).on(dept.id.eq(user.deptId))
        .leftJoin(parentDept).on(parentDept.id.eq(dept.parentId))
        .leftJoin(grandParent).on(grandParent.id.eq(parentDept.parentId));
}
```

### 2. 레벨별 부서명을 SQL 표현식으로

조인된 세 부서 중 **몇 개가 실제로 존재하느냐**로 깊이를 판별한다.

```java
/** 경로가 3단계 이하면 최상위 부서명이 곧 1레벨(본부)이 된다 */
private StringExpression divisionNameExpression() {
    return new CaseBuilder()
        .when(grandParent.id.isNotNull()).then(grandParent.deptNm)
        .when(parentDept.id.isNotNull()).then(parentDept.deptNm)
        .otherwise(dept.deptNm);
}

private StringExpression teamNameExpression() {
    return new CaseBuilder()
        .when(grandParent.id.isNotNull()).then(parentDept.deptNm)
        .when(parentDept.id.isNotNull()).then(dept.deptNm)
        .otherwise("");
}

private StringExpression partNameExpression() {
    return new CaseBuilder()
        .when(grandParent.id.isNotNull()).then(dept.deptNm)
        .otherwise("");
}
```

`CASE`는 남아 있지만 **분기가 3개로 고정**된다. 부서가 몇 개든 쿼리 크기는 그대로다.

### 3. 정렬·필터를 표현식 기준으로 교체

```java
// 정렬
case "divisionNm" -> new OrderSpecifier<>(direction, divisionNameExpression());
case "teamNm"     -> new OrderSpecifier<>(direction, teamNameExpression());
case "partNm"     -> new OrderSpecifier<>(direction, partNameExpression());

// 필터 (부분일치 + 헤더필터 포함/제외를 공용 유틸로 위임)
private void addDeptNameCondition(BooleanBuilder builder, StringExpression expression,
    GridSearch search, String keyword, String includeCsv, String excludeCsv, String field) {
    if (StringUtils.hasText(keyword)) {
        builder.and(expression.containsIgnoreCase(keyword.trim()));
    }
    GridFilter.applyIncludeExclude(builder, expression,
        csvValues(includeCsv), csvValues(excludeCsv), search, field);
}
```

부서 트리를 순회하던 메서드 4개와 `DeptTreeService` 의존성은 Repository에서 통째로 제거했다.

### 4. 등가성 검증 — 루트의 `parent_id` 확인이 핵심

교체 후 SQL은 이렇게 바뀐다.

```sql
left join dept d1 on d1.id = u.dept_id
left join dept d2 on d2.id = d1.parent_id
left join dept d3 on d3.id = d2.parent_id
where h.del_fl = 'N'
  and case when (d3.id is not null) then d2.dept_nm
           when (d2.id is not null) then d1.dept_nm
           else '' end = '영업2팀'
```

**이게 기존 `IN (4, 10, 11, 12, 13, 14, 7, 8, 9)`와 같은 집합을 잡는가?** 여기서 반드시
확인해야 할 게 루트 부서의 `parent_id` 값이다. 이 테이블은 루트를 `parent_id = -1`로
표현하고 있었다.

| 대상 | d1 | d2 | d3 | CASE 결과 |
|---|---|---|---|---|
| 영업2팀 (4) | 4 | 본사 | `id = -1` → 없음 | `d1.dept_nm` = 영업2팀 ✓ |
| 하위 파트 (10~14, 7~9) | 파트 | 4 | 본사 | `d2.dept_nm` = 영업2팀 ✓ |
| 본사 (루트) | 본사 | `id = -1` → 없음 | 없음 | `''` |

루트의 `parent_id`가 `-1`이라 존재하지 않는 ID를 조인하게 되고, `d3`가 자연스럽게 NULL이
되면서 깊이 판별이 정확히 맞아떨어진다. 만약 이 값이 `0`이고 `id = 0`인 더미 행이 실제로
있었다면 한 레벨씩 밀려서 **조용히 틀린 결과**가 나왔을 것이다. self-join으로 계층을 다룰
때는 루트 표현 규약(`NULL` / `-1` / `0` / 자기 자신)을 먼저 확인해야 한다.

## 결과

- `ORDER BY`의 `CASE`: **WHEN 62개 → 3개**. 부서가 늘어도 쿼리 크기 불변
- `WHERE`: 부서 ID 나열 `IN` 절 소멸
- 부수 효과로 **2레벨·3레벨 컬럼 정렬이 실제로 동작**하게 됐다 (기존에는 상수 정렬이라 무의미)

### 덤으로 발견한 ORDER BY 중복

교체 후 SQL을 다시 보다가 이런 걸 발견했다.

```sql
order by h.id desc, h.id desc
```

동점 처리를 위해 tie-breaker를 무조건 덧붙이던 코드가 원인이었다.

```java
if (orders.isEmpty()) return defaultOrder;
orders.add(record.id.desc());   // ← 이미 id 정렬 중이어도 또 붙는다
```

이미 `id`로 정렬 중이면 붙이지 않도록 고쳤다.

```java
if (sortOrders.stream().noneMatch(sort -> "id".equals(sort.getProperty()))) {
    orders.add(record.id.desc());
}
```

### 남는 트레이드오프

`case ... end = '영업2팀'`은 조인 결과에 대한 연산이라 **인덱스를 타지 못한다.** 기존
`u.dept_id IN (...)`은 `dept_id` 인덱스로 선필터가 가능했던 것과 대비된다.

- 구동 테이블이 삭제 플래그 + 날짜 범위로 이미 좁혀지고
- `dept` 조인 3개가 전부 PK 룩업이며
- 같은 프로젝트의 다른 화면들이 동일 방식으로 운영 중

이라 현 규모에서는 문제가 없다고 판단했다. 대상 테이블이 수십만 건 이상으로 커지면
그때는 부서 경로를 비정규화한 컬럼(`path`, `level1_nm` 등)을 두거나 재귀 CTE를 검토할
지점이다.

### 정리

인접 리스트 계층 데이터를 다룰 때 **"메모리에서 계산해 SQL로 펼치는" 방식은 거의 항상
잘못된 선택**이다. 쿼리 크기가 데이터 건수에 비례하는 순간을 경고 신호로 보면 된다.

리뷰 체크리스트로 삼을 만한 것:

- 반복문으로 QueryDSL `CaseBuilder`나 `IN` 목록을 쌓고 있지 않은가
- 같은 계층 규칙이 표시용(Service)과 조건용(Repository) 두 곳에 중복되어 있지 않은가
- self-join으로 바꾼다면 루트 노드의 `parent_id` 규약이 무엇인가
- 조건 빌더를 목록/카운트 쿼리가 공유한다면 **조인도 양쪽에 똑같이** 들어갔는가
