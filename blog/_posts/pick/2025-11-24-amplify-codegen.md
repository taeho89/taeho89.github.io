---
layout: post
title: "개발 중 버그 정리"
date: 2025-11-23 01:16:00 +0900
description: >
categories: [pick]
tags: [amplify, data, schema]
---

### GraphQL Input 타입 오류(NonInputTypeOnVariable)

**오류 메세지**

```bash
Validation error of type NonInputTypeOnVariable: Wrong type for a variable
```

**문제 상황**
프론트에서 다음과 같은 mutation 쿼리문을 작성했는데, 위와 같은 오류가 발생했다.

```dart
// 실제 요청
final request = GraphQLRequest<ChatMessage>(
    document: '''
        mutation SendMessage(\$input: SendMessageInputInput!) {
        sendMessage(input: \$input) {
            id
            receiver
            sentFrom
            content {
            messageId
            type
            text
            downloadUrl
            storagePath
            }
            createdAt
            updatedAt
        }
        }
        ''',
    variables: {'input': input},
    modelType: ChatMessage.classType,
    decodePath: "sendMessage",
    );
```

이때의 스키마 정의는 아래와 같이 하였다.

```ts
// Amplify Data 정의 스키마
SendMessageInput: a.customType({
        roomId: a.id().required(),
        receiver: a.id().required(),
        type: a.ref("MessageContentType").required(),
        text: a.string(),
        downloadUrl: a.string(),
        storagePath: a.string(),
    }),
sendMessage: a
    .mutation()
    .arguments({ input: a.ref("SendMessageInput").required() })
    .returns(a.ref("ChatMessage"))
    .handler(a.handler.function(sendMessageFn))
    .authorization((allow) => [allow.authenticated()]),
```

당연히 `SendMessageInput` 타입을 명시하면 정상동작 할줄 알았으나,

- 실제 생성된 GraphQL 스키마
  ![alt text](image-11.png)

해당 오류가 계속 발생하여 실제 생성된 스키마를 살펴보니 `SendMessageInput`은 `type`으로 정의된다.
하지만, GraphQL 스펙에서 mutation 변수 타입은 반드시 `input`타입이어야 한다.

**해결**
GraphQL 변수 타입에 `SendMessageInputInput`을 사용

혹은 변수를 나열하여 사용

```dart
final request = GraphQLRequest<ChatMessage>(
    document: '''mutation SendMessage(
        $roomId: ID!,
        $receiver: ID!,
        $type: MessageContentType!,
        $text: String,
        $downloadUrl: String,
        $storagePath: String
    ) {
        sendMessage(
            input: {
                roomId: $roomId,
                receiver: $receiver,
                type: $type,
                text: $text,
                downloadUrl: $downloadUrl,
                storagePath: $storagePath
            }
        ) {
            id
            receiver
            sentFrom
            content {
            messageId
            type
            text
            downloadUrl
            storagePath
            }
            createdAt
            updatedAt
        }
        }''',
    variables: {
        'roomId': input['roomId'],
        'receiver': input['receiver'],
        'type': input['type'],
        'text': input['text'],
        'downloadUrl': input['downloadUrl'],
        'storagePath': input['storagePath'],
    },
    modelType: ChatMessage.classType,
    decodePath: "sendMessage",
);
```

---
