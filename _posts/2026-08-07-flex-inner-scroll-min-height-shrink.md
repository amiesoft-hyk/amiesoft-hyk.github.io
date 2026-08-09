---
layout: post
title: "overflow-y: auto인데 스크롤이 안 된다 — flex 레이아웃의 min-height와 flex-shrink"
date: 2026-08-07 18:20:00 +0900
categories: [오류해결, CSS]
tags: [CSS, Flexbox, 스크롤, 레이아웃, Vue]
---

## 문제 상황

한 화면 안에 여러 패널이 들어가는 전체 높이 레이아웃에서, **내부 영역 스크롤이 동작하지 않았다.**

- 스크롤이 필요한 영역에 `overflow-y: auto`는 분명히 걸려 있음
- 그런데 내용이 넘쳐도 그 영역 안에서 스크롤되지 않고 **페이지 전체가 늘어남**
- 스크롤바가 보이는데 드래그해도 움직이지 않는 경우도 있었음

같은 화면에서 두 영역이 각각 다른 이유로 막혀 있었다.

## 원인

### 원인 1 — 스크롤 컨테이너에 `min-height: 0`이 없다

flex 컬럼의 자식은 기본값이 `min-height: auto`다. **콘텐츠 높이보다 작아질 수 없다는 뜻**이다.

```scss
.panel {
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.card-list {
  flex: 1;
  overflow-y: auto;   // 걸려 있지만 발동하지 않음
  // min-height: 0;   ← 없음
}
```

`.card-list`가 콘텐츠 높이만큼 늘어나 버리므로 **컨테이너를 넘치는 상황 자체가 만들어지지 않는다.** 넘치지 않으니 `overflow-y: auto`가 할 일이 없다.

### 원인 2 — 스크롤 컨테이너의 자식에 `flex-shrink: 0`이 없다

반대 방향의 문제다. 스크롤 컨테이너 자신이 `display: flex; flex-direction: column`이면, **자식들이 기본적으로 축소된다**(`flex-shrink: 1`).

```scss
.item-container {
  flex: 1;
  min-height: 0;      // 여기는 되어 있었음
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.item-block {
  // flex-shrink: 0;  ← 없음
}
```

자식들이 눌려서 컨테이너 안에 억지로 들어가므로 역시 **넘치지 않고**, 스크롤이 생기지 않는다. 화면에는 항목들이 찌그러진 채로 표시된다.

### 정리하면

flex 컨테이너에서 내부 스크롤이 성립하려면 **양쪽 조건이 모두** 필요하다.

| 대상 | 필요한 속성 | 없으면 |
|---|---|---|
| 스크롤 컨테이너 | `min-height: 0` | 줄어들지 못해 넘치지 않음 → 스크롤 없음 |
| 스크롤 컨테이너의 자식 | `flex-shrink: 0` | 눌려 들어가 넘치지 않음 → 스크롤 없음 |

문제의 화면에서는 한 영역에 `flex-shrink: 0`만 있고 `min-height: 0`이 없었고, 다른 영역에는 `min-height: 0`만 있고 `flex-shrink: 0`이 없었다. **서로 반대쪽 한 개씩만 빠져 있어서** 증상은 같은데 원인이 달랐다.

### 추가 — 페이지 전체 스크롤이 생기던 이유

최상위 컨테이너에 `min-height`가 걸려 있었다.

```scss
.page-layout {
  height: calc(100vh - 130px);
  min-height: 700px;      // ← 창이 작으면 화면보다 커진다
  overflow: hidden;
}
```

브라우저 창 높이가 `700px + 130px`보다 작아지면 컨테이너가 뷰포트를 넘어서므로 페이지 자체에 스크롤이 생긴다. 레이아웃이 뭉개지는 걸 막으려는 하한선이었지만, "페이지 스크롤 없이 내부에서만 스크롤"이 요구사항이라면 이 값과 공존할 수 없다.

## 해결 방법

```scss
// 1) 스크롤 컨테이너: 줄어들 수 있게
.card-list {
  flex: 1;

  // flex 자식의 기본 min-height(auto)를 해제해야 overflow-y가 동작한다.
  min-height: 0;
  overflow-y: auto;
}

// 2) 스크롤 컨테이너의 자식: 눌리지 않게
.item-block {
  // flex 컬럼의 자식은 기본으로 축소되어 컨테이너를 넘치지 않는다.
  flex-shrink: 0;
}

// 3) 최상위: 뷰포트에 고정, min-height 제거
.page-layout {
  height: calc(100vh - 130px);
  overflow: hidden;
}
```

## 결과

- 각 영역이 자기 안에서만 스크롤
- 창 높이를 줄여도 페이지 전체 스크롤이 생기지 않음
- 헤더·버튼 등 고정 영역은 그대로 유지

`min-height` 제거의 대가로, 창이 아주 짧아지면 페이지가 밀리는 대신 **패널 중 하나가 눌린다.** 어떤 패널이 먼저 줄어들지는 형제 요소의 `flex-shrink` 설정이 결정하므로, 중요한 영역에는 `flex-shrink: 0`을, 양보해도 되는 영역에는 `flex: 1`을 주는 식으로 배분하면 된다.

## 체크리스트

내부 스크롤이 안 될 때 위에서부터 훑으면 대체로 걸린다.

1. 스크롤 컨테이너에 `min-height: 0`이 있는가 (`flex-direction: row`라면 `min-width: 0`)
2. 스크롤 컨테이너의 **자식**에 `flex-shrink: 0`이 있는가
3. 조상 중에 높이가 확정된 요소가 있는가 (`height: 100%` 체인이 끊기지 않았는지)
4. 조상에 `min-height`가 걸려 뷰포트를 넘기고 있지 않은가
5. 애초에 넘칠 만큼 콘텐츠가 있는가 — 항목이 적으면 스크롤바가 보여도 움직일 범위가 없다
