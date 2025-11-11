---
layout: post
title: "AWS 백엔드 아키텍처 설계 가이드"
date: 2025-10-28 01:06:00 +0900
description: >
  AWS Lambda, API Gateway, Amplify를 이용해 백엔드 아키텍처를 설계하고 코드를 작성해봅니다.
categories: [pick]
tags: [amplify]
---

# AWS 백엔드 아키텍처 설계 가이드

Amplify Gen 2, Lambda, API Gateway, 도메인 분리, Controller/Service 패턴 등 백엔드 아키텍처 설계 방법을 코드와 함께 정리합니다.

## 1. 도메인 기반 서비스 분리

- 각 비즈니스 도메인(users, chats, match 등)은 별도의 Lambda 함수와 서비스 계층으로 분리
- 서비스별 resource.ts, index.ts, router.ts, controllers, services 디렉토리 구조

```
amplify/functions/
├── users/    # 유저 기능
├── chats/    # 채팅 기능
└── match/    # 매칭 기능
```

---

## 2. Controller/Service/Model 패턴

### Controller

- HTTP 요청 검증, 서비스 호출, 에러 핸들링 담당

```typescript
// amplify/functions/users/controllers/updateProfileController.ts
export async function updateProfileController(event) {
  if (!event.body) return apiBadRequest("InvalidInput");
  const { userId, nickname } = JSON.parse(event.body);
  if (!userId || !nickname) return apiBadRequest("MissingFields");
  const updated = await userService.updateProfile(userId, nickname);
  return apiOk({ success: true, user: updated });
}
```

### Service

- DB 접근, 비즈니스 로직, 외부 연동 담당

```typescript
// amplify/functions/users/services/userService.ts
export async function updateProfile(userId, nickname) {
  // DB 쿼리 및 검증 등 처리
  return { userId, nickname };
}
```

### Model

- DTO 및 API 응답 형식 정의

```typescript
export type UserDTO = {
  userId: string;
  nickname: string;
};
```

---

## 3. Lambda 핸들러 공통화 및 라우팅

```typescript
// amplify/functions/chats/index.ts
import { APIGatewayProxyEventV2 } from "aws-lambda";
import { routeRequest } from "./router";

export async function handler(event: APIGatewayProxyEventV2) {
  return routeRequest(event);
}
```

---

## 4. API Gateway CORS 및 Stage 관리

- Stage(v1, v2 등)와 BasePathPrefix를 통해 환경별 경로 분리
- CORS 허용 도메인 자세히 관리

```typescript
import { HttpApi, CorsHttpMethod } from "aws-cdk-lib/aws-apigatewayv2";
const httpApi = new HttpApi(stack, "HttpApi", {
  corsPreflight: {
    allowOrigins: ["https://yourdomain.com"],
    allowMethods: [CorsHttpMethod.GET, CorsHttpMethod.POST],
    allowHeaders: ["Authorization", "Content-Type"],
  },
  createDefaultStage: false,
});
httpApi.addStage("v1", { stageName: "v1", autoDeploy: true });
```

---

## 5. 인프라/코드 연결 구조

- resource.ts, backend.ts에서 Lambda, HttpApi, output 통합 관리
- CDK/Amplify 코드로 모든 인프라를 Code-As-Infrastructure로 관리

```typescript
// amplify/backend.ts
import { defineBackend } from "@aws-amplify/backend";
import { usersFunction } from "./functions/users/resource";
import { chatsFunction } from "./functions/chats/resource";
const backend = defineBackend({ usersFunction, chatsFunction });
```

---

## 6. 데이터 연동 및 확장성

- Amplify Data, DynamoDB, RDS 등 다양한 저장소와 연동
- SNS, EventBridge, 외부 API 후킹 구조 확장 가능

---

## 7. 아키텍처의 주요 설계 포인트

- **도메인 분리**: 비즈니스 도메인별 코드/서비스 격리
- **컨트롤러/서비스/모델 패턴**: 유지보수/테스트 효율 증가
- **API Gateway 관리**: Stage별 환경분리, CORS 세부 관리
- **Code-As-Infrastructure**: CDK/Amplify 환경 내 모든 인프라 코드 관리
- **확장성**: 각 도메인 별 서비스/함수 독립, 외부 연동 허브로 Lambda 활용

이 방식으로 서버리스/마이크로서비스 아키텍처를 효과적으로 설계 및 운영할 수 있습니다.
