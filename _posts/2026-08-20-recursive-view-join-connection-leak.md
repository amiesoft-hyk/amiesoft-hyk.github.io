---
layout: post
title: "정렬에만 쓰는 재귀 뷰를 무조건 조인해 커넥션을 10초씩 붙잡던 문제"
date: 2026-08-20 09:30:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, MariaDB, HikariCP, 성능튜닝, 재귀CTE]
---

목록 화면 하나가 500을 뱉었다. 서버 로그에는 예외 대신 HikariCP의 커넥션 누수 경고가 찍혀 있었다.
원인은 **정렬할 때만 필요한 재귀 뷰를 정렬 여부와 무관하게 매번 3번 조인**한 것이었다.

## 문제 상황

브라우저에서는 단순히 500이었다.

```
GET /api/v1/certificates?pagesize=100&currentpage=1 500 (Internal Server Error)
```

서버 로그를 보니 예외가 아니라 이런 WARN이 남아 있었다.

```
WARN ProxyLeakTask : Connection leak detection triggered for ... stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:127)
    ...
    at com.example.certificate.api.CertificateApi.getCertificateList(CertificateApi.java:41)
```

여기서 한 번 헷갈렸다. `java.lang.Exception`이 찍혀 있어 예외처럼 보이지만, 이건 **HikariCP가 "커넥션을 언제 어디서 빌려갔는지" 기록하려고 인위적으로 생성한 스택**이다.
그래서 최상단이 `getConnection` → `JpaTransactionManager.doBegin`, 즉 트랜잭션 시작 지점이지 실패 지점이 아니다.

설정을 확인하니 임계치는 10초였다.

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 10000
```

즉 이 경고가 말하는 건 하나였다. **이 조회가 커넥션을 10초 넘게 붙잡고 있다.**

## 원인

조회 쿼리가 이렇게 생겼다.

```java
List<CertificateResult> rows = queryFactory
    .select(Projections.bean(CertificateResult.class,
        cert.id, cert.ownerId, cert.typeCode, cert.subTypeCode, cert.title, ...))
    .from(cert)
    .innerJoin(owner).on(cert.ownerId.eq(owner.ownerId))
    .leftJoin(typeCode).on(typeCode.id.eq(cert.typeCode.longValue()))
    .leftJoin(typeParentCode).on(typeParentCode.id.eq(typeCode.parentId))
    .leftJoin(subTypeCode).on(subTypeCode.id.eq(cert.subTypeCode.longValue()))
    .where(condition)
    .orderBy(getOrderSpecifiers(pageDto))
    .offset(...).limit(...)
    .fetch();
```

`typeCode`, `typeParentCode`, `subTypeCode` 세 개가 붙는데, 이들은 **공통코드 계층 뷰**다.
문제는 이 세 조인이 `select`에도 `where`에도 기본 `orderBy`에도 등장하지 않는다는 점이었다. 오직 코드명 기준 정렬 표현식에서만 쓰였다.

```java
// 정렬 요청이 typeCode / subTypeCode 일 때만 호출된다
private StringExpression typeNameExpression() {
    return new CaseBuilder()
        .when(typeCode.depth.eq(3).and(typeParentCode.codeName.isNotNull())).then(typeParentCode.codeName)
        .otherwise(typeCode.codeName.coalesce(cert.typeCode.stringValue()));
}
```

그리고 이 뷰가 왜 비싼지는 정의를 보고 확신했다.

```sql
CREATE OR REPLACE VIEW code_tree_v AS
WITH RECURSIVE code_hierarchy AS (
    SELECT id, code_name, parent_id, 1 AS depth,
           CAST(id AS CHAR(1000)) AS path
      FROM code_master
     WHERE parent_id = -1 AND use_yn = 'Y'
    UNION ALL
    SELECT c.id, c.code_name, c.parent_id, h.depth + 1,
           CONCAT(h.path, '/', c.id)
      FROM code_master c
      JOIN code_hierarchy h ON c.parent_id = h.id
     WHERE c.use_yn = 'Y'
)
SELECT * FROM code_hierarchy
ORDER BY path;   -- ← 결정타
```

| 특성 | 결과 |
|---|---|
| `WITH RECURSIVE` | 조인할 때마다 코드 테이블 전체를 재귀 순회 |
| 뷰 안의 `ORDER BY path` | **MERGE 알고리즘을 쓸 수 없어** 매번 임시 테이블로 만들어짐 |
| `path`가 `CHAR(1000)` | 긴 문자열 키로 filesort |
| 임시 테이블 | **인덱스가 없어** 조인이 전부 풀스캔 |

여기에 `typeParentCode ↔ typeCode` 조인은 **임시 테이블끼리의 조인**이라 인덱스 없는 nested loop가 된다.
게다가 `cert.typeCode.longValue()`는 SQL에서 `CAST(...)`로 나가 인덱스를 탈 수 없다.

정리하면 **기본 조회에서 단 한 번도 참조하지 않는 값을 위해, 인덱스 없는 임시 테이블 3개를 만들고 버리고 있었다.**

결정적인 힌트는 같은 뷰를 쓰는 다른 화면이었다. 그쪽은 정렬이 걸렸을 때만 조인하고 있었다.

```java
if (requiresSort(pageDto, "typeCode") || requiresSort(pageDto, "subTypeCode")) {
    query.leftJoin(typeCode).on(...)
         .leftJoin(typeParentCode).on(...)
         .leftJoin(subTypeCode).on(...);
}
```

## 해결 방법

같은 방식으로 조인을 조건부로 바꿨다. 체이닝으로 한 번에 만들던 쿼리를 `JPAQuery`로 단계 조립하도록 고쳤다.

```java
JPAQuery<CertificateResult> query = queryFactory
    .select(Projections.bean(CertificateResult.class, ...))
    .from(cert)
    .innerJoin(owner).on(cert.ownerId.eq(owner.ownerId));

// 공통코드 뷰는 재귀 CTE + ORDER BY 구조라 조인할 때마다 임시 테이블로 만들어져 비용이 크다.
// 코드명 기준 정렬일 때만 필요하므로 해당 정렬에서만 조인한다.
if (requiresCodeNameSort(pageDto)) {
    query.leftJoin(typeCode).on(typeCode.id.eq(cert.typeCode.longValue()))
         .leftJoin(typeParentCode).on(typeParentCode.id.eq(typeCode.parentId))
         .leftJoin(subTypeCode).on(subTypeCode.id.eq(cert.subTypeCode.longValue()));
}

List<CertificateResult> rows = query
    .where(condition)
    .orderBy(getOrderSpecifiers(pageDto))
    .offset(...).limit(...)
    .fetch();
```

게이트 조건은 정렬 분기와 정확히 1:1로 맞췄다.

```java
/** 코드명 기준 정렬 여부. getOrderSpecifiers 의 분기와 1:1 대응해야 한다. */
private boolean requiresCodeNameSort(CommonPageDto pageDto) {
    return requiresSort(pageDto, "typeCode") || requiresSort(pageDto, "subTypeCode");
}

private boolean requiresSort(CommonPageDto pageDto, String property) {
    if (pageDto == null || pageDto.getPageable() == null) {
        return false;
    }
    return pageDto.getPageable().getSort().stream()
        .anyMatch(sort -> property.equals(sort.getProperty()));
}
```

**여기가 함정이다.** 조인을 뺐는데 정렬 표현식이 그대로 평가되면 QueryDSL이 암묵적 cross join을 만들어 오히려 더 나빠진다.
그래서 조인 게이트와 `getOrderSpecifiers`의 `case` 분기가 같은 프로퍼티 집합을 봐야 한다.

인식되지 않는 정렬 프로퍼티는 기본 정렬로 폴백되는데, 거기엔 뷰 참조가 없어 안전하다.

## 결과

- 기본 조회 SQL에서 뷰 조인 3개가 사라졌다.
- 커넥션 누수 경고가 나오지 않는다.
- 코드명 정렬 기능은 그대로 동작한다. 정렬을 걸었을 때만 예전 비용이 발생한다.

## 남는 교훈

**HikariCP leak 경고의 스택은 실패 지점이 아니라 커넥션 획득 지점이다.** 이걸 예외 스택으로 오독하면 트랜잭션 설정이나 JPA 쪽을 엉뚱하게 뒤지게 된다.
경고가 뜨면 "무엇이 실패했나"가 아니라 "무엇이 오래 걸리나"를 먼저 봐야 한다.

그리고 **뷰 안의 `ORDER BY`는 조용한 성능 폭탄이다.** MySQL/MariaDB에서 뷰에 `ORDER BY`가 있으면 MERGE 최적화가 막혀 매번 임시 테이블이 만들어진다.
조회 편의로 넣은 정렬이 조인 대상이 되는 순간 비용이 곱해진다.
