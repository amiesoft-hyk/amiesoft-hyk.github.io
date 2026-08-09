---
layout: post
title: "조회를 눌러도 그리드가 그대로다 — 조용히 실패하는 옵셔널 체이닝과 서버 기본 정렬"
date: 2026-08-07 18:40:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, CustomStore, ref, 정렬]
---

리포트 화면에서 두 가지 증상을 한 번에 받았다. 하나는 "상단 검색 조건을 바꾸고 조회를 눌러도 아래 그리드가 그대로"이고, 다른 하나는 "템플릿에 등록한 항목 순서대로 리포트가 정렬되지 않는다"였다. 원인은 완전히 달랐고, 하나는 프론트, 하나는 백엔드였다.

## 문제 상황

서버 페이징(`remoteOperations`) + `CustomStore` 조합으로 만든 리포트 그리드다.

1. 상단에서 대상 계획을 바꾸고 조회 버튼을 눌러도 그리드 행이 이전 조회 결과로 남아 있다. **첫 조회는 정상**이고 두 번째 조회부터 반응이 없다. 콘솔에 에러도 없다.
2. 상세 필터(대상자 태그)를 추가·삭제한 직후 조회하면 방금 바꾼 조건이 아니라 **이전 조건**으로 결과가 나온다.
3. 템플릿에서 1번, 2번, 3번 순으로 만든 항목이 리포트에서는 뒤죽박죽으로 나온다.

## 원인 1 — `?.()` 가 실패를 삼킨다

조회 함수는 대략 이런 모양이었다.

```js
let storeInitialized = false;

const search = async () => {
  Object.assign(searchParams, { planId, userIds, deptIds });

  if (!storeInitialized) {
    // 첫 조회: dataSource 를 교체한다 → 정상 동작
    gridConfig.dataSource = createStore();
    storeInitialized = true;
    return;
  }

  // 두 번째 조회부터는 이 두 줄이 전부다
  reportGridRef.value?.getInstance?.pageIndex?.(0);
  await reportGridRef.value?.refreshData?.();
};
```

첫 조회는 `gridConfig.dataSource`를 갈아끼우므로 컴포넌트의 watch가 받아서 재조회한다. 문제는 두 번째부터다. 갱신이 전적으로 `reportGridRef`에 달려 있는데, 이 화면은 ref를 이렇게 걸고 있었다.

```html
<!-- 문자열을 동적으로 바인딩 -->
<common-data-grid :ref="gridConfig.refName" :data-grid="gridConfig" />
```

`ref="reportGridRef"`처럼 **정적 문자열**로 쓰면 Vue 컴파일러가 그 이름이 setup 바인딩이라는 걸 알고 직접 연결해 준다. 반면 `:ref="gridConfig.refName"`은 컴파일 시점에 값을 알 수 없어서, 런타임이 문자열 ref를 setup 상태에 반영할 수 있는지에 의존하게 된다. 빌드 모드에 따라 이 연결이 성립하지 않으면 `reportGridRef.value`는 계속 `null`이다.

여기서 진짜 문제는 ref 자체가 아니라 **옵셔널 체이닝**이다.

```js
reportGridRef.value?.getInstance?.pageIndex?.(0);
await reportGridRef.value?.refreshData?.();
```

참조가 비어 있으면 이 두 줄은 예외도 없이, 경고도 없이, 네트워크 요청도 없이 그냥 `undefined`를 반환하고 지나간다. 화면에는 "첫 조회 결과가 영구히 고정된 그리드"만 남는다. `?.`는 방어 코드처럼 보이지만, 이렇게 **성공해야만 의미가 있는 호출**에 붙이면 실패를 조용히 삼키는 장치가 된다.

## 원인 2 — watch로 갱신되는 파생 상태를 조회 파라미터로 썼다

상세 필터는 태그박스에 담기고, 조회 파라미터는 별도 상태에서 꺼내 쓰고 있었다.

```js
// 태그 목록이 바뀌면 watch 가 파생 상태를 다시 계산한다
watch(tagItems, items => {
  selectedUserIds.value = items.filter(t => t.type === 'user').map(t => t.id);
  selectedDeptIds.value = items.filter(t => t.type === 'dept').map(t => t.id);
}, { deep: true });

const onTagValueChanged = e => {
  removeUnselected(e).then(() => {
    search();   // ← watch flush 보다 먼저 실행될 수 있다
  });
};

const search = async () => {
  Object.assign(searchParams, {
    userIds: [...selectedUserIds.value],   // 아직 이전 값일 수 있음
    deptIds: [...selectedDeptIds.value],
  });
  // ...
};
```

Vue의 watch 콜백은 기본적으로 컴포넌트 업데이트 전(pre)에 비동기로 flush된다. 태그를 지운 직후 `.then()` 안에서 곧바로 조회하면, 파생 상태가 갱신되기 전 값으로 요청이 나갈 수 있다. 화면에 보이는 태그와 서버로 가는 조건이 어긋나는 것이다.

## 원인 3 — 서버 기본 정렬이 PK 순이었다

정렬 쪽은 백엔드 문제였다. 리포트 목록을 만드는 서비스가 이렇게 되어 있었다.

```java
// 응답 목록에 쓸 항목 ID를 "응답 데이터"에서 뽑는다 → 순서 정보가 없다
List<Long> itemIds = answerRepository.findItemIdsByPlanId(planId);

return itemIds.stream()
    .sorted()                 // ← PK(itemId) 오름차순
    .map(itemId -> { ... })
    .toList();
```

정렬을 지정하지 않았을 때의 기본 comparator도 마찬가지로 PK였다.

```java
if (comparator == null) {
    comparator = Comparator.comparing(SearchResult::getItemId);
}
```

정작 화면이 기대하는 "템플릿에 등록한 순서"는 템플릿-항목 매핑 테이블의 순서 컬럼(`itemOrd`)에 이미 저장되어 있었는데, 이 서비스가 그 값을 아예 조회하지 않고 있었다. 항목을 만든 순서와 템플릿에 배치한 순서가 다르면 그대로 어긋난다. 같은 순서 컬럼을 쓰는 다른 상세 조회 화면은 이미 `itemOrd`로 정렬하고 있어서, 화면 간 기준도 통일되지 않은 상태였다.

## 해결 방법

### 1. 그리드 인스턴스를 이벤트로 직접 보관한다

`initialized` 이벤트는 ref 바인딩 방식과 무관하게 항상 컴포넌트 인스턴스를 넘겨준다. 이걸 붙잡아두면 참조 해석에 기대지 않아도 된다.

```js
const gridInstance = ref(null);

const onGridInitialized = e => {
  gridInstance.value = e?.component ?? null;
  // 기존 초기화 로직...
};
```

그리고 조회는 **실패할 수 없는 경로**로 폴백을 둔다. 첫 조회에서 이미 검증된 `dataSource` 교체 경로를 그대로 쓰는 것이다.

```js
const search = async () => {
  resetCaches();

  const gridComponent = reportGridRef.value;
  if (storeInitialized && typeof gridComponent?.refreshData === 'function') {
    gridInstance.value?.pageIndex?.(0);
    await gridComponent.refreshData();
    return;
  }

  // 참조를 얻지 못했으면 스토어를 새로 만들어 재조회한다.
  // dataSource 교체는 공통 그리드가 감지해 헤더필터 초기화와 재조회까지 처리한다.
  gridConfig.dataSource = createStore();
  storeInitialized = true;
};
```

`typeof ... === 'function'`으로 **쓸 수 있는지 명시적으로 확인**하고, 못 쓰면 다른 길로 간다. `?.()`로 조용히 넘어가는 것과의 차이가 이 버그의 핵심이었다.

### 2. 조회 파라미터를 화면 상태에서 직접 계산한다

파생 상태를 거치지 않고, 화면에 실제로 보이는 태그 목록에서 그때그때 계산한다.

```js
Object.assign(searchParams, {
  planId,
  userIds: tagItems.value.filter(t => t.type === 'user').map(t => t.id),
  deptIds: tagItems.value.filter(t => t.type === 'dept').map(t => t.id),
});
```

watch 타이밍과 무관해지고, "화면에 보이는 조건 = 서버로 가는 조건"이 구조적으로 보장된다. 파생 상태는 모달의 초기 선택값 같은 용도로만 남긴다.

### 3. 기본 정렬을 순서 컬럼으로 바꾼다

먼저 순서 값을 담은 매핑을 조회한다. 삭제 플래그까지 걸러서 가져오는 게 안전하다.

```java
@Query("SELECT reg FROM Plan p " +
    " JOIN p.template t " +
    " JOIN t.templateItems reg " +
    " WHERE p.id = :planId " +
    " AND p.delFl = 'N' AND t.delFl = 'N' " +
    " AND reg.delFl = 'N' AND reg.item.delFl = 'N'")
List<TemplateItem> findActiveTemplateItemsByPlanId(@Param("planId") Long planId);
```

```java
private Map<Long, Long> getItemOrdMap(final Long planId) {
    return templateItemRepository.findActiveTemplateItemsByPlanId(planId).stream()
        .filter(reg -> reg.getItem() != null && reg.getItemOrd() != null)
        .collect(Collectors.toMap(reg -> reg.getItem().getId(), TemplateItem::getItemOrd,
            (left, right) -> left));
}
```

목록을 만들 때 이 순서로 정렬하고, 응답 DTO에도 순서 값을 담는다. 템플릿에서 빠진 항목은 순서가 없으니 `nullsLast`로 뒤로 보낸다.

```java
Map<Long, Long> itemOrdMap = getItemOrdMap(planId);
Comparator<Long> itemOrder = Comparator
    .comparing((Long itemId) -> itemOrdMap.get(itemId),
        Comparator.nullsLast(Comparator.<Long>naturalOrder()))
    .thenComparing(Comparator.<Long>naturalOrder());

return itemIds.stream()
    .sorted(itemOrder)
    .map(itemId -> SearchResult.builder()
        .itemId(itemId)
        .itemOrd(itemOrdMap.get(itemId))
        // ...
        .build())
    .toList();
```

정렬을 지정하지 않았을 때의 기본 comparator도 같이 바꾼다. 사용자가 컬럼 헤더로 정렬을 지정하면 그 기준이 우선하는 기존 동작은 유지하고, **tie-breaker로 PK를 남겨** 순서 값이 같을 때도 결과가 안정적으로 나오게 한다.

```java
if (comparator == null) {
    // 정렬 미지정 시 템플릿에 등록된 순서를 기본 정렬로 사용
    comparator = Comparator.comparing(SearchResult::getItemOrd,
        Comparator.nullsLast(Comparator.naturalOrder()));
}
sorted.sort(comparator.thenComparing(SearchResult::getItemId));
```

순서 컬럼은 정렬 기준으로만 쓰고 그리드 컬럼으로 노출하지 않았다. 화면에 순번을 보여줄 필요가 없다면 응답에만 담아두는 편이 화면 변경 폭을 줄인다.

## 결과

- 조회 버튼을 누를 때마다 재조회가 실제로 일어난다. 참조가 비어 있어도 `dataSource` 교체 경로로 갱신된다.
- 태그를 바꾼 직후 조회해도 화면의 태그와 서버 조건이 일치한다.
- 정렬을 지정하지 않으면 템플릿 등록 순서로 나오고, 헤더로 정렬하면 그 기준이 우선한다.

## 남은 점검 포인트

정렬 수정은 **DB의 순서 컬럼이 실제로 채워져 있다는 전제** 위에 있다. 값이 전부 `NULL`이거나 모두 같은 값이면 tie-breaker인 PK 순으로 되돌아가 증상이 그대로 보인다. 정렬이 여전히 어긋나면 서비스 코드보다 먼저 데이터를 확인하는 게 빠르다.

```sql
SELECT reg.item_id, reg.item_ord, reg.del_fl
FROM template_item reg
JOIN plan p ON p.template_id = reg.template_id
WHERE p.id = ? AND reg.del_fl = 'N'
ORDER BY reg.item_ord
```

값이 비어 있다면 문제는 리포트가 아니라 템플릿 저장 쪽이다.

## 정리

- `obj?.method?.()`는 **호출이 실패해도 괜찮을 때만** 쓴다. 반드시 성공해야 하는 호출에 붙이면 버그가 로그도 없이 숨는다. 쓸 수 있는지 확인하고, 못 쓰면 다른 경로로 가거나 최소한 로그를 남긴다.
- 컴포넌트 인스턴스가 필요하면 `initialized` 같은 **이벤트로 직접 받는 편이 ref 문자열 바인딩보다 안전하다.** 특히 ref 이름을 동적으로 바인딩하는 구조라면 더 그렇다.
- 조회 파라미터는 watch로 갱신되는 파생 상태 대신 **화면 상태에서 직접 계산**한다. "보이는 것 = 보내는 것"을 구조로 보장하는 편이 타이밍을 맞추는 것보다 낫다.
- 목록의 기본 정렬은 PK가 아니라 **업무가 기대하는 순서 컬럼**이어야 한다. 같은 순서를 쓰는 화면이 이미 있다면 그 기준을 먼저 찾아보는 게 좋다.
