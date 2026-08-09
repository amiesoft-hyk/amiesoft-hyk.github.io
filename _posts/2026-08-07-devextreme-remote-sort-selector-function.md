---
layout: post
title: "DevExtreme 서버 정렬이 조용히 무시된다 — sort 파라미터에 실려 나간 함수 소스"
date: 2026-08-07 13:10:00 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, DataGrid, 정렬, remoteOperations]
---

## 문제 상황

서버 정렬(`remoteOperations.sorting: true`)을 쓰는 DataGrid에서 일부 컬럼만 정렬이 먹지 않았다.
부서명처럼 화면에 표시할 값을 따로 가공하는 컬럼들이었다.

증상이 고약했다.

- 헤더를 클릭하면 정렬 화살표는 정상적으로 바뀐다
- 그런데 목록 순서는 그대로다
- 콘솔에 에러가 없다
- 서버 응답도 200이다

정렬이 "실패"하는 게 아니라 "무시"되는 상황이라, 어디서부터 봐야 할지 잡히지 않았다.

개발자도구 Network 탭에서 조회 요청의 쿼리 파라미터를 열어보고서야 정체가 드러났다.

```
sort=+function (rowData) { return getDisplayName(rowData); }
```

컬럼명이 있어야 할 자리에 **함수 소스 문자열**이 통째로 들어가 있었다.

## 원인

### 1. DevExtreme이 정렬 selector를 고르는 순서

DevExtreme은 정렬 파라미터를 만들 때 컬럼에서 selector를 이 순서로 찾는다.

```
calculateSortValue → displayField → calculateDisplayValue → dataField
```

앞쪽이 비어 있어야 뒤쪽으로 넘어간다. 즉 `calculateDisplayValue`가 정의되어 있으면 `dataField`까지 도달하지 못한다.

### 2. 공통 그리드가 표시값 함수를 자동 주입하고 있었다

문제의 컬럼들은 서버가 내려준 이름을 그대로 보여주기 위해, 공통 그리드 컴포넌트가 컬럼 전처리 단계에서 `calculateDisplayValue`를 자동으로 붙이고 있었다.

```js
// 공통 그리드의 컬럼 전처리
if (isDeptColumn(column.dataField)) {
  column.lookup ??= buildDeptLookup(column.dataField);
  // 표시값은 서버가 내려준 이름을 그대로 쓴다
  column.calculateDisplayValue ??= rowData => getDeptDisplayName(column.dataField, rowData);
}
```

화면 개발자 입장에서는 컬럼에 `dataField`만 적었는데, 런타임에는 selector 후보가 하나 더 끼어든 상태가 된다.

### 3. 함수가 문자열 결합에 그대로 들어갔다

서버 전송용 파라미터를 만드는 코드는 selector를 문자열로 가정하고 있었다.

```js
// loadOptions.sort → "+field,-field" 형태로 변환
params.sort = loadOptions.sort
  .map(item => `${item.desc ? '-' : '+'}${item.selector}`)
  .join(',');
```

`item.selector`가 함수면 JavaScript가 암묵적으로 `Function.prototype.toString()`을 호출한다. 그래서 함수 소스가 컬럼명 자리에 실린다.

### 4. 서버가 조용히 버렸다

여기가 증상을 어렵게 만든 지점이다. 백엔드는 정렬 속성명을 화이트리스트와 대조한 뒤, **모르는 속성은 예외를 던지지 않고 무시**하고 기본 정렬로 대체한다.

```java
private String toSortColumn(String property) {
    return switch (property) {
        case "ymd" -> "t.ymd";
        case "agtId" -> "u.AGT_ID";
        // ...
        default -> null;   // 모르는 속성 → 조용히 제외
    };
}
```

방어적으로는 옳은 설계다. 임의의 문자열이 SQL `ORDER BY`에 들어가는 걸 막아야 하니까. 하지만 그 덕분에 **에러 없이 정렬만 사라지는** 상태가 만들어졌다.

정리하면 이렇다.

| 단계 | 동작 |
|---|---|
| 컬럼 전처리 | 표시값 함수 자동 주입 |
| DevExtreme | 그 함수를 정렬 selector로 선택 |
| 파라미터 조립 | 함수 → 문자열 강제 변환 |
| 서버 | 모르는 속성이므로 무시, 기본 정렬 적용 |

네 단계 모두 각자는 합리적인데, 이어 붙이니 조용히 실패하는 경로가 됐다.

## 해결 방법

원인은 1번 단계에 있으므로 거기서 끊는다. 컬럼 전처리 때 **정렬 기준을 `dataField`로 명시**해서 selector 탐색이 함수까지 가지 않게 했다.

```js
// 서버 정렬에서는 selector가 서버가 아는 필드명이어야 한다.
// 표시값이 함수면 그 함수가 selector로 잡히므로 정렬 기준을 dataField로 고정한다.
// (로컬 정렬은 표시값 기준 동작을 유지해야 하므로 건드리지 않는다.)
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

조건을 네 개 붙인 이유가 각각 있다.

- `remoteOperations.sorting` — 로컬 정렬은 표시값 기준으로 정렬하는 게 오히려 자연스럽다. 건드리면 안 된다
- `column.dataField` — 계산 전용 컬럼은 대체할 필드명이 없다
- `!column.calculateSortValue` — 화면이 직접 지정한 정렬 기준이 있으면 그쪽을 존중한다
- `typeof ... === 'function'` — 함수일 때만 문제가 되므로 그때만 개입한다

### 우회 방식과 비교

같은 증상을 두고 다른 접근도 나왔었다. `loadOptions.sort`를 아예 무시하고, 그리드 인스턴스에서 컬럼 상태를 직접 읽어 파라미터를 조립하는 방식이다.

```js
// 이렇게도 고칠 수 있지만 권하지 않는다
const buildSortParam = () => {
  return gridInstance
    .getVisibleColumns()
    .filter(column => column.sortOrder && column.dataField)
    .sort((a, b) => (a.sortIndex ?? 0) - (b.sortIndex ?? 0))
    .map(column => `${column.sortOrder === 'desc' ? '-' : '+'}${column.dataField}`)
    .join(',');
};
```

동작은 한다. 하지만 문제가 세 가지 있다.

1. **화면이 지정한 `calculateSortValue`를 무시한다.** 표시 컬럼과 정렬 필드를 다르게 두려는 의도가 통하지 않는다
2. **`dataField` 없는 컬럼은 정렬이 통째로 사라진다.** 필터에서 걸러지고, 그러면 또 조용히 기본 정렬로 폴백한다
3. **정렬 상태의 소스가 둘로 갈라진다.** `loadOptions`와 컴포넌트 인스턴스 상태가 어긋날 여지가 생긴다

특히 3번은 헤더 클릭 정렬을 "오름차순 → 내림차순 → 해제" 3단계로 커스터마이징하면서 마이크로태스크로 `sortOrder`를 지연 변경하는 코드가 있었기에 더 위험했다. 결국 우회 방식은 걷어내고, 원인 지점을 고친 쪽만 남겼다.

방어선 하나는 남겨뒀다. 파라미터 조립 유틸에서 문자열이 아닌 selector를 걸러낸다.

```js
params.sort = loadOptions.sort
  .filter(item => typeof item.selector === 'string')   // 함수는 제외
  .map(item => `${item.desc ? '-' : '+'}${item.selector}`)
  .join(',');
```

정상 경로에는 영향이 없고, 예외적으로 함수가 흘러들어와도 함수 소스가 서버로 나가지는 않는다.

## 결과

- 부서 컬럼들의 서버 정렬이 정상 동작하게 됐다
- 화면마다 `calculateSortValue: 'centerNm'`처럼 필드명을 손으로 적어두던 우회 코드를 걷어냈다. 공통에서 같은 값을 자동으로 넣어주므로 중복이었다
- 다만 lookup만 있고 표시값 함수가 없는 컬럼은 위 조건에 걸리지 않으므로, 그런 컬럼의 명시적 지정은 남겨뒀다

## 교훈

**"에러가 없다"가 "정상이다"는 아니다.** 서버가 알 수 없는 입력을 방어적으로 무시하도록 설계돼 있으면, 잘못된 요청은 조용한 실패로 나타난다. 이런 상황에서는 콘솔보다 **실제로 나간 요청 파라미터**를 먼저 봐야 한다.

그리고 프레임워크가 옵션을 여러 후보 중에서 고르는 구조(`A → B → C → D`)일 때, 공통 레이어가 중간 후보를 자동으로 채우면 하위 레이어의 의도가 가려진다. 자동 주입을 넣을 때는 그 값이 다른 용도로도 쓰이는지 확인해야 한다.
