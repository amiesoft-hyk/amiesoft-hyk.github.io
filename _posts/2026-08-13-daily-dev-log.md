---
layout: post
title: "일일 개발 로그 (2026-08-13)"
date: 2026-08-13 18:00:00 +0900
categories: [개발작업, 일일로그]
tags: [일일로그, SpringBoot, QueryDSL, Vue, ExcelJS]
---

## 오늘 한 일

### 백엔드 API

- **회원 등록 시 선택한 권한이 저장되지 않던 문제 수정** (완료)
  - 등록 서비스가 전달받은 권한 값을 확인하지 않고 부서 기반 기본값으로 무조건
    덮어쓰고 있었다. 선택값이 있으면 그대로 쓰고 미선택일 때만 기본값을 부여하도록
    조건부로 변경
  - 상세: [수정은 되는데 신규 저장만 안 된다]({{ site.baseurl }}{% post_url 2026-08-13-service-overwrites-user-selected-value %})

- **스킬 현황 조회 시 `SemanticException` 수정** (완료)
  - 목록 쿼리와 count 쿼리가 같은 조건 빌더를 공유하는데 count 쿼리에만 join이 빠져
    있어, 조인 대상 엔티티를 참조하는 조건이 들어오면 조회 자체가 실패했다
  - 상세: [QueryDSL 목록 쿼리와 count 쿼리의 join 불일치]({{ site.baseurl }}{% post_url 2026-08-13-querydsl-count-query-join-mismatch %})

- **리포트 부서 매칭 기준 변경** (완료)
  - 부서 식별을 경로 문자열에서 부서 ID로 변경. 경로 문자열이 미세하게 달라지면
    대상자가 집계에서 빠지는 문제가 있었다
  - 부서별 평균 점수 목록을 내림차순 정렬해 반환하도록 함께 수정

### 프론트엔드

- **엑셀 다운로드에 필터·정렬(autoFilter) 추가** (완료 — 4개 화면)
  - ExcelJS 직접 작성 방식과 `exportDataGrid()` 방식이 섞여 있어 설정 지점이 달랐다
  - 날짜 컬럼을 표시 형식으로 통일해 엑셀 정렬이 어긋나지 않게 처리
  - 상세: [ExcelJS 직접 작성과 exportDataGrid 두 경로]({{ site.baseurl }}{% post_url 2026-08-13-excel-autofilter-two-export-paths %})

- **좌우 분할 화면 레이아웃 검토** (롤백)
  - 폼 패널이 열릴 때 그리드가 밀리는 현상을 오버레이 방식(`position: absolute`)으로
    바꿔봤으나, 그리드를 가리는 동작이 맞지 않아 기존 좌우 분할 방식으로 되돌림
  - 검토 과정에서 확인한 것: flex 컨테이너 안에서 `float` + `width %`만 지정하면
    flex 아이템의 기본 `min-width: auto` 때문에 콘텐츠 최소 폭만큼 패널이 부풀어
    레이아웃이 넘친다. 같은 구조를 쓰는 다른 화면은 `flex-basis`와 `min-width: 0`을
    명시해 해결해 두었다

## 이슈 / 특이사항

### 편집 도구가 파일 전체 줄바꿈을 바꿔버린 건

2줄만 추가한 커밋인데 `git diff`가 **2294줄 변경**으로 잡혔다. 원인은 줄바꿈이었다.

```text
warning: in the working copy of 'src/pages/.../index.vue',
         LF will be replaced by CRLF the next time Git touches it
```

저장소의 blob은 LF인데 편집 과정에서 파일 전체가 CRLF로 저장되면서, 실질 변경은 2줄인데
모든 줄이 변경으로 기록된 것이다. `core.autocrlf=true`이고 `.gitattributes`가 없는
환경이라 도구마다 다르게 동작할 여지가 있었다.

해결 순서는 이랬다.

1. 필터를 우회해 저장소 원본 그대로 복원
   ```bash
   git -c core.autocrlf=false checkout -- <파일>
   ```
2. 줄바꿈을 건드리지 않는 방법으로 해당 줄만 삽입 (Node 스크립트로 `split('\n')` →
   `splice` → `join('\n')`)
3. 다시 스테이징해 변경량 확인 — `1 file changed, 2 insertions(+)`

`sed -i`도 시도했지만 이 환경(Git Bash)에서는 결과 파일이 다시 CRLF가 되어 소용없었다.
줄바꿈이 섞이면 안 되는 파일을 다룰 때는 **스테이징 후 `git diff --cached --stat`으로
변경량을 먼저 확인**하는 습관이 확실히 도움이 된다.

### 커밋에 다른 파일이 딸려 들어간 건

앞선 시도에서 스테이징해둔 파일이 인덱스에 남아 있어, 커밋 메시지와 맞지 않는 파일이
함께 들어갔다. push 전이라 아래로 분리했다.

```bash
git reset --soft HEAD~1
```

`--soft`는 커밋만 해제하고 인덱스·작업 트리는 그대로 두기 때문에, 원하는 파일만 남겨
다시 커밋하면 된다. 커밋 전에 `git diff --cached --stat`으로 **대상 파일 목록을 한 번
확인**하는 것이 결국 가장 싸게 먹힌다.
