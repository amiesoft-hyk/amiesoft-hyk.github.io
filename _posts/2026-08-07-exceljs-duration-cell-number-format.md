---
layout: post
title: "다운로드한 엑셀에서 시간 컬럼이 계산되지 않는다 — 표시형식 '일반'의 정체"
date: 2026-08-07 15:30:00 +0900
categories: [오류해결, ExcelJS]
tags: [ExcelJS, DevExtreme, Vue, 엑셀, numFmt]
---

## 문제 상황

리포트 화면에서 엑셀을 내려받았더니, 시간 컬럼이 계산에 쓸 수 없는 상태로 저장돼 있었다.

- 셀에는 `07:37:10`처럼 제대로 보인다
- 그런데 셀 서식을 열어보면 **표시형식이 '일반'**이다
- `SUM()`을 걸면 0이 나온다

값이 시간처럼 보일 뿐 실제로는 텍스트였다.

## 원인

### 엑셀 내보내기 경로가 두 갈래였다

코드를 훑어보니 프로젝트에 내보내기 경로가 둘 있었다.

**경로 A — 그리드 라이브러리의 내보내기 함수**

```js
exportDataGrid({
  component,
  worksheet,
  customizeCell: ({ gridCell, excelCell }) => {
    excelCell.border = { /* 테두리 */ };
    // 스타일만 얹는다
  },
});
```

이 경로는 라이브러리가 컬럼의 `dataType`(`number` / `date` / `boolean` / `string`)과 `format`을 읽어 **셀 값 타입과 `numFmt`를 자동으로 지정**한다. `customizeCell`은 그 위에 테두리·배경 같은 스타일만 덧입힌다.

**경로 B — 화면이 exceljs로 직접 만드는 내보내기**

```js
const onExporting = async e => {
  e.cancel = true;                       // 라이브러리 경로를 끊는다
  const rows = await fetchAllRows();     // 서버 페이징으로 전체 행 수집
  await exportRowsToExcel(rows);
};

const exportRowsToExcel = async rows => {
  const workbook = new Workbook();
  const worksheet = workbook.addWorksheet(title);

  rows.forEach((row, rowIndex) => {
    columns.forEach((column, colIndex) => {
      const cell = worksheet.getRow(rowIndex + 4).getCell(colIndex + 1);
      cell.value = getDisplayValue(row, column);   // ← 표시용 "문자열"
      applyCellStyle(cell);
    });
  });
  // ...
};
```

문제가 난 화면은 경로 B였다. `getDisplayValue`가 코드값을 코드명으로, 날짜를 표시 포맷 문자열로 바꾼 **문자열**을 돌려주고, 그걸 그대로 `cell.value`에 넣는다. 타입도 서식도 지정하지 않으니 전부 텍스트로 저장된다.

경로 B를 쓰는 이유도 분명했다. 서버 페이징 상태에서 전체 행을 내보내야 하는데, 라이브러리 기본 내보내기는 화면에 로드된 데이터만 처리하기 때문이다.

### dataType이 string이라 경로 A로 바꿔도 마찬가지였다

시간 컬럼 정의를 보니 이렇게 돼 있었다.

```js
const timeColumnOptions = () => ({
  dataType: 'string',                                  // ← 여기
  isDuration: true,
  customizeText: info => formatSecondsToTime(info.value),
});
```

서버는 초 단위 숫자를 주고, 화면 표기만 `customizeText`로 만든다. `dataType`이 `string`이라 경로 A로 내보내도 텍스트가 된다. 자동 타입 지정의 혜택을 받으려면 `dataType: 'number'`로 바꾸고 값 자체를 숫자로 흘려야 하는데, 그러면 화면 표시 로직을 다 손봐야 한다.

## 해결 방법

경로 B에 빠져 있던 타입 지정을 시간 컬럼에 한해 채워 넣기로 했다.

### 엑셀이 시간을 저장하는 방식

엑셀은 시간을 **"하루를 1로 보는 소수"**로 저장한다. 1시간은 `1/24`, 1초는 `1/86400`이다. 그 숫자에 표시형식(`numFmt`)을 씌워 `hh:mm:ss`처럼 보여준다.

그래서 문자열을 초로 파싱한 뒤 86400으로 나누고, 서식을 지정하면 된다.

```js
const SECONDS_PER_DAY = 24 * 60 * 60;
// 누적 시간을 표시하는 사용자 지정 서식.
// 대괄호를 씌운 [h]는 24시간에서 순환하지 않으므로 합계가 하루를 넘어도
// 25:30:00 처럼 그대로 표시된다(hh:mm:ss 는 24:00:00 이 00:00:00 으로 보인다).
const DURATION_FORMAT = '[h]:mm:ss';
const DURATION_PATTERN = /^(\d{2,}):([0-5]\d):([0-5]\d)$/;

const parseDurationSeconds = value => {
  if (typeof value === 'number') {
    return Number.isFinite(value) && value >= 0 ? value : null;
  }
  if (typeof value !== 'string' || value.length === 0) {
    return null;
  }

  const match = DURATION_PATTERN.exec(value);
  if (!match) {
    return null;
  }

  const [, hours, minutes, seconds] = match;
  return Number(hours) * 3600 + Number(minutes) * 60 + Number(seconds);
};

/**
 * 시간 문자열(hh:mm:ss)을 엑셀이 인식하는 숫자 시간값으로 바꾸고
 * 누적 시간 서식을 지정한다.
 *
 * @returns {boolean} 값을 변환했는지 여부
 */
export const applyDurationCell = (excelCell, column, value) => {
  if (!excelCell || column?.isDuration !== true) {
    return false;
  }

  const seconds = parseDurationSeconds(value);
  if (seconds === null) {
    return false;
  }

  excelCell.value = seconds / SECONDS_PER_DAY;
  excelCell.numFmt = DURATION_FORMAT;
  return true;
};
```

호출부는 한 줄 추가하면 된다.

```js
cell.value = value;
applyDurationCell(cell, column, value);   // duration 컬럼이면 숫자 + 서식으로 덮어쓴다
```

`isDuration` 같은 커스텀 플래그를 컬럼에 두면, 어떤 컬럼을 시간으로 볼지 화면이 선언하고 유틸은 판단하지 않아도 된다.

### `hh:mm:ss`가 아니라 `[h]:mm:ss`인 이유

처음에는 시스템 시간 서식(`[$-F400]h:mm:ss AM/PM`)을 썼다가 사용자 지정 서식으로 바꿨다. 대괄호 유무가 중요하다.

| 서식 | 86400초(24시간) 표시 | 90000초(25시간) 표시 |
|---|---|---|
| `hh:mm:ss` | `00:00:00` | `01:00:00` |
| `[h]:mm:ss` | `24:00:00` | `25:00:00` |

대괄호가 없으면 **하루를 넘는 순간 순환**해서 0시로 돌아간다. 시각(時刻)이 아니라 누적 시간(duration)을 다룰 때는 반드시 `[h]`를 써야 한다. 실제로 합계가 24시간을 넘는 행이 있었기 때문에 이 차이가 그대로 버그가 될 뻔했다.

### 왕복 테스트로 확인

`numFmt`가 실제로 파일에 보존되는지는 짐작하지 말고 확인하는 게 좋다. 워크북을 버퍼로 쓴 뒤 다시 읽으면 된다.

```js
it('XLSX 왕복 후에도 시간값과 서식이 유지된다', async () => {
  const workbook = new Workbook();
  const worksheet = workbook.addWorksheet('duration');
  const cell = worksheet.getCell('A1');

  applyDurationCell(cell, { isDuration: true }, '07:37:10');

  const buffer = await workbook.xlsx.writeBuffer();
  const restored = new Workbook();
  await restored.xlsx.load(buffer);
  const restoredCell = restored.getWorksheet('duration').getCell('A1');

  expect(restoredCell.value).toBeInstanceOf(Date);   // 숫자 시간값 → Date로 복원된다
  expect(restoredCell.numFmt).toBe('[h]:mm:ss');
});
```

읽어 들일 때 `Date` 인스턴스로 복원되는 게 포인트다. 텍스트로 저장됐다면 문자열이 나왔을 것이다.

## 결과

- 엑셀에서 시간 컬럼의 셀 서식이 '사용자 지정 · `[h]:mm:ss`'로 잡히고, `SUM()`·차이 계산이 정상 동작한다
- 24시간을 넘는 누적 시간도 순환 없이 그대로 표시된다
- 별도로 만들었던 유틸 파일을 없애고 기존 엑셀 공통 유틸로 합쳐, 서식을 바꿀 때 한 파일만 고치면 되도록 정리했다

## 교훈

**"화면에 제대로 보인다"와 "엑셀이 그 값을 이해한다"는 별개다.** exceljs로 셀을 직접 쓰면 타입 지정은 전부 직접 해야 한다. 그리드 라이브러리의 내보내기 함수가 알아서 해주던 일이라, 직접 작성 경로로 갈아탈 때 조용히 빠지기 쉽다.

내보내기 경로가 두 갈래로 존재하는 프로젝트라면, "이 화면은 어느 경로인가"를 먼저 확인하는 게 디버깅의 출발점이다.
