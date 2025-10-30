---
layout: post
title: "AppSync를 이용한 채팅 서비스 구현"
date: 2025-10-30 22:00:00 +0900
description: >
  Amplify에서의 GraphQL API 처리의 방법 2가지를 비교해봅니다.
categories: [pick]
tags: [amplify, graphql]
---

# Amplify Gen 2: AppSync JS Resolver vs Lambda Resolver

## 아키텍처 개요

```mermaid
graph TD
    A[Client GraphQL Request] --> B[AppSync Resolver]
    B -->|1. request()| C[Data Source (DynamoDB / Lambda)]
    C -->|2. Response| D[response()]
    D --> E[GraphQL Engine]
    E --> F[Client Filtered Response]
```

**아키텍처 설명**: AWS AppSync는 관리형 GraphQL 서비스로서, 클라이언트의 요청과 백엔드 데이터 소스 사이에서 중재자 역할을 수행합니다. 위 다이어그램은 단일 GraphQL 요청이 처리되는 전체 흐름을 보여줍니다. 클라이언트가 GraphQL 쿼리나 뮤테이션을 보내면, AppSync는 해당 스키마 필드에 연결된 리졸버를 실행하고, 리졸버는 데이터 소스와 통신하여 결과를 가져온 뒤, GraphQL 엔진이 최종적으로 클라이언트가 요청한 필드만 필터링하여 응답을 반환합니다.

---

## 1. AppSync와 Amplify Gen 2 핸들러 구조

Amplify Gen 2에서는 두 가지 주요 리졸버 타입을 지원합니다:

- **`a.handler.function()`** → **Lambda 리졸버**: AWS Lambda 컨테이너에서 실행되는 함수입니다. 완전한 Node.js 런타임을 사용하며, 외부 SDK와 라이브러리를 자유롭게 활용할 수 있습니다.
- **`a.handler.custom()`** → **AppSync JS 리졸버**: AppSync의 자체 JavaScript 런타임에서 직접 실행되는 경량 함수입니다. 빠른 실행 속도와 무료 비용이 장점이지만, 제한된 기능만 사용할 수 있습니다.

### 비교

| 구분            | Lambda 리졸버                           | AppSync JS 리졸버                     |
| --------------- | --------------------------------------- | ------------------------------------- |
| **실행 환경**   | Lambda 컨테이너 (완전한 Node.js 런타임) | AppSync JS 런타임 (제한된 JavaScript) |
| **호출 비용**   | 유료 (Lambda 호출 비용 발생)            | 무료 (AppSync에 포함)                 |
| **콜드스타트**  | 있음 (첫 호출 시 100ms~1초)             | 없음 (사전 초기화됨)                  |
| **코드 형태**   | `handler(event)` 형식                   | `request(ctx)` / `response(ctx)` 형식 |
| **외부 SDK**    | 가능 (AWS SDK, Axios 등)                | 불가 (`util.*` 유틸리티만)            |
| **비동기 처리** | 가능 (`async/await`)                    | 불가 (동기 처리만)                    |
| **사용 시점**   | 복잡한 비즈니스 로직, 외부 API 통신     | 단순 CRUD, 데이터 매핑                |

**콜드스타트란?** Lambda 함수가 처음 호출되거나 장시간 미사용 후 호출될 때, AWS가 새로운 실행 환경을 생성하는 과정입니다. 이 과정에서 코드 다운로드, 런타임 초기화, 의존성 로딩 등이 발생하여 100ms에서 수 초까지 지연이 발생할 수 있습니다. AppSync JS 리졸버는 사전 초기화된 환경에서 실행되므로 콜드스타트가 없습니다.

**적용 전략**:

- **데이터베이스 CRUD 작업** → AppSync JS 리졸버 (빠르고 비용 효율적)
- **결제 처리, 외부 API 호출** → Lambda 리졸버 (복잡한 로직 처리 가능)
- **이메일 발송, 이미지 처리** → Lambda 리졸버 (외부 SDK 필요)

---

## 2. AppSync JS 리졸버 기본 구조

AppSync JS 리졸버는 **request/response 기반의 데이터 매핑 계층**입니다. GraphQL 세계와 데이터 소스 사이를 연결하는 번역기 역할을 수행합니다.

### 기본 코드 구조

```typescript
export function request(ctx) {
  return {
    operation: "PutItem",
    key: util.dynamodb.toMapValues({ id: util.autoId() }),
    attributeValues: util.dynamodb.toMapValues({
      content: ctx.arguments.content,
      createdAt: util.time.nowISO8601(),
    }),
  };
}

export function response(ctx) {
  if (ctx.error) util.error(ctx.error.message, ctx.error.type);
  return ctx.result;
}
```

### 함수 역할 상세 설명

**1. request(ctx) 함수**:

- **목적**: GraphQL 인자를 데이터 소스가 이해할 수 있는 형식으로 변환
- **ctx.arguments**: 클라이언트가 GraphQL 쿼리에서 전달한 인자 (예: `{ content: "Hello World" }`)
- **반환값**: 데이터 소스 작업 명세 (DynamoDB의 경우 `operation`, `key`, `attributeValues` 등)
- **util 객체**: AppSync가 제공하는 유틸리티 함수 모음
  - `util.autoId()`: UUID 자동 생성
  - `util.time.nowISO8601()`: 현재 시간을 ISO 8601 형식으로 반환 (예: "2025-10-30T13:45:00Z")
  - `util.dynamodb.toMapValues()`: JavaScript 객체를 DynamoDB 형식으로 변환

**2. response(ctx) 함수**:

- **목적**: 데이터 소스 응답을 GraphQL 반환 타입으로 변환
- **ctx.error**: 데이터 소스에서 발생한 에러 정보
- **ctx.result**: 데이터 소스로부터 받은 응답 데이터
- **에러 처리**: `util.error()`를 사용하여 GraphQL 에러로 변환

**실무 포인트**: request 함수는 데이터 검증 및 변환 로직을 담당하고, response 함수는 에러 핸들링과 데이터 정규화를 담당합니다. 이 분리된 구조 덕분에 코드의 가독성과 유지보수성이 향상됩니다.

---

## 3. DynamoDB 데이터소스와 연동

### 데이터소스 참조 방식

`dataSource: a.ref('Post')`는 Amplify가 자동으로 생성한 DynamoDB 테이블(Post)을 참조합니다.

```typescript
.handler(a.handler.custom({
  dataSource: a.ref('Post'),
  entry: './increment-like.js'
}))
```

**데이터소스 타입**:

- **DynamoDB**: NoSQL 데이터베이스 (가장 일반적)
- **Lambda**: 커스텀 로직 실행
- **HTTP Endpoint**: 외부 REST API 호출
- **RDS**: 관계형 데이터베이스 (Aurora Serverless)
- **OpenSearch**: 검색 엔진

### DynamoDB 작업 예시: 좋아요 증가 기능

```javascript
export function request(ctx) {
  return {
    operation: "UpdateItem",
    key: util.dynamodb.toMapValues({ id: ctx.arguments.postId }),
    update: {
      expression: "ADD likes :plusOne",
      expressionNames: util.dynamodb.toMapValues({ "#likes": "likes" }),
      expressionValues: util.dynamodb.toMapValues({ ":plusOne": 1 }),
    },
  };
}

export function response(ctx) {
  if (ctx.error) util.error(ctx.error.message, ctx.error.type);
  return ctx.result;
}
```

**DynamoDB Update Expression 상세 설명**:

- **operation: 'UpdateItem'**: 기존 항목을 수정하는 작업 (PutItem은 덮어쓰기, UpdateItem은 부분 수정)
- **key**: 수정할 항목의 기본 키 (DynamoDB의 Primary Key)
- **update.expression**: 수정 작업을 정의하는 표현식
  - `ADD likes :plusOne`: likes 필드에 1을 더하기 (원자적 연산)
  - `SET name = :newName`: 필드 값 설정
  - `REMOVE oldField`: 필드 삭제
- **expressionNames**: 필드 이름 치환 (예약어 충돌 방지)
- **expressionValues**: 표현식에서 사용할 값 정의

**원자적 연산(Atomic Operation)이란?** 여러 클라이언트가 동시에 같은 항목을 수정해도 데이터 일관성이 보장되는 연산입니다. 예를 들어, 두 사용자가 동시에 좋아요 버튼을 누르면 최종 좋아요 수는 정확히 +2가 됩니다.

**참고 문서**:

- [AppSync JS Resolver Reference](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-overview-js.html)
- [DynamoDB Operations Reference](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-dynamodb-js.html)

---

## 4. 채팅 서비스 sendMessage Mutation 예시

실시간 채팅 애플리케이션에서 메시지를 전송하는 실제 사용 사례입니다.

### GraphQL 스키마 정의

```typescript
sendMessage: a.mutation()
  .arguments({
    roomId: a.string().required(),
    sender: a.string().required(),
    payload: a.ref("Payload").required(),
  })
  .returns(a.ref("ChatMessage"))
  .handler(
    a.handler.custom({
      dataSource: a.ref("ChatMessage"),
      entry: "./chatHandler/sendMessageHandler.ts",
    })
  )
  .authorization((allow) => [allow.publicApiKey()]);
```

**스키마 구성 요소 설명**:

- **mutation()**: 데이터를 변경하는 작업 (생성, 수정, 삭제)
  - Query: 데이터 조회 (읽기 전용)
  - Mutation: 데이터 변경 (쓰기 작업)
  - Subscription: 실시간 데이터 구독
- **arguments()**: 클라이언트가 전달해야 할 입력 값 정의
- **returns()**: 반환할 데이터 타입
- **authorization()**: 접근 권한 설정
  - `allow.publicApiKey()`: API 키로 인증 (개발/테스트용)
  - `allow.authenticated()`: 로그인 사용자만
  - `allow.owner()`: 데이터 소유자만
  - `allow.groups(['admin'])`: 특정 그룹만

### 리졸버 구현

```typescript
import { util } from "@aws-appsync/utils";

export function request(ctx) {
  const { roomId, sender, payload } = ctx.arguments;
  const id = util.autoId();
  const sentAt = util.time.nowISO8601();

  // stash에 데이터 저장 (response 함수에서 사용 가능)
  ctx.stash.newItem = { id, roomId, sender, payload, sentAt };

  return {
    operation: "PutItem",
    key: util.dynamodb.toMapValues({ id }),
    attributeValues: util.dynamodb.toMapValues({
      roomId,
      sender,
      payload,
      sentAt,
    }),
  };
}

export function response(ctx) {
  if (ctx.error) util.error(ctx.error.message, ctx.error.type);
  // stash에서 저장된 데이터 반환
  return ctx.stash.newItem;
}
```

**ctx.stash의 역할**:

- **목적**: request와 response 함수 간에 데이터를 공유하는 임시 저장소
- **사용 사례**:
  - 생성된 ID를 response에서 재사용
  - 중간 계산 결과 저장
  - 파이프라인 리졸버에서 함수 간 데이터 전달
- **생명주기**: 단일 리졸버 실행 동안만 유효 (요청 간 공유 안 됨)

**실무 활용**: stash를 활용하면 DynamoDB 응답이 예상과 다를 때도 일관된 데이터 구조를 클라이언트에 반환할 수 있습니다. 예를 들어, DynamoDB가 속성을 누락하거나 형식이 다를 때 stash에 저장된 원본 데이터를 반환하여 안정성을 확보합니다.

### GraphQL 요청 예시

```graphql
mutation {
  sendMessage(roomId: "room-123", sender: "alice", payload: { type: "text", text: "hello" }) {
    id
    sender
    payload {
      text
    }
  }
}
```

**중요**: GraphQL은 리졸버가 반환한 전체 객체 중에서 **클라이언트가 요청한 필드만** 응답합니다. 위 예시에서는 `sentAt`을 요청하지 않았으므로 응답에 포함되지 않습니다.

---

## 5. GraphQL의 필드 필터링 동작

GraphQL의 핵심 기능 중 하나는 **선택적 필드 조회(Field Selection)**입니다.

### 동작 원리

1. **리졸버 실행**: 리졸버는 전체 데이터 객체를 반환

   ```javascript
   return {
     id: "msg-123",
     roomId: "room-123",
     sender: "alice",
     payload: { type: "text", text: "hello" },
     sentAt: "2025-10-30T13:45:00Z",
   };
   ```

2. **GraphQL 엔진 필터링**: 클라이언트가 요청한 필드만 선택

   ```graphql
   {
     id
     sender
   }
   ```

3. **최종 응답**: 필터링된 결과만 전송
   ```json
   {
     "id": "msg-123",
     "sender": "alice"
   }
   ```

### 필터링의 장점

- **네트워크 대역폭 절약**: 불필요한 데이터 전송 방지
- **클라이언트 유연성**: 필요한 데이터만 정확히 요청
- **보안 강화**: 민감한 필드는 요청하지 않으면 노출되지 않음
- **성능 최적화**: 작은 페이로드로 빠른 응답

**REST API와의 차이**:

- REST: `/api/messages/123` 호출 시 모든 필드 반환
- GraphQL: 필요한 필드만 요청하여 받음 (Over-fetching 방지)

**실무 팁**: 모바일 앱처럼 네트워크 환경이 불안정한 경우, 필요 최소한의 필드만 요청하여 데이터 전송량을 줄이고 응답 속도를 개선할 수 있습니다.

---

## 6. Lambda 리졸버 vs AppSync JS 리졸버 심층 비교

### 상세 비교표

| 항목               | AppSync JS 리졸버                 | Lambda 리졸버                        |
| ------------------ | --------------------------------- | ------------------------------------ |
| **실행 위치**      | AppSync JS 런타임 (AWS 관리)      | AWS Lambda (독립 컨테이너)           |
| **호출 비용**      | 무료 (AppSync 요금에 포함)        | Lambda 호출당 $0.0000002             |
| **콜드스타트**     | 없음                              | 있음 (첫 호출 또는 장기간 미사용 시) |
| **최대 실행 시간** | 30초                              | 15분 (설정 가능)                     |
| **외부 SDK**       | 불가                              | 가능 (npm 패키지 자유롭게 사용)      |
| **데이터 접근**    | `util.dynamodb.*` 유틸리티만      | AWS SDK 전체 사용 가능               |
| **비동기 처리**    | 불가 (동기 처리만)                | 가능 (`async/await`)                 |
| **에러 처리**      | `util.error()`                    | `throw new Error()`, try-catch       |
| **루프 제한**      | `while`, `do-while` 불가          | 모든 루프 가능                       |
| **적합한 로직**    | 단순 CRUD, 권한 검증, 데이터 변환 | 복잡 로직, 외부 API, 파일 처리       |

### 실전 의사결정 가이드

**AppSync JS 리졸버를 선택해야 할 경우**:

1. DynamoDB CRUD 작업 (GetItem, PutItem, UpdateItem, Query, Scan)
2. 데이터 검증 및 변환 (입력 값 정규화, 타임스탬프 추가)
3. 권한 필터링 (사용자별 데이터 접근 제어)
4. 단순 계산 (필드 값 합산, 조건부 로직)
5. 높은 트래픽 환경 (콜드스타트 없이 빠른 응답)

**Lambda 리졸버를 선택해야 할 경우**:

1. 외부 API 호출 (결제 게이트웨이, 이메일 발송, SMS)
2. 복잡한 비즈니스 로직 (다단계 데이터 검증, 복잡한 계산)
3. 파일 처리 (이미지 리사이징, PDF 생성, S3 업로드)
4. 여러 데이터 소스 통합 (RDS + DynamoDB + 외부 API)
5. 장시간 실행 작업 (데이터 마이그레이션, 배치 처리)

### 비용 시뮬레이션 예시

**시나리오**: 월 1,000만 회 호출되는 간단한 조회 API

- **AppSync JS 리졸버**:
  - 리졸버 비용: $0 (무료)
  - AppSync 요청 비용: $0.08/백만 요청 × 10 = **$0.80**
- **Lambda 리졸버**:
  - Lambda 호출 비용: $0.20/백만 요청 × 10 = $2.00
  - Lambda 실행 비용: 128MB × 100ms × 10M 회 = 약 $0.20
  - AppSync 요청 비용: $0.80
  - **총 비용: $3.00**

**결론**: 단순 CRUD의 경우 AppSync JS 리졸버가 약 73% 비용 절감 효과를 가져옵니다.

### 하이브리드 전략

실제 프로덕션 환경에서는 두 리졸버를 혼합하여 사용합니다:

```typescript
// AppSync JS 리졸버로 빠른 조회
getPost: a.query()
  .returns(a.ref("Post"))
  .handler(
    a.handler.custom({
      dataSource: a.ref("Post"),
      entry: "./getPost.js",
    })
  );

// Lambda 리졸버로 복잡한 생성 로직 (이미지 업로드 + 썸네일 생성)
createPost: a.mutation().returns(a.ref("Post")).handler(a.handler.function(createPostFunction));
```

---

## 7. 참고 문서 및 학습 자료

### 공식 문서

| 항목                         | 링크                                                                                                 | 설명                     |
| ---------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------ |
| AppSync JS Resolver Overview | [공식 문서](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-overview-js.html) | 리졸버 개요 및 기본 개념 |
| DynamoDB Resolver Reference  | [공식 문서](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-dynamodb-js.html) | DynamoDB 작업 전체 목록  |
| Context Reference            | [공식 문서](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference-js.html)  | ctx 객체 상세 명세       |
| Utility Reference (`util.*`) | [공식 문서](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-util-reference.html)        | 유틸리티 함수 전체 목록  |
| Pipeline Resolvers           | [공식 문서](https://docs.aws.amazon.com/appsync/latest/devguide/pipeline-resolvers-js.html)          | 다단계 리졸버 구성 방법  |

### 추가 학습 리소스

**파이프라인 리졸버(Pipeline Resolver)**:

- 여러 함수를 순차적으로 실행하는 리졸버 패턴
- 사용 사례: 인증 확인 → 권한 검증 → 데이터 조회 → 로그 기록
- 각 단계의 출력이 다음 단계의 입력으로 전달됨

**배치 리졸버(Batch Resolver)**:

- 여러 항목을 한 번에 조회하여 N+1 쿼리 문제 해결
- DynamoDB BatchGetItem, BatchWriteItem 활용
- 성능 최적화에 필수적

---

## 8. 핵심 요약 및 면접 대비 포인트

### 핵심 개념 정리

| 핵심 포인트                      | 설명                                     | 실무 적용                     |
| -------------------------------- | ---------------------------------------- | ----------------------------- |
| **`a.handler.custom`**           | AppSync JS 리졸버, request/response 기반 | 단순 CRUD, 빠른 응답 필요 시  |
| **`a.handler.function`**         | Lambda 리졸버, 완전한 Node.js 런타임     | 복잡한 로직, 외부 SDK 필요 시 |
| **`dataSource: a.ref('Model')`** | DynamoDB 테이블 참조                     | 데이터 소스와 리졸버 연결     |
| **`ctx.arguments`**              | GraphQL 인자 객체                        | 클라이언트 입력 값 접근       |
| **`ctx.result`**                 | 데이터 소스 응답 객체                    | 데이터 가공 및 반환           |
| **`ctx.stash`**                  | 함수 간 데이터 공유                      | 파이프라인에서 상태 유지      |
| **필드 선택**                    | GraphQL 엔진의 자동 필터링               | Over-fetching 방지            |

### 면접 예상 질문 및 모범 답변

**Q1. AppSync JS 리졸버와 Lambda 리졸버의 차이점은 무엇인가요?**

**A**: AppSync JS 리졸버는 AppSync의 경량 JavaScript 런타임에서 실행되어 콜드스타트가 없고 비용이 무료이지만, 제한된 기능만 사용 가능합니다. 반면 Lambda 리졸버는 완전한 Node.js 환경에서 실행되어 외부 SDK와 복잡한 로직을 구현할 수 있지만, 콜드스타트와 비용이 발생합니다. 실무에서는 단순 CRUD는 AppSync JS로, 외부 API 호출이나 복잡한 비즈니스 로직은 Lambda로 구현하는 하이브리드 접근이 효과적입니다.

**Q2. ctx.stash는 언제 사용하나요?**

**A**: ctx.stash는 request와 response 함수 간에 데이터를 공유할 때 사용합니다. 예를 들어, request에서 생성한 ID나 타임스탬프를 response에서도 사용해야 할 때 stash에 저장합니다. 또한 파이프라인 리졸버에서 여러 함수가 순차적으로 실행될 때, 앞 단계의 결과를 다음 단계로 전달하는 용도로도 활용됩니다.

**Q3. GraphQL의 필드 필터링이 왜 중요한가요?**

**A**: GraphQL의 필드 필터링은 클라이언트가 필요한 데이터만 정확히 요청할 수 있게 하여, Over-fetching(과도한 데이터 전송)과 Under-fetching(부족한 데이터로 인한 추가 요청) 문제를 동시에 해결합니다. 특히 모바일 환경에서 네트워크 대역폭을 절약하고, 보안 측면에서도 민감한 필드는 요청하지 않으면 노출되지 않아 안전합니다.

**Q4. 콜드스타트 문제를 어떻게 완화할 수 있나요?**

**A**: Lambda의 콜드스타트를 완화하는 방법은 여러 가지입니다: (1) 함수 패키지 크기를 10MB 이하로 유지, (2) 프로비저닝된 동시성(Provisioned Concurrency) 사용, (3) 메모리 할당 증가(더 많은 CPU 할당), (4) Lambda 레이어로 공통 의존성 분리. 하지만 가장 근본적인 해결책은 단순 작업에는 콜드스타트가 없는 AppSync JS 리졸버를 사용하는 것입니다.

### 프로젝트 경험 작성 가이드

취업 준비 시 포트폴리오나 자기소개서에 다음과 같이 작성할 수 있습니다:

**프로젝트 설명 예시**:

> "실시간 채팅 애플리케이션을 구축하면서 AWS AppSync와 DynamoDB를 활용하여 서버리스 GraphQL API를 설계하고 구현했습니다. 메시지 조회 같은 단순 작업은 AppSync JS 리졸버로 구현하여 응답 속도를 최적화하고 비용을 절감했으며, 이미지 업로드 및 썸네일 생성 같은 복잡한 작업은 Lambda 리졸버로 분리하여 확장성을 확보했습니다. 또한 파이프라인 리졸버를 활용하여 인증 → 권한 검증 → 데이터 조회 순서로 다단계 로직을 구현했습니다."

**성과 작성 예시**:

- AppSync JS 리졸버 도입으로 API 응답 시간 평균 150ms → 50ms로 66% 개선
- Lambda 비용 월 $100 → $35로 65% 절감 (단순 조회를 AppSync JS로 전환)
- GraphQL 필드 선택을 활용하여 모바일 앱 데이터 전송량 40% 감소

---

## 9. 실전 코딩 패턴 모음

### 패턴 1: 타임스탬프 자동 추가

```javascript
export function request(ctx) {
  const now = util.time.nowISO8601();

  return {
    operation: "PutItem",
    key: util.dynamodb.toMapValues({ id: util.autoId() }),
    attributeValues: util.dynamodb.toMapValues({
      ...ctx.arguments,
      createdAt: now,
      updatedAt: now,
    }),
  };
}
```

### 패턴 2: 조건부 업데이트 (낙관적 잠금)

```javascript
export function request(ctx) {
  return {
    operation: "UpdateItem",
    key: util.dynamodb.toMapValues({ id: ctx.arguments.id }),
    update: {
      expression: "SET #name = :name, #version = #version + :inc",
      expressionNames: { "#name": "name", "#version": "version" },
      expressionValues: util.dynamodb.toMapValues({
        ":name": ctx.arguments.name,
        ":inc": 1,
      }),
    },
    condition: {
      expression: "#version = :expectedVersion",
      expressionValues: util.dynamodb.toMapValues({
        ":expectedVersion": ctx.arguments.expectedVersion,
      }),
    },
  };
}
```

**낙관적 잠금(Optimistic Locking)**: 버전 번호를 확인하여 동시 수정을 방지하는 기법입니다. 수정 시점의 버전이 예상한 버전과 다르면 업데이트가 실패합니다.

### 패턴 3: 소유자 검증

```javascript
export function request(ctx) {
  const userId = ctx.identity.sub; // Cognito User ID

  return {
    operation: "UpdateItem",
    key: util.dynamodb.toMapValues({ id: ctx.arguments.id }),
    update: {
      expression: "SET #content = :content",
      expressionNames: { "#content": "content" },
      expressionValues: util.dynamodb.toMapValues({
        ":content": ctx.arguments.content,
      }),
    },
    condition: {
      expression: "#owner = :userId",
      expressionNames: { "#owner": "owner" },
      expressionValues: util.dynamodb.toMapValues({ ":userId": userId }),
    },
  };
}
```

### 패턴 4: 페이지네이션

```javascript
export function request(ctx) {
  const limit = ctx.arguments.limit || 20;
  const nextToken = ctx.arguments.nextToken;

  return {
    operation: "Query",
    query: {
      expression: "#pk = :pk",
      expressionNames: { "#pk": "PK" },
      expressionValues: util.dynamodb.toMapValues({
        ":pk": ctx.arguments.roomId,
      }),
    },
    limit: limit,
    nextToken: nextToken,
    scanIndexForward: false, // 최신 메시지부터 조회
  };
}

export function response(ctx) {
  return {
    items: ctx.result.items,
    nextToken: ctx.result.nextToken,
  };
}
```

---

## 10. 트러블슈팅 가이드

### 문제 1: "util is not defined" 에러

**원인**: `@aws-appsync/utils` 임포트 누락

**해결**:

```javascript
import { util } from "@aws-appsync/utils";
```

### 문제 2: DynamoDB 타입 변환 에러

**원인**: JavaScript 객체를 DynamoDB 형식으로 변환하지 않음

**해결**:

```javascript
// 잘못된 예
key: {
  id: ctx.arguments.id;
}

// 올바른 예
key: util.dynamodb.toMapValues({ id: ctx.arguments.id });
```

### 문제 3: 콜드스타트로 인한 타임아웃

**원인**: Lambda 함수 패키지가 크거나, 많은 의존성이 있음

**해결**:

1. Tree shaking으로 불필요한 코드 제거
2. Lambda 레이어로 공통 의존성 분리
3. 메모리 할당 증가 (256MB → 512MB)
4. 프로비저닝된 동시성 적용

### 문제 4: 권한 오류 (AccessDenied)

**원인**: IAM 정책 또는 리소스 정책 미설정

**해결**:

```typescript
// Amplify에서 자동으로 권한 부여
.authorization((allow) => [
  allow.owner(),           // 소유자만
  allow.groups(['admin']), // 특정 그룹만
  allow.authenticated(),   // 모든 인증 사용자
])
```

---

## 마치며

이 문서는 AWS AppSync와 Amplify Gen 2의 리졸버 개념을 취업 준비 관점에서 상세히 정리한 학습 자료입니다. 실무에서 바로 적용 가능한 패턴과 의사결정 기준, 그리고 면접 대비 포인트를 포함하여 구성했습니다.

**계속 학습해야 할 주제**:

- GraphQL Subscription (실시간 데이터 구독)
- AppSync Resolver Mapping Template (VTL → JS 마이그레이션)
- DynamoDB Single Table Design (확장 가능한 데이터 모델링)
- AWS Lambda Best Practices (성능 최적화, 모니터링)
- Cognito User Pools 인증 통합

**실전 프로젝트 아이디어**:

1. 실시간 채팅 애플리케이션
2. 소셜 미디어 피드 (좋아요, 댓글 기능)
3. 전자상거래 장바구니 시스템
4. 협업 문서 편집기 (Google Docs 스타일)
5. 실시간 대시보드 (IoT 데이터 시각화)

이 개념들을 실제 프로젝트에 적용하면서 경험을 쌓고, 발생한 문제와 해결 과정을 정리하면 취업 준비에 큰 도움이 될 것입니다.
