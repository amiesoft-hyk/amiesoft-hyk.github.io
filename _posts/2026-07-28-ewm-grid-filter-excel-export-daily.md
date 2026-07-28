---
layout: post
title: "DevExtreme 그리드 헤더 필터 · 트리 리스트 엑셀 내보내기 · 근태 검색 정리"
date: 2026-07-28 20:50:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, ExcelJS, QueryDSL, Spring Boot]
---

프론트엔드(Vue + DevExtreme)와 백엔드(Spring Boot + QueryDSL)에 걸쳐
그리드 필터·엑셀 내보내기·검색 조건을 다듬은 하루였다. 여러 화면에 흩어져
있던 작업이라 개별 회고 대신, 나중에 같은 패턴을 다시 쓸 수 있게 **주제별
레퍼런스** 형태로 정리한다. 코드는 실제 업무 화면 대신 같은 문제/해결을
보여주도록 각색한 예시다.

## 오늘 한 일

- **공통 트리 리스트 컴포넌트에 엑셀 내보내기 공통화** — `dxDropDownButton`
  기반 툴바 버튼(전체/선택 데이터 다운로드)과 `exporting` 이벤트를 추가하고,
  밴드(멀티) 헤더를 그대로 엑셀로 출력하도록 처리. 재직기간별 통계 화면을 이
  공통 이벤트에 연동. (완료)
- **여러 그리드의 헤더 필터 활성화** — 평가 실시 현황, 부서별/문항별 보고서,
  근무시간 편집 그리드에서 막혀 있던 `allowHeaderFiltering` / `headerFilter`를
  일괄 활성화. (완료)
- **개인 평가 보고서 레이아웃 개선** — 요약/추이/차트 패널을 뷰포트 기준 flex
  비율 레이아웃으로 재구성하고 그리드·차트 높이를 100% 기준으로 조정. (완료)
- **휴가 이력 근태 필터링 개선(프론트+백엔드)** — 근태 유형 헤더 필터를
  코드값/코드명 어느 쪽으로 걸어도 매칭되도록 정규화하고, 근무일수 컬럼을
  number 타입 필터로 열고, 수정일시/수정자 텍스트 필터를 추가. 백엔드는
  근무일수 목록 검색 조건과 `editDt` 문자열 검색을 함께 손봄. (완료)
- **개인 설문 상세 방어 처리** — 잘못된 날짜/응답 구조에 대한 방어 코드와
  결과 화면 진입 라우팅 정리. (완료)

## 트리 리스트 밴드 헤더를 엑셀로 내보내기

기존 엑셀 내보내기는 그리드의 보이는 컬럼을 1차원으로만 뽑아, 부모-자식으로
묶인 **밴드 헤더**(예: `근무시간` 아래 `이전 / 수정 후`)가 평평하게 풀려버렸다.
컬럼 정의 트리를 직접 순회하면서, 자식이 있는 밴드는 가로 병합, 단일 컬럼은
세로 병합으로 헤더를 그려주면 화면과 동일한 2단 헤더가 나온다.

```js
// columns: [{ caption, columns? }] 형태의 컬럼 정의 트리
const BAND_ROW = 2; // 밴드(부모) 행
const LEAF_ROW = 3; // 실제 데이터(자식) 행
let colIdx = 1;

columns.forEach(col => {
  const children = col.columns;
  if (children?.length) {
    // 부모 밴드: 자식 개수만큼 가로 병합
    worksheet.mergeCells(BAND_ROW, colIdx, BAND_ROW, colIdx + children.length - 1);
    worksheet.getCell(BAND_ROW, colIdx).value = captionOf(col);
    children.forEach((child, i) => {
      worksheet.getCell(LEAF_ROW, colIdx + i).value = captionOf(child);
    });
    colIdx += children.length;
  } else {
    // 단일 컬럼: 밴드행~리프행 세로 병합
    worksheet.mergeCells(BAND_ROW, colIdx, LEAF_ROW, colIdx);
    worksheet.getCell(BAND_ROW, colIdx).value = captionOf(col);
    colIdx += 1;
  }
});
```

`captionOf`는 다국어 캡션(`i18n`)이 있으면 번역값을, 없으면 기본 캡션을
반환하도록 감싼 헬퍼다. leaf 컬럼만 따로 평탄화해 두면 데이터 행을 채울 때
표시 순서를 그대로 유지할 수 있다.

## 헤더 필터: 코드값과 코드명, 어느 쪽으로 걸어도 매칭

근태 유형처럼 화면에는 **코드명**이 보이지만 데이터/서버 파라미터로는
**코드값**을 쓰는 컬럼은, 헤더 필터 값이 코드명으로 넘어와 서버 조건과
어긋나는 경우가 있다. 코드값이든 코드명이든 하나의 코드로 정규화한 뒤,
서버로 보낼 때는 둘 다 후보에 넣어 안전하게 매칭했다.

```js
const findTypeCode = value =>
  typeList.find(
    c => String(c.codeValue) === String(value) || c.codeNm === String(value),
  );

// 화면 데이터: 항상 코드값으로 정규화
const normalizeType = value => findTypeCode(value)?.codeValue ?? value;

// 서버 필터: 코드값/코드명 둘 다 후보로
const typeFilterValues = value => {
  const code = findTypeCode(value);
  return code ? [...new Set([code.codeValue, code.codeNm].filter(Boolean))] : [value];
};
```

## QueryDSL: DATE_FORMAT 템플릿 대신 stringValue()

수정일시(`editDt`)를 문자열 `contains`로 검색할 때, 기존에는
`DATE_FORMAT(...)` SQL 템플릿으로 포맷 문자열을 만들어 비교했다. DB 함수에
의존하다 보니 포맷/DB 방언에 민감했는데, QueryDSL의 `stringValue()`로
바꿔 단순화했다.

```java
// 변경 전: DB의 DATE_FORMAT 함수에 의존
StringExpression formattedEditDt = Expressions.stringTemplate(
    "DATE_FORMAT({0}, '%Y-%m-%d %H:%i:%s')", attendance.editDt);

// 변경 후: 컬럼 값을 문자열로 변환
StringExpression formattedEditDt = attendance.editDt.stringValue();

if (search.getEditDt() != null && !search.getEditDt().isBlank()) {
    builder.and(formattedEditDt.contains(search.getEditDt().trim()));
}
```

근무일수 같은 숫자 컬럼 검색은 포함/제외 목록을 받아 재사용 헬퍼로 처리했다.

```java
// 검색 DTO: 포함/제외 목록
private List<BigDecimal> workDaysList = Collections.emptyList();
private List<BigDecimal> workDaysExcludeList = Collections.emptyList();

// 검색 조건 조립
applyNumberListCondition(
    builder, attendance.workDays,
    search.getWorkDaysList(), search.getWorkDaysExcludeList());
```

## 이슈 / 특이사항

- **헤더 필터의 값 표현이 화면과 서버에서 다를 수 있다.** 코드명 표시 +
  코드값 전송 구조에서는 필터 값을 한 방향으로 정규화하지 않으면 필터가
  "먹지 않는" 것처럼 보인다. 정규화 지점을 데이터 로드와 필터 조립 양쪽에
  두는 게 안전했다.
- **밴드 헤더 엑셀은 컬럼 정의 트리를 신뢰 소스로 삼는 게 깔끔하다.**
  런타임 그리드 인스턴스에서 보이는 컬럼을 뽑는 방식은 밴드 구조가 쉽게
  풀리므로, 원본 컬럼 정의를 순회하는 편이 화면과의 일치도가 높다.
- DB 함수 기반 문자열 검색은 편하지만 방언·포맷 변화에 취약하다. 단순
  `contains` 정도면 컬럼의 `stringValue()`로 충분한 경우가 많다.
