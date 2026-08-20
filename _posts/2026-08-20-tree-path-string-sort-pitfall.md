---
layout: post
title: "ID를 이은 path를 문자열 정렬해 트리 순서가 뒤집히던 문제"
date: 2026-08-20 10:00:00 +0900
categories: [오류해결, SQL]
tags: [SQL, MariaDB, QueryDSL, 정렬, 트리구조]
---

부서 선택 드롭다운과 헤더필터의 순서가 관리 화면에서 지정한 표시 순서와 전혀 달랐다.
원인은 **ID를 이어 만든 경로 문자열을 사전순으로 정렬**한 것이었다. `"10"`이 `"9"`보다 앞에 온다.

## 문제 상황

부서 트리를 이런 순서로 보여주고 있었다.

```
기획팀 → META → MPC → KH → NICE → 운영1팀 → 운영3팀 → 운영2팀
```

관리 화면에서 지정한 순서(`dept_ord`)는 이랬다.

```
기획팀(1) → 운영1팀(2) → 운영2팀(3) → 운영3팀(4) → META(5) → MPC(6) → KH(7) → NICE(8)
```

하위 섹션도 마찬가지였다. `지원(1) → 교육(2) → 운영1(3) …` 순이어야 하는데 `운영3 → 운영2 → 운영1 → 운영5 → 주문 → 지원 → 교육` 처럼 뒤죽박죽이었다.

## 원인

부서 목록 조회가 경로 문자열로 정렬하고 있었다.

```java
List<DeptCode> codes = deptMap.getValues().stream()
    .sorted(Comparator
        .comparing(DeptEntity::getPath, Comparator.nullsLast(String::compareTo))
        .thenComparing(DeptEntity::getId, Comparator.nullsLast(Long::compareTo)))
    .map(this::toDeptCode)
    .toList();
```

`path`는 루트부터 자기 자신까지 **ID를 구분자로 이은 문자열**이다.

```
기획팀   → "1/2"
운영1팀  → "1/4"
META     → "1/32"
```

이걸 문자열로 비교하면 이렇게 된다.

```
"1/2" < "1/32" < "1/33" < "1/34" < "1/35" < "1/4" < "1/5" < "1/6"
```

`"1/32"`가 `"1/4"`보다 앞선다. 두 번째 문자가 `3` vs `4`이기 때문이다.
화면에 보이던 순서와 정확히 일치했다.

문제는 두 가지였다.

| 문제 | 설명 |
|---|---|
| `dept_ord` 미반영 | 관리자가 지정한 표시 순서가 정렬에 전혀 쓰이지 않음 |
| 자릿수 역전 | ID 사전순이라 `"10"` < `"9"` |

## 왜 단순히 dept_ord로 정렬하면 안 되는가

처음엔 `Comparator.comparing(DeptEntity::getDeptOrd)`로 바꾸면 될 것 같았다. 하지만 **계층이 무너진다.**

`dept_ord`는 **형제 그룹 안에서만 유일**하기 때문이다. 실제 데이터를 보면 이랬다.

| 부서 | 상위 | dept_ord |
|---|---|---|
| 기획팀 | 센터 | 1 |
| 지원 | 운영1팀 | **1** |
| 지원 | 운영2팀 | **1** |
| META1 | META | **1** |

`dept_ord = 1`인 부서가 여럿이라, 계층을 무시하고 한 줄로 정렬하면 서로 다른 팀의 섹션이 뒤섞인다.

## 해결 방법

**경로를 앞에서부터 레벨별로 훑어, 처음 갈라지는 지점의 `dept_ord`로 순서를 정하는** 비교자를 만들었다.

```java
/**
 * 부서 트리 정렬 비교자.
 * 경로를 앞에서부터 훑어 처음으로 갈라지는 레벨의 순서로 앞뒤를 정한다.
 * 계층 구조를 유지한 채 같은 상위 부서 안에서만 dept_ord 순으로 정렬된다.
 */
private int compareByDeptOrdPath(DeptEntity left, DeptEntity right, DeptMap deptMap) {
    List<Long> leftIds = parseDeptPathIds(left == null ? null : left.getPath());
    List<Long> rightIds = parseDeptPathIds(right == null ? null : right.getPath());

    int commonSize = Math.min(leftIds.size(), rightIds.size());
    for (int level = 0; level < commonSize; level++) {
        Long leftId = leftIds.get(level);
        Long rightId = rightIds.get(level);
        if (Objects.equals(leftId, rightId)) {
            continue;
        }
        // 같은 레벨에서 갈라지면 dept_ord → 부서명 → id 순으로 앞뒤를 정한다.
        int compared = Integer.compare(deptOrdOf(deptMap, leftId), deptOrdOf(deptMap, rightId));
        if (compared != 0) {
            return compared;
        }
        compared = compareDeptName(deptMap, leftId, rightId);
        return compared != 0 ? compared : Long.compare(leftId, rightId);
    }
    // 한쪽이 다른 쪽의 상위 부서이면 상위가 먼저 온다.
    return Integer.compare(leftIds.size(), rightIds.size());
}

/** 표시 순서 미지정이면 맨 뒤로 보낸다. */
private int deptOrdOf(DeptMap deptMap, Long deptId) {
    DeptEntity dept = deptId == null ? null : deptMap.findOneById(deptId);
    return dept == null || dept.getDeptOrd() == null ? Integer.MAX_VALUE : dept.getDeptOrd();
}
```

동작은 이렇다.

```
센터A(ord=2) > 팀X(ord=1)   path = "10/30"
센터B(ord=1) > 팀Y(ord=5)   path = "7/41"

레벨1에서 갈라짐 → ord 1(센터B) < ord 2(센터A) → 센터B 계열이 앞
같은 센터 안에서는 레벨2의 ord 로 비교
```

### 문자열 키 대신 비교자를 쓴 이유

`String.format("%010d", ord)`로 zero-padding한 정렬 키를 만드는 방법도 있다. 하지만 **`dept_ord`가 음수이면 사전순이 깨진다.** `-000000001`이 `0000000001`보다 앞서기 때문이다.
자릿수를 넘는 값에도 취약하다. 비교자를 직접 구현하면 이런 걱정이 없다.

## 프론트에서 다시 정렬하고 있었다

백엔드를 고쳤는데도 헤더필터 순서가 그대로였다. 프론트가 응답을 받아 **이름순으로 재정렬**하고 있었다.

```javascript
// 헤더필터 드롭다운
return [...names]
  .sort((left, right) => left.localeCompare(right, 'ko', { numeric: true }))
  .map(name => ({ text: name, value: name }));

// 필터행 SelectBox
return [...itemMap.values()].sort((left, right) =>
  left.text.localeCompare(right.text, 'ko', { numeric: true }),
);
```

두 `sort`를 제거하니 서버 순서가 그대로 흘렀다. 다행히 중간 단계가 전부 순서를 보존하는 구조였다.

| 단계 | 순서 보존 | 근거 |
|---|---|---|
| 후보 생성 | O | `map`/`filter`만 사용 |
| 상위 선택 필터링 | O | `filter`만 사용 |
| 항목 맵 | O | `Map`은 삽입 순서 유지 |
| 스토어 | O | `sort` 옵션 없음 |

```javascript
// Set 은 삽입 순서를 유지하므로 서버가 내려준 계층별 dept_ord 순서가 그대로 남는다.
// 이름순으로 다시 정렬하면 관리 화면에서 지정한 표시 순서가 사라진다.
return [...names].map(name => ({ text: name, value: name }));
```

## 결과

- 부서 목록이 관리 화면에서 지정한 순서대로 나온다.
- 계층 구조가 유지된 채 같은 상위 부서 안에서만 `dept_ord` 순으로 정렬된다.
- `dept_ord`가 같으면 부서명, 그래도 같으면 ID로 순서가 고정된다.

## 남는 교훈

**ID를 이은 경로를 문자열로 정렬하는 코드는 자릿수가 늘어나는 순간 조용히 깨진다.** ID가 한 자리일 때는 우연히 맞아 보이다가, 10을 넘으면서 순서가 뒤집힌다. 초기에는 발견되지 않는 종류의 버그다.

그리고 **정렬은 한 곳에서만 해야 한다.** 서버에서 제대로 정렬해도 클라이언트가 다시 정렬하면 무의미하다. "서버 순서를 그대로 쓴다"는 계약이라면 주석으로 남겨두는 편이 좋다. 다음 사람이 무심코 `sort`를 추가하지 않도록.
