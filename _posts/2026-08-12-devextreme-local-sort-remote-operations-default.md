---
layout: post
title: "정렬 가드가 로컬 데이터에서 뒤통수를 쳤다 — remoteOperations 기본값이 true였다"
date: 2026-08-12 14:20:00 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, DataGrid, 정렬, remoteOperations, ArrayStore]
---

## 문제 상황

대상자를 고르는 선택 팝업에서 정렬이 일부만 동작하지 않는다는 제보를 받았다.

- 사번, 성명, 재직상태 컬럼 → 헤더 클릭하면 정상적으로 오름/내림차순 전환
- 팀, 섹션 컬럼 → **헤더 클릭해도 순서가 그대로**

이전에 겪었던 [서버 정렬이 조용히 무시되던 문제]({{ site.baseurl }}{% post_url 2026-08-07-devextreme-remote-sort-selector-function %})와 증상이 비슷했다. 정렬 화살표는 바뀌는데 목록만 그대로다. 콘솔 에러도 없다.

그런데 이번 팝업은 **서버 조회를 하지 않는다.** 부모 화면이 한 번 조회한 배열을 그대로 `dataSource`에 꽂아주는, 완전한 로컬 그리드였다. 서버 정렬 파라미터 문제일 리가 없었다.

## 원인

### 1. 팀·섹션은 원본 데이터에 없는 "계산 컬럼"이었다

팝업에 내려오는 행은 조직 경로를 하나의 문자열로만 들고 있었다.

```js
// 서버가 내려주는 행
{
  empNo: '1234567',
  name: '홍길동',
  orgPath: '본부 > 지원센터 > 1팀 > A섹션',
  // teamName, sectionName 같은 필드는 없다
}
```

화면에 팀/섹션을 따로 보여주기 위해, 컬럼 정의는 경로 문자열을 쪼개는 `calculateCellValue`를 쓰고 있었다.

```js
const createOrgLevelColumns = () =>
  ['centerName', 'teamName', 'sectionName'].map(field => ({
    dataField: field,
    caption: LABEL[field],
    // 행에는 field가 없다. 경로를 쪼개서 표시값을 만든다
    calculateCellValue: rowData => splitOrgPath(rowData?.orgPath)[field],
  }));
```

즉 `dataField: 'teamName'`은 **화면상의 식별자일 뿐, 실제 데이터에는 존재하지 않는 키**다.

### 2. 공통 그리드가 정렬 기준을 dataField로 강제하고 있었다

여기서 이전 글의 수정이 등장한다. 서버 정렬에서 표시값 함수가 selector로 잡히는 걸 막으려고, 공통 그리드 컴포넌트의 컬럼 전처리에 이런 코드가 들어가 있었다.

```js
// 서버 정렬에서는 selector가 서버가 아는 필드명이어야 한다
if (
  gridConfig.remoteOperations?.sorting &&
  column.dataField &&
  column.allowSorting !== false &&
  !column.calculateSortValue &&
  typeof getDisplayValueOption(column) === 'function'
) {
  column.calculateSortValue = column.dataField;
}
```

첫 번째 조건이 `remoteOperations.sorting`이다. 작성 당시 의도는 분명했다. **"로컬 정렬은 표시값 기준으로 도는 게 자연스러우니 건드리지 말자."** 가드는 제대로 붙어 있었다.

### 3. 그런데 그 플래그가 로컬 그리드에서도 true였다

팝업 컴포넌트는 `remoteOperations`를 아예 설정하지 않았다. 서버 조회를 안 하니 설정할 이유가 없었다.

문제는 공통 그리드가 **설정하지 않은 옵션을 기본값과 병합**한다는 점이다.

```js
// 공통 그리드의 기본 옵션
const gridDefaultOptions = {
  remoteOperations: {
    filtering: true,
    sorting: true,   // ← 기본값이 true
    grouping: false,
    paging: true,
  },
  // ...
};

Object.assign(gridConfig, { ...gridDefaultOptions, ...props.dataGrid, ...mergedObjects });
```

`remoteOperations`를 안 넘긴 팝업은 조용히 `sorting: true`를 물려받았다. 가드는 "이 그리드는 서버 정렬이다"라고 판단하고 `calculateSortValue = 'teamName'`을 심었다.

DevExtreme은 로컬 배열이든 원격이든 selector를 똑같이 해석한다. 문자열 selector면 `row['teamName']`을 읽는다.

```js
// 실제로 벌어진 일
rows.sort((a, b) => compare(a['teamName'], b['teamName']));
// 모든 행에서 undefined → 비교 결과가 늘 동일 → 순서 그대로
```

**모든 행의 정렬 키가 `undefined`**라서, 정렬은 "실패"한 게 아니라 "아무것도 바꾸지 않은 채 성공"했다. 에러가 없는 이유다.

사번·성명·재직상태가 멀쩡했던 이유도 같은 논리다. 그 컬럼들은 `dataField`가 실제 데이터의 키와 일치했다.

정리하면 이렇다.

| 단계 | 동작 | 판단 |
|---|---|---|
| 팝업 컴포넌트 | `remoteOperations` 미설정 | 서버 조회를 안 하니 합리적 |
| 공통 그리드 | 기본값 `sorting: true` 병합 | 대부분의 목록 화면 기준으로 합리적 |
| 컬럼 전처리 | 서버 정렬로 보고 selector를 dataField로 고정 | 이전 버그 수정으로서 합리적 |
| 로컬 정렬 | 없는 필드로 정렬 → 무변화 | 에러 없음 |

각 단계는 여전히 다 맞다. 어긋난 건 **"remoteOperations 플래그"와 "데이터 소스가 실제로 원격인가"가 같은 뜻이 아니라는 점**이었다.

## 해결 방법

가장 좁은 지점에서 끊었다. 팝업의 데이터는 항상 로컬 배열이므로, 부서 컬럼의 정렬 기준을 **표시값 계산 함수로 직접 지정**했다.

전처리 코드가 `!column.calculateSortValue`일 때만 덮어쓰기 때문에, 화면이 명시한 값이 항상 이긴다.

```js
columns: [
  // 팝업 목록은 항상 로컬 배열이고 행에는 orgPath만 있다.
  // 공통 그리드는 remoteOperations.sorting 기본값(true) 때문에 정렬 기준을
  // dataField로 바꾸는데, 행에 그 필드가 없어 정렬 키가 전부 undefined가 된다.
  // 경로에서 계산한 표시값을 정렬 기준으로 명시한다.
  ...createOrgLevelColumns().map(column => ({
    ...column,
    calculateSortValue: column.calculateCellValue,
  })),
  // ...
]
```

`calculateCellValue`를 그대로 재사용했다. 화면에 보이는 값과 정렬 기준이 같은 함수에서 나오므로 어긋날 여지가 없다.

### 다른 선택지도 있었다

| 방법 | 장점 | 안 고른 이유 |
|---|---|---|
| 팝업에 `remoteOperations: { sorting: false, ... }` 지정 | 원인을 정확히 겨냥 | 이 플래그는 헤더필터 후보 누적 방식도 함께 바꾼다. 팝업을 쓰는 모든 화면의 필터 동작이 바뀔 위험 |
| 전처리에서 데이터 소스가 배열인지 검사 | 근본 해결 | 런타임에 dataSource 타입이 바뀔 수 있어 판정이 불안정. 공통 컴포넌트라 파급이 큼 |
| 컬럼 팩토리에 `calculateSortValue` 기본 탑재 | 한 곳만 고침 | 이 팩토리는 서버 정렬 화면들도 쓴다. 그쪽에서는 다시 함수가 selector로 나가는 원래 버그가 재발 |

결국 "이 팝업은 로컬 데이터가 확정"이라는 사실을 아는 유일한 지점에서 고치는 게 파급이 가장 작았다.

## 결과

- 팝업의 센터/팀/섹션 컬럼 정렬이 정상 동작
- 다른 화면의 서버 정렬 동작은 그대로 (전처리 코드 미변경)
- 이 팝업을 공유하는 화면들도 함께 수정 효과

## 남는 교훈

**`remoteOperations.sorting`은 "서버가 정렬한다"는 선언이지, "데이터 소스가 원격이다"라는 사실이 아니다.** 공통 컴포넌트가 기본값을 병합하는 구조에서는 이 둘이 쉽게 어긋난다.

기본값이 `true`인 옵션을 조건문에 쓸 때는 한 번 더 물어보는 게 좋다.

> 이 옵션을 **명시하지 않은** 사용처에서도 이 분기가 맞는가?

이번 경우 답은 "아니오"였고, 그 간극이 정확히 하나의 화면에서 터졌다.
