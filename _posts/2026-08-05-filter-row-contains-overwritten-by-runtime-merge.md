---
layout: post
title: "사번 부분 검색이 안 된다 — 런타임 필터 복원이 contains를 =로 덮어쓴 문제"
date: 2026-08-05 11:20:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, 서버페이징, 필터]
---

## 문제 상황

대상자 추가 팝업(서버 페이징 그리드)에서 필터 행에 사번 일부를 입력해도 결과가 나오지 않았다.

- 사번 `10023`을 **전체 입력**하면 정상 조회된다
- 앞자리 `100`만 입력하면 **결과 0건**
- 이름·직급 등 다른 텍스트 컬럼도 마찬가지로 부분 검색이 되지 않는다
- 반면 코드성 컬럼(계약형태, 재직상태 등 lookup 컬럼)의 필터는 정상 동작한다

DevExtreme 필터 행에서 텍스트 컬럼의 기본 연산자는 `contains`다. UI에도 "포함" 아이콘이 그대로 떠 있으니 화면상으로는 이상한 점이 없었다. 그런데 실제 요청 파라미터를 까 보니 이렇게 나갔다.

```
empNo=100
empNoList=100
```

같은 값이 **부분 검색용 파라미터와 정확일치 목록 파라미터로 동시에** 전송되고 있었다. 서버는 둘을 AND로 묶으므로 `empNo LIKE '%100%' AND empNo IN ('100')` 이 되어 0건이 나온 것이다.

## 원인

### 서버 페이징 그리드의 필터 복원 로직

이 그리드는 원격 필터링을 쓰기 때문에 `loadOptions.filter` 트리를 파싱해 서버 파라미터로 변환한다. 여기에 더해, 조회 시점에 그리드 런타임 컬럼 상태(`columnOption`)에서 필터 조건을 **복원해 병합**하는 유틸을 함께 쓰고 있었다. 헤더 필터 선택값이 `loadOptions.filter`에 실리지 않는 타이밍이 있어 이를 보정하려고 만든 장치다.

```javascript
// 런타임 컬럼 상태에서 필터 조건을 복원한다
const runtimeConditions = fields.flatMap(field => {
  const column = gridInstance.columnOption(field);
  if (!column) {
    return [];
  }

  const conditions = [];

  // 필터 행(단일 값)
  if (column.filterValue !== null && column.filterValue !== undefined && column.filterValue !== '') {
    conditions.push({
      field,
      operator: column.selectedFilterOperation || '=',   // ← 문제 지점
      value: column.filterValue,
    });
  }

  // 헤더 필터(다중 값)
  if (Array.isArray(column.filterValues) && column.filterValues.length) {
    conditions.push({
      field,
      operator: column.filterType === 'exclude' ? 'noneof' : 'anyof',
      value: column.filterValues,
    });
  }

  return conditions;
});
```

복원한 필드는 파싱 결과보다 **우선 적용**된다. 즉 런타임 조건이 있는 필드는 `loadOptions.filter`에서 나온 조건을 버린다.

### `selectedFilterOperation`은 기본값일 때 비어 있다

핵심은 `column.selectedFilterOperation || '='` 이 한 줄이다.

DevExtreme은 사용자가 필터 행의 연산자를 **명시적으로 바꾸지 않으면** `selectedFilterOperation`을 채워주지 않는다. 텍스트 컬럼에서 그냥 값만 입력한 경우 이 값은 `undefined`다. 실제 적용되는 연산자는 `contains`인데도 그렇다.

그래서 복원 과정은 이렇게 흘러갔다.

```
1. 사용자가 사번 컬럼에 "100" 입력
2. loadOptions.filter → ['empNo', 'contains', '100']   ✔ 올바른 조건
3. 런타임 복원 → selectedFilterOperation 이 undefined 이므로 '=' 로 복원
4. 복원 조건이 우선 적용되어 2번 조건을 덮어씀
5. '=' 조건은 정확일치 목록 파라미터(empNoList)로 변환
6. 파싱 단계에서 만든 empNo 도 남아 있어 두 파라미터가 함께 전송
```

lookup 컬럼에서 문제가 없었던 이유도 같은 맥락이다. 코드성 컬럼은 애초에 `filterOperations: []`, `selectedFilterOperation: '='` 로 고정해 두었기 때문에 복원값과 실제 연산자가 일치한다.

## 해결 방법

런타임 복원은 **애초에 이 장치가 필요했던 코드(lookup) 컬럼에만** 적용하도록 대상 필드를 좁혔다. 텍스트 컬럼은 `loadOptions.filter` 파싱 결과를 그대로 쓴다.

```diff
+ // 런타임 복구는 코드(lookup) 컬럼에만 적용한다.
+ // 텍스트 컬럼까지 넘기면 필터행의 contains 조건이 '='로 덮어써져 정확일치 파라미터가 생성된다.
  const conditions = mergeRuntimeFilterConditions(
    parseFilterConditions(loadOptions.filter),
    () => gridRef.value,
-   ALL_FILTER_FIELDS,          // 텍스트 컬럼까지 전부 포함
+   LOOKUP_FIELDS,         // lookup 컬럼만
  );
```

파라미터가 중복 전송되던 부분도 같이 정리했다. 코드 컬럼은 목록 파라미터로 변환하기 전에 원본 키를 지운다.

```javascript
LOOKUP_FIELDS.forEach(field => {
  delete params[field];                        // 단일 파라미터 제거
  applyListParams(params, conditions, field);   // fieldList / fieldExcludeList 생성
});
```

기존에는 이 변환을 필드 종류별로 직접 구현해 두었는데, 공통 유틸로 통일하면서 텍스트 컬럼용 분기 자체가 사라졌다.

## 결과

- 사번·이름 등 텍스트 컬럼의 부분 검색이 정상 동작
- 요청 파라미터에서 `empNo`와 `empNoList`가 동시에 나가던 중복 제거
- lookup 컬럼의 헤더 필터 복원 동작은 그대로 유지

## 정리

DevExtreme 그리드 상태를 직접 읽어 필터를 복원할 때는 `selectedFilterOperation`이 **비어 있을 수 있다**는 점을 전제로 해야 한다. 이 값이 없다는 건 "연산자가 없다"가 아니라 "**컬럼 타입의 기본 연산자를 쓰고 있다**"는 뜻이다. `|| '='` 같은 폴백은 코드 컬럼에서는 우연히 맞지만 텍스트 컬럼에서는 의미를 바꿔버린다.

폴백을 굳이 둬야 한다면 컬럼 타입에 맞춰야 한다.

```javascript
const defaultOperation = column.lookup || column.dataType === 'number' ? '=' : 'contains';
const operator = column.selectedFilterOperation || defaultOperation;
```

다만 이번에는 복원 자체가 헤더 필터 보정용이었으므로, 폴백을 고치기보다 **적용 범위를 줄이는 쪽**이 부작용이 적다고 판단했다.
