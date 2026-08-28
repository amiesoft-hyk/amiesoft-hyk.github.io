---
layout: post
title: "공유 조회 서비스의 권한 검사, 라우터 메뉴 가드, DevExtreme lookup 기본값 3가지"
date: 2026-08-28 19:40:00 +0900
categories: [오류해결]
tags: [Spring, JPA, Vue, vue-router, DevExtreme, 권한]
---

관리자 화면에서 쓰던 조회 로직을 "마이페이지"(로그인한 본인 전용 화면)에서 재사용하면서 겪은 문제들을 정리한다.
증상은 셋 다 "화면은 떴는데 데이터가 안 나온다 / 엉뚱하게 나온다" 였지만 원인 계층은 각각 서비스, 라우팅, 그리드 에디터로 달랐다.

## 문제 상황

- 로그인 사용자가 본인에게 배정된 검토 대상 목록을 보는 마이페이지 화면을 새로 만들었다.
- 목록에서 "상세 진행" 버튼을 누르면, 관리자용 상세 화면과 **완전히 동일한 컴포넌트**를 재사용해 점수 입력 화면으로 이동시켰다.
- 이때 세 가지가 연쇄로 터졌다.
  1. 상세 화면 진입 자체가 `401 - 권한 없는 페이지` 로 막힘
  2. 401을 뚫고 나니, 상세 화면의 대상자 그리드가 **항상 빈 목록**
  3. 별개로, 일정(회차) 추가 그리드의 lookup 컬럼 기본값이 표시값과 에디터 값이 **서로 다르게** 보임

---

## 원인 1: 관리자 화면과 공유하는 조회 서비스가 "메뉴 권한 없으면 빈 결과"

상세 화면이 호출하는 대상자 목록 조회 서비스는 원래 관리자 화면 전용이었고,
맨 앞에서 로그인 사용자의 **메뉴 권한 범위**를 확인해 없으면 그냥 빈 리스트를 반환하고 있었다.

```java
// 재작성 예시
public List<TargetView> getTargetList(TargetSearch search) {
    MenuScope scope = authScopeService.getMenuScope(); // 현재 메뉴의 권한 등급 조회
    if (scope.isEmpty() || scope.isUnresolved()) {
        return List.of();              // ← 마이페이지 메뉴엔 권한 데이터가 없어 여기서 끝
    }
    if (!scope.isFull()) {
        // 부서/조직 범위로 접근 가능한 계획만 필터
        ...
    }
    return repositorySupport.getTargetList(search);
}
```

마이페이지 메뉴는 "로그인한 본인"이 대상이라서 별도 메뉴 권한(등급) 데이터를 부여하지 않는 경우가 많다.
그래서 `getMenuScope()` 가 빈 값을 돌려주고, 조회는 시작도 못 하고 `List.of()` 로 끝난다.
로그에는 이런 경고만 남았다.

```
WARN  AuthScopeService : 메뉴 권한 없음 - loginId=xxx, menuId=481
INFO  LogAspect        : method=RecordTargetApi.getTargetList ... execution time : 0 ms
```

### 해결 1: "본인이 담당 검토자면 권한 검사 건너뛰기"

이 프로젝트에는 이미 선례가 있었다 — 마이페이지 근태 내역 화면도 같은 이유로 관리자 조회 메서드에서 분리한 적이 있다.
같은 원칙(교육/평가/근태 등 마이페이지 화면은 "본인 고정"이므로 부서 권한을 적용하지 않는다)을 이 서비스에도 적용했다.

굳이 메서드를 통째로 복제하지 않고, **로그인 사용자가 해당 계획의 담당 검토자(리뷰어)로 지정돼 있으면 권한 검사를 건너뛰는** 분기를 넣었다.

```java
public List<TargetView> getTargetList(TargetSearch search) {
    String loginId = ContextUtils.getLoginId();

    // 로그인 사용자가 이 계획의 담당 검토자(리뷰어)면 마이페이지 진입으로 보고
    // 메뉴 권한(scope) 검사를 건너뛴다. 관리자 화면과 조회 메서드를 공유하기 때문에 필요한 예외.
    boolean isAssignedReviewer = targetRepository.existsByPlanIdAndReviewerIdAndDelFl(
        search.getPlanId(), loginId, "N");

    if (!isAssignedReviewer) {
        MenuScope scope = authScopeService.getMenuScope();
        if (scope.isEmpty() || scope.isUnresolved()) {
            return List.of();
        }
        if (!scope.isFull()) {
            // 기존 부서/조직 범위 필터 로직 그대로
            ...
        }
    }

    return repositorySupport.getTargetList(search);
}
```

핵심은 "관리자 화면 경로는 `isAssignedReviewer == false` 라 기존 로직을 100% 유지"하는 것이다.
공유 메서드에 예외를 넣을 때는 **원래 호출자의 동작이 바뀌지 않는지**를 먼저 확인해야 한다.

---

## 원인 2: 라우팅되는 모든 화면이 "등록 + 권한부여된 메뉴"여야 하는 프레임워크

프론트엔드 라우터 가드는 이렇게 생겼다.

```js
// 재작성 예시 - 일반 페이지 접근 권한 검사
const isAuthorizedRoute = (to) => {
  return menuStore.getMenuList
    .filter(menu => menu.authUseFl === 'Y' && menu.useFl === 'Y') // 권한 부여 + 사용중
    .some(menu => menu.pageUrl === to.path);
};
```

즉 `requiresAuth: true` 인 경로는 **메뉴 목록에 `pageUrl` 이 정확히 일치하는 활성 메뉴가 있고, 그 메뉴가 로그인 계정 권한에 부여돼 있어야** 통과한다.
새로 만든 상세 경로(`/my/task-progress/detail`)는 아직 메뉴로 등록되지 않아 무조건 401 → `/error/401` 로 튕겼다.

"마이페이지인데 왜 메뉴 권한이 필요하냐"는 반문이 나올 수 있는데, 이 프레임워크에서는
**라우팅되는 모든 화면**(팝업이 아닌 이상)이 메뉴 트리의 노드여야 한다.
다른 마이페이지 상세 화면들(평가 상세, 휴가 신청 상세 등)도 전부 부모 목록 메뉴 아래
`NORMAL_PAGE` 타입 자식 메뉴로 등록돼 있고, 권한도 부모와 함께 부여돼 있어서 "그냥 되는" 것처럼 보였을 뿐이다.

### 메뉴를 등록하는 두 가지 방식

메뉴 상세 화면을 등록하는 방식이 두 가지 있었고, 처음에 잘못된 쪽을 골랐다.

| 방식 | 테이블 | 특징 |
|---|---|---|
| 상세페이지 테이블 | `page_url` **UNIQUE** | 한 URL을 한 부모 메뉴의 상세로만 등록 가능 |
| 메뉴 자식 노드 | 메뉴 테이블 (`NORMAL_PAGE`) | 부모 아래 일반 자식 메뉴로 등록, 자기 `menuId` 보유 |

관리자용 상세 화면은 이미 상세페이지 테이블에 등록돼 있었는데, `page_url` 이 UNIQUE라 **같은 URL을 마이페이지 메뉴의 상세로 중복 등록할 수 없었다.**
그래서 마이페이지용으로는 **별도 URL**(`/my/task-progress/detail`)을 만들어 같은 컴포넌트를 재사용하고,
이 URL을 마이페이지 목록 메뉴의 **자식 `NORMAL_PAGE` 메뉴**로 등록하는 방식(다른 마이페이지 상세와 동일)으로 정리했다.

### 부수 버그 2-1: `NULL + 1` 로 만들어진 고아 메뉴

메뉴를 SQL로 넣을 때, 부모 조회 결과를 바인딩하지 않고 템플릿을 그대로 실행했다.

```sql
-- 잘못 실행된 형태
INSERT INTO menu (menu_depth, menu_nm, ..., parent_id)
VALUES (NULL + 1, '상세', ..., NULL);   -- NULL + 1 = NULL
```

`NULL + 1` 은 `NULL` 이라 `menu_depth`, `parent_id` 가 전부 비어버렸다.
`page_url` 은 UNIQUE라 재 INSERT도 불가. 결국 **UPDATE로 부모에 다시 연결**하는 게 정답이었다.

```sql
UPDATE menu AS c
JOIN   menu AS p ON p.page_url = '/my/task-progress'
SET    c.parent_id  = p.id,
       c.menu_depth = p.menu_depth + 1
WHERE  c.page_url = '/my/task-progress/detail';
```

### 부수 버그 2-2: 탭 닫기 시 `TypeError`

관리자 상세 화면을 열었다 닫으면 탭 시스템에서 에러가 났다. 원인은 탭 ID 생성 로직이었다.

```js
// 재작성 예시
const menuId = getRootMenuId(tabInfo);      // 상세페이지면 부모, 아니면 자기 menuId
const tabId  = stringToHash('TAB_' + menuId);
```

`getRootMenuId()` 가 `tabInfo.menuId` 를 못 찾으면 `undefined` 를 반환하고,
`stringToHash('TAB_undefined')` 는 **고정 해시**가 된다.
메뉴 데이터 없이 렌더된 탭은 브레드크럼 계산에서 이렇게 터진다.

```js
const getBreadCrumb = (tabInfo) => {
  let menu = getMenuById(tabInfo.menuId);   // undefined
  while (menu.menuDepth > 1) { ... }        // ← Cannot read properties of undefined
};
```

근본 해결은 **로그인 계정에 그 상세 메뉴 노드까지 권한을 부여**해서 `getMenuById` 가 정상 노드를 돌려주게 하는 것이다.
(방어 코드로 `getBreadCrumb` 를 null-safe 하게 만드는 것도 병행 가능하지만, 데이터가 맞아야 정상 동작한다.)

### 정리: 이 프레임워크에서 마이페이지 상세 화면 추가 체크리스트

1. 라우트는 부모 목록과 동일 규칙(`meta: { requiresAuth: true }`, `isDetailPage` 없이) 으로 등록
2. 메뉴 테이블에 부모 목록 메뉴의 **자식 `NORMAL_PAGE`** 로 등록 (`parent_id`, `menu_depth` 반드시 채우기)
3. 권한 관리에서 부모 메뉴 부여 시 **하위 포함**으로 자식까지 부여
4. 재로그인(메뉴 스토어 갱신)

---

## 원인 3: DevExtreme 그리드 lookup 컬럼 신규 행 기본값이 표시/에디터 불일치

일정(회차) 추가 그리드에서 "추출 방식" 컬럼(코드 lookup)의 신규 행 기본값을 넣으려고 이렇게 했다.

```js
const onInitNewRow = (e) => {
  if (e.data.extractMethodCd == null) {
    const auto = lookups.extractMethod.dataSource.find(code => code.codeNm === '자동');
    if (auto) e.data.extractMethodCd = auto.codeId;
  }
};
```

증상이 이상했다. **표시는 "자동"인데, 셀을 클릭해 에디터를 열면 "수동"으로 보였다.**

원인은 `codeNm === '자동'` 매칭이 (공백/표기 차이 등으로) 실패해 `auto` 가 `undefined` 가 되거나,
매칭돼도 `codeId` 의 타입이 lookup dataSource의 `codeId` 타입과 미묘하게 달라
표시용 lookup(관대한 매칭)과 에디터 SelectBox(엄격한 매칭)의 결과가 갈리는 것이었다.

### 해결 3: 이름 대신 "안정 코드값"으로 매칭하고, dataSource에서 직접 꺼내기

- `codeNm`(표시명, 바뀔 수 있음) 대신 **`codeValue`(고정 키)** 로 매칭
- lookup `dataSource` 에서 직접 객체를 찾아 `codeId` 타입이 그대로 일치하게

```js
const onInitNewRow = (e) => {
  if (e.data.extractMethodCd == null) {
    const list = lookups.extractMethod.dataSource || [];
    const manual =
      list.find(code => code.codeValue === 'extract_method_manual') ||
      list.find(code => (code.codeNm || '').trim() === '수동');   // 폴백
    if (manual) e.data.extractMethodCd = manual.codeId;
  }
};
```

`onInitNewRow` 로 안 잡히는 경우를 대비해 `onEditorPreparing` 에서도 값이 비어 있으면 같은 기본값을 세팅하고
`onValueChanged` 로 셀에 커밋되도록 이중으로 걸었다.

```js
const onEditorPreparing = (e) => {
  if (e.parentType === 'dataRow' && e.dataField === 'extractMethodCd' && e.value == null) {
    const manual = /* 위와 동일하게 codeValue 기준으로 조회 */;
    if (manual) {
      e.editorOptions.value = manual.codeId;
      const orig = e.editorOptions.onValueChanged;
      e.editorOptions.onValueChanged = (args) => { orig?.(args); e.setValue(args.value); };
    }
  }
};
```

---

## 결과

- **공유 조회 서비스**: "본인이 담당자면 권한 검사 스킵" 한 줄 분기로, 관리자 경로 동작을 유지한 채 마이페이지에서 데이터가 나오게 됨.
- **라우터 메뉴 가드**: 이 프레임워크에서 라우팅 화면 = 메뉴 노드. 마이페이지 상세도 부모 아래 `NORMAL_PAGE` 자식 메뉴로 등록 + 하위 포함 권한 부여 + 재로그인이 필요.
- **그리드 lookup 기본값**: 표시명(`codeNm`)이 아니라 고정 키(`codeValue`)로 매칭하고, `dataSource` 에서 꺼낸 값을 그대로 써서 타입 불일치를 없앰.

### 배운 것

- 관리자용 로직을 마이페이지에서 재사용할 때 첫 번째 장애물은 대부분 **권한 검사**다. 검사를 없애는 게 아니라 "본인 고정" 이라는 근거로 우회 조건을 명시한다.
- 프레임워크가 "모든 화면 = 메뉴" 라는 전제를 깔고 있으면, 새 화면 추가는 코드 + 메뉴 등록 + 권한 부여가 한 세트다.
- lookup/코드 값은 사람이 읽는 이름이 아니라 **불변 키**로 다뤄야 한다.
