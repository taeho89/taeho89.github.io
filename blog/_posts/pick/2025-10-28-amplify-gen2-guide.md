---
layout: post
title: "Amplify Gen2 실전 사용 가이드"
date: 2025-10-28 01:04:00 +0900
description: >
  Pick 데이팅 앱의 인프라로 사용할 Amplify에 대해 실전 예제 중심으로 정리합니다.
categories: [pick]
tags: [amplify]
---

# AWS Amplify Gen 2 실전 사용 가이드

AWS Amplify Gen 2를 사용해 Lambda 기반 REST API를 구축하고 Flutter 앱과 연동하는 방법을 코드 예제와 함께 정리합니다.

## 1. Amplify Gen 2 프로젝트 구조

Amplify Gen 2는 CDK 기반으로 백엔드 리소스를 코드로 정의합니다. 기본 디렉토리 구조는 다음과 같습니다:

```
amplify/
├── backend.ts                    # 백엔드 진입점
├── auth/
│   └── resource.ts              # Cognito 인증 설정
├── data/
│   └── resource.ts              # GraphQL API/DB 스키마
├── functions/
│   ├── chats/
│   │   ├── resource.ts          # Lambda 함수 정의
│   │   ├── index.ts             # Lambda 핸들러 진입점
│   │   ├── router.ts            # HTTP 라우팅 로직
│   │   ├── controllers/         # 컨트롤러 레이어
│   │   ├── services/            # 비즈니스 로직 레이어
│   │   └── models/              # DTO 및 타입 정의
│   └── users/
│       └── ...
└── infrastructure/
    ├── api-gateway.ts           # API Gateway 설정
    └── outputs.ts               # 프론트엔드 출력 설정
```

## 2. Lambda 함수 정의 (resource.ts)

각 Lambda 함수는 `defineFunction`을 사용해 정의합니다.

### amplify/functions/chats/resource.ts

```typescript
import { defineFunction } from "@aws-amplify/backend";

export const chatsFunction = defineFunction({
  name: "chatsServiceLambda",
  entry: "./index.ts",
  runtime: 20, // Node.js 20
  environment: {
    // 환경 변수 설정
  },
  timeoutSeconds: 10,
});
```

## 3. Lambda 핸들러 구현 (index.ts)

Lambda의 진입점에서는 API Gateway 이벤트를 받아 라우터로 전달합니다.

### amplify/functions/chats/index.ts

```typescript
import { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from "aws-lambda";
import { routeRequest } from "./router";

export async function handler(event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> {
  try {
    const result = await routeRequest(event);
    return {
      statusCode: result.statusCode,
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(result.body),
    };
  } catch (err: any) {
    console.error("Unhandled error in chats handler:", err);
    return {
      statusCode: 500,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        success: false,
        error: "InternalServerError",
        message: "Something went wrong in chats service.",
      }),
    };
  }
}
```

## 4. HTTP 라우팅 구현 (router.ts)

Path와 HTTP Method를 기반으로 적절한 컨트롤러로 요청을 라우팅합니다.

### amplify/functions/chats/router.ts

```typescript
import { APIGatewayProxyEventV2 } from "aws-lambda";
import { apiOk, apiBadRequest, ApiResult } from "./models/ApiResponse";
import { sendMessageController } from "./controllers/sendMessageController";
import { listMessagesController } from "./controllers/listMessagesController";
import { getChatRoomController } from "./controllers/getChatRoomController";

// Path 파싱 헬퍼 함수
function parsePathSegments(path: string | undefined): string[] {
  if (!path) return [];
  // 예: "/chats/rooms/123/messages" -> ["chats", "rooms", "123", "messages"]
  return path.replace(/^\/|\/$/g, "").split("/");
}

export async function routeRequest(event: APIGatewayProxyEventV2): Promise<ApiResult> {
  const method = event.requestContext.http.method; // GET, POST, etc.
  const rawPath = event.rawPath; // e.g., "/chats/send" or "/chats/rooms/123/messages"
  const segments = parsePathSegments(rawPath);

  // Path 검증
  if (segments[0] !== "chats") {
    return apiBadRequest("InvalidRoute", "Path must start with /chats");
  }

  // POST /chats/send - 메시지 전송
  if (method === "POST" && segments[1] === "send") {
    return sendMessageController(event);
  }

  // GET /chats/rooms/:roomId/messages - 메시지 목록 조회
  if (method === "GET" && segments[1] === "rooms" && segments[3] === "messages") {
    const roomId = segments[2];
    return listMessagesController(event, roomId);
  }

  // GET /chats/rooms/:roomId - 채팅방 정보 조회
  if (method === "GET" && segments[1] === "rooms" && segments.length === 3) {
    const roomId = segments[2];
    return getChatRoomController(event, roomId);
  }

  // 기타 라우트는 404 처리
  return apiBadRequest("NotFound", `No route for ${method} ${rawPath}`);
}
```

## 5. 컨트롤러 구현

컨트롤러는 HTTP 요청을 검증하고 서비스 레이어를 호출합니다.

### amplify/functions/chats/controllers/sendMessageController.ts

```typescript
import { APIGatewayProxyEventV2 } from "aws-lambda";
import { apiOk, apiBadRequest, ApiResult } from "../models/ApiResponse";
import { chatService } from "../services/chatService";

export async function sendMessageController(event: APIGatewayProxyEventV2): Promise<ApiResult> {
  // Body 검증
  if (!event.body) {
    return apiBadRequest("InvalidInput", "Request body is required");
  }

  let parsed: any;
  try {
    parsed = JSON.parse(event.body);
  } catch {
    return apiBadRequest("InvalidJSON", "Body must be valid JSON");
  }

  const { roomId, senderId, text } = parsed;
  if (!roomId || !senderId || !text) {
    return apiBadRequest("MissingFields", "roomId, senderId, text are required");
  }

  // 서비스 호출
  const created = await chatService.sendMessage({ roomId, senderId, text });

  return apiOk({
    success: true,
    message: created, // ChatMessageDTO
  });
}
```

### amplify/functions/chats/controllers/listMessagesController.ts

```typescript
import { APIGatewayProxyEventV2 } from "aws-lambda";
import { apiOk, apiBadRequest, ApiResult } from "../models/ApiResponse";
import { chatService } from "../services/chatService";

export async function listMessagesController(event: APIGatewayProxyEventV2, roomId: string): Promise<ApiResult> {
  if (!roomId) {
    return apiBadRequest("MissingRoomId", "roomId is required");
  }

  // 쿼리 파라미터에서 페이지네이션 정보 추출
  const limit = event.queryStringParameters?.limit;
  const cursor = event.queryStringParameters?.cursor;

  const result = await chatService.listMessages({
    roomId,
    limit: limit ? parseInt(limit, 10) : 20,
    cursor: cursor ?? null,
  });

  return apiOk({
    success: true,
    messages: result.messages, // ChatMessageDTO[]
    nextCursor: result.nextCursor, // 페이지네이션용
  });
}
```

## 6. 서비스 레이어 구현

서비스 레이어는 실제 비즈니스 로직과 데이터베이스 접근을 처리합니다.

### amplify/functions/chats/services/chatService.ts

```typescript
// Amplify Gen 2 Data를 사용하는 경우 GraphQL SDK import
// import { ChatMessageModel, ChatRoomModel } from '../../data/...';

type SendMessageInput = {
  roomId: string;
  senderId: string;
  text: string;
};

type ListMessagesInput = {
  roomId: string;
  limit: number;
  cursor: string | null;
};

async function sendMessage(input: SendMessageInput) {
  // 1. 입력 검증
  // 2. senderId가 roomId의 멤버인지 확인
  // 3. Amplify Data 또는 DynamoDB에 메시지 insert
  // 4. 생성된 메시지를 ChatMessageDTO로 반환

  // Pseudo 코드 (실제로는 Amplify Data SDK 사용)
  const result = {
    id: "msg12345",
    roomId: input.roomId,
    senderId: input.senderId,
    text: input.text,
    sentAt: Date.now(),
  };

  return result;
}

async function listMessages(input: ListMessagesInput) {
  // 1. roomId 검증
  // 2. DB에서 roomId로 메시지 목록 조회
  // 3. ChatMessageDTO[] 형태로 반환

  return {
    messages: [
      {
        id: "msg111",
        roomId: input.roomId,
        senderId: "user1",
        text: "hello",
        sentAt: 1690000000000,
      },
      {
        id: "msg112",
        roomId: input.roomId,
        senderId: "user2",
        text: "hi",
        sentAt: 1690000001000,
      },
    ],
    nextCursor: null, // 더 이상 데이터가 없으면 null
  };
}

async function getRoomInfo(roomId: string) {
  // 1. DB에서 room 조회
  // 2. 멤버 정보, 마지막 메시지 등 조회
  // 3. ChatRoomDTO로 반환

  return {
    roomId,
    memberIds: ["user1", "user2"],
    lastMessagePreview: "hi",
    unreadCountForRequester: 3,
  };
}

export const chatService = {
  sendMessage,
  listMessages,
  getRoomInfo,
};
```

## 7. DTO 및 API Response 모델

### amplify/functions/chats/models/ChatMessageDTO.ts

```typescript
export type ChatMessageDTO = {
  id: string;
  roomId: string;
  senderId: string;
  text: string;
  sentAt: number; // epoch ms
};
```

### amplify/functions/chats/models/ChatRoomDTO.ts

```typescript
export type ChatRoomDTO = {
  roomId: string;
  memberIds: string[];
  lastMessagePreview: string;
  unreadCountForRequester: number;
};
```

### amplify/functions/chats/models/ApiResponse.ts

```typescript
export type ApiResult = {
  statusCode: number;
  body: any; // JSON으로 직렬화될 객체
};

export function apiOk(body: any): ApiResult {
  return {
    statusCode: 200,
    body,
  };
}

export function apiBadRequest(code: string, message: string): ApiResult {
  return {
    statusCode: 400,
    body: {
      success: false,
      error: code,
      message,
    },
  };
}
```

## 8. backend.ts에서 리소스 통합

모든 리소스를 `defineBackend`로 통합하고, 인프라 설정을 추가합니다.

### amplify/backend.ts

```typescript
import { defineBackend } from "@aws-amplify/backend";
import { auth } from "./auth/resource";
import { data } from "./data/resource";
import { chatsFunction } from "./functions/chats/resource";
import { attachApiGatewayWithChatsRoutes } from "./infrastructure/api-gateway";
import { exposeFrontendOutputs } from "./infrastructure/outputs";

// 백엔드 리소스 정의
const backend = defineBackend({
  auth,
  data,
  chatsFunction,
});

// API Gateway 설정
const { apiStack, httpApi } = attachApiGatewayWithChatsRoutes(backend);

// Flutter 앱을 위한 출력 설정
exposeFrontendOutputs(backend, { apiStack, httpApi });

export default backend;
```

## 9. API Gateway 인프라 설정

CDK를 사용해 HttpApi를 생성하고 Lambda와 연결합니다.

### amplify/infrastructure/api-gateway.ts

```typescript
import { Stack } from "aws-cdk-lib";
import { CorsHttpMethod, HttpApi, HttpMethod } from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";

export function attachApiGatewayWithChatsRoutes(backend: any) {
  // 1. API Stack 생성
  const apiStack = backend.createStack("api-stack");

  // 2. HTTP API 생성 (CORS 설정 포함)
  const httpApi = new HttpApi(apiStack, "HttpApi", {
    corsPreflight: {
      allowOrigins: ["*"],
      allowMethods: [CorsHttpMethod.GET, CorsHttpMethod.POST, CorsHttpMethod.OPTIONS],
      allowHeaders: ["*"],
    },
  });

  // 3. Lambda Integration 생성
  const chatsIntegration = new HttpLambdaIntegration("ChatsIntegration", backend.chatsFunction.resources.lambda);

  // 4. Route 추가: /chats/{proxy+} 경로를 chatsFunction에 연결
  httpApi.addRoutes({
    path: "/chats/{proxy+}",
    methods: [HttpMethod.GET, HttpMethod.POST],
    integration: chatsIntegration,
  });

  return { apiStack, httpApi };
}
```

**주요 포인트:**

- `{proxy+}`: `/chats/send`, `/chats/rooms/123/messages` 등 모든 하위 경로를 하나의 Lambda로 라우팅
- Lambda 내부에서 `event.rawPath`로 세부 경로를 파싱하여 컨트롤러 분기 처리

## 10. 프론트엔드 출력 설정

Flutter 앱이 API를 호출할 수 있도록 URL과 Region 정보를 출력합니다.

### amplify/infrastructure/outputs.ts

```typescript
import { Stack } from "aws-cdk-lib";

export function exposeFrontendOutputs(backend: any, deps: { apiStack: any; httpApi: any }) {
  const { apiStack, httpApi } = deps;

  backend.addOutput({
    custom: {
      restapi: {
        chats: {
          chatsApi: {
            awsregion: Stack.of(apiStack).region,
            url: httpApi.apiEndpoint, // "https://abc123.execute-api.ap-northeast-2.amazonaws.com"
            basepath: "chats",
            authorizationType: "COGNITO_JWT", // 또는 'NONE'
          },
        },
        // users, match 등 다른 API도 추가 가능
      },
    },
  });
}
```

이 설정은 `amplify_outputs.json`으로 출력되어 Flutter 앱에서 사용할 수 있습니다.

## 11. Stage와 Prefix 설정 (선택사항)

API 버전 관리를 위해 `/v1` prefix를 추가하려면:

### api-gateway.ts (Stage 추가 버전)

```typescript
import { HttpApi, HttpMethod, CorsHttpMethod } from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";

export function attachApiGatewayWithChatsRoutes(backend: any) {
  const apiStack = backend.createStack("api-stack");

  const httpApi = new HttpApi(apiStack, "HttpApi", {
    corsPreflight: {
      allowOrigins: ["*"],
      allowMethods: [CorsHttpMethod.GET, CorsHttpMethod.POST, CorsHttpMethod.OPTIONS],
      allowHeaders: ["*"],
    },
    // 기본 Stage 생성 비활성화
    createDefaultStage: false,
  });

  // v1 Stage 생성
  const stage = httpApi.addStage("v1", {
    stageName: "v1",
    autoDeploy: true,
  });

  const chatsIntegration = new HttpLambdaIntegration("ChatsIntegration", backend.chatsFunction.resources.lambda);

  // Route: /chats/{proxy+}
  httpApi.addRoutes({
    path: "/chats/{proxy+}",
    methods: [HttpMethod.GET, HttpMethod.POST],
    integration: chatsIntegration,
  });

  // 최종 URL: https://xxx.execute-api.ap-northeast-2.amazonaws.com/v1/chats/...
  return { apiStack, httpApi, stage };
}
```

**outputs.ts 수정:**

```typescript
backend.addOutput({
  custom: {
    restapi: {
      chats: {
        chatsApi: {
          awsregion: Stack.of(apiStack).region,
          url: `${httpApi.apiEndpoint}/v1`, // Stage prefix 포함
          basepath: "chats",
          apiversion: stage.stageName, // "v1"
        },
      },
    },
  },
});
```

## 12. Flutter 앱에서 API 호출

### ApiConfig 설정

```dart
class ApiConfig {
  static const String region = 'ap-northeast-2';
  static const String baseUrl = 'https://abc123.execute-api.ap-northeast-2.amazonaws.com/v1/chats';
}
```

### HTTP Client 구현

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class ChatApiClient {
  final http.Client http;

  ChatApiClient({http.Client? httpClient}) : http = httpClient ?? http.Client();

  Uri _uri(String path, [Map<String, String>? query]) {
    // baseUrl: "https://.../v1/chats"
    // path: "send" or "rooms/roomId/messages"
    final full = '${ApiConfig.baseUrl}/$path';
    return Uri.parse(full).replace(queryParameters: query);
  }

  /// POST /chats/send - 메시지 전송
  Future<Map<String, dynamic>> sendMessage({
    required String roomId,
    required String senderId,
    required String text,
    String? authToken,
  }) async {
    final body = jsonEncode({
      'roomId': roomId,
      'senderId': senderId,
      'text': text,
    });

    final response = await http.post(
      _uri('send'),
      headers: {
        'Content-Type': 'application/json',
        if (authToken != null) 'Authorization': 'Bearer $authToken',
      },
      body: body,
    );

    return jsonDecode(response.body) as Map<String, dynamic>;
  }

  /// GET /chats/rooms/:roomId/messages - 메시지 목록 조회
  Future<Map<String, dynamic>> listMessages({
    required String roomId,
    int limit = 20,
    String? cursor,
    String? authToken,
  }) async {
    final response = await http.get(
      _uri('rooms/$roomId/messages', {
        'limit': limit.toString(),
        if (cursor != null) 'cursor': cursor,
      }),
      headers: {
        if (authToken != null) 'Authorization': 'Bearer $authToken',
      },
    );

    return jsonDecode(response.body) as Map<String, dynamic>;
  }

  /// GET /chats/rooms/:roomId - 채팅방 정보 조회
  Future<Map<String, dynamic>> getChatRoom({
    required String roomId,
    String? authToken,
  }) async {
    final response = await http.get(
      _uri('rooms/$roomId'),
      headers: {
        if (authToken != null) 'Authorization': 'Bearer $authToken',
      },
    );

    return jsonDecode(response.body) as Map<String, dynamic>;
  }
}
```

### Flutter 모델 클래스

```dart
class ChatMessage {
  final String id;
  final String roomId;
  final String senderId;
  final String text;
  final int sentAt;

  ChatMessage({
    required this.id,
    required this.roomId,
    required this.senderId,
    required this.text,
    required this.sentAt,
  });

  factory ChatMessage.fromJson(Map<String, dynamic> json) {
    return ChatMessage(
      id: json['id'] as String,
      roomId: json['roomId'] as String,
      senderId: json['senderId'] as String,
      text: json['text'] as String,
      sentAt: json['sentAt'] as int,
    );
  }
}
```

### 사용 예시

```dart
final api = ChatApiClient();

// 메시지 전송
final result = await api.sendMessage(
  roomId: 'room123',
  senderId: 'user1',
  text: 'Hello!',
  authToken: '<JWT_TOKEN>',
);

if (result['success'] == true) {
  final message = ChatMessage.fromJson(result['message']);
  print('Message sent: ${message.id}');
}

// 메시지 목록 조회
final messagesResult = await api.listMessages(
  roomId: 'room123',
  limit: 20,
  authToken: '<JWT_TOKEN>',
);

final messages = (messagesResult['messages'] as List)
    .map((json) => ChatMessage.fromJson(json))
    .toList();
```

## 13. JWT 인증 (Cognito)

Amplify Auth를 사용하는 경우 idToken을 추출하여 API 호출에 포함합니다.

```dart
import 'package:amplify_flutter/amplify_flutter.dart';

Future<String> getIdToken() async {
  final session = await Amplify.Auth.fetchAuthSession() as CognitoAuthSession;
  return session.userPoolTokensResult.value.idToken.raw;
}

// 사용 예시
final token = await getIdToken();
final result = await api.sendMessage(
  roomId: 'room123',
  senderId: 'user1',
  text: 'Hello!',
  authToken: token,
);
```

Lambda 내부에서는 `event.requestContext.authorizer.jwt.claims.sub`로 사용자 ID를 추출할 수 있습니다 (API Gateway Cognito Authorizer 설정 시).

## 14. 배포 및 테스트

```bash
# Amplify 프로젝트 배포
npx amplify sandbox

# 또는 프로덕션 배포
npx amplify deploy
```

배포 후 생성된 `amplify_outputs.json` 파일을 Flutter 프로젝트에 복사하여 사용합니다.

## 요약

1. **Lambda 함수**: `defineFunction`으로 정의하고, `index.ts`에서 핸들러 구현
2. **라우팅**: `router.ts`에서 path/method 기반으로 컨트롤러 분기
3. **컨트롤러**: HTTP 요청 검증 및 서비스 호출
4. **서비스**: 비즈니스 로직 및 DB 접근
5. **API Gateway**: CDK로 HttpApi 생성 및 Lambda 연결 (`{proxy+}` 패턴)
6. **출력**: `backend.addOutput`으로 Flutter 앱에 API 정보 제공
7. **Flutter**: HTTP 클라이언트로 REST API 호출, JWT 토큰 전달

이 구조를 통해 확장 가능하고 유지보수가 쉬운 서버리스 백엔드를 구축할 수 있습니다.
