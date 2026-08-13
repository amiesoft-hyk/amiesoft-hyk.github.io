---
layout: post
title: "수정은 되는데 신규 저장만 안 된다 — 서비스가 선택값을 기본값으로 덮어쓰던 문제"
date: 2026-08-13 15:10:00 +0900
categories: [오류해결, Spring]
tags: [SpringBoot, JPA, 디버깅, 백엔드]
---

## 문제 상황

회원 등록 화면에서 **권한 그룹을 선택해 저장하면 반영되지 않는데, 저장된 회원을
수정할 때 같은 값을 고르면 정상 반영**되는 현상이 보고됐다.

신규와 수정이 서로 다르게 동작한다는 것은 화면(폼)이 아니라 **서버의 등록/수정 경로가
갈라지는 지점**에 원인이 있을 가능성이 높다. 실제로 프론트 payload를 확인해 보니
신규·수정 모두 선택한 값을 그대로 전송하고 있었다.

```js
// 신규 저장 payload — 선택값을 그대로 보내고, 미선택이면 -1
const data = {
  ...formData,
  roleId: !isEmpty(formData.roleId) ? formData.roleId : -1,
};
```

## 원인

서비스의 등록 메서드가 **전달받은 값을 확인하지 않고 무조건 기본값으로 덮어쓰고**
있었다.

```java
@Transactional
public Member create(MemberDto.Add dto) {
    validate(dto);

    Long roleId = resolveDefaultRoleByDept(dto.getDeptId());  // 부서 기준 기본 권한
    dto.setRoleId(roleId);                                    // ← 선택값을 덮어씀

    Member member = dto.toEntity();
    memberRepository.save(member);
    ...
}
```

`resolveDefaultRoleByDept()`는 부서 경로명에 따라 협력사별 기본 권한을 돌려주는
메서드다. 처음 만들 때는 "등록 시 권한을 고르는 UI가 없으니 부서로 자동 부여한다"는
의도였는데, 이후 화면에 권한 선택 항목이 추가되면서 **선택값이 항상 무시되는** 상태가
됐다.

수정 경로는 전달받은 값을 그대로 반영하고 있었기 때문에 정상 동작했다.

```java
public void update(MemberDto.Update dto) {
    this.roleId = dto.getRoleId();   // ← 전달값 그대로 반영
    ...
}
```

기능이 추가되면서 **기존의 "기본값 자동 부여" 로직이 그대로 남아 새 입력을 잡아먹은**
전형적인 케이스다.

## 해결 방법

선택값이 있으면 그대로 쓰고, 미선택일 때만 기본값을 부여하도록 조건부로 바꿨다.

```java
/**
 * 화면에서 권한을 실제로 선택했는지 판별한다.
 * 미선택 시 프론트는 -1을 전송하므로 양수만 유효한 선택값으로 본다.
 */
private boolean isSelectedRoleId(Long roleId) {
    return roleId != null && roleId > 0;
}

@Transactional
public Member create(MemberDto.Add dto) {
    validate(dto);

    // 선택값 우선, 미선택일 때만 부서 기반 기본 권한 부여
    if (!isSelectedRoleId(dto.getRoleId())) {
        dto.setRoleId(resolveDefaultRoleByDept(dto.getDeptId()));
    }

    Member member = dto.toEntity();
    memberRepository.save(member);
    ...
}
```

`-1`을 미선택 표식으로 쓰는 이유는 DTO에 `@NotNull` 제약이 걸려 있어 프론트가 `null`을
보낼 수 없기 때문이다. 그래서 서버는 `null`과 `-1`을 모두 "미선택"으로 취급한다.

한 가지 더 확인해야 했던 것은 **결정된 값이 어디까지 쓰이는가**였다. 이 DTO의
`roleId`는 엔티티 생성뿐 아니라 연동 계정 생성 DTO에도 함께 쓰이고 있었다.

```java
public LoginAccountDto.Add toLoginAccountDto() {
    return LoginAccountDto.Add.builder()
        .loginId(this.loginId)
        .roleId(this.roleId)   // ← 같은 값이 연동 계정에도 반영된다
        .build();
}
```

DTO 필드를 최종값으로 세팅하는 방식을 유지했기 때문에, 선택한 권한이 엔티티와 연동
계정 양쪽에 동일하게 반영된다. 수정 경로의 동기화 정책과도 일치한다.

## 결과

- 권한을 선택해 등록하면 선택값이 그대로 저장된다
- 권한을 비운 채 등록하면 기존처럼 부서 기반 기본 권한이 부여된다 (하위호환 유지)

### 정리하며

"수정은 되는데 등록은 안 된다"처럼 **CRUD 중 일부 경로만 다르게 동작하는 증상**은
프론트보다 서버의 분기 지점을 먼저 보는 편이 빠르다. 이번 건도 payload 로그를 확인해
프론트를 후보에서 제외한 뒤, 등록 서비스만 읽어서 5분 만에 원인을 특정했다.

그리고 `setXxx()`로 입력 DTO를 덮어쓰는 코드는 나중에 이런 식으로 조용히 기능을
잡아먹기 쉽다. 기본값 부여가 필요하다면 **"값이 없을 때만"이라는 조건을 처음부터 명시**해
두는 편이 안전하다.
