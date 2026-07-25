---
layout: post
title: "DevExtreme 목록이 20건에서 끊기고 페이저가 사라진 문제 해결"
date: 2026-07-25 21:00:00 +0900
categories: [오류해결]
tags: [Vue, DevExtreme, Spring Boot, QueryDSL, 페이징]
---

원격 페이징을 쓰는 관리자 목록 화면에서 두 가지 증상이 함께 나타났다. 하나는 "목록이 항상 20건에서 끊긴다", 다른 하나는 "선택 옵션을 바꾸면 페이저(페이지 이동 UI)가 사라진다"였다. 둘 다 데이터가 아니라 **DevExtreme 그리드의 동작 방식**에서 비롯된 문제였다.

이 글에서는 특정 업무 화면 대신, 같은 유형의 원격 페이징 그리드에 재사용할 수 있도록 각색한 예시로 원인과 해결을 정리한다.

## 문제 상황

- 서버는 요청받은 페이지 크기(예: 100건)만큼 내려줄 수 있는데도, 화면 목록은 늘 **20건**에서 잘렸다.
- 화면에서 정렬 화살표를 눌러도 순서가 바뀌지 않았다. 서버 정렬이 반영되지 않는 것처럼 보였다.
- 그리드의 선택 모드(체크박스 등) 옵션을 바꾸는 순간 하단 **페이저가 통째로 사라졌다**.

## 원인

### 1. remoteOperations가 로컬 상태면 원격 페이징을 스스로 끈다

DevExtreme의 `CustomStore`는 서버로 `take`/`skip`(페이지 크기·오프셋)을 보내 원격 페이징을 한다. 그런데 **정렬·필터가 로컬(remoteOperations에서 꺼진 상태)이면**, DevExtreme은 "어차피 전체를 받아 클라이언트에서 정렬/필터해야 한다"고 판단하고 `take`/`skip`을 아예 보내지 않는다.

```js
// ❌ 페이징만 원격, 정렬·필터는 로컬
remoteOperations: {
  filtering: false,
  sorting: false,
  paging: true,
},
```

이 조합에서는 서버로 페이지 크기가 전달되지 않고, 서버는 파라미터가 없을 때의 **기본 pagesize(예: 20)**로 잘라서 응답한다. 그래서 목록이 늘 20건에서 끊겼고, 화면 정렬도 서버에 도달하지 못했다.

### 2. repaint()가 헤더 툴바를 재생성하면서 페이저를 날린다

공통 그리드 래퍼는 페이저를 헤더 툴바 영역으로 옮겨 배치한다. 이 상태에서 옵션 변경 후 `repaint()`를 호출하면 **헤더 툴바가 통째로 다시 그려지면서**, 툴바로 옮겨 두었던 페이저까지 함께 사라진다.

```js
// ❌ 선택 옵션만 바꾸면 되는데 전체 repaint까지 호출
grid.option('selection.mode', mode);
grid.updateDimensions();
grid.repaint(); // 헤더 툴바 재생성 → 페이저 유실
```

### 3. 정렬값이 같은 행은 페이지마다 순서가 흔들린다

서버 정렬을 붙여도, 정렬 키 값이 같은 행들(예: 같은 시작일)의 상대 순서는 DB가 보장하지 않는다. 그래서 페이지를 넘길 때 경계에서 같은 행이 다시 보이거나 빠지는 현상이 생길 수 있다.

## 해결 방법

### 1. 페이징이 필요한 원격 조합을 함께 켠다

정렬·필터도 서버가 처리하도록 `remoteOperations`를 함께 켠다. 그래야 DevExtreme이 `take`/`skip`을 정상적으로 실어 보낸다.

```js
// ✅ 셋을 함께 켜야 원격 페이징이 유지된다
remoteOperations: {
  filtering: true,
  sorting: true,
  grouping: false,
  paging: true,
},
```

단, 정렬·필터를 원격으로 켜면 **서버가 처리할 수 없는 컬럼**(계산·조합 표시 컬럼 등)은 조건을 열어둬도 조용히 무시된다. 이런 컬럼은 아예 막아 사용자가 헷갈리지 않게 한다.

```js
// 표시값이 "참여/대상 ( n% )" 같은 조합 문자열이라 서버 조건으로 다룰 수 없는 컬럼
{
  dataField: 'ratio',
  allowFiltering: false,
  allowHeaderFiltering: false,
  allowSorting: false,
  calculateCellValue: row => `${row.done}/${row.total} ( ${row.rate}% )`,
},
```

### 2. repaint() 대신 필요한 option()만 갱신한다

선택 UI는 `option()` 변경만으로 갱신된다. 전체 `repaint()`를 부르지 않으면 헤더 툴바가 재생성되지 않아 페이저도 유지된다.

```js
// ✅ 필요한 옵션만 바꾸고 repaint()는 호출하지 않는다
grid.option('selection.mode', mode);
grid.option('selection.showCheckBoxesMode', checkBoxesMode);
grid.option('selection.allowSelectAll', allowSelectAll);
grid.updateDimensions();
// grid.repaint();  ← 제거: 이 호출이 페이저를 날린다
```

### 3. 서버 정렬 끝에 고유키(id)를 tiebreaker로 붙인다

화면 정렬을 서버 정렬로 변환하되, 마지막에 항상 고유키를 덧붙여 순서를 고정한다. QueryDSL 예시(각색):

```java
// 화면에서 넘어온 정렬을 지원 컬럼만 서버 정렬로 변환
private OrderSpecifier<?>[] toOrderSpecifiers(Pageable pageable) {
    OrderSpecifier<?>[] defaultOrder = {
        record.startDt.desc(), record.regDt.desc(), record.id.desc()
    };
    if (pageable == null || pageable.getSort().isUnsorted()) {
        return defaultOrder;
    }

    List<OrderSpecifier<?>> orders = pageable.getSort().stream()
        .map(this::toOrderSpecifier)
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
    if (orders.isEmpty()) {
        return defaultOrder;
    }

    // 정렬값이 같은 행의 순서가 페이지마다 흔들리지 않도록 고유키를 마지막에 덧붙인다
    orders.add(record.id.desc());
    return orders.toArray(OrderSpecifier[]::new);
}

private OrderSpecifier<?> toOrderSpecifier(Sort.Order sort) {
    Order dir = sort.isAscending() ? Order.ASC : Order.DESC;
    return switch (sort.getProperty()) {
        case "name"     -> new OrderSpecifier<>(dir, record.name);
        case "statusCd" -> new OrderSpecifier<>(dir, record.statusCd);
        case "startDt"  -> new OrderSpecifier<>(dir, record.startDt);
        // 조인 테이블 컬럼도 별칭과 동일한 경로로 정렬
        case "registrant" -> new OrderSpecifier<>(dir, userMaster.name);
        // 비율 등 서브쿼리 계산값은 정렬 대상에서 제외 (화면에서도 정렬을 막아둔다)
        default -> null;
    };
}
```

### 참고: 필터값의 연산자 규약을 맞춘다

원격 필터를 켜면 프론트가 연산자에 따라 값에 접두/래핑 문자열을 붙여 서버로 보낸다(예: contains → `%값%`, notcontains → `<>%값%`). 서버가 이 규약을 파싱하는 공통 빌더를 거치지 않고 원시 `like(value)`로 직접 처리하면, 기본 연산자(contains)는 동작해도 다른 연산자에서는 문자열이 그대로 패턴이 되어 결과가 어긋난다. 조인 테이블 컬럼을 직접 조건으로 만들 때는 **연산자를 하나로 고정**하거나 같은 파싱 규약을 재사용해야 한다.

```js
// ✅ 서버가 원시 like로 처리하는 컬럼은 UI 연산자를 contains로 고정
{
  dataField: 'registrant',
  allowFiltering: true,
  allowHeaderFiltering: false, // 표시값에 사번이 붙어 헤더필터 선택값과 서버값이 어긋남
  filterOperations: ['contains'],
}
```

## 결과

- 서버로 페이지 크기가 정상 전달되어 목록이 **100건 단위**로 조회되고, 화면 정렬이 서버 정렬로 반영됐다.
- 선택 옵션을 바꿔도 페이저가 유지된다.
- 페이지 경계에서 같은 정렬값을 가진 행의 순서가 흔들리지 않는다.

정리하면 이번 문제의 핵심은 세 가지다.

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| 목록이 20건에서 끊김 | 로컬 정렬·필터면 remoteOperations가 take/skip을 안 보냄 | filtering·sorting·paging을 함께 원격으로 켬 |
| 페이저 사라짐 | repaint()가 헤더 툴바를 재생성 | option() 갱신만 하고 repaint() 제거 |
| 페이지마다 순서 흔들림 | 정렬 키 동값 행의 순서 미보장 | 정렬 끝에 고유키 tiebreaker 추가 |
