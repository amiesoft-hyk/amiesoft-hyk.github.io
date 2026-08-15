---
layout: post
title: "집계 인라인 뷰 목록 조회가 느릴 때 — 대상자 push-down과 불필요한 COUNT 집계 제거"
date: 2026-08-15 18:40:00 +0900
categories: [오류해결, SQL]
tags: [SQL, MariaDB, Oracle, Hibernate, JPA, 성능튜닝]
---

30분 단위 통계 테이블을 일자·담당자별로 합산해 보여주는 목록 화면이 있었다.
조회 기간을 늘리면 급격히 느려지고, 필터를 좁혀도 빨라지지 않고, 페이지를
넘길 때마다 첫 조회와 똑같이 기다려야 했다. 원인은 하나가 아니라 네 가지가
겹쳐 있었다.

## 문제 상황

화면은 다음과 같은 구조로 조회한다.

- 원천: 30분 단위 통계 테이블(하루 담당자 1명당 48행)
- 표시: 일자 × 담당자로 묶어 특정 시간대만 합산한 값
- 그리드: 서버 페이징 100건, 컬럼 필터·정렬 지원, 엑셀 다운로드

증상은 세 가지였다.

1. 조회 기간을 1일 → 45일로 늘리면 대기 시간이 거의 선형으로 늘어남
2. 사번 하나로 필터를 걸어도 조회 시간이 줄지 않음
3. 2페이지, 3페이지로 넘길 때마다 첫 조회와 동일하게 느림

## 원인

### 1. 파생 테이블이 조인·필터보다 먼저 전량 집계된다

기존 쿼리는 이런 모양이었다.

```sql
SELECT s.ymd, u.login_id, u.user_nm,
       COALESCE(s.type_a_sec, 0) AS type_a_sec,
       COALESCE(s.type_b_sec, 0) AS type_b_sec
FROM  (SELECT i.ymd, i.agent_id,
              SUM(CASE WHEN i.hh24mi BETWEEN '0800' AND '1700'
                       THEN i.type_a_sec ELSE 0 END) AS type_a_sec,
              SUM(CASE WHEN i.hh24mi BETWEEN '0800' AND '1700'
                       THEN i.type_b_sec ELSE 0 END) AS type_b_sec
       FROM   stat_30m i
       WHERE  i.ymd BETWEEN :fromYmd AND :toYmd
       GROUP  BY i.ymd, i.agent_id) s
INNER JOIN user_master u ON u.ext_id = s.agent_id
WHERE  s.ymd BETWEEN :fromYmd AND :toYmd
  AND  u.del_yn = 'N'
  AND  u.ext_id IS NOT NULL
  AND  u.dept_id IN (:allowedDeptIds)
ORDER  BY s.ymd, s.agent_id
```

파생 테이블(인라인 뷰)은 **바깥 조인·필터와 무관하게 먼저 만들어진다.**
`user_master` 조인도, 권한 부서 조건도, 사번 필터도 전부 집계가 끝난 뒤에
적용된다. 그래서 담당자 1명만 조회해도 집계 대상 행 수는 전체와 같다.

읽는 양은 대략 `담당자 수 × 조회일수 × 48`. 45일이면 1일 대비 45배다.

### 2. COUNT 쿼리가 쓰지도 않는 SUM을 계산한다

서버 페이징이라 총 건수 쿼리가 별도로 나간다. 그런데 그 쿼리가 목록 쿼리와
**같은 파생 테이블**을 그대로 쓰고 있었다.

```sql
SELECT COUNT(*)
FROM  (SELECT i.ymd, i.agent_id,
              SUM(CASE WHEN ... THEN i.type_a_sec ELSE 0 END) AS type_a_sec,  -- 안 씀
              SUM(CASE WHEN ... THEN i.type_b_sec ELSE 0 END) AS type_b_sec   -- 안 씀
       FROM   stat_30m i
       WHERE  i.ymd BETWEEN :fromYmd AND :toYmd
       GROUP  BY i.ymd, i.agent_id) s
INNER JOIN user_master u ON u.ext_id = s.agent_id
WHERE  ...
```

COUNT에 필요한 건 `(ymd, agent_id)` 그룹의 **개수**뿐이다. 합산값은 어디에서도
참조하지 않는다. 그런데 `hh24mi`, `type_a_sec`, `type_b_sec` 세 컬럼을 전부 읽고
CASE 평가와 SUM까지 수행하고 있었다.

### 3. 리팩터링 잔재로 남은 중복 술어

`s.ymd BETWEEN :fromYmd AND :toYmd` 가 파생 테이블 안팎에 **두 번** 있었다.

이력을 보니 원인이 명확했다. 원래 `s` 는 파생 테이블이 아니라 일별 누적
테이블이었고, 그때는 바깥 `WHERE` 가 유일한 기간 제한이었다. 이후 30분 단위
집계로 원천을 바꾸면서 파생 테이블 안에 조건을 넣었는데(안 넣으면 전체 기간을
집계한다) **바깥 조건을 지우지 않아** 남은 것이다.

`AND u.ext_id IS NOT NULL` 도 마찬가지다. `INNER JOIN u.ext_id = s.agent_id` 가
이미 NULL을 배제하므로 아무 일도 하지 않는 조건이었다.

### 4. NULL 보정 COALESCE가 필터·정렬 표현식마다 복제된다

집계 컬럼을 참조하는 표현식이 코드에서 이렇게 만들어지고 있었다.

```java
private static String sumColumn(String column) {
    return "COALESCE(s." + column + ", 0)";
}
```

이 표현식은 SELECT 뿐 아니라 **필터·정렬 SQL을 조립할 때도 그대로 쓰인다.**
합계 컬럼으로 정렬하면 이렇게 나간다.

```sql
ORDER BY COALESCE(s.type_a_sec, 0) + COALESCE(s.type_b_sec, 0) DESC, s.ymd, s.agent_id
```

시간 문자열(`HH:MM:SS`) 형태로 LIKE 필터를 걸면 더 심해진다.

```sql
AND CONCAT(CASE WHEN FLOOR((COALESCE(s.type_a_sec, 0)) / 3600) < 10 THEN '0' ELSE '' END,
    CAST(FLOOR((COALESCE(s.type_a_sec, 0)) / 3600) AS CHAR), ':',
    LPAD(CAST(FLOOR(MOD((COALESCE(s.type_a_sec, 0)), 3600) / 60) AS CHAR), 2, '0'), ':',
    ...) LIKE :typeATime
```

### 5. 페이지를 넘길 때마다 COUNT가 다시 실행된다

서버는 `includeCount` 파라미터가 없으면 기본값을 `true` 로 해석했고, 프론트는
페이지 이동 요청에서 이 값을 아예 보내지 않았다. 결과적으로 **2페이지, 3페이지로
넘길 때마다 위의 무거운 COUNT가 다시** 돌았다.

## 해결 방법

### 1. EXISTS로 대상자를 파생 테이블 안으로 밀어 넣기

바깥 `INNER JOIN` + `del_yn` + 권한 부서 조건과 **의미가 완전히 같은** 술어를
파생 테이블 안에 넣는다. 결과 집합이 바뀌지 않는 등가 재작성이다.

```sql
FROM (SELECT i.ymd, i.agent_id,
             COALESCE(SUM(CASE WHEN i.hh24mi BETWEEN :startHhmi AND :endHhmi
                               THEN i.type_a_sec ELSE 0 END), 0) AS type_a_sec,
             COALESCE(SUM(CASE WHEN i.hh24mi BETWEEN :startHhmi AND :endHhmi
                               THEN i.type_b_sec ELSE 0 END), 0) AS type_b_sec
      FROM   stat_30m i
      WHERE  i.ymd BETWEEN :fromYmd AND :toYmd
        AND  EXISTS (SELECT 1
                     FROM   user_master u2
                     WHERE  u2.ext_id  = i.agent_id
                       AND  u2.del_yn  = 'N'
                       AND  u2.dept_id IN (:allowedDeptIds))
      GROUP  BY i.ymd, i.agent_id) s
```

퇴사자·미매핑 담당자의 통계가 `GROUP BY` **이전에** 걸러진다. 권한 범위가 좁은
계정일수록 효과가 크다.

권한 조건은 있을 때만 붙여야 하므로 술어를 동적으로 만든다.

```java
private String targetPredicate(Collection<Long> allowedDeptIds) {
    String authPredicate = (allowedDeptIds == null || allowedDeptIds.isEmpty())
        ? ""
        : " AND u2.dept_id IN (:allowedDeptIds)";
    return "AND EXISTS (SELECT 1 FROM user_master u2"
        + " WHERE u2.ext_id = i.agent_id AND u2.del_yn = 'N'" + authPredicate + ")";
}
```

### 2. COUNT 전용 파생 테이블 분리

집계값을 참조하는 필터가 없으면 SUM을 만들지 않는다.

```java
/** 건수 조회용 인라인 뷰 선택 */
private String countSourceSql(Collection<Long> allowedDeptIds, SearchCondition search) {
    return hasValueFilter(search)
        ? aggregateSourceSql(allowedDeptIds)   // 기존: SUM 포함
        : keyOnlySourceSql(allowedDeptIds);    // 신규: 키만
}

private boolean hasValueFilter(SearchCondition search) {
    return search != null && (
        hasText(search.getTypeATime())  || hasText(search.getTypeATimeList())
            || hasText(search.getTypeBTime())  || hasText(search.getTypeBTimeList())
            || hasText(search.getTotalTime())  || hasText(search.getTotalTimeList())
    );
}
```

```sql
-- keyOnlySourceSql
(SELECT i.ymd, i.agent_id
 FROM   stat_30m i
 WHERE  i.ymd BETWEEN :fromYmd AND :toYmd
   AND  EXISTS (...)
 GROUP  BY i.ymd, i.agent_id)
```

**여기에 함정이 하나 있다.** SQL에서 `:startHhmi` 가 사라졌는데 바인딩 코드는
그대로면 JPA가 `Could not locate named parameter` 로 터진다. 파라미터 바인딩도
조건부로 바꿔야 한다.

```java
Query query = em.createNativeQuery(sql)
    .setParameter("fromYmd", from)
    .setParameter("toYmd", to);
// 건수 조회는 집계를 생략한 인라인 뷰를 쓰므로 시간 파라미터가 SQL 에 없을 수 있다
if (sql.contains(":startHhmi")) {
    query.setParameter("startHhmi", START_HHMI)
         .setParameter("endHhmi", END_HHMI);
}
```

### 3. 중복 술어 제거

```diff
-WHERE  s.ymd BETWEEN :fromYmd AND :toYmd
-  AND  u.del_yn = 'N'
-  AND  u.ext_id IS NOT NULL
+WHERE  u.del_yn = 'N'
```

두 조건 모두 **다른 조건이 이미 보장하던 중복**이라 결과에 영향이 없다.

### 4. COALESCE를 파생 테이블 안으로 이동

여기서 주의할 점: **그냥 지우면 안 된다.**

`SUM(CASE WHEN cond THEN col ELSE 0 END)` 는 보통 NULL이 될 수 없지만, 그룹의
모든 행이 `cond = true` 이면서 `col` 이 NULL이면 SUM이 NULL을 반환한다. 보정 자체는
필요하다. **위치만** 옮기는 것이 정답이다.

```diff
-// 파생 테이블
-SUM(CASE WHEN ... THEN i.type_a_sec ELSE 0 END) AS type_a_sec
-// 바깥 참조
-COALESCE(s.type_a_sec, 0)
+// 파생 테이블 — NULL 보정을 한 곳으로
+COALESCE(SUM(CASE WHEN ... THEN i.type_a_sec ELSE 0 END), 0) AS type_a_sec
+// 바깥 참조 — 컬럼 참조만 남는다
+s.type_a_sec
```

```java
private static String sumColumn(String column) {
    return "s." + column;   // COALESCE 제거
}
```

정렬 표현식이 이렇게 짧아진다.

```sql
ORDER BY s.type_a_sec + s.type_b_sec DESC, s.ymd, s.agent_id
```

### 5. 총 건수 재조회 생략

페이지·정렬을 뺀 조회 조건만으로 시그니처를 만들고, 직전과 같으면 서버 COUNT를
건너뛰고 캐시된 값을 쓴다. 총 건수는 정렬과 무관하므로 정렬도 시그니처에서 뺀다.

```js
// 총건수는 페이지·정렬과 무관하므로 그 외 조건만으로 키를 만든다
const buildCountSignature = params => {
  const { currentpage, pagesize, sort, ...rest } = params;
  return JSON.stringify({ ...rest, fromYmd: search.from, toYmd: search.to });
};

// CustomStore load
const signature = buildCountSignature(params);
const reuseCount = lastCountSignature.value === signature;

const res = await callApi({
  actionName: 'GET_DAILY_SUMMARY',
  data: reuseCount ? { ...params, includeCount: false } : params,
});
if (!isSuccess(res)) {
  lastCountSignature.value = null;      // 실패 시 캐시 무효화
  return { data: [], totalCount: 0 };
}

const totalCount = reuseCount ? cachedTotalCount.value : getTotalCount(res);
lastCountSignature.value = signature;
setTotalCount(totalCount);
return { data, totalCount };
```

조회 버튼을 다시 누를 때는 시그니처를 초기화해 총 건수를 새로 계산한다.

## 결과

호출·연산 구조가 이렇게 바뀐다.

| 동작 | 이전 | 이후 |
|---|---|---|
| 최초 조회 | COUNT + 목록 | COUNT + 목록 (동일) |
| 페이지 이동 | COUNT + 목록 | **목록만** |
| 정렬 변경 | COUNT + 목록 | **목록만** |
| COUNT 쿼리 | 구간·값 컬럼 3개 읽고 SUM | **키 2개만 읽음** |
| 집계 대상 | 전 담당자 | **권한·재직 대상자만** |

조회 결과와 건수는 이전과 완전히 동일하다. `EXISTS` 는 바깥 `INNER JOIN` 조건과
같은 집합을 만들고, 삭제한 술어는 다른 조건이 이미 보장하던 중복이며, COALESCE는
위치만 옮겼기 때문이다.

다만 **실제 소요 시간은 실행계획으로 확인해야 한다.** 구조를 바꿨다고 옵티마이저가
반드시 더 나은 계획을 고르는 것은 아니므로, 개선 전후로 `EXPLAIN` 을 떠서 파생
테이블 접근이 풀 스캔에서 인덱스 스캔으로 바뀌었는지, 그리고 **결과 건수가 같은지**
반드시 대조한다.

## 인덱스

구조 변경만으로는 부족하고, 커버링 인덱스가 있어야 파생 테이블 집계가 테이블
액세스 없이 끝난다.

```sql
-- WHERE ymd 범위 → GROUP BY (ymd, agent_id) 순서를 인덱스가 그대로 제공하고,
-- 나머지 컬럼까지 포함해 테이블 랜덤 액세스를 없앤다
CREATE INDEX ix_stat_30m_ymd_agent
    ON stat_30m (ymd, agent_id, hh24mi, type_a_sec, type_b_sec);

-- EXISTS push-down / INNER JOIN 용
CREATE INDEX ix_user_master_ext
    ON user_master (ext_id, del_yn, dept_id);
```

`ymd` 가 범위 조건이지만 `GROUP BY` 의 선두 컬럼과 같아 인덱스 정렬 순서를 그대로
쓸 수 있다.

**반영 전 확인할 것**

- 조인 양쪽 컬럼(`stat_30m.agent_id`, `user_master.ext_id`)의 **타입과 collation이
  일치**하는지. 다르면 암묵 형변환으로 인덱스를 쓰지 못한다. 이 경우 인덱스보다
  컬럼 정의 정정이 우선이다.
- `hh24mi` 가 문자형인지. 숫자형인데 `'0800'` 과 비교하면 역시 형변환이 일어난다.
- 이미 `(ymd, agent_id)` 로 시작하는 인덱스가 있으면 새로 만들지 말고 **기존
  인덱스에 컬럼을 추가**하는 편이 낫다.

## 건드리지 말아야 했던 것

시간대 조건이 `WHERE` 가 아니라 `CASE WHEN` 안에 있는 것이 비효율처럼 보인다.
`WHERE i.hh24mi BETWEEN ...` 로 옮기면 스캔량이 확 줄기 때문이다.

하지만 그러면 **해당 시간대 밖에만 활동한 담당자의 행이 목록에서 통째로
사라진다.** 그 사람은 0으로 표시되어야 하는 것이지 안 보여야 하는 게 아니다.
요구사항이 걸린 부분이라 구조를 유지하고, 대신 커버링 인덱스로 해결했다.

성능 개선에서 제일 위험한 건 "빨라 보이는데 결과가 달라지는" 변경이다.
등가 변환인지 아닌지를 먼저 구분하고 손대야 한다.

## 정리 — 파생 테이블 집계 쿼리 점검 목록

- [ ] 파생 테이블 안에서 대상을 좁힐 수 있는가? (바깥 조인·필터와 등가인 `EXISTS`)
- [ ] COUNT 쿼리가 목록 쿼리와 같은 집계를 쓰고 있지 않은가? 그 집계값을 실제로 참조하는가?
- [ ] 파생 테이블 안팎에 같은 조건이 중복돼 있지 않은가? (원천을 바꾼 리팩터링 흔적)
- [ ] `INNER JOIN` 이 이미 보장하는 `IS NOT NULL` 같은 무의미한 술어가 남아 있지 않은가?
- [ ] NULL 보정이 필터·정렬 표현식마다 복제되고 있지 않은가? 파생 테이블 안으로 옮길 수 있는가?
- [ ] 페이지 이동마다 총 건수를 다시 계산하고 있지 않은가?
- [ ] SQL을 동적으로 바꿨다면 **파라미터 바인딩도 같이** 조건부로 바꿨는가?
- [ ] 조회 기간에 상한이 있는가? (기간이 길어질수록 비용이 선형으로 늘어난다면)
