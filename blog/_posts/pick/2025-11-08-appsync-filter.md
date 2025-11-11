---
layout: post
title: "AppSync Filter"
date: 2025-11-08 20:00:00 +0900
description: >
  AppSync JS Resolver Subscription 필터링에 대해서 정리합니다.
categories: [pick]
tags: [amplify, graphql, appsync]
---

> AppSync를 통한 graphQL API에서 subsciption을 커스텀하는 방법과 원하는 대상에게만 구독 알림을 전송하는 필터에 대해서 알아봅시다.

## AppSync 요청이 처리되는 간단한 흐름

해당 예제에서는 DynamoDB와의 소통이 이루어지는 API를 예시로 다루겠습니다.

우선 스키마를 정의해야 합니다.
watchMessage 구독은 Message가 생성될 때 트리거가 발동하며 생성된 Message 객체를 구독중인 사용자에게 전송합니다.

```typescript
// amplify의 스키마 정의법에 따름
Message: a.model({
    text: a.string().required(),
    receiver: a.id().required(),
    sentFrom: a.id().required(),
})

watchMessage: a
    .subsciption()
    .for(a.ref("Message").mutations(["create"]))
    .arguments(
        userId: a.id().required(),
    )
    .handler(
        a.handler.custom({
            entry: "watchMessageHandler.js",
        })
    )
```

이럴 경우, Message가 생성될때 마다 구독 트리거가 발생하게 되고 필터링이 없다면 내가 속해있지 않은 채팅방에 대한 메세지도 모두 전송받는 상황이 발생하게 됩니다.

따라서 우리는 구독 API에 userId를 인자로 전달받아 해당 message의 recevier가 userId와 같은 경우에만 메세지를 전송하도록 필터링을 추가하여야 합니다.

## AppSync JS Resolver 작성 (필터링)

현재는 간단한 예제이므로 핸들러를 스키마를 정의한 디렉토리와 같은 위치에 정의하겠습니다.

[참고 문서][aws-appsync/js-resolver]

subscription의 필터를 등록하는 handler는 반드시 다음 내용을 지켜야합니다.

- `request()` => `return ( {payload: null })`
- `reponse()` => `return null`

```js
// watchMessageHandler.js
import { util } from "@aws-appsync/utils";

export function reqeust() {
  paydload: null;
}

export function response(ctx) {
  const filter = {
    receiver: {
      eq: ctx.argument.userId,
    },
  };
  extensions.setSubscriptionFilter(util.transform.toSubscriptionFilter(filter));
  return null;
}
```

## 필터 표현식

필터는 다음 형태의 구조로 정의됩니다.
[공식 문서][aws-appsync/filter-extension]

```js
{
    <field>: {
        <operator>: <value>
    }
}
```

field: 조건 검사 기준이 되는 필드를 명시한다.
expression: 어떤 조건으로 필터링을 진행할지 명시한다.

### operator

- `eq(equal)`, `ne(not equal)`

```js
//schema
message = a.model({
  text: a.string().required(),
  receiver: a.id().required(),
  sentFrom: a.id().required(),
});

const filter = {
  recevier: {
    eq: userId,
  },
};
```

- `gt(greater than)`, `ge(greater than or equal)`, `lt(less than)`, `le(less than or equal)`

- `in`, `notIn`
  field의 **값**이 **배열 value**에 속하는지를 검사

```js
// UserA 혹은 UserB로부터 온 메세지인지 확인
const filter = {
  sentFrom: {
    in: ["UserA", "UserB"],
  },
};
```

- `between`

```js
// cnt가 1~10사이라면 수신 (임의 예시)
const filter = {
  cnt: {
    between: [1, 10],
  },
};
```

### function

- `beginsWith`
  field의 값이 value의 값으로 시작하는지 검사

```js
// Hello로 시작하는 메세지라면 수신
const filter = {
  text: {
    beginsWith: "Hello",
  },
};
```

- `contains`, `notContains`, `containsAny`
  field의 배열 속에 value의 값이 속하는지 검사

```js
// schema
message = a.model({
  text: a.string(),
  sentFrom: a.id(),
  participants = a.id().array(),
});

// 내 userId가 참가자 배열에 속해있으면 수신
const filter = {
  participants: {
    contains: userId,
  },
};
```

추가적으로 or 연산자 and연산자들을 이용하여 더 복합적인 조건 검사도 가능하다.

```js
// 내 userId가 참가자 배열에 속해있고 보낸 사람이 내가 아니라면 우신
const filter = {
  participants: { contains: userId },
  sentFrom: { ne: userId },
};
```

```js
// 내 userId가 참가자 배열에 속해있거나 보낸 사람이 나라면 수신
const filter = {
  or: [
    participants: { contains: userId },
    sentFrom: { ne: userId },
  ]
};
```

[aws-appsync/js-resolver](https://docs.aws.amazon.com/ko_kr/appsync/latest/devguide/aws-appsync-real-time-enhanced-filtering.html)
[aws-appsync/filter-extension](https://docs.aws.amazon.com/appsync/latest/devguide/extensions.html)
