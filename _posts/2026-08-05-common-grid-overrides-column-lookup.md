---
layout: post
title: "화면에서 지정한 컬럼 lookup이 무시된다 — 공통 그리드 컴포넌트의 무조건 덮어쓰기"
date: 2026-08-05 13:10:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, lookup, 공통컴포넌트]
---

## 문제 상황

그룹별 리포트 화면에서 **팀 필터 목록을 3개로 제한**해 달라는 요청이 들어왔다. 전체 부서 트리를 다 보여줄 필요 없이 운영 조직 세 팀만 고르게 하면 되는 건이었다.

컬럼 정의에 `lookup`을 직접 지정했다.

```javascript
const TEAM_OPTIONS = [
  { text: '운영1팀', value: '운영1팀' },
  { text: '운영2팀', value: '운영2팀' },
  { text: '운영3팀', value: '운영3팀' },
];

{
  caption: '팀',
  dataField: 'teamNm',
  allowFiltering: true,
  allowHeaderFiltering: true,
  lookup: {
    dataSource: TEAM_OPTIONS,
    displayExpr: 'text',
    valueExpr: 'value',
  },
}
```

그런데 화면을 열어보면 팀 필터 드롭다운에 **여전히 전체 부서 목록**이 나왔다. 헤더 필터도 마찬가지였다. 컬럼 정의를 아무리 고쳐도 반영되지 않아 처음에는 오타나 캐시를 의심했다.

## 원인

### 1. 공통 컴포넌트가 부서 컬럼의 lookup을 무조건 덮어쓴다

이 프로젝트는 DevExtreme DataGrid를 감싼 공통 컴포넌트를 쓴다. 컴포넌트는 컬럼 전처리 단계에서 **부서 계열 컬럼(센터/팀/섹션)을 식별해 lookup을 자동 주입**한다. 부서 목록은 별도 API로 한 번 받아와 레벨별로 캐싱해 둔 것이다.

```javascript
// 컬럼 전처리
const deptLevel = DEPT_LEVEL_BY_FIELD[column.dataField];
if (deptLevel) {
  // 화면이 무엇을 지정했든 그대로 덮어쓴다
  column.lookup = {
    dataSource: deptItemsByLevel[deptLevel],
    displayExpr: 'text',
    valueExpr: 'value',
    allowClearing: false,
  };
}
```

부서 컬럼은 어느 화면에서나 같은 목록을 쓸 거라는 전제로 짠 코드였다. 화면이 목록을 좁히고 싶어 하는 경우가 이번에 처음 나왔다.

### 2. 필터 행 에디터도 부서 컬럼을 우선 분기하고 있었다

같은 전제가 필터 행 에디터 준비 로직에도 들어 있었다. 부서 컬럼이면 컬럼에 설정된 lookup을 **확인조차 하지 않고** 공통 목록을 꽂는다.

```javascript
if (e.parentType === 'filterRow' && (e.column?.lookup || deptLevel)) {
  const runtimeColumn = readRuntimeColumn(e.column);

  // 부서 컬럼이면 무조건 공통 목록
  const lookupItems = deptLevel
    ? deptItemsByLevel[deptLevel]
    : readLookupItems(runtimeColumn);

  e.editorName = 'dxSelectBox';
  e.editorOptions = {
    ...e.editorOptions,
    dataSource: lookupItems,
    displayExpr: deptLevel ? 'text' : runtimeColumn.lookup?.displayExpr,
    valueExpr: deptLevel ? 'value' : runtimeColumn.lookup?.valueExpr,
  };
}
```

`displayExpr` / `valueExpr`까지 `deptLevel` 여부로 갈라놓아서, 화면이 다른 표시 필드를 쓰는 경우에도 공통 규칙이 강제됐다.

### 3. 런타임 컬럼에는 화면 설정이 남아 있지 않다

런타임 컬럼 조회 함수는 그리드 인스턴스의 `columnOption()` 결과를 읽는다. 그런데 인스턴스에 이미 전처리된(=덮어써진) 컬럼이 반영된 뒤라, 여기서 화면이 준 원본 설정을 되찾을 수 없었다. 필터 행 쪽을 고치더라도 참조 대상을 바꾸지 않으면 같은 목록이 나온다.

## 해결 방법

### 주입을 "없을 때만"으로 바꾼다

전처리에서 화면이 이미 lookup을 지정했으면 그대로 두도록 조건을 붙였다. 한 줄짜리 변경이지만 우선순위 규칙 자체가 뒤집힌다.

```diff
  const deptLevel = DEPT_LEVEL_BY_FIELD[column.dataField];
  if (deptLevel) {
-   column.lookup = {
-     dataSource: deptItemsByLevel[deptLevel],
-     displayExpr: 'text',
-     valueExpr: 'value',
-     allowClearing: false,
-   };
+   if (!column.lookup) {
+     column.lookup = {
+       dataSource: deptItemsByLevel[deptLevel],
+       displayExpr: 'text',
+       valueExpr: 'value',
+       allowClearing: false,
+     };
+   }
```

### 필터 행은 화면 설정 컬럼을 먼저 찾는다

런타임 컬럼 대신 **설정 원본(컬럼 정의 배열)** 에서 컬럼을 찾아 참조하도록 바꿨다. 그리고 부서 여부로 분기하던 걸 "화면 설정이 있으면 그것, 없으면 공통 목록" 순서로 뒤집었다.

```diff
- const runtimeColumn = readRuntimeColumn(e.column);
- const lookupItems = deptLevel
-   ? deptItemsByLevel[deptLevel]
-   : readLookupItems(runtimeColumn);
+ const configuredColumn =
+   flattenColumns(gridOptions.columns).find(column => column.dataField === e.column?.dataField)
+   || e.column;
+ const runtimeColumn = readRuntimeColumn(configuredColumn);
+ const lookupItems =
+   readLookupItems(runtimeColumn) || (deptLevel ? deptItemsByLevel[deptLevel] : null);

  e.editorOptions = {
    ...e.editorOptions,
    dataSource: lookupItems,
-   displayExpr: deptLevel ? 'text' : runtimeColumn.lookup?.displayExpr,
-   valueExpr: deptLevel ? 'value' : runtimeColumn.lookup?.valueExpr,
+   displayExpr: runtimeColumn.lookup?.displayExpr || (deptLevel ? 'text' : undefined),
+   valueExpr: runtimeColumn.lookup?.valueExpr || (deptLevel ? 'value' : undefined),
  };
```

`displayExpr` / `valueExpr`도 같은 순서로 통일했다. 화면이 준 값이 있으면 그걸 쓰고, 없을 때만 부서 기본값(`text` / `value`)으로 떨어진다.

### 헤더 필터는 별도로 지정한다

DevExtreme에서 헤더 필터 목록은 `lookup.dataSource`를 따라가지만, 명시적으로 제한하려면 `headerFilter.dataSource`를 따로 주는 편이 확실하다. 화면 쪽에는 둘 다 지정했다.

```javascript
{
  caption: '팀',
  dataField: 'teamNm',
  lookup: {
    dataSource: TEAM_OPTIONS,
    displayExpr: 'text',
    valueExpr: 'value',
  },
  headerFilter: {
    dataSource: TEAM_OPTIONS,
  },
}
```

## 결과

- 해당 화면의 팀 필터에 지정한 3개 항목만 노출
- 필터 행 드롭다운과 헤더 필터 목록이 동일하게 제한됨
- lookup을 지정하지 않은 다른 화면은 기존대로 공통 부서 목록 사용 (동작 변화 없음)

## 정리

공통 컴포넌트가 특정 컬럼을 특별 취급할 때는 **주입(inject)인지 강제(override)인지** 를 처음부터 구분해 두는 게 좋다. 이번 코드는 의도는 주입이었는데 구현은 강제였고, 그 차이가 드러나기까지 화면이 여러 개 쌓인 뒤였다.

경험칙으로 정리하면 이렇다.

- 공통 기본값은 `if (!column.xxx)` 또는 `column.xxx ??= ...` 로 넣는다 — 실제로 같은 파일에서 `calculateDisplayValue`와 `cellTemplate`은 이미 `??=` 로 되어 있어 문제가 없었다
- 화면 설정을 참조할 때는 런타임 상태가 아니라 **설정 원본**을 본다. 런타임에는 이미 공통 규칙이 반영돼 있다
- "이 컬럼이면 A, 아니면 B" 형태의 분기는 "지정값이 있으면 그것, 없으면 기본값" 순서로 뒤집을 수 있는지 먼저 검토한다
