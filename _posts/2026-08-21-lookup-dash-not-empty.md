---
layout: post
title: "화면의 '-'는 빈 값이 아니었다 - lookup 컬럼의 표시 기준과 필터 기준"
date: 2026-08-21 15:30:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, QueryDSL, 공통코드]
---

## 문제 상황

그리드 헤더필터의 "(필드 값 없음)" 처리를 수정한 뒤였다. 서버는 조건을 정확히 만들고 있었다.

```sql
where del_fl = 'N'
  and (change_type_cd is null or trim(change_type_cd) = '')
  and last_chng_time between '2026-07-01' and '2026-08-21'
```

그런데 화면에서는 여전히 "적용이 안 된다"는 얘기가 나왔다. 목록에 `-`로 표시된 행이 분명히 있는데, **"(필드 값 없음)"을 체크하면 그 행들이 조회되지 않았다.**

SQL은 맞는데 결과가 기대와 다르다. 이 경우 의심할 곳은 하나뿐이다 — **화면에 보이는 값과 DB에 저장된 값이 다른가?**

## 원인 — `-`는 두 가지를 뜻하고 있었다

공통 그리드 컴포넌트의 셀 렌더링 로직을 보니 이런 코드가 있었다.

```js
// 값이 있을 때만 진입한다
if (e.value !== null && e.value !== undefined && e.value !== '') {
    const lookupItems = resolveLookupItems(column);
    const valueExpr = column.lookup?.valueExpr || 'this';
    const matched = lookupItems.some(
        item => String(getLookupValue(item, valueExpr)) === String(e.value),
    );
    if (!matched && e.cellElement) {
        e.cellElement.textContent = '-';   // ← lookup 목록에 없으면 '-' 로 표시
    }
}

// 진짜 빈 셀도 '-' 로 표기
if (e.rowType === 'data' && e.cellElement?.innerHTML === '&nbsp;') {
    e.cellElement.textContent = '-';
}
```

두 블록이 **같은 `-`를 찍고 있었다.**

| 화면 | 실제 DB 값 |
|---|---|
| `-` | 값이 진짜 없음 (NULL / 빈 문자열) |
| `-` | **값은 있는데 공통코드 목록에 없음** |

문제의 행들은 후자였다. 전자였다면 `is null or trim(...) = ''` 조건에 걸려 조회됐을 테니까.

DevExtreme에서 lookup 컬럼은 `valueExpr`로 매칭한 항목의 `displayExpr`를 보여준다. 매칭이 안 되면 아무것도 표시하지 않고, 위 로직이 그 빈 칸을 `-`로 채운다. 코드값이 바뀌었거나 공통코드가 정리되면서 목록에서 빠진 값이 데이터에 남아 있으면 이렇게 된다.

### 헤더필터 목록에도 안 뜬다

여기에 하나가 더 겹친다. DevExtreme은 lookup 컬럼의 헤더필터 항목을 **lookup 목록 기준으로** 만든다. 목록에 없는 코드값을 가진 행은 어떤 항목으로도 선택할 수 없다.

즉 그 행들은 **일반 값으로도, "(필드 값 없음)"으로도 걸러지지 않는 사각지대**에 있었다.

## 이미 정렬은 다른 기준을 쓰고 있었다

같은 파일의 정렬 코드를 보니 흥미로웠다.

```java
private StringExpression codeNameExpression(String codeKey, StringPath codePath) {
    CaseBuilder.Cases<String, StringExpression> cases = new CaseBuilder()
        .when(codePath.isNull()).then("");
    for (Code code : codeList(codeKey)) {
        cases = cases.when(codePath.eq(code.getCodeValue())).then(code.getCodeNm());
    }
    return cases.otherwise("");   // ← 코드 목록에 없으면 빈 값으로 본다
}
```

정렬은 코드명 기준, 즉 **화면 표시 기준**으로 동작하고 있었다. 목록에 없는 값은 빈 문자열로 취급해서 맨 앞이나 맨 뒤로 몰린다.

**정렬은 화면 기준, 필터는 DB 원본 기준.** 같은 컬럼에 잣대가 둘이었고, 이 불일치가 증상으로 드러난 것이다.

## 해결 방향

필터의 빈 값 판정을 정렬과 같은 기준으로 맞춘다.

```
현재: change_type_cd IS NULL OR TRIM(change_type_cd) = ''
수정: 위 조건 OR change_type_cd NOT IN ('유효코드1', '유효코드2', ...)
```

값 선택 필터는 그대로 두고 **빈 값 판정에만** 코드 목록을 사용한다. 공통코드 조회 클라이언트는 이미 해당 Repository에 주입되어 있어(정렬 표현식이 쓰고 있다) 추가 의존성이 필요 없다.

구현은 빈 값 표현식을 외부에서 주입할 수 있도록 필터 유틸에 오버로드를 두는 방향이 깔끔하다.

```java
// 기본: 컬럼 타입에 따른 판정
applyIncludeExclude(condition, expression, includes, excludes, search, field);

// lookup 컬럼: 빈 값 판정을 바꿔서 전달
applyIncludeExclude(condition, expression, includes, excludes, search, field,
    codeDisplayEmpty(codeKey, codePath));
```

## 정리하며

같은 컬럼을 두고 **화면·정렬·필터가 각각 다른 기준**을 쓰면, 셋 중 둘이 맞아도 나머지 하나에서 반드시 이상하게 보인다.

특히 lookup 컬럼은 "표시값"과 "저장값"이 분리되어 있어 이 함정에 빠지기 쉽다. 필터를 만들 때 **사용자가 화면에서 보고 고르는 것은 표시값**이라는 점을 놓치면, 서버 SQL이 아무리 정확해도 "필터가 안 먹는다"는 얘기를 듣게 된다.

디버깅 관점에서 얻은 것도 있다. SQL 로그에 조건이 정확히 찍혀 있는데 결과가 기대와 다르면, 그때부터는 **SQL이 아니라 데이터와 표시 계층을 의심**해야 한다. 이번에는 `-`라는 표시 하나가 두 가지 상태를 덮고 있었던 게 원인이었다.
