---
layout: post
title: "저장은 성공하는데 수정일시가 안 바뀐다 — JPA 더티 체킹과 감사 컬럼"
date: 2026-07-30 17:20:00 +0900
categories: [오류해결, JPA]
tags: [JPA, Hibernate, Spring Data, Auditing]
---

## 문제 상황

본인 정보 수정 화면에 "마지막 저장 일시 / 저장자" 항목을 추가했다. 저장 버튼을 누르면 이 값이 방금 시각으로 갱신되어야 한다.

그런데 저장을 눌러도 값이 그대로였다.

- 저장 완료 토스트는 정상적으로 뜬다
- DB를 직접 조회해도 `updated_at` 컬럼이 이전 값 그대로다
- 예외도 없고 로그도 깨끗하다

엔티티는 Spring Data JPA Auditing을 쓰는 공통 베이스 클래스를 상속하고 있었다.

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "updated_by")
    private String updatedBy;

    @PreUpdate
    public void preUpdate() {
        // 요청 컨텍스트에서 로그인 ID를 꺼내 updatedBy 에 채운다
        this.updatedBy = currentLoginId();
    }
}
```

저장 서비스는 이런 흐름이었다.

```java
@Transactional
public Profile save(String userId, ProfileUpdateRequest request) {
    Profile profile = profileRepository.findByUserId(userId).orElseGet(...);
    profile.apply(request);              // 필드 대입
    return profileRepository.saveAndFlush(profile);
}
```

## 원인

**값이 실제로 바뀌지 않으면 Hibernate는 UPDATE 문 자체를 발행하지 않는다.**

Hibernate는 flush 시점에 영속 상태 엔티티의 현재 값과 로딩 시점의 스냅샷을 비교한다(더티 체킹). 비교 결과 달라진 필드가 하나도 없으면 실행할 SQL이 없다고 판단하고 아무 것도 보내지 않는다.

문제는 `@LastModifiedDate`와 `@PreUpdate`가 **UPDATE가 발행될 때 호출되는 콜백**이라는 점이다.

```
값 변경 없음 → 더티 아님 → UPDATE 미발행 → @PreUpdate / @LastModifiedDate 미실행 → 감사 컬럼 그대로
```

재현 조건이 명확하다. 사용자가 화면에서 아무 값도 바꾸지 않고 저장만 눌렀거나, 바꿨다가 원래 값으로 되돌린 경우다. 실제로 처음 확인했을 때 폼이 대부분 비어 있는 상태에서 저장을 눌렀고, DB 값도 이미 전부 `null`이라 변경분이 하나도 없었다.

`saveAndFlush()`를 호출했으니 뭔가 나갈 것 같지만, `flush()`는 "보류 중인 변경을 지금 반영하라"는 의미일 뿐 **변경이 없으면 보낼 것도 없다.**

### @DynamicUpdate와 헷갈리지 말 것

엔티티에 `@DynamicUpdate`가 붙어 있어서 처음엔 이 애노테이션을 의심했다. 하지만 둘은 다른 이야기다.

| 항목 | 역할 |
|---|---|
| 더티 체킹 | 변경이 **있는지** 판단 → 없으면 UPDATE 자체가 없음 |
| `@DynamicUpdate` | UPDATE를 낼 때 **어떤 컬럼을 넣을지** 결정 (변경된 컬럼만) |

즉 `@DynamicUpdate`를 떼도 증상은 그대로다. 원인은 더티 체킹이다.

## 해결 방법

"사용자가 저장 버튼을 눌렀다"는 사실 자체를 기록해야 하므로, 값 변경 여부와 무관하게 감사 필드를 **명시적으로** 채워 엔티티를 더티 상태로 만들었다.

```java
@Transactional
public Profile save(String userId, ProfileUpdateRequest request) {
    Profile profile = profileRepository.findByUserId(userId).orElseGet(...);
    profile.apply(request);

    // 입력값이 이전과 같아도 저장 이력이 남도록 감사 정보를 직접 기록한다.
    profile.markUpdated(LocalDateTime.now(), currentLoginId());

    return profileRepository.saveAndFlush(profile);
}
```

`updatedAt`에 현재 시각을 넣는 순간 스냅샷과 값이 달라지므로 더티로 판정되고, UPDATE가 발행되면서 나머지 변경분도 함께 반영된다.

### 주의: 자동 세팅을 건너뛰는 플래그

쓰던 공통 베이스 클래스에는 감사 필드를 수동으로 세팅하면 자동 세팅을 건너뛰는 플래그가 있었다.

```java
public void setManualUpdatedAt(LocalDateTime updatedAt) {
    this.updatedAt = updatedAt;
    this.manuallySetAudit = true;   // 이후 @PreUpdate 자동 처리를 스킵
}
```

`updatedAt`만 수동으로 넣으면 `updatedBy`를 채우는 자동 로직이 건너뛰어져, **시각은 갱신되는데 저장자는 이전 사람으로 남는** 상태가 된다. 그래서 두 값을 항상 함께 세팅했다.

### 신규 등록 케이스

행이 없어서 새로 INSERT 되는 경우도 확인이 필요하다. Spring Data JPA Auditing은 INSERT 시점에 `@CreatedDate`와 `@LastModifiedDate`를 **둘 다** 채워준다. 반면 `@PreUpdate`에서만 채우던 `updatedBy`는 `null`로 남는다.

조회 응답에서는 이 점을 감안해 폴백을 뒀다.

```java
.lastSavedAt(entity.getUpdatedAt() != null ? entity.getUpdatedAt() : entity.getCreatedAt())
.lastSavedBy(entity.getUpdatedBy() != null ? entity.getUpdatedBy() : entity.getCreatedBy())
```

## 결과

- 값을 바꾸지 않고 저장만 눌러도 마지막 저장 일시/저장자가 갱신됨
- 관리자가 다른 화면에서 같은 데이터를 수정하면 그 사람의 ID로 기록됨
- 신규 등록 직후에도 등록 정보로 폴백되어 빈칸이 노출되지 않음

## 정리

> `@LastModifiedDate`, `@PreUpdate`는 **UPDATE가 실제로 나갈 때만** 동작한다. "저장 시도" 자체를 이력으로 남겨야 한다면 더티 체킹에 의존하지 말고 감사 필드를 직접 세팅해야 한다. `saveAndFlush()`를 불렀다고 UPDATE가 보장되는 것이 아니다.
