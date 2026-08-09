---
layout: post
title: "저장했는데 셀에 연두색이 남는다 — DevExtreme batch 모드의 cellValue()"
date: 2026-08-07 18:40:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, batch, cellValue]
---

## 문제 상황

목록 그리드의 상태 컬럼에서 이전/다음 버튼으로 상태를 바꾸는 기능이 있다. 버튼을 누르면 서버 저장은 정상적으로 끝나는데, **해당 셀에 연두색 배경이 남아 지워지지 않았다.**

- 저장 API는 성공, "정상적으로 저장되었습니다" 토스트도 정상
- 그런데 셀만 계속 강조 상태
- 다른 행을 클릭해도, 그리드를 다시 그려도 사라지지 않음
- 검색을 다시 하면 그때 사라짐

사용자 입장에서는 "저장이 안 된 것 같다"로 읽히는 상태였다.

## 원인

### 1. 연두색의 정체는 "미저장 변경" 표시였다

DevExtreme 테마 CSS를 뒤져 보면 해당 색이 나온다.

```css
.dx-datagrid-rowsview .dx-data-row:not(.dx-edit-row) .dx-cell-modified {
    background-color: rgba(139, 195, 74, .32);
}
```

`dx-cell-modified`는 **아직 저장되지 않은 편집이 있는 셀**에 붙는 클래스다. 즉 hover 효과나 커스텀 스타일이 아니라, 그리드가 "이 셀은 변경됐지만 저장 안 됨"이라고 표시하고 있던 것이다.

### 2. `cellValue()`는 값 대입이 아니라 편집 등록이다

문제의 코드다.

```javascript
const res = await callApi(payload);
if (isSuccess(res)) {
  options.component.cellValue(options.rowIndex, 'statusCd', nextStatus);
  options.component.repaint();
  showToast('정상적으로 저장되었습니다.');
}
```

이 그리드의 편집 모드는 `batch`였다.

```javascript
editing: {
  allowUpdating: false,   // 저장/취소 툴바 버튼 없음
  allowAdding: false,
  allowDeleting: false,
  mode: 'batch',
}
```

`batch` 모드에서 `cellValue(rowIndex, field, value)`는 **단순 값 대입이 아니라 "저장 대기 중인 편집"으로 등록**된다. 그 결과 셀에 `dx-cell-modified`가 붙는다.

### 3. 지울 방법이 없었다

세 가지가 겹쳐 상태가 고착됐다.

- `repaint()`는 다시 그리기만 할 뿐 **편집 상태를 지우지 않는다**
- `allowUpdating: false`라 저장/취소 툴바 버튼이 없어 **사용자가 해제할 수 없다**
- 재조회 시에만 사라졌는데, 조회 함수가 내부적으로 `cancelEditData()`를 호출하고 있었기 때문

즉 **서버 저장은 끝났는데 화면만 "저장 안 됨" 상태로 남는** 모순이 생겼다.

## 해결 방법

서버 저장이 이미 끝난 값이므로 편집 상태를 만들 이유가 없다. **행 데이터를 직접 갱신하고 해당 행만 다시 그린다.**

```javascript
const res = await callApi(payload);
if (isSuccess(res)) {
  // batch 편집 모드에서 cellValue()는 '미저장 변경'으로 기록되어
  // dx-cell-modified(연두색) 표시가 남는다.
  // 서버 저장이 끝난 값이므로 행 데이터를 직접 갱신하고 해당 행만 다시 그린다.
  options.row.data.statusCd = nextStatus;
  options.component.repaintRows([options.rowIndex]);

  showToast('정상적으로 저장되었습니다.');
}
```

`repaintRows`로 해당 행만 다시 그리면 `cellTemplate`이 재실행되므로, 상태에 따라 달라지는 버튼의 `disabled` 처리도 함께 갱신된다. 전체 `repaint()` + `updateDimensions()` 호출이 불필요해져 코드도 짧아졌다.

같은 화면의 다른 토글 기능은 이미 `row.data`를 직접 고치는 방식이었다. 한 파일 안에서 두 방식이 섞여 있었고, 문제가 난 쪽만 `cellValue()`를 쓰고 있었다.

### `cancelEditData()`를 붙이면 안 되는 이유

"편집 상태를 지우면 되지 않나" 싶어 `cancelEditData()`를 떠올리기 쉽지만 **틀린 해법이다.** batch 모드에서 `cellValue()`로 넣은 값 자체가 취소 대상이므로, 호출하면 방금 반영한 값까지 되돌아가 화면이 이전 상태로 돌아간다.

## 결과

- 상태 변경 후 연두색 강조가 남지 않음
- 버튼 활성/비활성 상태도 같이 갱신됨
- 재조회 없이도 화면과 서버 상태가 일치

## 정리

`batch` / `cell` 편집 모드에서 **서버에 이미 저장한 값을 화면에 반영할 때는 `cellValue()`를 쓰면 안 된다.** 이 API는 "사용자가 셀을 편집했다"는 의미이지 "데이터가 이미 바뀌었다"는 의미가 아니다.

| 상황 | 적절한 방법 |
|---|---|
| 사용자 편집을 코드로 대신 입력 (저장 전) | `cellValue()` |
| 서버 저장이 끝난 값을 화면에 반영 | `row.data` 직접 갱신 + `repaintRows()` |
| 서버 상태를 통째로 다시 맞춤 | 재조회 (`refresh()` 등) |
