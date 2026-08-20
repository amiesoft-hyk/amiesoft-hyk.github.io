---
layout: post
title: "URLSearchParams가 null을 문자열 \"null\"로 보내 서버 타입 변환이 실패하던 문제"
date: 2026-08-20 09:50:00 +0900
categories: [오류해결, JavaScript]
tags: [JavaScript, axios, Spring, 쿼리스트링, 타입변환]
---

엑셀 다운로드가 400을 뱉었다. 서버 메시지는 타입 변환 실패였다.
원인은 **`URLSearchParams`가 `null`을 문자열 `"null"`로 직렬화**하고, 서버가 그걸 숫자로 바꾸려다 실패한 것이었다.

## 문제 상황

프론트 콘솔에는 이렇게 찍혔다.

```
Uncaught (in promise) Error: Data Loading Error
    at fetchAllExportRows (report/index.vue:...)
```

서버 응답 메시지는 이랬다.

```
Failed to convert value of type 'java.lang.String' to required type 'java.lang.Long'; For input string: "null"
```

`"null"`이라는 **문자열**이 `Long` 파라미터로 들어온 것이다.

## 원인

### 1. 공통 API 계층의 GET 직렬화

```javascript
// api/index.js
default:
    return httpClient.get(`${path}?${new URLSearchParams(payload?.data)}`, config);
```

`URLSearchParams`는 값을 문자열로 강제 변환한다. `null`도 예외가 아니다.

```javascript
new URLSearchParams({ sheetId: null }).toString();   // "sheetId=null"
new URLSearchParams({ sheetId: undefined }).toString(); // "sheetId=undefined"
```

`axios`의 `params` 옵션은 `null`/`undefined` 키를 알아서 빼주지만, 이렇게 직접 쿼리스트링을 만들면 그 처리가 없다.

### 2. 검색 조건 객체가 초기값 그대로 전송됐다

```javascript
const reportSearch = reactive({ scheduleId: null, sheetId: null, targetIds: [], deptIds: [] });
```

조회 버튼을 누르면 `Object.assign`으로 값이 채워지지만, **조회를 거치지 않고 엑셀 다운로드를 실행하면 초기값 그대로** 서버로 갔다.

실제 전송 파라미터를 찍어보니 명확했다.

```
{"currentpage":1,"scheduleId":null,"sheetId":null,"targetIds":[],"deptIds":[]}
```

### 3. 파라미터 조립 과정에서 정제를 우회했다

```javascript
const buildParams = (loadOptions = {}) => {
  return {
    ...buildGridParams(loadOptions, { ... }),   // 여기까진 정제됨
    ...reportSearch,                            // ← 정제 없이 그대로 덮어씀
  };
};
```

그리드 파라미터 생성 유틸에는 정제가 있었지만 두 가지 한계가 있었다.

```javascript
Object.keys(params).forEach(key => {
  if (params[key] === undefined) {
    delete params[key];     // undefined 만 제거, null 은 통과
  }
});
```

- `undefined`만 지우고 `null`은 그대로 둔다
- 이 정제는 `loadOptions` 기반 파라미터에만 적용되고, 검색 조건은 **그 뒤에** 스프레드되므로 아예 거치지 않는다

빈 배열도 위험했다. `deptIds: []`는 `deptIds=`(빈 문자열)로 나가는데, 서버가 `List<Long>`으로 받으면 같은 유형의 변환 오류가 난다.

## 해결 방법

### 1. 빈 값을 전송 대상에서 제외

```javascript
const buildParams = (loadOptions = {}) => {
  const params = {
    ...buildGridParams(loadOptions, { ... }),
    ...reportSearch,
  };

  // 요청은 URLSearchParams 로 직렬화되어 null/undefined 가 "null"/"undefined" 문자열로 전달된다.
  // 서버의 숫자 타입 파라미터는 이 문자열을 변환하지 못하므로 값이 비어 있는 조건은 아예 보내지 않는다.
  Object.keys(params).forEach(key => {
    const value = params[key];
    if (value === null || value === undefined || (Array.isArray(value) && !value.length)) {
      delete params[key];
    }
  });

  return params;
};
```

빈 배열도 함께 제거했다. "조건 없음"과 의미가 같아 서버 결과가 달라지지 않는다.

### 2. 실행 전 필수 조건 검증

애초에 잘못된 요청이 서버까지 가지 않게 막았다. 조회 버튼에는 있던 검증이 엑셀 경로에는 없었다.

```javascript
const onExporting = async e => {
  e.cancel = true;
  // 조회를 거치지 않으면 검색 조건이 비어 서버 파라미터 검증에서 실패한다.
  // 선택 행 다운로드는 이미 조회된 데이터만 쓰므로 전체 다운로드에만 적용한다.
  if (!e.selectedRowsOnly && !reportSearch.scheduleId) {
    showToast('조회 조건을 먼저 선택해 주세요.');
    return;
  }
  ...
};
```

## 결과

- 조회 없이 엑셀을 실행하면 400 대신 안내 메시지가 뜬다.
- 값이 없는 조건은 쿼리스트링에 아예 실리지 않는다.
- 같은 유형의 실수를 파라미터 조립 단계에서 한 번 더 막아준다.

## 디버깅에서 효과적이었던 것

추측을 반복하는 대신 **양쪽 경로의 파라미터를 직접 찍어 비교**한 게 결정적이었다.

```javascript
console.log('[조회 params]', JSON.stringify(params));
console.log('[엑셀 params]', JSON.stringify(baseParams));
```

여기서 한 가지 더 배운 게 있다. `JSON.stringify`는 **값이 `undefined`인 키를 생략한다.**

```javascript
JSON.stringify({ sort: undefined, filter: undefined });   // "{}"
```

빈 객체 `{}`가 찍히길래 "값이 없다"고 읽었는데, 실제로는 "키는 있고 값이 `undefined`"인 상태였다. 로그만 보고 상태를 단정하면 안 된다.

## 남는 교훈

**쿼리스트링을 직접 조립하는 코드는 `null` 처리를 별도로 해야 한다.** HTTP 클라이언트가 알아서 해줄 거라 가정하기 쉬운데, `new URLSearchParams(obj)`를 직접 쓰는 순간 그 보호막이 사라진다.

그리고 **DTO의 스프레드 순서가 정제 로직을 무력화할 수 있다.** 정제된 객체 뒤에 원본을 스프레드하면 정제가 통째로 덮인다.
