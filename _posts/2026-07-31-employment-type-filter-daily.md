---
layout: post
title: "계약형태 필터 추가 · 공통코드 조회 예외 · 그리드 헤더 필터 정리"
date: 2026-07-31 19:00:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, Spring Boot, QueryDSL, Jackson]
---

프론트엔드(Vue + DevExtreme)와 백엔드(Spring Boot + QueryDSL)에 걸쳐 월별
퇴사율 리포트에 **계약형태 필터**를 붙인 하루였다. 필터 하나를 추가하는 작업이
경계값 처리·비동기 경쟁 상태·예외 처리 범위까지 건드리게 된 흐름을 주제별로
정리한다. 코드는 실제 업무 코드 대신 같은 문제/해결을 재구성한 예시로 대체했다.

## 1. 계약형태 필터 — 필터 값을 enum으로 고정하기

리포트 조회 조건에 **전체 / 센터 / 외주** 세 가지를 추가했다. 요청 문자열을
그대로 쿼리 조건에 흘려보내지 않고, 먼저 enum으로 좁혔다.

```java
public enum EmploymentFilter {

    ALL,
    CENTER,
    OUTSOURCED;

    /**
     * 요청값을 계약형태 필터로 변환한다. (미지정 시 ALL)
     */
    public static EmploymentFilter from(String value) {
        if (value == null) {
            return ALL;
        }
        return valueOf(value);   // 허용되지 않은 값이면 IllegalArgumentException
    }
}
```

요청 DTO에는 기본값과 검증을 함께 건다. 문자열 필드로 받되 enum 유효성만
애노테이션으로 검사하는 방식이라, 잘못된 값이 서비스 계층까지 내려오지 않는다.

```java
@NotBlank(message = "계약형태는 필수값 입니다.")
@EnumValid(enumClass = EmploymentFilter.class)
private String employmentFilter = EmploymentFilter.ALL.name();
```

요약 조회와 상세 조회 DTO **양쪽 모두**에 같은 필드를 넣은 게 포인트다.
상세(드릴다운)가 요약과 다른 조건으로 조회되면 합계가 안 맞기 때문이다.

## 2. QueryDSL 동적 조건 — `null` 반환으로 조건 생략

계약형태 조건은 `ALL`일 때 아예 걸지 않아야 한다. QueryDSL에서는 `where()`에
`null`을 넘기면 **해당 조건이 무시**되므로, 조건 빌더가 `null`을 반환하도록
설계했다.

```java
public List<Employee> findActiveBetween(LocalDate startDt, LocalDate endDt,
        Collection<Long> deptIds, EmploymentFilter filter, Long outsourcedCodeId) {

    return queryFactory.selectFrom(employee)
        .where(
            employee.deletedFl.eq("N"),
            employee.joinDt.loe(endDt),
            employee.retireDt.isNull().or(employee.retireDt.goe(startDt)),
            employee.deptCd.in(deptIds),
            employmentCondition(filter, outsourcedCodeId)   // null 이면 조건 생략
        )
        .fetch();
}

private BooleanExpression employmentCondition(EmploymentFilter filter, Long outsourcedCodeId) {
    if (filter == null || filter == EmploymentFilter.ALL) {
        return null;
    }
    return switch (filter) {
        case CENTER -> employee.employmentTypeCd.isNotNull()
            .and(employee.employmentTypeCd.ne(outsourcedCodeId));
        case OUTSOURCED -> employee.employmentTypeCd.eq(outsourcedCodeId);
        case ALL -> null;
    };
}
```

`CENTER`(= 외주가 아닌 것) 조건에서 `isNotNull()`을 함께 건 이유는 SQL의
3값 논리 때문이다. `col <> 100`은 `col`이 `NULL`인 행을 **참으로 만들지 않는다.**
계약형태가 미지정인 인원을 "센터"로 볼지 말지는 정책 문제인데, 이번 화면에서는
**명시적으로 제외**하는 것이 맞아 조건을 붙였다. 의도를 코드에 드러내는 쪽이
나중에 읽을 때 헷갈리지 않는다.

## 3. 코드 조회 실패를 "전체 조회"로 흘리지 않기

외주 여부 판정에 쓰는 공통코드 ID를 코드 테이블에서 가져오는데, 이게 **없을 때
`null`을 그대로 넘기면 조건이 무력화**되어 전체 조회가 되어버린다. 필터를 걸었는데
전체가 나오는 건 최악의 실패 방식이라, 즉시 예외로 끊었다.

```java
/**
 * 외주 계약형태 공통코드 ID를 조회한다.
 * CENTER/OUTSOURCED 필터에서 코드가 없으면 전체 조회로 오인되지 않도록 즉시 실패한다.
 */
private Long resolveOutsourcedCodeId(EmploymentFilter filter) {
    if (filter == EmploymentFilter.ALL) {
        return null;
    }
    CommonCode code = codeService.findByValue(EmploymentTypeCode.OUTSOURCED);
    if (code == null || code.getCodeId() == null) {
        throw new IllegalStateException("외주 계약형태 공통코드를 찾을 수 없습니다.");
    }
    return code.getCodeId();
}
```

**조용히 잘못된 결과를 주는 것보다 시끄럽게 실패하는 편이 낫다**는 원칙을 적용한
지점. 필터 조건이 사라진 결과는 화면상 정상처럼 보이기 때문에 더 위험하다.

## 4. 검색 조건 변경 시 이전 응답이 화면을 덮는 문제

프론트에서는 조회할 때마다 새 `CustomStore`를 만들어 꽂는 구조였는데, 이전 요청이
취소되지 않아 **늦게 도착한 옛 응답이 화면을 덮어쓰는** 경쟁 상태가 있었다.
조회마다 일련번호를 부여하고, 응답 처리 직전에 최신 여부를 확인해 버리도록 했다.

```javascript
let currentSearchId = 0;

const searchList = async searchData => {
  const searchId = ++currentSearchId;   // 클로저로 캡처

  dataGrid.dataSource = new CustomStore({
    key: 'baseDt',
    load: async loadOptions => {
      const res = await callApi({ /* ... */ });

      // 더 최신 조회가 시작됐다면 이 응답은 버린다 (throw 하지 않고 빈 결과 반환)
      if (searchId !== currentSearchId) {
        return { data: [], totalCount: 0 };
      }
      return { data: buildRows(res), totalCount: res.data.header.totalCount };
    },
  });
};
```

마스터-디테일로 펼쳐둔 상세 그리드도 재조회 시 옛 조건 데이터를 그대로 들고
있어서, 조회 시작 시점에 상태를 초기화했다.

```javascript
pendingCategoryId = null;
detailPropsMap.clear();
gridRef.value?.getInstance?.collapseAll(-1);   // -1: 모든 마스터-디테일 행 접기
```

상세 그리드 쪽 watch도 분류만 보던 것을 조회 조건까지 함께 보도록 확장했다.

```javascript
watch(() => [props.category, props.employmentFilter], fetchDetail, { immediate: true });
```

> 이 항목은 별도 글로도 정리했다 —
> [검색 조건을 바꿨는데 이전 결과가 뜬다]({% post_url 2026-07-31-devextreme-customstore-stale-response-race %})

## 5. 요청 payload 스프레드 순서

같은 화면에서, 기간 조건이 그리드 필터 파라미터에 덮이는 문제도 있었다.

```javascript
// 변경 전: 그리드 필터가 뒤에 와서 기간을 덮을 수 있음
data: { startDt: start, endDt: end, ...gridParams }

// 변경 후: 화면 검색 조건이 항상 우선
data: { ...gridParams, startDt: start, endDt: end, employmentFilter }
```

"화면 검색 조건 > 그리드 필터"라는 우선순위를 코드 순서로 고정해두면, 필터 키가
늘어나도 같은 문제가 재발하지 않는다.

## 6. 마스터-디테일 상세 그리드에 닫기 버튼 추가

상세 그리드를 접으려면 마스터 행의 펼침 아이콘을 다시 눌러야 해서, 상세 툴바에
닫기 버튼을 넣었다. 부모 그리드 인스턴스는 `provide`/`inject`로 받는다.

```javascript
const getParentGridInstance = inject('getParentGridInstance', () => null);

const closeDetail = () => {
  getParentGridInstance()?.collapseRow(props.baseDt);   // 자기 자신을 접는다
};
```

툴바 커스텀 버튼은 기본적으로 `location: 'after'` 구역의 끝에 붙는데, 페이저
바로 옆에 두고 싶어서 렌더 이후 DOM 순서를 한 번 조정했다.

```javascript
const moveCloseButtonNextToPager = gridInstance => {
  const root = gridInstance?.element?.();
  const pagerItem = root?.querySelector?.('.grid-pager-item');
  const closeItem = root?.querySelector?.('.preset-close-plain')?.closest?.('.dx-toolbar-item');

  if (pagerItem && closeItem && pagerItem.nextElementSibling !== closeItem) {
    pagerItem.after(closeItem);
  }
};
```

`contentReady` 시점에 `nextTick()` 이후 호출하고, 이미 원하는 위치면 건드리지
않도록 가드를 뒀다(그렇지 않으면 재렌더마다 DOM을 흔든다). 컴포넌트 라이브러리가
제공하지 않는 배치라 DOM을 직접 만졌지만, 클래스명 의존이 생기는 만큼 **버전 업
시 깨질 수 있는 지점**으로 기록해둔다.

## 7. 그 밖에

- 근무현황 그리드에 `headerFilter.visible`을 켜서 컬럼별 값 필터링 지원
- 공통코드 조회 API가 **빈 결과**를 내려줄 때 `convertValue`가 `null`을 반환해
  `stream()`에서 NPE가 나고, 이게 광범위한 `catch (Exception e)`에 잡혀
  "API 통신 오류"로 둔갑하던 문제 수정 —
  [별도 글로 정리]({% post_url 2026-07-31-catch-exception-npe-masked-as-api-error %})

## 이슈 / 특이사항

- 계약형태 미지정(`NULL`) 인원을 센터로 볼지 여부는 **정책 확인이 필요**한
  부분이라, 현재는 명시적으로 제외하도록 구현하고 조건에 주석을 남겼다.
- 툴바 버튼 위치 조정은 라이브러리 내부 클래스명에 의존하므로, 라이브러리 업그레이드
  시 회귀 테스트 대상.
