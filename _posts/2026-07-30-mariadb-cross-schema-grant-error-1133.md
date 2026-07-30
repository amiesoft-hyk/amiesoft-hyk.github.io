---
layout: post
title: "MariaDB 크로스 스키마 조회 권한과 ERROR 1133 Can't find any matching row in the user table"
date: 2026-07-30 17:40:00 +0900
categories: [오류해결, MariaDB]
tags: [MariaDB, MySQL, JPA, 권한]
---

## 문제 상황

한 애플리케이션에서 **다른 스키마의 테이블**을 조회하는 기능을 추가했다. 같은 DB 인스턴스에 스키마만 다른 구조라, 애플리케이션 계정 하나로 `other_schema.batch_history`를 읽는 방식이다.

로컬에서 SQL 클라이언트로 쿼리를 돌렸을 때는 잘 나왔는데, 애플리케이션에서 호출하니 실패했다.

```
SELECT command denied to user 'app_user'@'192.0.2.10' for table `other_schema`.`batch_history`
```

```
org.hibernate.exception.SQLGrammarException: JDBC exception executing SQL
[SELECT target_date, ended_at, started_at FROM OTHER_SCHEMA.batch_history
 WHERE job_name = ? AND status = ? ORDER BY started_at DESC LIMIT 1]
```

같은 스키마의 **다른 테이블은 이미 잘 조회되고 있었다.** 코드 경로도, 커넥션도, 계정도 동일했다.

## 원인 1: 테이블 단위 권한 부여

`SQLGrammarException`이라는 예외 타입 때문에 SQL 문법 문제로 오해하기 쉽지만, 원인 메시지는 명확히 `SELECT command denied`다. **권한 문제**다.

확인해 보니 이 환경은 스키마 전체(`other_schema.*`)가 아니라 **테이블 단위로 권한을 부여**하고 있었다. 기존에 쓰던 테이블 2개에만 `SELECT`가 들어가 있었고, 새로 참조한 테이블은 빠져 있었다.

여기서 얻은 진단 요령이 하나 있다.

> 같은 스키마의 다른 테이블은 되는데 특정 테이블만 거부된다면, 연결·설정 문제가 아니라 **테이블 단위 GRANT 누락**을 먼저 의심한다.

애플리케이션은 하나의 DataSource로 동작했고, 스키마명은 설정값으로 주입해 테이블명을 조립하는 방식이었다. 즉 접근 경로가 완전히 동일했으므로 코드 쪽에서 우회할 여지가 없었다.

```java
private String batchHistoryTableName() {
    String schemaName = properties.getOtherSchemaName();
    // 설정값이 그대로 SQL 에 들어가므로 화이트리스트 검증은 필수
    if (schemaName == null || !schemaName.matches("[A-Za-z0-9_]+")) {
        throw new IllegalStateException("Invalid schema name: " + schemaName);
    }
    return schemaName + ".batch_history";
}
```

## 원인 2: GRANT 대상 계정을 잘못 지정 (ERROR 1133)

오류 메시지에 나온 그대로 권한을 주려고 하니 이번엔 다른 오류가 났다.

```sql
grant select on `other_schema`.`batch_history` to 'app_user'@'192.0.2.10';
```

```
SQL Error [1133] [28000]: Can't find any matching row in the user table
```

`1133`은 **지정한 계정 행이 `mysql.user`에 없다**는 뜻이다.

핵심은 이것이다.

> MariaDB의 권한 거부 메시지는 `계정명@접속한_클라이언트_IP` 형식으로 출력되지만, `mysql.user`에 저장된 계정 행의 `host`는 `%`나 `10.0.0.%` 같은 **와일드카드일 수 있다.**

즉 `192.0.2.10`은 애플리케이션 서버가 접속해 온 실제 IP일 뿐, 계정을 식별하는 값이 아니다. GRANT는 **저장된 계정 행과 정확히 일치하는 `user@host`** 를 요구한다.

| 값 | 의미 |
|---|---|
| 오류 메시지의 `@` 뒤 | 클라이언트가 접속해 온 IP |
| `mysql.user`의 `host` | 계정 정의에 등록된 패턴 (`%` 등 와일드카드 가능) |
| GRANT에 써야 하는 값 | **후자** |

## 해결 방법

### 1단계 — 계정 행의 실제 host 확인

```sql
select `user`, `host` from `mysql`.`user` where `user` = 'app_user';
```

결과가 비어 있다면 계정명 표기(대소문자 포함)가 다른 경우이므로 범위를 넓혀 확인한다.

```sql
select `user`, `host` from `mysql`.`user` where lower(`user`) like '%app%';
```

애플리케이션 커넥션에서 직접 확인할 수 있다면 이 두 함수의 차이가 결정적이다.

```sql
select user(), current_user();
```

| 함수 | 반환값 |
|---|---|
| `user()` | 접속 시 사용한 계정 + **클라이언트 호스트** |
| `current_user()` | 실제로 매칭된 **`mysql.user` 행** ← GRANT 대상 |

### 2단계 — 조회된 host로 GRANT

계정이 `'app_user'@'%'`로 등록되어 있었다면 이렇게 준다.

```sql
grant select on `other_schema`.`batch_history` to 'app_user'@'%';
flush privileges;
```

### 3단계 — 확인

```sql
show grants for 'app_user'@'%';
```

이미 동작하던 테이블들의 GRANT가 같은 목록에 보이므로, **그 항목들과 동일한 host 표기를 쓰면 확실하다.**

## 부가 조치: 부가 기능이 본 기능을 막지 않게

이 조회는 화면 상단에 참고용으로 표시되는 부가 정보였다. 권한이 부여되기 전이나, 환경에 따라 두 스키마가 아예 다른 DB 인스턴스에 있는 경우에도 **목록 화면 자체는 떠야 한다.**

그래서 서비스 계층에서 데이터 접근 예외를 흡수하도록 했다.

```java
public LastRunInfo getLastRun() {
    try {
        return repository.findLastRun(JOB_NAME, STATUS_SUCCESS)
            .orElseGet(LastRunInfo::empty);
    } catch (DataAccessException e) {
        log.warn("배치 이력 조회 실패. 스키마 접근 권한/연결을 확인하세요. job={}", JOB_NAME, e);
        return LastRunInfo.empty();
    }
}
```

화면에는 값 대신 `-`가 표시되고, 원인은 서버 로그에 `WARN`으로 남는다. 다만 이렇게 삼키면 설정 문제를 영영 못 볼 수 있으므로, **로그를 남기는 것과 부가 기능에만 적용하는 것**이 전제다.

## 결과

- 누락된 테이블 GRANT를 추가해 정상 조회
- `1133` 오류는 계정 행의 실제 `host`를 확인해 해소
- 권한 문제가 있어도 본 기능(목록 조회)은 영향받지 않도록 방어

## 정리

> `SELECT command denied` 메시지의 `@` 뒤 값은 **클라이언트 IP**일 뿐 계정 식별자가 아니다. GRANT는 `mysql.user`에 저장된 `user@host`와 정확히 일치해야 하며, 실제 매칭된 계정은 `current_user()`로 확인할 수 있다. 그리고 같은 스키마의 다른 테이블이 잘 된다면 연결이 아니라 **테이블 단위 권한 누락**을 먼저 의심하자.
