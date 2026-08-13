---
layout: post
title: "Could not interpret path expression — QueryDSL 목록 쿼리와 count 쿼리의 join 불일치"
date: 2026-08-13 14:20:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, Hibernate, JPA, SpringBoot]
---

## 문제 상황

목록 조회 화면에서 특정 검색 조건을 넣으면 조회가 통째로 실패했다. 서버 로그에는
Hibernate의 `SemanticException`이 찍혔다.

```text
org.springframework.dao.InvalidDataAccessApiUsageException:
  org.hibernate.query.SemanticException:
  Could not interpret path expression 'skill.acqDt01'
```

특이한 점이 두 가지 있었다.

- 기본 조회(조건 없음)는 정상인데, **기준일자 조건을 넣으면** 실패한다
- 로그를 보면 SQL이 한 번은 실행된 뒤에 예외가 발생했다 (p6spy 로그가 예외 앞에 찍힘)

즉 목록 쿼리는 성공했고, 그 **다음에 나가는 쿼리**에서 터진 것이다.

## 원인

이 Repository는 목록과 전체 건수를 각각 다른 쿼리로 조회하면서, `where` 조건만
같은 `BooleanBuilder`를 공유하고 있었다.

```java
public Page<MemberSkillDto> search(CommonPageDto pageDto, SearchDto search) {
    BooleanBuilder condition = buildCondition(search);   // ← 두 쿼리가 공유

    // 목록 쿼리 : skill 을 left join 한다
    List<MemberSkillDto> list = queryFactory
        .select(Projections.constructor(MemberSkillDto.class,
            member.loginId, member.name, skill.currentLevel, skill.acqDt01))
        .from(member)
        .leftJoin(skill).on(skill.loginId.eq(member.loginId))   // ← join 있음
        .where(condition)
        .offset(...).limit(...)
        .fetch();

    // count 쿼리 : skill join 이 빠져 있다
    JPAQuery<Long> countQuery = queryFactory
        .select(member.count())
        .from(member);                                          // ← join 없음

    return pageDto.getPageResult(list, () -> countQuery.where(condition).fetchOne());
}
```

`buildCondition()`은 기준일자가 들어오면 아래 조건을 추가한다.

```java
private BooleanBuilder buildCondition(SearchDto search) {
    BooleanBuilder condition = new BooleanBuilder();
    condition.and(member.delFlag.eq("N"));

    LocalDate baseDt = parseDate(search.getBaseDt());
    if (baseDt != null) {
        // 스킬 획득일자 컬럼 중 하나라도 기준일자 이하인지
        BooleanBuilder anyHeld = new BooleanBuilder();
        anyHeld.or(skill.acqDt01.loe(baseDt));   // ← skill 경로 참조
        anyHeld.or(skill.acqDt02.loe(baseDt));
        condition.and(anyHeld);
    }
    return condition;
}
```

`skill`은 count 쿼리의 `from`/`join` 어디에도 등장하지 않는다. Hibernate는 JPQL을
파싱할 때 선언되지 않은 alias를 만나면 경로를 해석하지 못하고 `SemanticException`을
던진다. 조건이 없을 때는 `skill` 경로를 참조하지 않으니 통과했고, 기준일자를 넣는
순간 터진 것이다.

**핵심은 "조건을 공유하면 join 구성도 공유해야 한다"는 점**이다. 조건 빌더를 분리하지
않고 재사용하는 순간, 두 쿼리의 from/join 절은 같은 alias 집합을 제공해야 한다.

## 해결 방법

count 쿼리에도 동일한 join을 추가했다.

```java
JPAQuery<Long> countQuery = queryFactory
    .select(member.count())
    .from(member)
    .leftJoin(skill).on(skill.loginId.eq(member.loginId));   // ← 추가
```

이 테이블은 로그인 ID가 PK인 1:1 관계라 `left join`을 추가해도 행이 늘어나지 않아
집계 값이 달라지지 않는다. **1:N 관계였다면** 단순히 join만 추가하면 count가 부풀기
때문에 `member.countDistinct()`를 쓰거나, 조건을 `exists` 서브쿼리로 바꾸는 편이 안전하다.

```java
// 1:N 인 경우의 대안
condition.and(JPAExpressions.selectOne()
    .from(skill)
    .where(skill.loginId.eq(member.loginId).and(skill.acqDt01.loe(baseDt)))
    .exists());
```

## 결과

기준일자 조건, 스킬 헤더 필터 등 조인 대상 엔티티를 참조하는 모든 조건에서 조회가
정상 동작하게 됐다.

이 유형의 버그는 **조건 없이 테스트하면 절대 재현되지 않는다**는 게 함정이다.
목록/count를 나눠 작성하는 QueryDSL Repository를 리뷰할 때는 다음 두 가지를 같이 본다.

- 두 쿼리가 같은 조건 빌더를 공유하는가
- 공유한다면 from/join 절의 alias 집합이 동일한가

조건 빌더가 참조할 수 있는 alias를 애초에 join으로 강제하고 싶다면, 쿼리 시작부를
공통 메서드로 빼서 두 쿼리가 같은 baseQuery에서 출발하게 만드는 것도 방법이다.

```java
private <T> JPAQuery<T> baseQuery(Expression<T> select) {
    return queryFactory.select(select)
        .from(member)
        .leftJoin(skill).on(skill.loginId.eq(member.loginId));
}
```
