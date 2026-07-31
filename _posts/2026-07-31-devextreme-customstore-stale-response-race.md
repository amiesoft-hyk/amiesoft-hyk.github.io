---
layout: post
title: "검색 조건을 바꿨는데 이전 결과가 뜬다 — CustomStore 재조회 경쟁 상태 정리"
date: 2026-07-31 18:30:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, CustomStore, 비동기]
---

## 문제 상황

월별 집계 그리드에 **계약형태(전체 / 센터 / 외주)** 라는 검색 조건을 새로 추가했다. 조건을 바꿔 조회하면 세 가지 증상이 나타났다.

1. 조건을 빠르게 연속으로 바꾸면 **마지막에 선택한 조건이 아닌 결과**가 남는다
2. 마스터-디테일(행 펼치기)로 열어둔 상세 그리드가 **이전 조건 기준 데이터**를 그대로 보여준다
3. 특정 조합에서는 검색 조건의 기간이 **무시된 것처럼** 보인다

셋 다 "요청은 제대로 나갔는데 화면이 안 맞는" 유형이라 서버부터 의심하기 쉬웠지만, 실제 원인은 전부 프론트에 있었다.

## 원인

### 1. 늦게 도착한 이전 응답이 화면을 덮어쓴다 (stale response)

조회 버튼을 누를 때마다 `CustomStore`를 새로 만들어 `dataSource`에 꽂는 구조였다.

```javascript
const searchList = async searchData => {
  const { startDt, endDt, employmentFilter } = normalize(searchData);

  dataGrid.dataSource = new CustomStore({
    key: 'baseDt',
    load: async loadOptions => {
      const res = await callApi({
        actionName: 'GET_MONTHLY_SUMMARY',
        data: { ...toGridParams(loadOptions), startDt, endDt, employmentFilter },
      });
      return { data: res.data.data, totalCount: res.data.header.totalCount };
    },
  });
};
```

문제는 `dataSource`를 교체해도 **이미 날아간 이전 요청이 취소되지 않는다**는 점이다. DevExtreme은 새 `load()`를 호출하지만, 직전 store의 `load()`는 여전히 `await` 상태로 살아 있다.

```
t0  조건=CENTER 로 조회 → 요청 A 발사 (느림, 900ms)
t1  조건=OUTSOURCED 로 조회 → dataSource 교체 → 요청 B 발사 (빠름, 200ms)
t2  B 응답 도착 → 화면에 OUTSOURCED 결과 표시  ✔
t3  A 응답 도착 → 화면에 CENTER 결과 표시      ✘  ← 나중에 온 옛 응답이 이긴다
```

전형적인 **last-write-wins 경쟁 상태**다. 조건이 하나뿐일 때는 티가 안 나다가, 조건을 추가하고 사용자가 토글하듯 바꾸기 시작하면서 드러났다.

### 2. 마스터-디테일 props를 재조회 시 정리하지 않았다

상세 그리드는 부모가 넘겨준 reactive props(기준월, 분류, 검색조건)를 그대로 받아 자기 데이터를 조회한다. 부모는 이걸 `Map`에 보관하며 셀 링크 클릭 시 갱신한다.

```javascript
const detailPropsMap = new Map();

// 행마다 상세 컴포넌트에 넘길 props 를 만들어 보관
const detailProps = reactive({ baseDt: row.baseDt, category: null });
detailPropsMap.set(row.baseDt, detailProps);
```

그런데 **재조회 시 이 Map도, 펼쳐진 행도 초기화되지 않았다.** 이미 펼쳐진 상세 그리드 컴포넌트는 그대로 살아 있고, 새 조건은 새로 만들어진 props에만 들어가므로 화면에 떠 있는 상세는 옛 조건 기준으로 남는다.

게다가 상세 그리드의 재조회 트리거가 분류(`category`) 한 가지만 보고 있었다.

```javascript
// 분류가 바뀔 때만 재조회 — 검색 조건이 바뀌어도 그대로다
watch(() => props.category, fetchDetail, { immediate: true });
```

### 3. 스프레드 순서 때문에 기간 조건이 덮였다

세 번째 증상의 원인은 훨씬 단순했다. 요청 payload를 만드는 순서가 이랬다.

```javascript
data: {
  startDt: start,
  endDt: end,
  ...gridParams,   // ← 그리드 필터에서 온 값이 뒤에 온다
}
```

그리드 헤더 필터/필터행에서 만들어진 `gridParams`에 같은 키가 들어 있으면 **명시적으로 넣은 기간 값을 덮어쓴다.** 평소에는 겹치는 키가 없어 문제가 없다가, 필터 조합에 따라 조용히 기간이 바뀌었다.

## 해결 방법

### 1. 조회마다 일련번호를 부여해 늦은 응답을 버린다

요청 취소(AbortController)까지 가지 않고도, **가장 최근 조회인지 확인**하는 것만으로 충분하다.

```javascript
let currentSearchId = 0;

const searchList = async searchData => {
  const { startDt, endDt, employmentFilter } = normalize(searchData);
  const searchId = ++currentSearchId;      // 이번 조회의 일련번호

  dataGrid.dataSource = new CustomStore({
    key: 'baseDt',
    load: async loadOptions => {
      const res = await callApi({ /* ... */ });

      // 내가 발사한 뒤 더 최신 조회가 시작됐다면, 이 응답은 버린다
      if (searchId !== currentSearchId) {
        return { data: [], totalCount: 0 };
      }

      if (!isSuccess(res)) {
        return { data: [], totalCount: 0 };
      }
      return { data: buildRows(res), totalCount: res.data.header.totalCount };
    },
  });
};
```

포인트는 `searchId`를 **`load` 바깥(클로저)에서 캡처**한다는 것이다. `load`가 실행되는 시점이 아니라 조회가 시작된 시점의 번호를 기억해야 비교가 성립한다.

`load`가 예외를 던지면 그리드에 에러 상태가 뜨므로, 버릴 때도 던지지 않고 **빈 결과를 반환**한다. 어차피 최신 조회 결과가 곧 화면을 채운다.

### 2. 재조회 시 상세 상태를 초기화한다

새 `CustomStore`를 만들기 **전에** 펼쳐진 행과 보관 중인 props를 정리한다.

```javascript
const searchId = ++currentSearchId;

pendingCategoryId = null;
detailPropsMap.clear();
gridRef.value?.getInstance?.collapseAll(-1);   // -1: 모든 마스터-디테일 행 접기

dataGrid.dataSource = new CustomStore({ /* ... */ });
```

`collapseAll(-1)`은 DataGrid에서 **모든 그룹/마스터-디테일 행을 접는** 호출이다. 열려 있던 상세 컴포넌트가 파괴되므로 옛 조건 화면이 남을 여지가 사라진다.

그리고 검색 조건을 상세 props에도 함께 실어, 상세 그리드가 같은 조건으로 조회하도록 한다.

```javascript
const detailProps = reactive({
  baseDt: row.baseDt,
  employmentFilter,   // 조회 조건을 상세까지 전달
  category: null,
});
```

상세 쪽 watch도 조건을 함께 감시하도록 배열로 바꾼다.

```javascript
// 분류 또는 검색 조건이 바뀌면 재조회
watch(() => [props.category, props.employmentFilter], fetchDetail, { immediate: true });
```

> `watch`에 배열(게터가 배열을 반환)을 쓰면 원소 중 하나라도 바뀔 때 콜백이 실행된다. 다만 매번 새 배열이 만들어지므로 얕은 비교로 동작한다는 점은 알고 쓰는 게 좋다.

### 3. 스프레드 순서를 뒤집는다

명시적으로 지정한 값이 항상 이기도록 순서를 바꾼다.

```javascript
data: {
  ...gridParams,        // 그리드 필터가 먼저
  startDt: start,       // 화면 검색 조건이 나중 = 우선
  endDt: end,
  employmentFilter,
}
```

사소해 보이지만, "화면 검색 조건 > 그리드 필터"라는 우선순위를 코드에 고정해두는 편이 안전하다. 반대 순서는 필터 키가 하나 늘어날 때마다 재발 가능성이 생긴다.

## 결과

- 조건을 빠르게 바꿔도 **마지막 조건의 결과만** 화면에 남음
- 재조회 시 펼쳐져 있던 상세 그리드가 접히고, 다시 열면 새 조건으로 조회됨
- 그리드 필터 조합과 무관하게 기간 조건이 항상 적용됨

## 정리

> `dataSource`를 교체해도 **이미 발사된 요청은 취소되지 않는다.** 비동기 조회를 쓰는 그리드에서는 조회마다 일련번호를 부여하고, `load` 안에서 "내가 아직 최신인가"를 확인해 늦게 도착한 응답을 버려야 한다. 마스터-디테일처럼 화면에 상태가 남는 구조라면 재조회 시 `collapseAll(-1)`로 함께 초기화하고, 조회 조건은 상세 컴포넌트까지 내려 watch 대상에 포함시킨다.
