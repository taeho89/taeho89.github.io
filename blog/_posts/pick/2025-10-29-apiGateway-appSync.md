---
layout: post
title: "API Gateway WebSocket API, AppSync Realtime Endpoint 기능 차이"
date: 2025-10-29 04:06:00 +0900
description: >
  웹 소켓을 이용한 채팅 기능 구현을 위한 두 가지 AWS 기능과 차이, 선택 이유에 대해서 정리합니다.
categories: [pick]
tags: [amplify]
---

> AWS에서 웹 소켓 기능을 구현하기 위해서 API Gateway WebSocket API와 AppSync Realtime Endpoint 두 가지 기능에 대해서 알아보고, 각 두 기능의 특징이 무엇인지 정리합니다.

각 AppSync 서비스, API Gateway 서비스가 무엇인지에 관한 내용은 아래 문서에서 다룹니다.
[AWS AppSync][app-sync]
[AWS API Gateway][api-gateway]

# API Gateway WebSocket API

### 프로토콜 / 메시지 형식

API Gateway는 HTTP, HTTPS, WebSocket(ws, wss) 프로토콜을 직접 사용하며 라우트 기반의 요청 처리를 지원한다.
GraphQL이나 상위 프로토콜 없이 Raw WebSocket 자체만을 사용하며 그 외의 **메세지 포맷, pub/sub 기능 등을 직접 구현**하여 사용하여야 한다.
$connect, $disconnet, $defualt 같은 라우트 키 기반으로 Lambda 등의 외부 서비스를 직접 연결하여 처리하여야 한다.

```text
┌───────────────────────────────┐
│  Custom Messages (JSON/Text)  │   ← 직접 설계
│         WebSocket Layer       │   ← AWS가 연결만 관리
└───────────────────────────────┘
```

## 예시 코드

API Gateway 에서 WebSocket을 구현하기 위해서는 `$connect`, `$default`, `$disconnect`라는 최소 라우트가 존재하며 해당 라우트들은 Lambda와 연결된다.

`$connect`: 처음 클라이언트와 연결을 맺을 때 호출

`$defualt`(또는 커스텀, sendMessage): 클라이언트가 메세지를 보낼 때 호출

`$disconnect`: 클라이언트와 연결이 끊길 때 호출

1. CDK 인프라 정의

```ts
import * as apigwv2 from "aws-cdk-lib/aws-apigatewayv2";
import * as integrations from "aws-cdk-lib/aws-apigatewayv2-integrations";
import * as lambda from "aws-cdk-lib/aws-lambda";
import { Stack } from "aws-cdk-lib";

export class ChatWsStack extends Stack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    // Lambda들
    const connectHandler = new lambda.Function(this, "ConnectHandler", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "connectHandler.handler",
      code: lambda.Code.fromAsset("lambda"),
    });

    const messageHandler = new lambda.Function(this, "MessageHandler", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "messageHandler.handler",
      code: lambda.Code.fromAsset("lambda"),
    });

    // WebSocket API
    const wsApi = new apigwv2.WebSocketApi(this, "ChatWsApi", {
      connectRouteOptions: {
        // "$connect" 라우트
        integration: new integrations.WebSocketLambdaIntegration("ConnectIntegration", connectHandler),
      },
      routeSelectionExpression: "$request.body.action",
    });

    // 커스텀 라우트 "sendMessage"
    wsApi.addRoute("sendMessage", {
      integration: new integrations.WebSocketLambdaIntegration("MessageIntegration", messageHandler),
    });

    // 스테이지 배포
    new apigwv2.WebSocketStage(this, "ProdStage", {
      webSocketApi: wsApi,
      stageName: "prod",
      autoDeploy: true,
    });
  }
}
```

2. `$connect` 라우트용 Lambda (연결 등록)
   API Gateway와의 연결마다 고유한 `connectionId`를 발급받으며, 서버는 이를 통해 어떤 사용자와 연결이 되어 있는지 확인할 수 있다.
   아래 코드에선 다이나모 DB에 해당 정보를 저장하여 관리한다.

```js
// connectHandler.js
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const db = new DynamoDBClient({});

export const handler = async (event) => {
  const connectionId = event.requestContext.connectionId;

  // 예: 쿼리스트링 roomId=123 으로 접속했다고 치자
  const roomId = event.queryStringParameters?.roomId || "lobby";

  // 이 유저가 어느 방을 구독 중인지 직접 저장
  await db.send(
    new PutItemCommand({
      TableName: "Connections",
      Item: {
        connectionId: { S: connectionId },
        roomId: { S: roomId },
      },
    })
  );

  return { statusCode: 200 };
};
```

3. 메시지 수신 라우트 예: sendMessage
   클라이언트가 WebSocket으로 JSON을 보내면, API Gateway가 그걸 이 Lambda로 전달하게 설정해둘 수 있다.

```js
// messageHandler.js
import { DynamoDBClient, QueryCommand, PutItemCommand } from "@aws-sdk/client-dynamodb";
import { ApiGatewayManagementApiClient, PostToConnectionCommand } from "@aws-sdk/client-apigatewaymanagementapi";

const db = new DynamoDBClient({});

export const handler = async (event) => {
  const body = JSON.parse(event.body || "{}");
  // body 예: { "action": "sendMessage", "roomId": "123", "text": "hi" }

  const { roomId, text } = body;
  const senderConnectionId = event.requestContext.connectionId;

  // 1) 메시지를 DB에 저장 (채팅 로그 테이블 등)
  await db.send(
    new PutItemCommand({
      TableName: "ChatMessages",
      Item: {
        messageId: { S: Date.now().toString() },
        roomId: { S: roomId },
        senderId: { S: senderConnectionId },
        text: { S: text },
        createdAt: { S: new Date().toISOString() },
      },
    })
  );

  // 2) 이 roomId에 연결된 모든 connectionId를 조회
  const roomConnections = await db.send(
    new QueryCommand({
      TableName: "Connections",
      IndexName: "RoomIndex", // roomId로 찾을 수 있게 GSI 만든다고 가정
      KeyConditionExpression: "roomId = :r",
      ExpressionAttributeValues: {
        ":r": { S: roomId },
      },
    })
  );

  // 3) 각 connectionId 에게 메시지 푸시
  const apiClient = new ApiGatewayManagementApiClient({
    endpoint: `https://${event.requestContext.domainName}/${event.requestContext.stage}`,
  });

  const payload = {
    roomId,
    text,
    senderConnectionId,
    createdAt: new Date().toISOString(),
  };

  for (const item of roomConnections.Items ?? []) {
    const targetConnectionId = item.connectionId.S;

    await apiClient.send(
      new PostToConnectionCommand({
        ConnectionId: targetConnectionId,
        Data: JSON.stringify(payload),
      })
    );
  }

  return { statusCode: 200 };
};
```

4. 클라이언트 쪽 WebSocket 사용 예 (flutter)

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:web_socket_channel/web_socket_channel.dart';

class ChatPage extends StatefulWidget {
  const ChatPage({super.key});

  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  late WebSocketChannel _channel;
  final _controller = TextEditingController();
  final _messages = <String>[];

  @override
  void initState() {
    super.initState();
    final roomId = '123';
    final url =
        'wss://your-api-id.execute-api.ap-northeast-2.amazonaws.com/prod?roomId=$roomId';
    _channel = WebSocketChannel.connect(Uri.parse(url));

    _channel.stream.listen((event) {
      final data = jsonDecode(event);
      setState(() => _messages.add(data['text'] ?? event));
    });
  }

  void _sendMessage() {
    if (_controller.text.isEmpty) return;
    _channel.sink.add(jsonEncode({
      "action": "sendMessage",
      "roomId": "123",
      "text": _controller.text,
    }));
    _controller.clear();
  }

  @override
  void dispose() {
    _channel.sink.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("WebSocket Chat")),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: _messages.length,
              itemBuilder: (context, i) => ListTile(title: Text(_messages[i])),
            ),
          ),
          Row(
            children: [
              Expanded(
                child: TextField(controller: _controller),
              ),
              IconButton(
                icon: const Icon(Icons.send),
                onPressed: _sendMessage,
              ),
            ],
          )
        ],
      ),
    );
  }
}
```

Amplify Environment Variable: WebSocket URL을 Amplify 콘솔의 Environment variables 로 넣어두고,
Flutter 쪽에서 dotenv나 Amplify configuration JSON으로 불러와서 wss://... 주소를 주입하면 관리가 편하다.

# AppSync Realtime Endpoint

AWS AppSync는 GraphQL 스키마 기반으로 query, mutation, subscription 등을 지원하며, pub/sub 기반 실시간 소통을 지원한다.
아래 그림과 같이 AppSync가 소켓을 한번 감싸 요청을 처리하는 구조로 동작하며 웹 소켓 레벨 자체에서 동작하는 API Gateway와는 차이가 있다.

```text
┌───────────────────────────────┐
│         GraphQL Layer         │   ← AppSync가 제공 (Query/Mutation/Subscription)
│  ┌──────────────────────────┐ │
│  │   AppSync Realtime API   │ │   ← GraphQL-over-WebSocket 프로토콜
│  └──────────────────────────┘ │
│         WebSocket Layer       │   ← 실제 네트워크 전송 (wss://)
└───────────────────────────────┘
```

## 예시 코드

1. 스키마 정의
   우선, 채팅방 기능에 필요한 데이터의 type과 subscription을 정의하여야 한다.
   아래 코드의 정의는 sendMessage라는 mutation이 실행될 때마다, onNewMessage를 구독중인 클라이언트에게 메세지가 push된다.

```graphql
type Message {
  id: ID!
  roomId: ID!
  sender: String!
  content: String!
  createdAt: AWSDateTime!
}

type Mutation {
  sendMessage(roomId: ID!, content: String!): Message
}

type Subscription {
  onNewMessage(roomId: ID!): Message @aws_subscribe(mutations: ["sendMessage"])
}
```

2. 클라이언트의 구독
   클라이언트(웹/모바일)는 WebSocket으로 직접 JSON 프레임을 주고받지만 Amplify를 통해 이렇게 쓴다.

```graphql
subscription SubscribeNewMessages($roomId: ID!) {
  onNewMessage(roomId: $roomId) {
    id
    sender
    content
    createdAt
  }
}
```

```dart
// 1) Amplify를 configure (amplify pull or amplify init로 생성된 aws-exports.js / amplifyconfiguration.dart 같은 설정 사용)
Amplify.configure(amplifyGeneratedConfig);

// 2) 메시지 전송 (mutation)
await Amplify.API.mutate(
  request: GraphQLRequest<String>(
    document: '''
      mutation SendMessage($roomId: ID!, $content: String!) {
        sendMessage(roomId: $roomId, content: $content) {
          id
          roomId
          sender
          content
          createdAt
        }
      }
    ''',
    variables: {
      "roomId": "123",
      "content": "hi everyone"
    },
  ),
);

// 3) 실시간 구독 (subscription)
final subscriptionStream = Amplify.API.subscribe(
  request: GraphQLRequest<String>(
    document: '''
      subscription OnNewMessage($roomId: ID!) {
        onNewMessage(roomId: $roomId) {
          id
          sender
          content
          createdAt
        }
      }
    ''',
    variables: { "roomId": "123" },
  ),
  onEstablished: () {
    print("Realtime 연결됨");
  },
  onData: (event) {
    print("새 메시지: ${event.data}");
  },
  onError: (error) {
    print("에러: $error");
  },
  onDone: () {
    print("스트림 종료");
  },
);

```

## 그래서 뭐 써야되는데?

Amplify 환경에서는 AppSync가 GraphQL 기반의 Realtime Endpoint를 공식적으로 지원하기 때문에,
amplify_outputs.dart와 같은 구성 파일을 통해 WebSocket URL이 자동으로 주입되고,
인증·권한·구독 관리까지 통합적으로 관리된다는 점이 가장 큰 장점이다.

반면, API Gateway WebSocket은 Amplify가 직접 추적하지 않아
Stage 배포 시마다 URL을 수동으로 등록해야 하고,
연결·브로드캐스트 로직을 모두 Lambda와 DynamoDB로 직접 관리해야 한다.
따라서 운영 복잡도와 유지보수 비용을 고려했을 때, AppSync Realtime Endpoint를 선택하기로 했다.

다만, 팀 내에서 GraphQL 경험이 많지 않기 때문에 앞으로 스키마 설계, Subscription 필터링, Resolver 파이프라인 같은 부분을
체계적으로 학습해 나가면서 AppSync의 장점을 최대한 활용할 계획이다.

---

### 참고자료

[Reddit: AWS AppSync vs API Gateway - A Comprehensive Guide][Reddit: AWS AppSync vs API Gateway - A Comprehensive Guide]: https://www.reddit.com/r/aws/comments/14cfji8/aws_appsync_vs_api_gateway_a_comprehensive_guide/

[LearnAWS: AppSync vs API Gateway (WebSocket APIs Section)][LearnAWS: AppSync vs API Gateway (WebSocket APIs Section)]: https://learnaws.io/blog/appsync-vs-api-gateway/#web-socket-apis--the-cool-sibling

[Toktokhan Blog: REST API vs GraphQL 비교][Toktokhan Blog: REST API vs GraphQL 비교]: https://blog.toktokhan.dev/rest-api-vs-graphql-7348f54a220b

[AWS Docs: API Gateway WebSocket API Route Keys (CONNECT/DISCONNECT)][AWS Docs: API Gateway WebSocket API Route Keys (CONNECT/DISCONNECT)]: https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/apigateway-websocket-api-route-keys-connect-disconnect.html#apigateway-websocket-api-routes-about-connect

[AWS Docs: AppSync WebSocket Event API - Publish Messages][AWS Docs: AppSync WebSocket Event API - Publish Messages]: https://docs.aws.amazon.com/ko_kr/appsync/latest/eventapi/publish-websocket.html

[AWS Docs: What is AWS AppSync?][AWS Docs: What is AWS AppSync?]: https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/what-is-appsync.html

[app-sync]:

[api-gateway]:
