---
layout: post
title: "AppSync를 이용한 채팅 서비스 구현"
date: 2025-10-29 21:00:00 +0900
description: >
  AppSync Real-time Endpoint를 이용한 채팅 서비스의 구현 과정을 기록합니다.
categories: [pick]
tags: [amplify, graphql]
---

# Amplify Data에서 스키마 정의

Amplify Gen2 기반 문서에 따르면 기존의 Gen1 Amplify는 GraphQL 스키마 선언이 GraphQL Directive(`@model` 등)으로 이루어졌지만 이제는 `a.model()` 등과 방식으로 TS DSL을 통해 스키마를 정의할 수 있게 하였다. 기존의 `.graphql` 파일 없이 **TypeScript** 환경 내에서 백엔드, 스키마 정의를 모두 할 수 있게 된 것이다.

```ts
// amplify/data/chat.schema.ts
export const chatModels = {
  Payload: a.customType({
    type: a.enum(["text", "image", "audio", "video"]),
    text: a.string(),
    downloadUrl: a.string(),
    storagePath: a.string(),
  }),

  ChatMessage: a
    .model({
      id: a.id().required(),
      roomId: a.string().required(),
      sender: a.string().required(),
      payload: a.ref("Payload").required(),
      sentAt: a.datetime().required(),
    })
    .authorization((allow) => [allow.publicApiKey()]),

  sendMessage: a
    .mutation()
    .arguments({
      roomId: a.string().required(),
      sender: a.string().required(),
      payload: a.ref("Payload").required(),
    })
    .returns(a.ref("ChatMessage"))
    .handler(
      a.handler.custom({
        entry: "./sendMessageHandler.ts",
      })
    )
    .authorization((allow) => [allow.publicApiKey()]),

  watchMessage: a
    .subscription()
    .for(a.ref("sendMessage"))
    .arguments({
      roomId: a.string().required(),
    })
    .handler(
      a.handler.custom({
        entry: "./watchMessageHandler.ts",
      })
    )
    .authorization((allow) => [allow.publicApiKey()]),
};
```

> 이렇게 되면 사용자가 메세지를 보낼때마다 구독 handler가 호출되어 Lambda 기능의 요금 폭탄이 발생하는 것 아닌가?

# Amplify에서 GraphQL Request Handler 함수 구현

아래 핸들러 함수들은 추후 추가적인 비즈니스 로직이 있다면 덧붙여야 한다. 현재는 생성된 응답을 그대로 반환

```ts
// amplify/data/sendMessage.ts
// This handler simply passes through the arguments of the mutation through as the result
export function request() {
  return {};
}

/**
 * @param {import('@aws-appsync/utils').Context} ctx
 */
export function response(ctx: any) {
  return ctx.args;
}
```

```ts
// amplify/data/watchMessage.ts
export function request() {
  return {};
}

export const response = (ctx: any) => {
  return ctx.result;
};
```

# front(flutter)에서 요청 전송

GraphQL API, Amplify를 이용하여 백엔드에게 요청을 전송하고 응답을 받는 구조이다.

```dart
// lib/models/chat_repository.dart의 일부
Future<void> sendMessage(
    {required roomId, required sender, required payload}) async {
    final request = GraphQLRequest<String>(
        document: '''
        mutation SendMessage(\$roomId, \$sender, \$payload) {
            sendMessage(
                roomId: \$roomId,
                sender: \$sender,
                payload: \$payload) {
                id
                roomId
                sender
                payload {
                    type
                    text
                    downloadUrl
                    storagePath
                }
            }
        }
        ''',
        variables: {
            'roomId': roomId,
            'sender': sender,
            'payload': payload,
        },
    );
    final response = await Amplify.API.mutate(request: request).response;
    if (response.hasErrors) {
        safePrint('Creating Todo failed.');
    } else {
        safePrint('Creating Todo successful.');
    }
}

Stream<ChatMessage> watchMessages(String roomId) {
    final request = GraphQLRequest<String>(
        document: '''
        subscription OnCreateMessage(\$roomId) {
            onCreateMessage(roomId: \$roomId) {
                id
                roomId
                sender
                payload {
                    type
                    text
                    downloadUrl
                    storagePath
                }
            }
        }
        ''',
        variables: {
            'roomId': roomId,
        },
    );

    final Stream<GraphQLResponse> rawStream = Amplify.API.subscribe(
        request,
        onEstablished: () => safePrint('Subscription established'),
    );

    // Stream<GraphQLResponse> → Stream<Message>로 매핑
    return rawStream
        .where((result) => result.data != null)
        .map((result) => ChatMessage.fromJson(result.data!['onCreateMessage']));
}
```

```dart
// lib/viewmodels/chat/chatRoomPage.dart
// ViewModel 초기화 시 stream 구독

Future<void> init() async {
    _messages = await _chatRepo.fetchMessages(_pairId, limit: 50);
    _subscription = _chatRepo.watchMessages(_pairId).listen((message) {
        _messages.insert(0, message);
        notifyListeners();
    });
}

```

```dart
// lib/viewmodels/chat/chatRoomPage.dart
// 메세지 전송 버튼 클릭 시 호출되는 함수

Future<void> sendMessage(String payload, ChatMessagePayloadType type) async {
    UploadResult mediaResult;
    ChatMessage msg;

    msg = ChatMessage(
        pairId: _pairId,
        fromUid: _uid,
        payload: ChatMessagePayload(
            type: ChatMessagePayloadType.text,
            text: sanitized,
        ),
        sentAt: TemporalDateTime.now(),
    );

    try {
        await _chatRepo.sendMessage( roomId: _pairId, sender: _uid, payload: msg.payload);
    } catch (e) {
        rethrow;
    }
}
```
