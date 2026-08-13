---
layout: post
title: "다운로드한 엑셀에서 필터·정렬 쓰기 — ExcelJS 직접 작성과 exportDataGrid 두 경로"
date: 2026-08-13 16:05:00 +0900
categories: [오류해결, ExcelJS]
tags: [ExcelJS, DevExtreme, Vue, 엑셀다운로드]
---

## 문제 상황

그리드에서 내려받은 엑셀 파일을 열면 **헤더 행에 필터 드롭다운이 없어서 엑셀 안에서
정렬·필터를 할 수 없다**는 요청이 들어왔다. 같은 시스템의 다른 화면은 되는데 일부
화면만 안 되는 상황이었다.

확인해 보니 화면마다 엑셀 생성 방식이 두 가지로 나뉘어 있었고, 각각 설정 지점이 달랐다.

| 방식 | 설명 | 설정 위치 |
| --- | --- | --- |
| ExcelJS 직접 작성 | 워크시트에 셀을 하나씩 채움 | `worksheet.autoFilter` |
| `exportDataGrid()` | DevExtreme이 그리드를 워크시트에 씀 | `autoFilterEnabled` 옵션 |

## 원인

### 1. ExcelJS로 직접 작성하는 경우

서버 페이징(remoteOperations)이 켜진 그리드에서는 `exportDataGrid()`가 전체 데이터를
받아오지 못하는 문제 때문에, 일부 화면은 ExcelJS로 직접 워크시트를 채우고 있었다.

```js
const worksheet = workbook.addWorksheet(title);

// 1행: 병합된 제목, 2행: 컬럼 헤더, 3행~: 데이터
worksheet.mergeCells(1, 1, 1, columns.length);
worksheet.getCell(1, 1).value = title;

const headerRow = worksheet.getRow(2);
columns.forEach((column, index) => {
  headerRow.getCell(index + 1).value = column.caption;
});

rows.forEach((row, rowIndex) => { /* 3행부터 데이터 채우기 */ });
```

이 방식은 셀을 직접 쓰기 때문에 **아무것도 하지 않으면 autoFilter가 생기지 않는다.**

### 2. `exportDataGrid()`를 쓰는 경우

DevExtreme은 `autoFilterEnabled` 옵션을 지원하지만 **기본값이 `false`**다. 공통 그리드
컴포넌트가 설정값에서 이 값을 읽어 넘기는 구조였는데,

```js
autoFilterEnabled: dataGrid.excel.autoFilterEnabled ? dataGrid.excel.autoFilterEnabled : false,
```

해당 화면의 `dataGrid.excel`에 그 키 자체가 없어 항상 `false`로 동작하고 있었다.

## 해결 방법

### ExcelJS 직접 작성 — `worksheet.autoFilter` 지정

데이터를 다 채운 뒤 **헤더 행 범위**를 지정한다. 제목이 1행, 헤더가 2행이므로 2행을
기준으로 잡는다.

```js
worksheet.autoFilter = {
  from: { row: 2, column: 1 },
  to: { row: 2, column: columns.length },
};
```

`from`/`to`를 헤더 행만 지정해도 엑셀이 아래 데이터 범위를 자동으로 인식한다.

### exportDataGrid — 옵션만 켜기

그리드 설정에 값을 추가하면 된다.

```js
excel: {
  title: '처리 대장',
  autoFilterEnabled: true,   // ← 헤더 행에 필터 드롭다운 생성
},
```

`topLeftCell: { row: 4, column: 1 }`처럼 시작 위치를 옮긴 경우에도 DevExtreme이 실제
헤더 행 위치를 기준으로 필터를 붙여주기 때문에 별도 계산이 필요 없다.

### 날짜 컬럼 형식 통일

직접 작성 방식에서는 서버 원본값이 그대로 셀에 들어가면서 **엑셀에서 날짜 정렬이 엉키는**
문제도 같이 있었다. 값을 쓰기 직전에 표시 형식으로 통일했다.

```js
// 날짜 컬럼은 원본값 그대로 두면 엑셀 정렬 형식이 어긋난다
if (column.dataType === 'date' && value) {
  value = formatDate(value, undefined, 'YYYY-MM-DD');
}
cell.value = value ?? '';
```

`exportDataGrid()` 방식은 그리드 표시값 기준으로 내보내므로 이 처리가 필요 없다.

## 결과

두 방식 모두 헤더 행에 필터·정렬 드롭다운이 생겼고, 사용자가 다운로드한 파일에서
바로 정렬·필터를 쓸 수 있게 됐다.

정리하면 이렇다.

- **엑셀을 직접 조립하고 있다면** → `worksheet.autoFilter`를 헤더 행 범위로 지정
- **`exportDataGrid()`를 쓰고 있다면** → `autoFilterEnabled: true` (기본값이 false라는 점에 주의)
- CSV로 내려받는 경로에는 영향이 없다. autoFilter는 xlsx 전용 기능이다

같은 시스템 안에서도 엑셀 생성 경로가 여러 개면 "다른 화면은 되는데 여기만 안 된다"는
요청이 반복된다. 화면을 고칠 때 **어떤 경로로 만들어지는 엑셀인지부터 확인**하면 헤매지 않는다.
