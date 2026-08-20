---
layout: post
title: "빈 값 필터 조건이 값 파라미터로 새어 항상 0건이던 문제"
date: 2026-08-20 17:10:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, 그리드필터, Spring, 쿼리스트링]
---

그리드 헤더필터에서 "(필드 값 없음)"을 골랐는데 결과가 0건이었다.
요청을 보니 빈 값 계약(`blankFields`)은 정상적으로 나가는데 **엉뚱한 파라미터가 하나 더** 붙어 있었다.
원인은 값이 비워진 조건을 걸러내지 않아 `undefined`가 문자열 `"null"`로 직렬화된 것이었다.

## 문제 상황

조직 컬럼(부문/팀/유닛)에서 "(필드 값 없음)"을 선택하면 아무 데이터도 나오지 않았다.
같은 선택지를 다른 컬럼(직급, 재직상태 등)에서 고르면 정상 동작했다.

요청 쿼리스트링을 확인했다.

```
GET /api/v1/members?blankFields=divisionName&divisionName=null&userState=EMPLOYED
                    ^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^
                    빈 값 계약(정상)          이건 왜 붙었나
```

`blankFields`는 의도대로 나갔다. 문제는 `divisionName=null`이 함께 나간 것이다.
서버는 이걸 **"null"이라는 이름의 부문을 찾으라**는 뜻으로 받았고, 그런 조직이 없으니 조회가 즉시 끊겼다.

```java
Set<Long> orgIds = findOrgIdsByName(orgMap, search);
if (orgIds != null && orgIds.isEmpty()) {
    return Collections.emptyList();   // ← SQL 도 타지 못하고 여기서 반환
}
```

수정 전에는 SQL 로그 자체가 남지 않았다. 조회 쿼리가 실행되기 전에 빠져나갔기 때문이다.

## 원인

### 빈 값 조건은 값이 사라지고 플래그만 남는다

프론트는 "(필드 값 없음)" 선택을 내부 표식으로 다룬다.

```js
export const BLANK_MARKER = '__BLANK__';
```

이 표식은 화면 안에서만 쓰는 값이라 서버로 나가면 안 된다.
그래서 필터 트리를 조건 목록으로 정규화할 때, 표식을 값에서 떼어내 **플래그로 옮긴다**.

```js
const normalizeBlankCondition = condition => {
  const values = Array.isArray(condition.value) ? condition.value : [condition.value];
  const hasBlank = values.some(isBlankValue);
  const actualValues = values.filter(value => !isBlankValue(value));

  return {
    ...condition,
    value: Array.isArray(condition.value) ? actualValues : actualValues[0],
    hasBlank,
  };
};
```

여기서 나오는 결과가 핵심이다.

| 입력 | 정규화 후 `value` | `hasBlank` |
|---|---|---|
| 필터행에서 "(필드 값 없음)" | `undefined` | `true` |
| 헤더필터에서 "(필드 값 없음)"만 체크 | `[]` | `true` |
| "(필드 값 없음)" + 실제 값 | `['영업1']` | `true` |

값이 **비워진 채로 조건 객체는 남는다**. 의미는 `hasBlank` 플래그로 옮겨간 상태다.

### 값이 빈 조건을 값 파라미터로 내보내면 안 된다

문제는 그다음 단계였다. 조건을 서버 파라미터로 바꾸는 코드가 이 조건을 걸러내지 않았다.

```js
conditions
  .filter(condition => SUPPORTED_OPERATORS.has(condition.operator))
  .forEach(condition => {
    const converted = toRequestParams({ filter: [condition.field, condition.operator, condition.value] });
    if (converted[condition.field] !== undefined) {
      params[condition.field] = converted[condition.field];
    }
  });
```

`value`가 `undefined`인 조건이 그대로 변환 함수로 들어간다.
그리고 그 변환 함수에는 이런 계약이 있었다.

```js
const toFilterValue = (field, operator, value) => {
  // 헤더필터의 (공백) 항목은 값이 null 로 전달된다.
  // 서버는 문자열 'null' 을 IS NULL 로 해석하므로 변환한다.
  if (value === null || value === undefined) {
    if (operator === '=') return 'null';
    if (operator === '<>') return '<>null';
    return undefined;
  }
  ...
};
```

이건 **다른 그리드에서 쓰는 정상 계약**이다. 값이 없으면 문자열 `'null'`로 보내고 서버가 `IS NULL`로 해석한다.
문제는 이 화면이 `blankFields`라는 **다른 계약**을 쓰는데, 값이 빈 조건이 옛 계약 쪽으로 흘러 들어간 것이다.

```
BLANK_MARKER  →  blankFields=divisionName     ✅ 정상 전달
              →  divisionName=null            ❌ 이게 같이 나감
```

### 왜 이 컬럼만 그랬나

조직 컬럼은 화면이 필터 표현식을 직접 정의하고 있었다.

```js
// 화면에서 컬럼에 직접 지정
column.calculateFilterExpression = function (filterValue) {
  if (filterValue === null || filterValue === undefined || filterValue === '') return null;
  return [column.dataField, '=', filterValue];   // 빈 값 분기 없음
};
```

공통 그리드는 표현식을 직접 주지 않은 컬럼에만 자동으로 표현식을 붙이고, 그 표현식에는 빈 값 처리가 들어 있었다.
화면이 표현식을 덮어쓰면 그 처리를 통째로 잃는다.
게다가 "공통이 생성한 표현식"이라는 표시가 없어, 파라미터를 정리하는 후처리 단계도 건너뛴다.
그래서 잘못 만들어진 `divisionName=null`이 지워지지 않고 그대로 나갔다.

## 해결 방법

### 값이 남지 않은 조건은 값 파라미터로 내보내지 않는다

정규화가 만들 수 있는 "빈 모양"은 `undefined`와 `[]` 두 가지다. 셋 다 막았다.

```js
conditions
  .filter(condition => {
    // (공백) 선택과 값이 남지 않은 조건은 blankFields 로만 전달한다.
    // 값 파라미터로 내보내면 undefined 가 문자열 'null' 로 바뀌어
    // 서버가 실제 값 'null' 을 검색하고 항상 0건이 된다.
    if (
      condition.hasBlank ||
      condition.value === null ||
      condition.value === undefined ||
      (Array.isArray(condition.value) && condition.value.length === 0)
    ) {
      return false;
    }
    return SUPPORTED_OPERATORS.has(condition.operator);
  })
  .forEach(/* ... */);
```

`hasBlank` 하나로도 세 경우가 다 걸리지만, 나머지 두 조건을 함께 둔 것은
"값이 남지 않은 조건은 값 파라미터로 내보내지 않는다"는 규칙을 코드에 드러내기 위해서다.

이 가드는 공통 그리드에 있으므로 조직 컬럼뿐 아니라 **커스텀 표현식을 쓰는 모든 컬럼**에 함께 적용된다.

### 다중 선택은 값 배열이 아니라 OR 트리로 펼친다

같은 컬럼을 보다가 다중 선택도 깨져 있는 것을 발견했다.

헤더필터는 `filterValue`를 **배열**로 넘긴다. 그런데 화면 표현식이 배열을 값 자리에 그대로 넣고 있었다.

```js
return [column.dataField, '=', filterValue];
//   →  ['divisionName', '=', ['영업1', '영업2']]   ← 값이 배열
```

이 모양이면 필터 트리 파서가 "값 1개짜리 조건"으로 오인해서,
다중 선택과 빈 값 선택을 분리하지 못하고 통째로 스칼라 경로로 흘려보낸다.

값마다 조건을 만들어 `or`로 펼치도록 바꿨다.

```js
export const createSelectionExpression = (dataField, values) => {
  // 스칼라(필터행): 빈 값은 필터 해제
  if (!Array.isArray(values)) {
    return values === null || values === undefined || String(values).trim() === ''
      ? null
      : [dataField, '=', values];
  }

  // 배열(헤더필터): 원소의 null/'' 는 (공백) 선택이므로 버리지 않고 표식으로 정규화한다.
  // 버리면 조건이 조용히 사라져 필터가 걸리지 않은 전체 조회가 된다.
  const selected = values
    .filter(value => value !== undefined)
    .map(value => (value === null || String(value).trim() === '' ? BLANK_MARKER : value));
  if (!selected.length) return null;

  return selected
    .slice(1)
    .reduce((expr, value) => [expr, 'or', [dataField, '=', value]], [dataField, '=', selected[0]]);
};
```

스칼라와 배열의 빈 값이 **의미가 다르다**는 점이 중요하다.

- 스칼라 `null` = 필터행을 비웠다 → 필터 해제
- 배열 안의 `null` = 헤더필터의 (공백) 항목을 골랐다 → 빈 값 선택

배열 원소를 버리면 조건이 사라져 **전체가 조회된다**. 0건보다 위험한 실패 방식이다.
사용자는 필터가 걸린 줄 알고 잘못된 결과를 본다.

## 결과

| 선택 | 이전 | 이후 |
|---|---|---|
| "(필드 값 없음)" 단독 | `blankFields=divisionName`<br>**+ `divisionName=null`** → 0건 | `blankFields=divisionName` |
| "(필드 값 없음)" + 부문명 | 분리 실패 | `blankFields=...` + `divisionNameList=영업1` |
| 부문명 단독 | `divisionName=영업1` | 동일 (회귀 없음) |
| 부문명 다중 선택 | 값이 배열로 실려 깨짐 | `divisionNameList=영업1,영업2` |

수정 후에는 SQL 로그가 정상적으로 남았다. 조기 반환에 걸리지 않고 조회까지 도달했다는 뜻이다.

```sql
and ( case when ... end is null or trim(case when ... end) = '' )
```

## 정리

- 조건을 정규화하면서 값을 플래그로 옮겼다면, **값이 빈 조건을 이후 단계에서 반드시 걸러내야 한다**
- 한 시스템에 빈 값 계약이 둘 이상 있으면(문자열 `'null'` vs 별도 필드) 조건이 옛 계약으로 새기 쉽다
- 공통 처리를 화면에서 덮어쓸 때는 **무엇을 함께 잃는지** 확인해야 한다. 표현식만 바꾼 줄 알았는데 후처리까지 건너뛰고 있었다
- 배열 원소를 조용히 버리면 조건이 사라져 전체 조회가 된다. 0건은 눈에 띄지만 전체 조회는 눈치채기 어렵다
