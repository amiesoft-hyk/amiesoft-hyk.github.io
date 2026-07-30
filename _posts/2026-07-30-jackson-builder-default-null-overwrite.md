---
layout: post
title: "@Builder.Default는 역직렬화에 적용되지 않는다 — 부분 수정 API가 기존 데이터를 날린 사례"
date: 2026-07-30 17:30:00 +0900
categories: [오류해결, Jackson]
tags: [Jackson, Lombok, JPA, Spring Boot]
---

## 문제 상황

관리자용 상세 팝업에서 회원 정보를 수정했더니, **팝업에 없는 항목들이 전부 `NULL`로 초기화**되는 현상이 있었다.

- 팝업에서 편집한 항목(연락처, 주소 등)은 정상 저장
- 팝업에 표시되지 않는 항목(성별 코드, 학력 코드, 자격 보유 여부 등)은 저장 직후 `NULL`
- 예외도 없고 저장 성공 응답도 정상

팝업이 보내는 요청은 화면에 있는 필드만 담고 있었다.

```javascript
// 프론트: 화면에서 편집하는 항목만 전송
const toSaveData = () => ({
  userId: form.userId,
  phone: form.phone,
  email: form.email,
  address: form.address,
});
```

## 원인

원인은 **두 가지가 겹친 것**이었다.

### 1. 요청 DTO의 기본값이 적용되지 않았다

수정 요청 DTO는 Lombok 빌더를 쓰면서 일부 필드에 `@Builder.Default`로 기본값을 두고 있었다.

```java
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ProfileUpdateRequest {

    private String userId;
    private String phone;
    private String email;
    private String address;

    private Long genderCode;
    private Long educationCode;

    @Builder.Default
    private String certifiedFl = "N";   // ← 기본값 'N' 을 의도했지만...

    @Builder.Default
    private String deletedFl = "N";
}
```

`@Builder.Default`는 이름 그대로 **빌더로 객체를 만들 때만** 적용된다. Lombok은 이 애노테이션이 붙으면 **필드의 초기화식을 제거하고** 그 값을 빌더 전용 `$default$...()` 메서드로 옮긴다.

```java
// Lombok 이 만들어내는 형태 (개념적으로)
private String certifiedFl;                                  // 초기화식이 사라짐
private static String $default$certifiedFl() { return "N"; } // 빌더에서만 사용
```

Jackson은 빌더가 아니라 **기본 생성자 + 필드/세터**로 객체를 만든다. 따라서 요청 JSON에 `certifiedFl`이 없으면 `"N"`이 아니라 **`null`이 된다.** 빌더를 쓰지 않는 경로에서는 기본값이 사라지는 셈이다.

> 참고로 `@NoArgsConstructor(access = PRIVATE)`라도 Jackson은 리플렉션으로 접근하므로 객체 생성 자체는 문제없이 된다. 조용히 `null`이 될 뿐이다.

### 2. 엔티티가 전체 필드를 무조건 대입했다

여기에 엔티티의 병합 메서드가 기름을 부었다.

```java
public void merge(ProfileUpdateRequest request) {
    this.phone = request.getPhone();
    this.email = request.getEmail();
    this.address = request.getAddress();
    this.genderCode = request.getGenderCode();          // 전송 안 됨 → null
    this.educationCode = request.getEducationCode();    // 전송 안 됨 → null
    this.certifiedFl = request.getCertifiedFl();        // @Builder.Default 무력화 → null
    this.deletedFl = request.getDeletedFl();
}
```

전송되지 않은 필드가 `null`인 채로 **무조건 대입**되면서 DB의 기존 값을 덮어썼다.

`@DynamicUpdate`가 붙어 있으면 안전하지 않냐고 생각할 수 있는데, 그렇지 않다. `@DynamicUpdate`는 **변경된 컬럼만 UPDATE 문에 포함**시키는 옵션이다. `기존값 → null`도 엄연한 변경이므로 그대로 UPDATE에 실린다.

```
전송 안 함 → DTO 필드 null → 엔티티에 null 대입 → "변경됨" 판정 → UPDATE SET col = NULL
```

## 해결 방법

근본 원인은 "전체 필드 대입 DTO를 부분 수정에 재사용한 것"이었다. 그래서 **부분 수정 전용 요청 모델**을 따로 만들고, 엔티티도 그 항목만 갱신하도록 분리했다.

```java
// 화면에서 편집하는 항목만 담는다. 나머지는 아예 존재하지 않는다.
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ProfileEditRequest {

    @NotBlank(message = "사용자 ID는 필수입니다.")
    private String userId;

    @Pattern(regexp = "^(?:\\d{9,11})?$", message = "전화번호는 9~11자리 숫자만 입력할 수 있습니다.")
    private String phone;

    @Email @Size(max = 30)
    private String email;

    @Size(max = 50)
    private String address;
}
```

```java
// 엔티티도 편집 대상만 갱신 — 화면에 없는 항목은 손대지 않는다
public void applyEdit(ProfileEditRequest request) {
    this.phone = request.getPhone();
    this.email = request.getEmail();
    this.address = request.getAddress();
}
```

기존 API는 이 새 모델을 받도록 바꾸고, 전체 필드를 대입하던 `merge()`와 그 DTO는 제거했다.

### 부수 효과: 서버 검증도 일관성을 얻었다

기존 전체 대입 DTO에는 전화번호·우편번호 같은 형식 검증이 없어, 프론트 검증만 통과하면 서버는 무엇이든 받아들이는 상태였다. 부분 수정 모델을 새로 만들면서 같은 항목을 다루는 다른 화면과 **동일한 검증 애노테이션**을 붙일 수 있었다.

### 다른 선택지

상황에 따라 이런 방법도 가능하다.

| 방법 | 설명 | 주의점 |
|---|---|---|
| 부분 수정 전용 DTO | 이번에 택한 방법. 의도가 코드에 드러남 | DTO가 하나 늘어남 |
| `null`이면 대입 생략 | `if (v != null) this.x = v;` | "값을 비우는 수정"이 불가능해짐 |
| `Optional<T>` 필드 | 미전송/명시적 null 구분 가능 | DTO 필드로 `Optional`은 비권장 |
| `@JsonInclude` + 패치 맵 | JSON Merge Patch 방식 | 구현 복잡도 상승 |

"값을 비우는 것"과 "전송하지 않는 것"을 구분해야 하는지가 선택 기준이다. 이번 화면은 입력을 지우면 DB도 비워지는 것이 맞는 동작이라 부분 수정 DTO로 충분했다.

## 결과

- 화면에 없는 항목이 저장 시 `NULL`로 덮어써지지 않음
- 관리자 화면과 사용자 본인 화면이 같은 갱신 경로를 공유하게 되어 동작이 일치
- 서버 측 형식 검증이 두 화면에 동일하게 적용

## 정리

> `@Builder.Default`는 **빌더 경로 전용**이다. Jackson 역직렬화는 기본 생성자를 쓰므로 기본값이 적용되지 않고 `null`이 된다. 여기에 "전체 필드 무조건 대입" 병합이 겹치면 조용히 데이터가 사라진다. 부분 수정 API에는 **부분 수정 전용 요청 모델**을 쓰는 것이 가장 안전하다.
