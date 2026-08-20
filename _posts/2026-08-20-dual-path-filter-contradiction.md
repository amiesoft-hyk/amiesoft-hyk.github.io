---
layout: post
title: "같은 검색 조건을 두 경로로 판정하면 서로 모순된다"
date: 2026-08-20 17:50:00 +0900
categories: [오류해결, QueryDSL]
tags: [QueryDSL, Spring, DTO, 그리드필터, 검색조건]
---

헤더필터에서 "(필드 값 없음)"과 실제 값을 **함께** 고르면 아무 데이터도 나오지 않았다.
각각 따로 고르면 정상이었다.
원인은 한 컬럼의 조건이 인메모리 판정과 SQL 조건으로 갈라진 뒤, 두 결과가 AND로 묶여 서로 모순된 것이었다.

## 문제 상황

유닛 컬럼 헤더필터에서 세 개를 체크했다.

```
☑ (필드 값 없음)
☑ 영업1
☑ 영업2
```

기대는 "유닛이 비었거나 영업1이거나 영업2"인 인원이다. 결과는 0건이었다.
각각 단독으로 고르면 모두 정상이었다.

서버 파라미터는 정상적으로 도착했다.

```
unitNameList = 영업1,영업2
blankFields  = unitName
```

## 원인

### 조건이 두 경로로 갈라진다

조직명은 테이블에 없는 계산 필드라 SQL로 직접 거를 수 없다.
그래서 **조직 트리를 먼저 훑어 조건에 맞는 조직 ID를 모으고**, 그 집합을 `orgId IN (...)`으로 넘기는 구조였다.

```java
// 경로 A — 인메모리
Set<Long> orgIds = findOrgIdsByName(orgMap, search);   // unitNameList 처리

// 경로 B — SQL
Search querySearch = toQuerySearch(search);            // blankFields 처리
repository.findMembers(querySearch, pageDto, authScope, orgIds);
```

`unitNameList`는 경로 A로, `blankFields`는 경로 B로 갔다. 서로 다른 곳에서 판정된 것이다.

**경로 A**는 빈 값 여부를 아예 보지 않았다.

```java
if (!includes.isEmpty() && !includes.contains(actual)) {
    return false;   // 유닛이 빈 조직은 actual="" 이라 여기서 탈락
}
```

→ `orgId IN (영업1·영업2 조직들)`

**경로 B**는 빈 값 조건만 만들었다.

```java
GridFilter.applyIncludeExclude(predicate, unitName, includes, excludes, search, "unitName");
// includes/excludes 는 비어 있고 includesBlank("unitName") 만 true
```

→ `unitName IS NULL OR TRIM(unitName) = ''`

두 조건이 **AND**로 묶인다.

```sql
orgId IN (유닛이 영업1/영업2 인 조직)
AND (unitName IS NULL OR TRIM(unitName) = '')     -- 모순
```

"유닛이 영업1이면서 동시에 비어 있는" 행을 찾으니 결과가 있을 수 없다.

### 출발점은 DTO 재조립에서 필드가 빠진 것

왜 조건이 갈라졌나. `toQuerySearch`가 검색 조건을 새 객체로 옮길 때 **조직명 필드를 복사하지 않기** 때문이다.

이 함수는 [예전에도 한 번 문제가 됐던 자리]({% post_url 2026-08-18-builder-parent-field-loss %})다.
그때는 `@Builder`가 상위 클래스 필드를 만들어주지 않아 `blankFields`가 통째로 유실됐고, 명시 복사로 해결했다.

```java
Search querySearch = Search.builder()
    .memberId(search.getMemberId())
    .memberName(search.getMemberName())
    // ... 60여 개 필드
    .build();

// @Builder 는 부모 필드를 만들어주지 않으므로 반드시 함께 옮긴다
querySearch.setBlankFields(search.getBlankFields());
```

`blankFields`는 이렇게 살아남았지만 **조직명은 복사 대상에 없었다**.
경로 A가 처리하니 의도적으로 뺀 것이었는데, 그 결과 같은 컬럼의 조건이 절반씩 다른 곳으로 가게 됐다.

DTO 필드와 복사 목록을 대조해보니 누락이 더 있었다.

```
DTO 필드 75개 / 복사 63개

누락:
  divisionName, teamName, unitName (+List/ExcludeList)  → 경로 A 가 처리 (의도적)
  extensionNo (+List/ExcludeList)                        → 대체 경로 없음 ❌
  tenureBucketList, tenureBucketExcludeList              → 대체 경로 없음 ❌
```

내선번호와 근속 구간 필터는 **아무도 처리하지 않고 있었다**.
조건이 사라져도 오류가 나지 않고 조용히 전체가 조회되니 드러나지 않던 상태였다.

## 해결 방법

### 판정을 한 경로로 모은다

조직 조건은 인메모리 쪽으로 통일했다. 조건 객체가 빈 값 여부까지 함께 받도록 확장한다.

```java
private record OrgNameCondition(String expected, Set<String> includes, Set<String> excludes,
    boolean includeBlank, boolean excludeBlank) {

    private boolean matches(String actual) {
        String value = actual == null ? "" : actual.trim();
        boolean blank = value.isEmpty();

        if (expected != null && !expected.isBlank()
            && !value.equalsIgnoreCase(expected.trim())) {
            return false;
        }
        // (필드 값 없음)을 실제 값과 함께 고르면 둘의 합집합이어야 한다.
        // 빈 값을 여기서 판정하지 않으면 SQL 쪽 빈값 조건과 AND 로 묶여 항상 0건이 된다.
        if ((!includes.isEmpty() || includeBlank)
            && !(includes.contains(value) || (includeBlank && blank))) {
            return false;
        }
        if (excludeBlank && blank) {
            return false;
        }
        return !excludes.contains(value);
    }
}
```

그리고 인메모리가 판정을 마쳤다면 **SQL로 넘기는 빈 값 목록에서 조직 필드를 뺀다**.

```java
Set<Long> orgIds = findOrgIdsByName(orgMap, search);
Search querySearch = toQuerySearch(search);
if (orgIds != null) {
    // 조직 조건을 조직맵에서 이미 판정했으므로 SQL 쪽 (필드 값 없음) 조건은 뺀다.
    // 남겨두면 'orgId in (...)' 와 '유닛이 비어 있음' 이 AND 로 묶여 항상 0건이 된다.
    clearOrgBlankFields(querySearch);
}
```

`clearOrgBlankFields`는 CSV에서 조직 3필드만 제거하고 다른 컬럼(내선번호 등)의 빈 값 조건은 그대로 둔다.

**"(필드 값 없음)" 단독 선택은 기존 SQL 경로를 그대로 둔다.**
조직 자체가 배정되지 않은 인원은 조직 트리에 없어 `orgId IN (...)`으로 잡히지 않기 때문이다.
이 경우 인메모리 진입 조건에 걸리지 않으므로 자연히 SQL이 처리한다.

### 누락된 필드 복구

대체 경로가 없던 필드는 복사 목록에 추가하고, 조직명을 일부러 뺐다는 사실을 주석으로 남겼다.

```java
    .extensionNo(search.getExtensionNo())
    .extensionNoList(search.getExtensionNoList())
    .extensionNoExcludeList(search.getExtensionNoExcludeList())
    // ...
    .build();

// 조직명(divisionName 등)은 여기서 복사하지 않는다. findOrgIdsByName 이 원본 search 로
// 조직맵을 훑어 orgId 집합으로 좁히기 때문이다. 그 외 검색 필드는 이 변환을 거치지 않으면
// 조건이 조용히 사라져 필터를 걸어도 전체가 조회되므로 DTO 에 필드를 추가할 때 여기도 함께 채워야 한다.
querySearch.setBlankFields(search.getBlankFields());
```

### 겸사겸사 정리한 것

조건 판정 안에서 CSV를 매번 다시 쪼개고 있었다.

```java
private boolean matchesOrgName(String actual, String expected, String includeCsv, String excludeCsv) {
    Set<String> includes = splitCsv(includeCsv);       // 조직마다 재파싱
    return !splitCsv(excludeCsv).contains(actual);     // 조직마다 재파싱
}
```

검색 조건은 요청당 고정인데 조직 수만큼 반복된다. 조직이 100개면 3레벨 × include/exclude로 **최대 600회**다.
조건 객체를 순회 전에 한 번만 만들도록 옮겨 6회로 줄였다.

## 결과

| 선택 | 이전 | 이후 |
|---|---|---|
| "(필드 값 없음)" + `영업1` + `영업2` | AND 모순 → **0건** | 세 부류 모두 조회 |
| "(필드 값 없음)" 단독 | 정상 | 동일 (SQL 경로 유지) |
| `영업1` 단독 | 정상 | 동일 |
| 내선번호 헤더필터 | 조건 유실 → 전체 조회 | 정상 필터링 |

## 정리

- 한 컬럼의 조건이 **두 경로로 갈라지면** 두 결과는 AND로 만나고, 상보적인 조건일수록 모순이 되기 쉽다
- "이건 저쪽에서 처리하니까 여기선 빼자"는 판단은 **왜 뺐는지 주석으로 남겨야** 한다. 남기지 않으면 다음 사람이 같은 자리에서 또 빠뜨린다
- DTO를 손으로 재조립하는 코드는 필드가 늘어날 때마다 누락 위험이 쌓인다. 필드 대조를 한 번 돌려보면 놓친 게 드러난다
- 조건이 사라지면 **오류 없이 전체가 조회된다**. 0건보다 발견이 늦다
