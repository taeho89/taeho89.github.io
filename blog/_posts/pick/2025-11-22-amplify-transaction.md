---
layout: post
title: "Amplify 트랜잭션 커스텀 mutate"
date: 2025-11-21 00:21:00 +0900
description: >
  Amplify를 통해 트랜잭션을 수행하는 커스텀 Mutation을 작성해봅시다.
categories: [pick]
tags: [amplify, data, resolver, lambda]
---

내 서비스의 채팅메시지의 구조는 아래와 같이 설계되었다.

```ts
MessageContent: a
    .model({
      messageId: a
        .id()
        .required()
        .authorization((allow) => allow.authenticated().to(["read"])),
      message: a
        .belongsTo("ChatMessage", "messageId")
        .authorization((allow) => allow.authenticated().to(["read"])),
       text: a.string(),
      owner: a
        .string()
        .authorization((allow) => [
          allow.owner().to(["read", "delete", "create"]),
        ]),
    })
    .identifier(["messageId"])
    .authorization((allow) => [allow.owner(), allow.authenticated()]),

  ChatMessage: a
    .model({
      id: a
        .id()
        .required()
        .authorization((allow) => allow.authenticated().to(["read"])),
      roomId: a
        .id()
        .required()
        .authorization((allow) => allow.authenticated().to(["read"])),
      room: a.belongsTo("ChatRoom", "roomId"),
      receiver: a
        .id()
        .required()
        .authorization((allow) => allow.authenticated().to(["read"])),
      sentFrom: a
        .id()
        .required()
        .authorization((allow) => allow.authenticated().to(["read"])),
      content: a.hasOne("MessageContent", "messageId"),
      // 메시지 생성 시점으로 정렬된 결과를 받기 위해 createdAt 기준 정렬 사용
      createdAt: a.datetime().required(),
      updatedAt: a.datetime().required(),
      owner: a
        .string()
        .authorization((allow) => [
          allow.owner().to(["read", "delete", "create"]),
        ]),
    })
    .identifier(["id"])
    .secondaryIndexes((index) => [index("roomId").sortKeys(["createdAt"])])
    .authorization((allow) => [allow.owner(), allow.authenticated()]),

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
        .handler(a.handler.custom(
            entry: "./sendMessageHandler.js",
            dataSource: a.ref("ChatMessage")
        ))
        .authorization((allow) => [allow.authenticated()])

```

이렇게 메세지와 메세지의 내용을 분리하여 설계를 했기에 메세지 전송 메소드에서 두 개의 테이블에 대해서 객체를 생성할 필요가 있었다.
처음에는 Lambda의 요금을 최소화시키기 위해 AppSync JS Resolver를 통해 아래와 같이 구현을 진행했었다.

**참고 자료**
[DynamoDB용 AWS AppSync 해석기 매핑 템플릿 참조](https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/js-resolver-reference-dynamodb.html)
[AWS AppSync JavaScript 해석기 컨텍스트 객체 참조](https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/resolver-context-reference-js.html)
[해석기 및 함수를 위한AWS AppSync JavaScript 런타임 기능](https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/resolver-util-reference-js.html)

**주의할 점**

- owner, createdAt, updatedAt 기록
- DynamoDB 테이블이름을 명시할 때, AppSync가 파이프라인을 통해 기록해주는 `awsAppsyncApiId`, `amplifyApiEnvironmentName`을 읽어와 실제 Sandbox에 배포된 리소스의 이름을 만들어야 한다. (`"<테이블 이름>-<ApiID>-<EnvName>"`)

```ts
// sendMessage
import { util } from "@aws-appsync/utils";

export function request(ctx) {
  const { input } = ctx.arguments;
  const userId = ctx.identity.sub; // context(ctx) 객체의 identity를 통해 사용자의 UUID 가져오기

  // argument 검증
  if (!input.roomId || !userId || !input || !input.type) {
    util.error("Missing required arguments: roomId, userId, content.type", "BadRequest");
  }

  // util 런타임 기능을 이용하여 고유 UUID, 서버측 현재 시간 기록
  const now = util.time.nowISO8601();
  const messageId = util.autoId();

  // 실제 데이터베이스 저장될 객체(Item) 생성
  const messageContentItem = {
    messageId: messageId,
    text: input.text || null,
    owner: userId,
    createdAt: now,
    updatedAt: now,
  };
  const chatMessageItem = {
    id: messageId,
    roomId: input.roomId,
    receiver: input.receiver,
    sentFrom: userId,
    content: messageContentItem,
    owner: userId,
    createdAt: now,
    updatedAt: now,
  };

  ctx.stash.content = messageContentItem;
  ctx.stash.chatMessage = chatMessageItem;

  // AppSync API ID 및 환경 이름
  const apiId = ctx.stash.awsAppsyncApiId;
  const env = ctx.stash.amplifyApiEnvironmentName;
  const chatMessageTable = `ChatMessage-${apiId}-${env}`;
  const chatContentTable = `MessageContent-${apiId}-${env}`;

  // TransactWriteItems용 요청 객체 반환
  return {
    operation: "TransactWriteItems",
    transactItems: [
      {
        table: chatMessageTable,
        operation: "PutItem",
        key: util.dynamodb.toMapValues({ id: messageId }),
        attributeValues: util.dynamodb.toMapValues(chatMessageItem),
      },
      {
        table: chatContentTable,
        operation: "PutItem",
        key: util.dynamodb.toMapValues({ messageId: messageId }),
        attributeValues: util.dynamodb.toMapValues(messageContentItem),
      },
    ],
  };
}

export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }
  // stash에서 생성한 데이터를 조합하여 반환
  return { ...ctx.stash.chatMessage, content: ctx.stash.content };
}
```

이렇게 하면 될 줄 알았으나.. 권한의 문제가 발생했다.

다시 한번 sendMessage 스키마 정의를 살펴보자.
custom JS Resolver를 붙이기 위한 `handler.custom` 메소드는 `dataSource`를 하나밖에 정의하지 못한다.
dataSource를 정의해야만 AppSync가 해당 작업을 수행하는 객체에서 테이블에 대한 리소스를 제공해준다.

```ts
sendMessage: a
    .mutation()
    .arguments({ input: a.ref("SendMessageInput").required() })
    .returns(a.ref("ChatMessage"))
    .handler(a.handler.custom(
        entry: "./sendMessageHandler.js",
        dataSource: a.ref("ChatMessage")
    ))
    .authorization((allow) => [allow.authenticated()])
```

위와 같이 선언 시에 sendMessage를 실행하는 Role에는 다음과 같은 정책이 설정된다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "dynamodb:BatchGetItem",
        "dynamodb:BatchWriteItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:Scan",
        "dynamodb:Query",
        "dynamodb:UpdateItem",
        "dynamodb:ConditionCheckItem",
        "dynamodb:DescribeTable",
        "dynamodb:GetRecords",
        "dynamodb:GetShardIterator"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/ChatMessage-<ApiID>-<EnvName>",
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/ChatMessage-<ApiID>-<EnvName>/*"
      ],
      "Effect": "Allow"
    }
  ]
}
```

JS Resolver를 통해 트랜잭션을 수행하기 위해서는 Resource 부분에 MessageContent 리소스에 대한 권한도 추가해주어야 했다.

```json
"Resource": [
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/ChatMessage-<ApiID>-<EnvName>",
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/ChatMessage-<ApiID>-<EnvName>/*",
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/MessageContent-<ApiID>-<EnvName>",
        "arn:aws:dynamodb:us-east-1:<계정 번호>:table/MessageContent-<ApiID>-<EnvName>/*",
      ],
```

## Lambda를 이용하여 권한 문제 해결

하지만 매번 Sandbox를 재실행할때마다 콘솔에서 위와 같은 작업을 반복하는 것은 비효율적이라고 판단이 되어 해당 권한 문제를 해결하기 위해 Lambda를 이용하기로 했다.

우선 `amplify/functions/sendMessage` 디렉토리를 생성하고 디렉토리 하위에 `handler.ts`, `resource.ts` 파일을 생성하자.

### `resource.ts` 작성 및 권한 추가

Lambda를 이용하게 되면 아래와 같이 CDK를 통해 lambda 리소스의 정책을 직접 관리할 수 있기 때문에 위와 같은 권한 문제를 해결할 수 있었다.

```ts
///amplify/functions/sendMessage/resource.ts
import { defineFunction } from "@aws-amplify/backend";

export const sendMessageFn = defineFunction({
  name: "sendMessageFn",
  entry: "./handler.ts",
  resourceGroupName: "data",
});

///amplify/backend.ts
const backend = defineBackend({
  auth,
  data,
  sendMessageFn,
});

// Lambda 함수에 DynamoDB 테이블 쓰기 권한 부여 및 환경 변수 설정
const chatMessageTable = backend.data.resources.tables["ChatMessage"];
const messageContentTable = backend.data.resources.tables["MessageContent"];

backend.sendMessageFn.addEnvironment("CHAT_MESSAGE_TABLE_NAME", chatMessageTable!.tableName);

backend.sendMessageFn.addEnvironment("MESSAGE_CONTENT_TABLE_NAME", messageContentTable!.tableName);

chatMessageTable?.grantWriteData(backend.sendMessageFn.resources.lambda);
messageContentTable?.grantWriteData(backend.sendMessageFn.resources.lambda);
```

- `resourceGroupName`
  해당 lambda 리소스가 어느 stack에 포함시킬지를 결정하는 변수.
  Amplify는 사용자가 따로 stack을 명시하지 않으면 data, auth, functions라는 이름을 가진 stack에 해당 리소스들을 위치시킨다.
  때문에 해당 필드를 명시하지 않으면 Lambda 리소스와 Data 리소스가 다른 stack에 존재하게 되어 순환 참조 오류가 발생한다. (data stack은 function stack을 참조하고, function stack은 data stack을 참조하게 된다.)

```bash
[ERROR] [CloudformationStackCircularDependencyError] The CloudFormation deployment failed due to circular dependency found between nested stacks [data7552DF31, function1351588B]
Resolution: If you are using functions then you can assign them to existing nested stacks that are dependent on functions or functions depend on them, for example:
1. If your function is defined as auth triggers, you should assign this function to auth stack.
2. If your function is used as data resolver or calls data API, you should assign this function to data stack. To assign a function to a different stack, use the property 'resourceGroupName' in the defineFunction call and choose auth, data or any custom stack.
```

---

### `handler` 작성

AppSync JS Resolver를 이용할 때에는 Amplify가 `util.dynamodb.toMapValues` 메소드를 통해 요청을 DynamoDB 표현식으로 매핑 해주었지만, Lambda는 직접 DynamoDB에 요청을 보내기 때문에 해당 표현식으로 작성을 해야한다는 점이 존재했다.

**참고 자료**
[형식 시스템(요청 매핑)](https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/js-aws-appsync-resolver-reference-dynamodb-typed-values-request.html)

```ts
//amplify/functions/sendMessage/handler.ts
import { DynamoDBClient, TransactWriteItemsCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({});

const CHAT_MESSAGE_TABLE = process.env["CHAT_MESSAGE_TABLE_NAME"];
const MESSAGE_CONTENT_TABLE = process.env["MESSAGE_CONTENT_TABLE_NAME"];

type AppSyncEvent = {
  arguments: {
    input: {
      roomId: string;
      receiver: string;
      type: string;
      text?: string;
      downloadUrl?: string;
      storagePath?: string;
    };
  };
  identity?: {
    sub?: string;
    username?: string;
  };
};

export const handler = async (event: AppSyncEvent) => {
  const { input } = event.arguments;
  const senderId = event.identity?.sub ?? event.identity?.username ?? null;
  if (!senderId) {
    throw new Error("Unauthorized: senderId is required");
  }

  const now = new Date().toISOString();
  const messageId = crypto.randomUUID();

  const messageContentItem = {
    messageId: { S: messageId },
    text: input.text ? { S: input.text } : { NULL: true },
    owner: { S: senderId },
    createdAt: { S: now },
    updatedAt: { S: now },
  };

  const chatMessageItem = {
    id: { S: messageId },
    roomId: { S: input.roomId },
    receiver: { S: input.receiver },
    sentFrom: { S: senderId },
    content: { S: JSON.stringify(messageContentItem) },
    createdAt: { S: now },
    updatedAt: { S: now },
    owner: { S: senderId },
  };

  const transactItems = [
    {
      Put: {
        TableName: MESSAGE_CONTENT_TABLE,
        Key: { messageId: { S: messageId } },
        Item: messageContentItem,
        ConditionExpression: "attribute_not_exists(messageId)",
      },
    },
    {
      Put: {
        TableName: CHAT_MESSAGE_TABLE,
        Key: { id: { S: messageId } },
        Item: chatMessageItem,
        ConditionExpression: "attribute_not_exists(id)",
      },
    },
  ];

  try {
    await ddb.send(
      new TransactWriteItemsCommand({
        TransactItems: transactItems,
      })
    );
  } catch (error) {
    console.error("Error sending message:", error);
    throw new Error("Failed to send message");
  }

  return {
    id: messageId,
    roomId: input.roomId,
    receiver: input.receiver,
    sentFrom: senderId,
    content: {
      id: messageId,
      text: input.text,
      owner: senderId,
      createdAt: now,
      updatedAt: now,
    },
    owner: senderId,
    createdAt: now,
    updatedAt: now,
  };
};
```

---

**Issue**
처음 Lambda 핸들러의 반환값을 작성할 때 owner 필드를 빼먹고 작성을 하였었다.
때문에 해당 Mutation 호출 시 owner필드가 없기에 AppSync에서 Unauthorized 오류를 발생시켰다.

```bash
Error sending message: [
  {
    path: [ 'sendMessage', 'owner' ],
    data: null,
    errorType: 'Unauthorized',
    errorInfo: null,
    locations: [ [Object] ],
    message: 'Not Authorized to access owner on type ChatMessage'
  }
]
```

해당 오류를 처음 목격했을때, `owner` 필드가 잘못 기재되어 문제가 발생하는줄 알았으나 그냥 반환값에 해당 필드가 빠져있었다.
AppSync는 `owner` 필드에 **field-level authrization**이 적용되어 있기 때문에 현재 사용자와 반환받은 `owner`를 비교하여 권한 체크를 진행하는데, Lambda 핸들러의 반환값으로부터 `owner` 필드가 `null`로 돌아왔기때문에 일치하지 않은 사용자로 판단하여 해당 오류를 뱉은 것이라고 판단했다.
