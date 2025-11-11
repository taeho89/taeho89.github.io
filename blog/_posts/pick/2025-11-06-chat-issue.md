---
layout: post
title: "기존 chat 스키마의 문제 정의"
date: 2025-11-06 07:45:00 +0900
description: >
  기존 chat 스키마에서의 문제를 정의하고 어떻게 해결했는지 기술합니다.
categories: [pick]
tags: [amplify, graphql]
---

## AppSync + Amplify Gen 2로 채팅 읽음 처리 설계/구현기

### 배경

- 채팅방 입장 시 사용자가 “안 읽었던 메시지들”을 읽음 처리해야 했다.
- 데이터 스택은 Amplify Gen 2(AppSync + DynamoDB, JS Resolver) + Flutter(Amplify API SDK).

---

## 문제 정의

- 입장 시 최신 메시지를 페이지네이션으로 불러와 UI에 표시.
- 내 계정이 아직 읽지 않은 범위까지 “읽음 처리”를 수행하되:
  - 멱등적(여러 번 호출해도 결과가 한 번만 반영)
  - 경쟁 조건(멀티 디바이스/다중 탭)을 잘 처리
  - 쓰기 비용이 폭발하지 않음

---

## 대안 탐색

기존 스키마: 메시지별 ReadBy 개별 생성(클라이언트)

- 장점: 모델이 직관적(누가 어떤 메시지를 읽었는지 바로 조회)
- 단점: 쓰기 폭탄(페이지 당 50건이면 PutItem 50회), 중복/경쟁 조건 처리 난이도↑, 비용/지연↑

변경: “읽음 커서(ReadCursor)”만 서버에 저장

- 핵심: 사용자별 “마지막으로 읽은 시점/메시지ID”만 서버에 저장, unread 계산은 커서 기준으로 처리
- 장점: 멱등/저비용/간단, 실무 메신저 패턴과 일치
- 단점: “메시지별로 누가 읽었는지” 상세 뷰가 필요하면 추가 설계 필요

---

## 최종 설계

### 데이터 모델(Amplify Gen 2 defineData)

- ChatRoom, ChatMessage(이미 존재)
- ReadCursor(신규): roomId + userId 복합 식별자, readAt(또는 lastMessageId)

권장 스키마 개요:

- ReadCursor
  - roomId: ID!
  - userId: ID!
  - readAt: AWSDateTime!
  - identifier: ["roomId", "userId"] (멱등/업서트)
- 권한: ownersDefinedIn 또는 authenticated + 서버측 조건으로 보호

### 커스텀 Mutation: markReadUpTo

- 입력: roomId, upTo(AWSDateTime; 보통 첫 페이지에서 가장 최신 메시지 sentAt 또는 now)
- 로직: 기존 ReadCursor.readAt < upTo일 때만 Update(앞으로만 전진)
- 반환: 업데이트된 커서

---

## 구현

### 1) 스키마 요소(요약)

- ownersDefinedIn("memberIds") 등으로 Room/Message 접근 제어
- ReadCursor 모델 추가(복합 식별자)
- markReadUpTo mutation + JS handler 등록

예시(요지):

```ts
// amplify/data/chat.ts (개요)
export const chatModels = {
  // ChatRoom, ChatMessage 정의 생략...

  ReadCursor: a
    .model({
      roomId: a.id().required(),
      userId: a.id().required(),
      readAt: a.datetime().required(),
    })
    .identifier(["roomId", "userId"])
    .authorization((allow) => [
      allow.authenticated(), // 혹은 ownersDefinedIn("userId") 스타일
    ]),

  markReadUpTo: a
    .mutation()
    .arguments({
      roomId: a.id().required(),
      upTo: a.datetime().required(),
    })
    .returns(a.ref("ReadCursor"))
    .handler(
      a.handler.custom({
        entry: "./chat/markReadUpToHandler.js",
      })
    )
    .authorization((allow) => [allow.authenticated()]),
};
```

주의:

- custom handler에서는 dataSource를 지정하지 않는다(Gen 2의 JS resolver는 operation으로 데이터소스를 암시).
- Gen 2 권한 체계에서 allow.owner() 혼용을 피하고 ownersDefinedIn / authenticated 등 일관성 유지.

### 2) markReadUpTo Handler(JS Resolver)

```js
// amplify/data/chat/markReadUpToHandler.js
import { util } from "@aws-appsync/utils";

export function request(ctx) {
  const { roomId, upTo } = ctx.arguments;
  const userId = ctx.identity.sub; // Cognito sub (또는 적절한 식별자)

  if (!roomId || !upTo || !userId) {
    util.error("Missing required arguments", "BadRequest");
  }

  // 조건식: 기존 readAt < :upTo 일 때만 갱신
  return {
    operation: "UpdateItem",
    key: util.dynamodb.toMapValues({ roomId_userId: `${roomId}#${userId}` }), // 또는 PK/SK 모델링에 맞게
    update: {
      expression: "SET readAt = :upTo",
      expressionValues: util.dynamodb.toMapValues({ ":upTo": upTo }),
    },
    condition: {
      expression: "attribute_not_exists(readAt) OR readAt < :upTo",
      expressionValues: util.dynamodb.toMapValues({ ":upTo": upTo }),
    },
  };
}

export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }
  return ctx.result; // 업데이트된 ReadCursor
}
```

모델 키 모델링은 실제 테이블 스키마에 맞게 조정(PK/SK 분리 시 key에 각각 매핑).

### 3) watchMessage Handler(JS Resolver)

- setSubscriptionFilter로 roomId 일치 필터 적용

요지:

```js
import { util, extensions } from "@aws-appsync/utils";

export function request(ctx) {
  const { roomId } = ctx.arguments;
  if (!roomId) util.error("roomId required", "BadRequest");

  extensions.setSubscriptionFilter(
    util.transform.toSubscriptionFilter({
      filterGroup: [{ filters: [{ fieldName: "roomId", operator: "eq", value: roomId }] }],
    })
  );

  return {};
}

export function response(ctx) {
  if (ctx.error) util.error(ctx.error.message, ctx.error.type);
  return ctx.result;
}
```

### 4) 클라이언트(Flutter) 흐름

- 입장 시:

  1. 첫 페이지 메시지 목록을 list 쿼리로 가져옴(ModelQueries.list) → PaginatedResult에서 items/hasNextResult/requestForNextResult 사용
  2. 첫 페이지의 최상단(가장 최신) 메시지 sentAt을 upTo로 삼아 `markReadUpTo(roomId, upTo)` 호출
  3. 구독 시작(Subscription). 사용자가 하단에 있을 때 새 메시지 수신 시, 잠시 후 `markReadUpTo` 재호출(디바운스)

- 페이지네이션:
  - `requestForNextResult`를 그대로 저장해 다음 페이지를 요청
  - String 토큰이 아닌 “요청 객체” 자체를 보관하는 게 안전하고 편리

---

## 그 밖의 장애/배포 이슈 해결 메모

- Import 실수
  - ❌ `@aws-amplify/utils` → ✅ `@aws-appsync/utils`
- custom handler 설정
  - ❌ `dataSource: a.ref("...")` → ✅ 제거(Gen 2에서 operation으로 암시)
  - 그렇지 않으면 “No value provided for input HTTP label: functionId” 같은 스택 생성 오류
- DynamoDB 조건식
  - ❌ `==` 연산자, expressionNames/Values 누락 → ✅ `=` 사용, 필요한 ExpressionNames/Values 모두 지정
- 권한(Gen 2)
  - `allow.owner()` 혼용으로 배포 실패/권한 불일치 → ✅ `ownersDefinedIn("...")` 또는 `authenticated()`로 일관성
- Subscription vs Sync
  - 실시간은 Subscription이 담당. Sync는 오프라인/초기 대량 동기화 시 쓰는 다른 경로 → Subscription에 Sync 요청 불필요
- TS 설정
  - `allowImportingTsExtensions`는 `noEmit` 또는 `emitDeclarationOnly`와 함께 사용
- 포맷팅
  - TS는 Prettier가 줄바꿈을 가독성 기준으로 강제하는 경우가 있어 한 줄 강제는 `/* prettier-ignore */` 또는 체이닝 분해 리팩터링으로 처리

---

## 테스트 시나리오

- 입장 시:
  - 메시지 첫 페이지 50건 로드 → 최신 메시지 sentAt 기준 `markReadUpTo` 호출
  - DB에서 ReadCursor.readAt이 과거 → upTo로 업데이트
  - 동일 호츨 2회 이상 → 결과 불변(멱등)
- 멀티 디바이스:
  - A·B 동시에 입장. B가 더 멀리 스크롤하여 upTo를 더 미래로 업데이트 → A의 늦은 호출은 조건식에 의해 무시
- 구독:
  - 동일 roomId 구독자만 새 메시지 수신(setSubscriptionFilter 확인)

---

## 성능/비용

- 메시지 수 N과 무관하게 읽음 처리는 O(1) 호출(상수 횟수)
- 개별 ReadBy 생성 대비 쓰기, 네트워크, 레이턴시가 크게 감소
- 대규모 룸에서도 안정적으로 동작

---

## 회고

- “읽음”의 본질은 “커서 업데이트”였다. 메시지별 플래그를 전부 쓰는 접근은 비용과 복잡도만 높였다.
- AppSync Gen 2의 JS Resolver는 작지만 강력하다. 유틸 import, dataSource 설정, 조건식 같은 기본기를 번번이 확인하는 습관이 중요.
- 클라이언트는 “읽음 처리”를 이벤트 기반으로 조심스럽게(하단 유무, 디바운스) 호출하는 것이 UX/비용 모두에 이롭다.

---

이 과정을 통해 “안 읽은 메시지 읽음 처리”를 성능/정합성/유지보수성 모두 만족하는 구조로 안정화했습니다.

---

## 정량화(측정 가이드)

### 핵심 KPI

- 서버 쓰기량
  - DynamoDB WRU(ConsumedWriteCapacityUnits) 총합/세션/일
  - 쓰기 요청 수(WriteRequestCount) 또는 UpdateItem/PutItem 호출 수
  - 조건부 갱신 실패율(ConditionalCheckFailedRequests; 멱등·경쟁조건 지표)
- 지연 시간
  - AppSync markReadUpTo p50/p95 GraphQLLatency
  - 클라이언트에서 “입장 → 읽음 확정” 왕복시간 p50/p95
- 비용/네트워크
  - WRU 기준 예상 비용(온디맨드: 1 WRU ≈ 1KB 쓰기 1회)
  - 읽음 처리 관련 업/다운로드 바이트/세션
- 품질/정합성
  - 중복 방지율(조건부 갱신으로 스킵된 비율)
  - 읽음 불일치 인시던트 건수(버그 리포트/로그 카운트)

### 비교 방법(전/후 또는 A/B)

- 전/후 비교: 배포 타임라인을 기준으로 같은 기간(예: 7일) 전/후 지표 비교
- A/B: 일부 사용자/룸에 기존 방식 vs 커서 방식 동시 운영(헤더 플래그, 별도 스테이지) 후 지표 비교

### 계산식(템플릿)

기호:

- U = 입장 시 평균 미읽음 메시지 수
- P = 페이지 크기(예: 50)
- K = 세션당 커서 업데이트 호출 수(보통 1~3, 디바운스 포함)
- S_readby = 메시지별 ReadBy 아이템 평균 크기(KB)
- S_cursor = ReadCursor 아이템 평균 크기(KB, 보통 < 1KB)
- Price_WRU = WRU 단가(온디맨드 기준 1백만 WRU당 약 $1.25; 지역/요금제에 따라 상이)

쓰기 건수(세션 기준):

- 기존: Writes_before ≈ min(U, 실제로 생성한 ReadBy 개수) // 예: 첫 페이지만이면 ≈ P, 전체 미읽음이면 ≈ U
- 변경: Writes_after ≈ K // 커서 1~3회

WRU(세션 기준):

- WRU_before ≈ Writes_before × ceil(S_readby / 1KB)
- WRU_after ≈ K × ceil(S_cursor / 1KB)

절감률:

- WRU_Savings(%) = 1 − (WRU_after / WRU_before)
- 비용 절감액(일) ≈ (WRU_before_day − WRU_after_day) × Price_WRU

예시(치환):

- U=120, P=50, K=2, S_readby=0.4KB, S_cursor=0.2KB →
  - Writes_before(전체 미읽음) ≈ 120, Writes_after ≈ 2
  - WRU_before ≈ 120 × 1 = 120 WRU, WRU_after ≈ 2 × 1 = 2 WRU
  - 절감률 ≈ 98.3%

### 수집 방법

서버(DynamoDB)

- CloudWatch 메트릭
  - AWS/DynamoDB: ConsumedWriteCapacityUnits(Sum), ConditionalCheckFailedRequests(Sum), SuccessfulRequestLatency(p50/p95)
- 샘플 CLI
  - WRU 합계:
    aws cloudwatch get-metric-statistics \
     --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits \
     --dimensions Name=TableName,Value=<TableName> \
     --start-time <ISO8601> --end-time <ISO8601> --period 300 --statistics Sum
  - 조건부 실패:
    aws cloudwatch get-metric-statistics \
     --namespace AWS/DynamoDB --metric-name ConditionalCheckFailedRequests \
     --dimensions Name=TableName,Value=<TableName> \
     --start-time <ISO8601> --end-time <ISO8601> --period 300 --statistics Sum

서버(AppSync)

- 그래프QL 레이턴시/요청 수
  - AWS/AppSync: GraphQLLatency(p50/p95), GraphQLRequestCount(필요 시 Enhanced Metrics로 필드/리졸버 단위 활성화)
- 샘플 CLI
  - 지연시간:
    aws cloudwatch get-metric-statistics \
     --namespace AWS/AppSync --metric-name GraphQLLatency \
     --dimensions Name=GraphQLAPIId,Value=<ApiId> Name=Operation,Value=Mutation \
     --start-time <ISO8601> --end-time <ISO8601> --period 300 --statistics Average,Minimum,Maximum

클라이언트(Flutter)

- 계측 포인트
  - t0: 룸 입장 요청 직전
  - t1: 첫 페이지 메시지 수신 완료(최초 렌더)
  - t2: markReadUpTo 요청 전송
  - t3: markReadUpTo 응답 수신
- 기록 지표
  - API 호출 수(세션), t3−t2(읽음 왕복), t1−t0(최초 렌더), 업/다운로드 바이트(네트워크 인터셉터)

정합성/경쟁 조건

- 조건부 갱신 스킵율 = ConditionalCheckFailedRequests / (ConditionalCheckFailedRequests + 성공 UpdateItem)
- 목적: 뒤로 되돌아가지 않는지(스킵 발생은 정상)와 과도한 경쟁으로 재시도 폭증이 없는지 확인

### 대시보드 권장 위젯

- DynamoDB ConsumedWriteCapacityUnits(Sum) – 5분 bin, 전/후 비교
- DynamoDB ConditionalCheckFailedRequests(Sum) – 증가하면 멱등/경쟁 처리 정상 동작 지표
- AppSync GraphQLLatency p50/p95 – markReadUpTo 중심
- AppSync 4XX/5XX – 안정성 확인
- 세션당 API 호출 수/네트워크 바이트(클라이언트 집계치 업로드 시)
