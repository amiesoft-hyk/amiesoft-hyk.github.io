---
layout: post
title: "CustomStore를 쓰면 그리드 높이가 비어 페이지가 통째로 스크롤된다"
date: 2026-08-12 15:00:00 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, DataGrid, CustomStore, 스크롤, 레이아웃]
---

## 문제 상황

대상자를 등록·수정하는 목록 화면에서 스크롤 동작이 어긋났다.

- 행이 많아지면 **브라우저 오른쪽에 페이지 스크롤바**가 생긴다
- 스크롤을 내리면 컬럼 헤더, 필터행, 상단 툴바(추가/저장/삭제)가 화면 밖으로 사라진다
- 같은 프로젝트의 다른 목록 화면은 그리드 안에서만 스크롤되고 헤더가 고정된다

특히 "대상자 추가" 버튼으로 한 번에 수십 명을 신규 행으로 밀어 넣는 화면이라, 행을 추가하자마자 헤더가 위로 날아가 버려서 어떤 컬럼에 뭘 입력하는지 알 수 없는 상태가 됐다.

같은 공통 그리드 컴포넌트를 쓰는데 왜 이 화면만 다른지가 출발점이었다.

## 원인

### 1. 그리드 높이를 렌더 훅에서 계산하고 있었다

공통 그리드 컴포넌트는 높이를 이렇게 잡는다.

```js
const calculateHeight = propHeight => {
  if (propHeight === null || propHeight === undefined) {
    // 그리드 상단 위치를 기준으로 뷰포트 나머지를 채운다
    return `calc(100vh - ${gridTopOffset.value}px)`;
  }
  return propHeight;
};

onBeforeUpdate(() => {
  gridConfig.height = calculateHeight(gridConfig.height);
});
```

높이가 잡히면 DevExtreme이 내부 스크롤 영역을 만들고, 헤더는 고정된다. 높이가 비어 있으면 그리드는 **내용 높이만큼 그대로 늘어난다.** 그러면 스크롤 주체가 그리드가 아니라 `body`가 된다.

핵심은 이 계산이 `onBeforeUpdate`에 걸려 있다는 점이다. 이건 **컴포넌트가 다시 렌더링될 때만** 실행되는 훅이다.

### 2. CustomStore로 데이터를 채우면 렌더 갱신이 일어나지 않는다

정상 동작하던 화면들은 대개 이런 형태다.

```js
// 조회 결과를 반응형 배열에 대입 → Vue 렌더 갱신 → onBeforeUpdate 발동
gridConfig.dataSource = response.data.items;
```

문제의 화면은 서버 페이징·필터를 직접 다루려고 `CustomStore`를 그대로 넘기고 있었다.

```js
const dataSource = new CustomStore({
  key: 'id',
  async load(loadOptions) {
    const res = await callApi(buildParams(loadOptions));
    return { data: res.rows, totalCount: res.totalCount };
  },
});

const gridConfig = reactive({
  dataSource,   // 인스턴스를 한 번 넘기고 끝
  // height 미지정
  // ...
});
```

`CustomStore`는 **DevExtreme 내부에서 자기 데이터를 갱신**한다. Vue 입장에서는 `dataSource`가 가리키는 객체가 처음부터 끝까지 동일한 인스턴스이므로, 데이터가 몇 번 바뀌든 **컴포넌트를 다시 렌더링할 이유가 없다.**

그래서 `onBeforeUpdate`가 돌지 않고, `height`는 계속 비어 있었다.

### 3. 가끔 되고 가끔 안 되는 이유

이 화면도 아주 가끔은 높이가 잡혔다. 컬럼 lookup에 쓰는 별도 목록을 비동기로 채우고 있었는데, 그 값이 반응형이라 대입 시점에 렌더가 한 번 돌면서 `onBeforeUpdate`가 얻어걸린 것이다.

즉 **레이아웃이 무관한 상태 변화의 타이밍에 의존**하고 있었다. 재현이 들쭉날쭉했던 이유다.

정리하면 이렇다.

| 데이터 소스 형태 | 렌더 갱신 | height 계산 | 스크롤 주체 |
|---|---|---|---|
| 배열 대입 | 발생 | 실행됨 | 그리드 내부 |
| CustomStore 인스턴스 | 없음 | 실행 안 됨 | 페이지(body) |

## 해결 방법

공통 그리드는 이 상황을 위한 옵션을 이미 갖고 있었다. `autoHeight`를 켜면 렌더 훅과 무관하게 **마운트 직후 한 번**, 그리고 **창 크기가 바뀔 때마다** 높이를 다시 계산한다.

```js
const applyAutoHeight = async () => {
  if (!gridConfig.autoHeight || props.dataGrid.height != null) {
    return;
  }

  await nextTick();
  const height = calculateHeight(null);
  gridConfig.height = height;

  const instance = getGridInstance();
  instance?.option('height', height);
  instance?.updateDimensions();
};

onMounted(async () => {
  await applyAutoHeight();
  if (gridConfig.autoHeight) {
    window.addEventListener('resize', applyAutoHeight);
  }
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', applyAutoHeight);
});
```

화면 쪽에서는 옵션 한 줄이면 된다.

```js
const gridConfig = reactive({
  dataSource,
  // CustomStore를 dataSource로 직접 쓰면 렌더 갱신이 없어 높이가 비고
  // 페이지 전체가 스크롤된다. autoHeight로 뷰포트 기준 높이를 계산한다.
  autoHeight: true,
  scrolling: {
    mode: 'standard',
    showScrollbar: 'always',
  },
  // ...
});
```

`showScrollbar: 'always'`는 취향의 영역이지만, 대량 신규 행을 넣는 화면에서는 "더 아래에 행이 있다"는 신호가 있는 편이 낫다.

### 검증 포인트

높이 문제는 눈으로 확인하는 게 가장 빠르다.

- 브라우저 오른쪽에 **페이지 스크롤바가 생기지 않는지**
- 스크롤 시 컬럼 헤더·필터행·툴바가 고정되는지
- 창 크기를 바꿨을 때 그리드 하단이 뷰포트에 다시 맞는지
- 데이터 0건일 때 빈 영역 높이가 무너지지 않는지

## 결과

- 그리드가 뷰포트를 채우고, 스크롤이 그리드 내부에서만 발생
- 수십 건을 한 번에 추가해도 헤더와 툴바가 그대로 유지
- 같은 프로젝트의 다른 목록 화면들과 동작이 일치

## 남는 교훈

**렌더 훅에 레이아웃 계산을 얹으면, 데이터가 어떻게 들어오느냐에 따라 레이아웃이 달라진다.**

`onBeforeUpdate`에서 높이를 잡는 방식은 "데이터가 바뀌면 컴포넌트가 다시 렌더된다"는 가정 위에 서 있다. 반응형 배열을 쓰는 동안은 이 가정이 맞지만, 데이터 갱신을 라이브러리 내부로 넘기는 순간 조용히 깨진다.

레이아웃은 가능하면 **레이아웃 이벤트(mount, resize)** 에 붙이는 게 맞다. 이번 옵션이 하는 일이 정확히 그것이었다.
