---
layout: post
title: "DevExtreme DataGrid 컬럼이 minWidth 제거 후에도 넓게 보인 이유"
date: 2026-08-12 16:22:07 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, DataGrid, UI, ColumnWidth]
---

## 문제 상황

Vue 3와 DevExtreme DataGrid로 만든 조회 화면에 컬럼이 네 개뿐인 경우가 있었다. 각 컬럼에 지정된 `minWidth`가 너무 커 보여 이 값을 제거했지만, 넓은 모니터에서 확인해 보니 컬럼은 여전히 화면 전체를 채우고 있었다.

처음에는 최소 너비가 원인이라고 생각하기 쉽다. 하지만 `minWidth`는 컬럼이 줄어들 수 있는 하한만 정할 뿐, 남는 공간을 어떻게 나눌지는 결정하지 않는다. 따라서 최소 너비를 제거하는 것만으로는 원하는 결과가 나오지 않았다.

## 원인

### `minWidth`와 `width`의 역할 차이

`minWidth`는 컬럼의 최소 크기다. 화면에 여유 공간이 있으면 실제 컬럼은 이 값보다 얼마든지 넓어질 수 있다.

```js
const columns = [
  {
    dataField: 'workDate',
    minWidth: 130,
  },
];
```

반면 `width`는 해당 컬럼이 사용할 기준 너비를 명시한다.

```js
const columns = [
  {
    dataField: 'workDate',
    width: 170,
  },
];
```

### 자동 너비 옵션이 남는 공간을 분배한다

그리드에는 다음 설정이 적용되어 있었다.

```js
const gridOptions = {
  columnAutoWidth: true,
};
```

`columnAutoWidth`가 활성화된 상태에서는 콘텐츠 크기를 기준으로 컬럼 너비를 계산한다. 컬럼 수가 적고 그리드 컨테이너가 넓으면, 결과적으로 각 컬럼이 넓게 보일 수 있다.

즉 이번 현상은 다음 두 조건이 함께 만든 결과였다.

- 그리드는 화면 너비 전체를 사용한다.
- 표시할 컬럼은 네 개뿐이다.
- 자동 너비 계산이 활성화되어 있다.
- `minWidth`를 제거해도 남는 공간 자체는 그대로다.

## 해결 방법

자동 너비 분배를 끄고 각 컬럼에 의도한 너비를 명시했다.

```js
const gridOptions = {
  allowColumnResizing: true,
  columnResizingMode: 'widget',
  columnAutoWidth: false,
  columns: [
    {
      dataField: 'workDate',
      width: 170,
    },
    {
      dataField: 'groupDuration',
      width: 180,
    },
    {
      dataField: 'trainingDuration',
      width: 180,
    },
    {
      dataField: 'totalDuration',
      width: 170,
    },
  ],
};
```

네 컬럼의 전체 너비는 700px로 고정되고, 그리드 오른쪽의 남는 영역은 빈 공간으로 유지된다. 사용자는 필요할 때 컬럼 리사이징 기능으로 너비를 직접 조절할 수 있다.

조회 API, 서버 페이징, 필터 및 정렬 로직은 건드리지 않았다. 화면 표현에만 필요한 최소 변경으로 범위를 제한했다.

## 결과

- 넓은 화면에서도 네 컬럼이 전체 영역으로 과도하게 늘어나지 않는다.
- 날짜와 시간 값이 잘리지 않을 정도의 여유 너비를 확보했다.
- 컬럼 리사이징 기능은 그대로 유지했다.
- 데이터 조회와 서버 처리 로직에는 영향이 없다.

이번 작업에서 얻은 핵심은 간단하다. DevExtreme DataGrid의 컬럼이 예상보다 넓다면 `minWidth`만 확인하지 말고, `columnAutoWidth`와 명시적 `width`가 함께 어떻게 동작하는지 확인해야 한다.

> `minWidth`는 하한이고 `width`는 의도한 크기다. 적은 수의 컬럼을 좁게 모아 배치하려면 자동 너비 계산을 끄고 명시적 너비를 주는 방식이 가장 예측 가능하다.
