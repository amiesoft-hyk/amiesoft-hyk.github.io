---
layout: post
title: "insertable=false 컬럼이 권한 필터 기준이 되면서 신규 데이터가 목록에서 사라진 문제"
date: 2026-08-20 09:40:00 +0900
categories: [오류해결, JPA]
tags: [JPA, Hibernate, QueryDSL, 권한, 데이터정합성]
---

새로 등록한 데이터가 목록에 보이지 않았다. INSERT 로그는 정상이었고 DB에도 행이 있었다.
원인은 **`insertable = false`로 매핑돼 늘 NULL이던 컬럼이, 어느 시점부터 권한 필터의 기준으로 쓰이기 시작한** 것이었다.

## 문제 상황

등록은 성공했다. p6spy 로그에도 INSERT가 찍혔다.

```sql
insert into user_master
    (user_name, user_id, dept_cd, join_dt, use_yn, ...)
values
    ('test_user', 'test_user', 7, '2026-08-19', 'Y', ...)
```

DB를 직접 조회하면 행도 있었다.

```
id=1272, user_id=test_user, dept_cd=7,
center_dept_cd=NULL, team_dept_cd=NULL,
del_yn=N, use_yn=Y, state=EMPLOYED
```

그런데 목록 조회 SQL은 이랬다.

```sql
select ...
  from user_master um
 where um.state = 'EMPLOYED'
   and um.use_yn = 'Y'
   and um.del_yn = 'N'
   and um.center_dept_cd = 1     -- ← NULL 이라 탈락
 order by um.user_name
```

`NULL = 1`은 참이 아니므로 방금 만든 행은 결과에서 빠진다.
전체 권한 계정으로 보면 정상적으로 보였기 때문에 더 헷갈렸다.

## 원인

### 1. 엔티티가 두 컬럼을 쓰기 대상에서 제외하고 있었다

```java
/*
 * 부서트리(센터 > 팀 > 파트)의 상위 계층을 DEPT_CD와 별도로 보관하는 컬럼.
 * 조회 전용으로만 매핑하며, 갱신은 관리 화면 쪽 로직에서 담당한다.
 */
@Column(name = "CENTER_DEPT_CD", insertable = false, updatable = false)
private Long centerDeptCd;

@Column(name = "TEAM_DEPT_CD", insertable = false, updatable = false)
private Long teamDeptCd;
```

주석은 "갱신은 관리 화면 쪽 로직에서 담당한다"고 적혀 있었지만, **그 로직은 어디에도 없었다.**
소스 전체를 검색해도 이 두 필드에 값을 넣는 코드가 필드 선언 외에 한 곳도 없었다.

`insertable = false, updatable = false`라 JPA는 이 컬럼을 SQL에 아예 포함하지 않는다. 실제 INSERT 문에도 두 컬럼이 빠져 있었다.

### 2. 권한 필터가 그 컬럼을 직접 비교하기 시작했다

권한 체크 로직이 개편되면서 이런 조건이 들어왔다.

```java
public static BooleanExpression ofUserMaster(AuthScope scope, QUserMaster userMaster) {
    if (scope == null || scope.isEmpty()) {
        return Expressions.FALSE;
    }
    if (scope.isFull()) {
        return null;   // 전체 권한이면 조건 없음
    }
    Long scopeDeptId = scope.getScopeDeptId();
    return switch (scope.getRole()) {
        case CENTER  -> userMaster.centerDeptCd.eq(scopeDeptId);
        case TEAM    -> userMaster.teamDeptCd.eq(scopeDeptId);
        case SECTION -> userMaster.deptCd.eq(scopeDeptId);
        default -> null;
    };
}
```

`SECTION` 권한은 `dept_cd`를 보므로 멀쩡했지만, `CENTER`/`TEAM` 권한은 늘 NULL인 컬럼을 비교하게 됐다.

**두 조건이 겹쳐야 증상이 나타난다.**

- 값을 채우는 코드가 없다 (이건 원래부터 그랬다)
- 그 값이 필터 기준이 됐다 (이건 최근 변경이다)

값이 비어 있어도 아무도 쓰지 않을 때는 문제가 드러나지 않았다. 기존 데이터는 과거 마이그레이션으로 채워져 있어 정상 조회됐고, **개편 이후 새로 만든 데이터만** 사라졌다.

git으로 확인해보니 두 컬럼은 처음 추가될 때부터 `insertable = false`였다.

```bash
git log --oneline -S "insertable = false" -- src/main/java/.../UserMaster.java
```

## 해결 방법

### 1. 매핑 제한을 풀고 갱신 메서드를 만든다

```java
/*
 * 권한 범위 판정이 이 값을 직접 비교하므로,
 * DEPT_CD 가 정해지는 시점에 updateDeptHierarchy 로 함께 갱신해야 한다.
 */
@Column(name = "CENTER_DEPT_CD")
private Long centerDeptCd;

@Column(name = "TEAM_DEPT_CD")
private Long teamDeptCd;

public void updateDeptHierarchy(Long centerDeptCd, Long teamDeptCd) {
    this.centerDeptCd = centerDeptCd;
    this.teamDeptCd = teamDeptCd;
}
```

### 2. 부서 경로에서 상위 계층을 계산해 채운다

부서 트리는 루트부터 자기 자신까지의 ID 경로를 가지고 있으므로, 앞에서부터 잘라 쓰면 된다.

```java
private void applyDeptHierarchy(UserMaster userMaster, Long deptId) {
    if (deptId == null) {
        userMaster.updateDeptHierarchy(null, null);
        return;
    }

    DeptEntity dept = deptService.getActiveDeptMap().findOneById(deptId);
    List<Long> pathIds = parseDeptPathIds(dept == null ? null : dept.getPath());
    if (pathIds.isEmpty()) {
        log.warn("부서 경로를 찾을 수 없어 상위 계층을 설정하지 못했습니다. deptId: {}", deptId);
        userMaster.updateDeptHierarchy(null, null);
        return;
    }

    Long centerDeptCd = pathIds.getFirst();
    Long teamDeptCd = pathIds.size() > 1 ? pathIds.get(1) : null;
    userMaster.updateDeptHierarchy(centerDeptCd, teamDeptCd);
}
```

계층별로 이렇게 정리된다.

| 소속 부서 depth | center | team |
|---|---|---|
| 1 (센터 자체) | 자신 | `null` |
| 2 (팀) | 상위 | 자신 |
| 3 (섹션) | `path[0]` | `path[1]` |

### 3. 부서가 정해지는 모든 지점에서 호출한다

엔티티에서 `deptId`가 바뀌는 곳을 전부 찾아 짝을 맞췄다. 생성자와 정보 수정 메서드 두 곳이었다.

```java
// 등록
userMaster.updateState(EMPLOYED);
applyDeptHierarchy(userMaster, userMaster.getDeptId());
userMasterRepository.save(userMaster);

// 수정
userMaster.updateUserInfo(update);
applyDeptHierarchy(userMaster, userMaster.getDeptId());   // 부서 변경 시 함께 갱신
```

부서를 바꿨는데 상위 계층이 그대로면 다시 어긋나므로, 수정 경로 처리가 빠지면 안 된다.

## 결과

- 신규 등록 데이터가 목록에 바로 나온다.
- 부서를 변경하면 상위 계층도 함께 갱신된다.
- 부서를 못 찾는 경우엔 경고 로그를 남기고 `null`로 둔다. 값이 틀리게 들어가는 것보다 드러나는 편이 낫다.

기존 누락 데이터는 별도 UPDATE로 보정하면 된다.

## 남는 교훈

**"조회 전용 매핑"이라는 주석은 검증되지 않는다.** 누가 채우는지 코드로 강제되지 않으면, 그 컬럼은 조용히 비어 있다가 누군가 그걸 기준으로 쓰는 순간 터진다.

그리고 **이런 버그는 "기존 데이터는 되는데 신규만 안 되는" 형태로 나타난다.** 그 패턴이 보이면 조회 조건보다 "그 컬럼을 누가 채우는가"를 먼저 확인하는 게 빠르다.
