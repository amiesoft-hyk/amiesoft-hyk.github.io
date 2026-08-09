---
layout: post
title: "합계가 24:00:00으로 딱 떨어진다 — 화면에서 원본 테이블까지 거슬러 올라가기"
date: 2026-08-07 17:40:00 +0900
categories: [오류해결, 데이터분석]
tags: [SQL, 데이터검증, 디버깅, 통계배치]
---

## 문제 상황

리포트 화면에 "시간이 과다하게 집계된다"는 확인 요청이 들어왔다.

두 달치를 조회하니 이런 행들이 섞여 있었다.

| 교육시간 | 트레이닝시간 | 합계 |
|---|---|---|
| `00:00:00` | `24:00:00` | `24:00:00` |
| `00:00:00` | `24:00:00` | `24:00:00` |
| `00:00:00` | `21:21:29` | `21:21:29` |

하루 종일 교육만 받은 사람은 없을 테니 명백히 이상하다. 게다가 **`24:00:00`이 딱 떨어진다**는 게 걸렸다. 실제 측정값이라면 이렇게 정확히 나올 리가 없다.

## 추적 과정

### 1. 계산하는 지점을 찾는다

이상한 숫자를 만나면 먼저 "누가 이 값을 만들었는가"를 찾아야 한다. 화면 컬럼명부터 원본까지 한 단계씩 내려갔다.

**화면** — 컬럼의 `dataField`를 확인한다.

```js
{ caption: '교육시간',    dataField: 'groupTrainingTime' },
{ caption: '트레이닝시간', dataField: 'trainingTime' },
{ caption: '합계',        dataField: 'totalTrainingTime' },
```

**API DTO** — 같은 필드명으로 백엔드를 검색한다.

```java
public void setTrainingTime(Integer groupSec, Integer trainingSec) {
    int group = defaultZero(groupSec);
    int training = defaultZero(trainingSec);
    this.groupTrainingTime = formatSeconds(group);
    this.trainingTime      = formatSeconds(training);
    this.totalTrainingTime = formatSeconds(group + training);
}

private static String formatSeconds(int seconds) {
    int hours   = seconds / 3600;
    int minutes = (seconds % 3600) / 60;
    int remain  = seconds % 60;
    return String.format("%02d:%02d:%02d", hours, minutes, remain);
}
```

여기서 하는 일은 **초를 `hh:mm:ss` 문자열로 나누는 것뿐**이다. 상한 처리도, 보정도 없다.

**리포지토리 SQL** — 그럼 이 초는 어디서 오는가.

```sql
SELECT
    d.ymd                          AS ymd,
    u.agt_id                       AS agt_id,
    COALESCE(d.away_reason4_sec, 0) AS group_training_sec,
    COALESCE(d.away_reason5_sec, 0) AS training_sec
FROM stats_schema.agent_daily d
INNER JOIN hr_user_master u ON u.pbx_id = d.agt_no
WHERE d.ymd BETWEEN :startYmd AND :endYmd
```

`COALESCE`로 NULL만 0으로 바꿔서 **그대로 SELECT**한다. 집계도, 가공도 없다.

### 2. 계산이 없다면 원본이다

여기서 결론이 하나로 좁혀진다. 화면·API·SQL 어디에도 값을 키우는 로직이 없으므로, **원본 테이블에 그 값이 그대로 들어있다**는 뜻이다.

`24:00:00`은 86400초다. 확인은 쿼리 한 줄이면 된다.

```sql
SELECT ymd, agt_no, away_reason4_sec, away_reason5_sec
FROM stats_schema.agent_daily
WHERE ymd BETWEEN '20260601' AND '20260731'
  AND (away_reason4_sec >= 86400 OR away_reason5_sec >= 86400)
ORDER BY away_reason5_sec DESC;
```

결과는 예상대로였다. `away_reason5_sec`에 **정확히 `86400`**, 같은 행의 `away_reason4_sec`는 전부 `0`.

코드는 무죄였다.

### 3. 패턴을 찾는다

원본 문제로 확정됐다고 끝이 아니다. "왜 그런 값이 들어갔는가"까지 좁혀야 담당자에게 넘길 수 있다.

결과 행의 날짜를 보다가 규칙이 눈에 들어왔다. 06-06, 06-07, 06-13, 06-14, 06-20, 06-21. **전부 토·일**이었다.

가설을 세웠으면 검증한다.

```sql
SELECT ymd, DAYOFWEEK(ymd) AS dw, COUNT(*) AS cnt
FROM stats_schema.agent_daily
WHERE ymd BETWEEN '20260601' AND '20260731'
  AND away_reason5_sec = 86400
GROUP BY ymd
ORDER BY ymd;
```

`dw`가 `1`(일요일)과 `7`(토요일)만 나왔다. 평일은 한 건도 없었다.

## 원인

비근무일에 상담사 상태가 **종일 이석**으로 마감되고 있었다.

금요일 퇴근 시 이석 상태(사유 5번)를 해제하지 않은 채로 두면, 주말 동안 상태 변경 이벤트가 없다. 일별 통계 배치가 "그날 하루 내내 그 상태였다"고 판단해 86400초를 찍는다.

`away_reason4_sec`가 0이라는 점도 이 가설과 맞아떨어진다. 실제 교육 활동은 전혀 없었고, 미해제된 상태값만 하루치로 누적된 것이다.

따라서 이건 리포트 화면이나 조회 API의 버그가 아니라 **통계 배치의 마감 규칙 문제**다.

## 결과

- 조회 기간 두 달 기준 9건, 전부 주말이라는 사실과 함께 통계 담당자에게 이관했다
- 화면·API 코드는 수정하지 않았다. 원본을 그대로 보여주는 게 정확한 동작이고, 화면에서 값을 깎으면 이상 데이터가 가려져 정정 자체가 늦어진다

임시 대응으로 표시 단계에서 막는 방법도 검토했지만 채택하지 않았다.

| 방안 | 판단 |
|---|---|
| 86400 초과분을 잘라내기 | 데이터를 감추게 되어 부적절 |
| 이상값 셀 강조 표시 | 값은 보존하므로 차선책 |
| 배치 마감 규칙 수정 | 근본 해결. 이쪽으로 이관 |

## 교훈

**이상한 숫자를 만나면 계산 지점부터 찾는다.** 화면 → API → 쿼리 → 원본 순으로 한 단계씩 내려가면서 "여기서 값이 바뀌는가"만 확인하면 된다. 어느 단계에도 계산이 없다면 답은 하나, 원본이 그렇다는 뜻이다.

이 과정이 중요한 이유는 **아니라는 걸 증명해야 넘길 수 있기 때문**이다. "우리 코드 문제 아닙니다"만으로는 아무도 움직이지 않는다. 어느 테이블 어느 컬럼에 어떤 값이 몇 건 있고 어떤 패턴인지까지 정리해야 담당자가 바로 확인에 들어갈 수 있다.

그리고 **딱 떨어지는 값은 의심하라.** `86400`, `3600`, `100`, `0` 같은 경계값이 데이터에 나타나면 측정 결과가 아니라 기본값·상한·미처리 상태일 가능성이 높다. 이번에도 "정확히 24시간"이라는 점이 가장 빠른 단서였다.
