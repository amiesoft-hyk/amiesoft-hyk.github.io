---
layout: post
title: "DevExtreme 엑셀 헤더 필터 · 검색 필터 유실 · 재계산 캐시 정리"
date: 2026-07-29 21:30:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, ExcelJS, Spring Boot, JPA]
---

프론트엔드(Vue + DevExtreme)와 백엔드(Spring Boot + JPA)에 걸쳐 그리드 엑셀
내보내기와 조회 동작을 다듬은 하루였다. 여러 화면에 흩어진 작업이라, 나중에
같은 패턴을 다시 쓸 수 있도록 **주제별 레퍼런스** 형태로 정리한다. 코드는 실제
업무 화면 대신 같은 문제/해결을 재구성한 예시로 대체했다.

## 1. 엑셀 헤더에 필터 드롭다운 넣기 — 두 가지 경로

여러 목록/보고서 화면의 엑셀 다운로드 헤더에 "필터 드롭다운"(엑셀 autoFilter)을
공통 적용했다. 화면마다 export 구현 방식이 달라 경로가 두 가지였다.

### (a) 공통 그리드의 내장 export를 쓰는 경우

내부적으로 DevExtreme `exportDataGrid`를 호출하는 공통 컴포넌트를 쓰는 화면은,
export 설정 플래그 하나만 켜면 된다.

```js
// 그리드 설정
export: {
  enabled: true,
  title: '보고서',
  autoFilterEnabled: true, // 헤더 행에 필터 드롭다운 추가
},
```

컴포넌트 내부에서는 이 값을 `exportDataGrid`에 그대로 넘긴다.

```js
exportDataGrid({
  component: grid,
  worksheet,
  autoFilterEnabled: dataGridConfig.export.autoFilterEnabled,
});
```

### (b) ExcelJS로 워크시트를 직접 작성하는 경우

서버 페이징 등의 이유로 `exportDataGrid` 대신 ExcelJS로 직접 셀을 채우는 화면은,
플래그가 통하지 않는다. `worksheet.autoFilter`를 직접 지정해야 한다.

```js
// 1행: 병합 제목, 2행: 헤더, 3행~: 데이터 구조라면
worksheet.autoFilter = {
  from: { row: 2, column: 1 },
  to:   { row: 2, column: columns.length },
};
```

**주의 — autoFilter는 "실제 컬럼 헤더 행"에 걸어야 한다.** 상단에 병합 제목 행이
있으면 헤더는 2행, 데이터는 3행부터다. 밴드(멀티) 헤더라면 leaf 헤더 행(예: 3행)에
걸어야 각 컬럼 단위로 필터가 잡힌다. 제목 행에 걸면 엑셀이 "복구" 경고를 띄우기도
한다.

### 곁들여: 자동 열 너비에 최소 너비 보장하기

내장 export는 내용 길이 기반으로 열 너비를 자동 계산하는데, 숫자/코드 열이 너무
좁게 나왔다. 다른 화면에 영향 없이 특정 화면만 넓히려고, 자동 계산 함수에 opt-in
파라미터를 하나 추가했다.

```js
const adjustColumnWidths = (worksheet, headerRowIndex, minColumnWidth) => {
  // ...기존 자동 계산...

  // 값이 있을 때만 각 열의 최소 너비를 보장 (미설정 시 기존 동작 그대로)
  if (minColumnWidth) {
    const colCount = worksheet.columnCount;
    for (let i = 1; i <= colCount; i += 1) {
      const col = worksheet.getColumn(i);
      if (!col.width || col.width < minColumnWidth) col.width = minColumnWidth;
    }
  }
};
```

공통 코드를 건드릴 때는 **기본값 미설정 시 기존 동작 유지**를 원칙으로 두면
회귀 위험이 없다.

## 2. "검색 후 전체 다운로드"인데 검색 조건이 빠지는 문제

서버 페이징 그리드에서 필터로 검색한 뒤 전체 다운로드를 하면, 검색 결과가 아니라
**전체 데이터**가 받아졌다.

### 원인

전체 다운로드 핸들러가 필터를 얻으려고 `dataSource.loadOptions()`를 썼는데, 이 값은
그리드가 서버로 보내는 **결합 필터(filterRow + 헤더필터)를 담지 않는다.**

```js
// 문제 코드 — 결합 필터가 안 담김
const loadOptions = dataSource.loadOptions();
```

### 해결

그리드 인스턴스의 `getCombinedFilter(true)`로 실제 결합 필터를, `dataSource.sort()`로
정렬을 직접 얻어 store.load에 넘긴다.

```js
const loadOptions = {
  filter: gridInstance.getCombinedFilter(true), // filterRow + 헤더필터 결합
  sort: dataSource.sort(),
};

// 이후 서버 페이징 루프에서 loadOptions로 페이지 단위 수집
```

화면 조회와 export가 **완전히 동일한 필터 입력**을 쓰게 되어, 검색 결과만 정확히
내려받는다.

## 3. keep-alive 화면 재진입 시 검증 메시지가 남는 문제

탭을 캐시(keep-alive)하는 등록 화면에서, 저장을 눌러 검증을 한 번 실행한 뒤 목록으로
나갔다가 다시 "추가"로 들어오면 **빈 폼인데도 빨간 필수 메시지가 그대로** 떠 있었다.

### 원인

재진입 시 폼 데이터(값)만 초기화하고, DevExtreme validator의 invalid 상태는
초기화하지 않았다. `validation-message-mode="always"`라 invalid가 남아 있으면
메시지가 계속 보인다.

### 해결

값을 세팅한 **직후** validation group을 reset한다. (reset은 에디터 값도 비우므로
반드시 값 세팅 이후에 호출)

```js
const resetValidation = async () => {
  validationGroup.value?.reset();
  await nextTick();
  validationGroup.value?.reset();
};

// 신규 진입(수정 대상 없음) 분기에서
Object.assign(formData, cloneObj(initFormData));
await resetValidation();
```

수정 모드는 값이 채워지면 validator가 스스로 valid로 바뀌므로 건드리지 않는다.

## 4. 로딩 인디케이터가 두 번 뜨는 문제

화면 진입 시 자동 조회를 붙였더니, **전체 화면 로딩**과 **그리드 내부 loadPanel**이
연달아 떠서 두 번 깜빡였다.

- 데이터 fetch는 1회지만,
- API 호출의 `loading: true`가 전체 화면 로딩을 띄우고,
- fetch 완료 후 `dataSource`를 갱신할 때 그리드 loadPanel이 또 뜬다.

느린 조회 대기 피드백은 전체 화면 로딩이 담당하므로, 뒤따라 뜨는 그리드 loadPanel을
껐다.

```js
// 그리드 설정
loadPanel: { enabled: false },
```

## 5. 검증 메시지가 아예 안 보이는 문제

다른 등록 폼은 필수 검증 아이콘만 뜨고 메시지 텍스트가 안 보였다. 에디터에
`validation-message-mode`가 없어 기본값 `auto`(마우스 오버 시에만 표시)로 동작한
탓이다. 항상 보이게 바꿨다.

```html
<dx-date-box v-model="form.startDt" validation-message-mode="always">
  <dx-validator>
    <dx-required-rule message="필수입니다." />
  </dx-validator>
</dx-date-box>
```

좁은 날짜/셀렉트 박스에서는 항상 표시 메시지가 2줄로 접히며 아래 행과 겹칠 수 있어,
한 줄 유지 CSS를 함께 넣었다.

```css
:deep(.dx-invalid-message.dx-invalid-message-always > .dx-overlay-content) {
  max-width: none !important;
  white-space: nowrap;
}
```

## 6. 조회할 때마다 전체 재계산이 도는 백엔드 — TTL 캐시로 완화

한 목록 조회 API가 매 요청마다 "대상 전체를 재계산해서 테이블에 upsert"한 뒤 결과를
읽는 구조였다. 재계산이 상관 서브쿼리 다발이라 무거웠고, 진입 자동 조회까지 붙으니
체감이 더 나빠졌다.

### 해결 — 최근에 계산했으면 재계산 skip

계산 결과 테이블의 최신 갱신 시각(`edit_dt`)을 기준으로, **TTL 이내에 같은 기간을
이미 계산했으면 무거운 재계산을 건너뛰고** 저장된 결과만 조회한다. 서버 공유 메모리
상태 없이 DB로 판단하므로 다중 인스턴스에서도 안전하다.

```java
private static final Duration RECOMPUTE_TTL = Duration.ofMinutes(5);

public List<Result> findAll(Search search) {
    // ...기간 파라미터 확인...

    if (!isRecentlyRefreshed(startDt, endDt)) {
        refreshHeavyRecalc(startDt, endDt); // 무거운 전체 재계산
    }
    return selectResultList(search, startDt, endDt);
}

private boolean isRecentlyRefreshed(LocalDate startDt, LocalDate endDt) {
    LocalDateTime cutoff = LocalDateTime.now().minus(RECOMPUTE_TTL);
    Object cnt = em.createNativeQuery("""
        SELECT COUNT(*) FROM target_required_round r
        WHERE r.calc_start_dt = :startDt AND r.calc_end_dt = :endDt
          AND r.del_fl = 'N' AND r.edit_dt >= :cutoff
        """)
        .setParameter("startDt", startDt)
        .setParameter("endDt", endDt)
        .setParameter("cutoff", cutoff)
        .getSingleResult();
    return ((Number) cnt).longValue() > 0;
}
```

재계산은 idempotent(`INSERT ... ON DUPLICATE KEY UPDATE`)라 skip해도 정합성은
유지된다. 대신 **데이터 신선도 vs 속도 트레이드오프**가 생기므로 TTL은 업무 기준에
맞춰 조정한다.

### 곁들여: 상관 서브쿼리엔 인덱스

전체 대상을 도는 상관 서브쿼리는 각 서브쿼리의 조인/필터 컬럼(`대상ID`, `del_fl`,
기간 컬럼 등)에 복합 인덱스가 없으면 대상 수만큼 풀스캔이 반복된다. 조회 조건에
맞춰 `(대상ID, del_fl, 기간)` 순의 복합 인덱스를 잡아 두면 재계산·최종 조회가 함께
빨라진다. 인덱스 적용 전엔 `SHOW INDEX`로 선행 인덱스 중복을 확인하고, 스테이징에서
`EXPLAIN`으로 실제 사용 여부를 검증하는 게 안전하다.

## 정리

- 엑셀 필터 드롭다운은 **내장 export(플래그)** / **ExcelJS(worksheet.autoFilter)** 두 경로.
  후자는 제목/헤더 행 위치를 정확히 잡아야 한다.
- export의 필터는 `loadOptions()`가 아니라 `getCombinedFilter(true)`로 얻는다.
- keep-alive 재진입 검증 잔존은 값 세팅 후 `validationGroup.reset()`.
- 로딩 인디케이터 중복은 전역 loading과 그리드 loadPanel 중 하나만 남긴다.
- 매 조회 무거운 재계산은 최신 갱신 시각 기반 **TTL skip + 인덱스**로 완화한다.
