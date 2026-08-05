---
layout: post
title: "DevExtreme DataGrid 부서 필터에 불필요한 항목이 노출된 원인과 해결"
date: 2026-08-05 13:00:02 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, DataGrid, Lookup, HeaderFilter]
---

## 문제 상황

Vue 3와 DevExtreme DataGrid를 사용하는 목록 화면에서 조회 결과는 특정 팀으로 제한했지만,
`팀` 컬럼의 필터 목록에는 관계없는 조직까지 노출되는 문제가 있었다.

같은 컬럼에서 제공하는 두 필터 모두 동일한 증상을 보였다.

- 필터 행의 SelectBox에 전체 조직의 팀 목록이 표시됨
- 컬럼 헤더 필터의 체크 목록에도 전체 팀 목록이 표시됨
- 실제 조회 API는 허용된 팀만 반환하므로 화면 데이터와 필터 후보가 일치하지 않음

조회 결과를 제한했다고 해서 필터 후보도 자동으로 제한되는 것은 아니었다.
DataGrid가 표시 데이터와 별도로 `lookup.dataSource`를 필터 후보로 사용하고 있었기 때문이다.

## 원인

### 공통 그리드 래퍼가 화면별 lookup을 덮어씀

공통 그리드 래퍼는 부서 관련 컬럼명을 감지하면 조직 트리에서 만든 공통 lookup을 자동으로
주입하고 있었다. 여러 화면에서 반복 설정을 줄이는 데는 유용하지만, 특정 화면이 제한된
목록을 명시해도 공통 목록이 나중에 덮어쓰는 구조였다.

```js
// 문제가 되는 단순화된 예시
const level = departmentFieldLevels[column.dataField];

if (level) {
  column.lookup = {
    dataSource: organizationOptions[level],
    displayExpr: 'text',
    valueExpr: 'value',
  };
}
```

페이지에서 세 개의 허용 항목만 지정해도 컬럼 전처리 단계에서 전체 조직 목록으로 교체됐다.

### 필터 행과 헤더 필터는 서로 다른 경로를 사용함

DevExtreme에서 필터 행의 SelectBox와 헤더 필터 팝업은 겉보기에는 비슷하지만 데이터 소스를
결정하는 경로가 다르다.

- 필터 행: `lookup.dataSource` 또는 `editor-preparing`에서 지정한 데이터 사용
- 헤더 필터: `headerFilter.dataSource`가 있으면 이를 우선 사용
- 원격 조회 화면: 공통 래퍼가 현재 페이지 값이나 누적 값을 별도로 구성할 수도 있음

따라서 한쪽만 수정하면 다른 쪽에는 전체 목록이 계속 남을 수 있다.

또한 `editor-preparing` 이벤트를 페이지에 전달한 뒤 공통 래퍼가 다시 editor 옵션을 설정하면,
페이지에서 지정한 값이 최종적으로 유지되지 않는 순서 문제도 생길 수 있다.

## 해결 방법

### 1. 명시적인 화면 설정을 공통 기본값보다 우선시킴

공통 래퍼는 페이지가 lookup을 지정하지 않은 경우에만 기본 조직 목록을 주입하도록 변경했다.

```js
const level = departmentFieldLevels[column.dataField];

if (level && !column.lookup) {
  column.lookup = {
    dataSource: organizationOptions[level],
    displayExpr: 'text',
    valueExpr: 'value',
  };
}
```

이렇게 하면 기존 화면은 계속 공통 lookup을 사용하고, 특별한 제한이 필요한 화면만 자체
목록으로 재정의할 수 있다.

### 2. 필터 행에서도 화면별 lookup을 우선 사용함

필터 편집기를 만들 때 런타임 컬럼에 설정된 lookup을 먼저 해석하고, 없을 때만 공통 조직
목록으로 대체했다.

```js
const configuredColumn = findConfiguredColumn(event.dataField);
const runtimeColumn = resolveRuntimeColumn(configuredColumn);

const items = resolveLookupItems(runtimeColumn)
  || getDefaultDepartmentItems(event.dataField);

event.editorName = 'dxSelectBox';
event.editorOptions = {
  ...event.editorOptions,
  dataSource: items,
  displayExpr: runtimeColumn.lookup?.displayExpr ?? 'text',
  valueExpr: runtimeColumn.lookup?.valueExpr ?? 'value',
  searchEnabled: false,
};
```

핵심은 화면의 `editor-preparing` 설정을 무조건 덮어쓰지 않고, 최종 컬럼 설정에서 명시적
lookup을 다시 찾아 사용하는 것이다.

### 3. 제한된 목록을 필터 행과 헤더 필터에 각각 연결함

특정 화면에서는 허용된 팀 목록을 한 번 정의한 뒤 두 필터에 명시적으로 연결했다.

```js
const allowedTeamOptions = [
  { text: '운영 A팀', value: '운영 A팀' },
  { text: '운영 B팀', value: '운영 B팀' },
  { text: '운영 C팀', value: '운영 C팀' },
];

const teamColumn = {
  dataField: 'teamName',
  lookup: {
    dataSource: allowedTeamOptions,
    displayExpr: 'text',
    valueExpr: 'value',
  },
  headerFilter: {
    dataSource: allowedTeamOptions,
  },
};
```

필터 행은 `lookup`, 헤더 필터는 `headerFilter.dataSource`를 사용하므로 두 설정을 분리해
명시하는 것이 안전하다.

## 결과

- 조회 결과와 필터 후보가 동일한 허용 팀 범위로 일치함
- 필터 행 SelectBox에서 불필요한 조직이 제거됨
- 헤더 필터 체크 목록에서도 동일한 제한 목록만 표시됨
- lookup을 지정하지 않은 기존 화면은 전체 조직 목록을 사용하는 기존 동작 유지
- 화면별 예외를 지원하면서 공통 그리드 래퍼의 재사용성도 유지

DevExtreme 공통 래퍼를 설계할 때 자동 설정은 어디까지나 기본값이어야 한다. 화면에서
명시한 설정을 공통 전처리 로직이 덮어쓰지 않도록 `명시적 설정 > 공통 기본값` 우선순위를
지키는 것이 중요하다. 그리고 필터 문제를 확인할 때는 필터 행과 헤더 필터를 하나의 기능으로
보지 말고 각각의 데이터 소스와 생성 시점을 따로 추적해야 한다.
