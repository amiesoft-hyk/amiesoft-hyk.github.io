---
layout: post
title: "엑셀 다운로드를 누르면 브라우저가 멈춘다 — 내보내기용 임시 그리드에 딸려온 cellTemplate"
date: 2026-08-03 17:20:00 +0900
categories: [오류해결, DevExtreme]
tags: [DevExtreme, Vue, DataGrid, 엑셀, 성능]
---

## 문제 상황

목록 화면에서 엑셀 다운로드 버튼을 누르면 **"페이지가 응답하지 않습니다"** 알림이 뜨고 파일은 끝내 내려오지 않는다.

특이한 건 같은 다운로드 구조를 쓰는 다른 화면 4~5개는 전부 정상이라는 점이었다. 문제가 되는 화면 하나만 멈춘다.

여기서 먼저 짚고 갈 것: **"응답하지 않는 페이지"는 메인 스레드가 동기적으로 오래 붙잡혀 있을 때만 뜬다.** 서버 응답을 `await`로 기다리는 동안에는 절대 나타나지 않는다. 이 화면의 다운로드는 전체 데이터를 여러 페이지로 나눠 순차 조회하는 구조라 "서버가 느린 것"부터 의심하기 쉬웠지만, 그 구간은 전부 `await`라서 애초에 용의선상에서 빠진다. 범인은 데이터를 다 받은 **이후** 구간이다.

## 원인

### 내보내기용 임시 그리드가 화면용 cellTemplate을 그대로 물려받는다

이 프로젝트의 전체 다운로드는 "화면 그리드의 목록·선택 상태를 건드리지 않기 위해" 화면 밖에 임시 `DataGrid`를 만들어 내보내는 방식을 쓴다.

```javascript
const onExporting = async e => {
  e.cancel = true;
  const component = e.component;
  const rows = await fetchAllRows();          // 전체 데이터 수집 (여기는 전부 await)

  const container = document.createElement('div');
  container.style.cssText = 'position:fixed;left:-100000px;top:0;width:1920px;height:800px;';
  document.body.appendChild(container);

  const exportGrid = new DataGrid(container, {
    dataSource: rows,
    columns: component.getVisibleColumns(),   // ← 문제 지점
    remoteOperations: false,
    paging: { enabled: false },               // ← 문제를 증폭시키는 지점
    selection: { mode: 'none' },
  });
  await exportGrid.refresh();

  await exportDataGrid({ component: exportGrid, worksheet, keepColumnWidths: true });
  // ... 파일 저장
};
```

`getVisibleColumns()`가 돌려주는 컬럼 객체에는 **화면에서 쓰던 `cellTemplate`이 그대로 들어 있다.** 그리고 이 화면은 다른 화면과 달리 컬럼 7개에 `cellTemplate`이 걸려 있었다.

```javascript
{
  caption: '내용',
  dataField: 'contentHtml',
  customizeText: cellInfo => stripHtml(cellInfo.value),
  cellTemplate: (container, options) => {
    const aTag = document.createElement('a');
    aTag.textContent = stripHtml(options.value);   // 행마다 HTML 파싱
    aTag.addEventListener('click', () => openDetail(options.data));
    container.append(aTag);
  },
},
// + 분포 컬럼 6개: 행마다 배경색 지정하는 cellTemplate
```

그리고 `stripHtml`은 호출할 때마다 HTML 문서를 통째로 만든다.

```javascript
const stripHtml = value => {
  const html = String(value || '');
  if (!html) return '';
  const doc = new DOMParser().parseFromString(html, 'text/html');   // 매번 문서 생성
  return String(doc.body.textContent || '').replace(/\s+/g, ' ').trim();
};
```

여기까지 조합하면 답이 나온다.

- 화면 그리드는 `pageSize: 100`이라 한 번에 100행만 렌더링 → 견딘다
- 임시 그리드는 `paging: { enabled: false }` → **전체 행을 한 번에 동기 렌더링**
- 전체 행 × cellTemplate 7개 + 행마다 `DOMParser` 문서 생성

에디터로 작성된 리치 HTML이 들어 있는 컬럼이라 파싱 비용도 작지 않다. 결과적으로 메인 스레드가 수십 초 단위로 잠기고 브라우저가 "응답 없음"으로 판단한다.

### 왜 다른 화면은 멀쩡했나

같은 임시 그리드 패턴을 쓰는 다른 화면들은 표시값 가공을 전부 `calculateDisplayCellValue` / `customizeText`로만 하고 있었다. 이들은 값을 계산하는 함수라 DOM을 만들지 않는다. **`cellTemplate`을 쓰는 화면만 이 문제에 걸린다**는 게 "이 화면만 멈춘다"는 관찰과 정확히 맞아떨어졌다.

### 핵심: 엑셀 내보내기는 cellTemplate을 쓰지 않는다

`exportDataGrid`는 셀의 DOM을 읽지 않는다. 셀 값과 `customizeText` / `calculateDisplayCellValue` / `lookup`으로 텍스트를 만든다. 즉 **임시 그리드에 `cellTemplate`을 넘겨봐야 결과 파일에는 아무 영향이 없고, 렌더링 비용만 그대로 낸다.**

## 해결 방법

### 1. 내보내기용 컬럼에서 템플릿 제거

`getVisibleColumns()` 결과를 그대로 쓰지 않고, 내보내기에 필요 없는 속성을 걷어낸 컬럼을 만들어 넘긴다.

```javascript
/**
 * 엑셀 내보내기용 컬럼 목록을 만든다.
 * 내보내기는 cellTemplate을 사용하지 않는데(값은 customizeText/calculateDisplayCellValue/lookup으로 결정),
 * 화면용 cellTemplate이 그대로 넘어가면 임시 그리드가 전체 행을 렌더링하면서 화면이 멈춘다.
 */
const buildExportColumns = component =>
  component
    .getVisibleColumns()
    .filter(column => column.dataField && column.allowExporting !== false)
    .map(column => {
      const {
        cellTemplate,
        headerCellTemplate,
        editCellTemplate,
        groupCellTemplate,
        ownerBand,          // 원본 그리드 기준 내부 인덱스 — 임시 그리드에서는 엉뚱한 컬럼을 가리킨다
        index,
        visibleIndex,
        ...exportColumn
      } = column;
      return exportColumn;
    });
```

`ownerBand` 같은 내부 상태 속성도 같이 털어낸다. 밴드(멀티헤더) 컬럼이 있는 그리드에서 `getVisibleColumns()`는 말단 컬럼만 돌려주는데, 각 컬럼에 붙어 있는 `ownerBand`는 **원본 그리드의 컬럼 인덱스**다. 컬럼 구성이 다른 임시 그리드에 그대로 넘기면 존재하지 않거나 엉뚱한 부모를 참조하게 된다.

`caption`, `dataField`, `dataType`, `width`, `customizeText`, `calculateDisplayCellValue`, `lookup`은 남겨야 엑셀 표시값이 화면과 같게 나온다.

### 2. HTML 파싱 결과 캐싱

`stripHtml`은 셀 렌더링·헤더 필터·내보내기에서 같은 값으로 반복 호출된다. 원문을 키로 캐싱하면 `DOMParser` 호출 횟수가 크게 줄고, 화면 렌더링도 같이 빨라진다.

```javascript
const stripHtmlCache = new Map();

const stripHtml = value => {
  const html = String(value || '');
  if (!html) return '';
  if (stripHtmlCache.has(html)) return stripHtmlCache.get(html);

  const doc = new DOMParser().parseFromString(html, 'text/html');
  const text = String(doc.body.textContent || '').replace(/\s+/g, ' ').trim();
  stripHtmlCache.set(html, text);
  return text;
};
```

### 3. 진행 표시 추가

전체 다운로드는 서버를 여러 번 조회하는데 로딩 표시가 전혀 없었다. 데이터가 많으면 그것만으로도 "멈춘 것처럼" 보인다.

```javascript
component.beginCustomLoading('엑셀 다운로드');
try {
  // 데이터 수집 → 임시 그리드 생성 → 내보내기
} finally {
  exportGrid?.dispose();
  container.remove();
  component.endCustomLoading();
}
```

`finally`에서 반드시 풀어줘야 중간에 실패했을 때 로딩이 영구히 남지 않는다. 데이터 수집 단계의 예외, 조회 결과 0건으로 조기 리턴하는 경로에서도 빠짐없이 해제해야 한다.

## 결과

- 엑셀 다운로드 시 화면이 멈추지 않는다
- 결과 파일 내용은 이전과 동일 — `cellTemplate`은 애초에 엑셀에 반영되지 않던 속성이라 제거해도 표시값이 변하지 않는다
- 화면 그리드 렌더링도 캐싱 덕에 함께 빨라졌다

기억해둘 것.

- **"응답 없는 페이지"는 동기 블로킹의 신호다.** `await`가 걸린 네트워크 구간은 후보에서 빼고 시작하면 탐색 범위가 확 줄어든다.
- **화면용 컬럼 정의를 내보내기용 그리드에 그대로 재사용하지 말 것.** `getVisibleColumns()`는 편리하지만 렌더링 전용 속성과 내부 상태까지 함께 딸려온다.
- 임시 그리드를 `paging: { enabled: false }`로 만들면 전체 행이 한 번에 렌더링된다. 행 수가 늘어날수록 선형으로 비싸지는 구조라는 걸 의식해야 한다.
