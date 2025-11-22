---
layout: post
title: "Amplify Owner 권한과 필드 레벨 Authorization 경고 해결하기"
date: 2025-11-23 01:16:00 +0900
description: >
  Amplify Gen 2에서 owner 기반 권한을 사용할 때 발생하는 권고 메시지와 그 해결 방법을 알아봅시다.
categories: [pick]
tags: [amplify, data, authorization]
---

Amplify Data(Model)에서 `allow.owner()` 규칙을 사용하면, 사용자가 자신이 생성한 레코드를 읽고/수정하고/삭제할 수 있는 기능을 쉽게 적용할 수 있다.  
그런데 모델 안에 특정 필드(`owner`, `memberIds` 등)가 존재할 때, Amplify CLI는 소유권 재할당(Ownership Reassignment) 가능성을 경고하기도 한다.

아래는 `pnpm ampx sandbox` 또는 `ampx deploy` 과정에서 흔히 볼 수 있는 경고 메시지다.

```

WARNING: owners may reassign ownership for the following model(s) and role(s):
ChatRoom: [memberIds], ChatRoomMember: [owner], MessageContent: [owner].
If this is not intentional, you may want to apply field-level authorization rules to these fields.

```

이 글에서는 이 경고가 왜 나타나는지, 어떤 위험을 의미하는지, 그리고 어떻게 해결해야 안전한 권한 구성을 만들 수 있는지 정리해본다.

---

## 왜 경고가 발생하는가?

`allow.owner()`는 모델 전체에 대해 다음과 같은 권한을 부여한다.

- create
- read
- update
- delete

즉, 해당 레코드의 owner라면 update 요청을 보내 필드 값을 변경할 수 있게 된다.  
문제가 되는 부분은 모델 안에 다음과 같은 민감한 필드가 있을 때이다.

- `owner`
- `memberIds`
- `createdBy`
- `assignedTo`

이 필드들이 update 가능한 상태라면, 사용자가 자신의 레코드를 업데이트하면서 다음과 같은 동작을 수행할 수 있게 된다.

```graphql
updateMessageContent(
  id: "message-1",
  owner: "another-user-id"
)
```

즉, "이 레코드는 이제 다른 사람이 소유한 것으로 취급해라"라는 의미가 되며, 대부분의 서비스에서는 의도하지 않은 보안 위험이다.
Amplify는 이러한 가능성을 감지하면 경고 메시지를 출력한다.

---

## delete 권한은 왜 문제 없는가?

여기서 중요한 점은 다음과 같다.

**필드를 업데이트(update)하는 것과 레코드를 삭제(delete)하는 것은 완전히 다른 동작이다.**

- update → 특정 필드 값을 변경
- delete → 레코드 전체를 제거

특정 필드에 `read`만 허용되어 있어도, 레코드를 삭제할 권한이 있다면 해당 필드 값은 단순히 함께 사라질 뿐이며, 이는 보안상 문제되지 않는다.
경고 메시지가 문제 삼는 것은 delete가 아니라 update를 통한 필드 조작이다.

---

## 해결 방법

### 1. 필드 레벨 authorization을 통해 update 차단하기

가장 명확한 해결책은 문제될 수 있는 필드에 대해 update 권한을 제거하는 것이다.

```ts
owner: a.id().authorization((allow) => [
  allow.owner().to(["read"]), // update 차단
]),
```

이렇게 하면:

- owner는 레코드를 생성/삭제할 수 있으나
- owner 필드 자체는 더 이상 수정할 수 없다

따라서 소유권 재할당 문제가 해결된다.

---

### 2. 클라이언트의 mutation input에서 owner 필드를 제거하기

많은 서비스에서는 owner 값을 클라이언트가 직접 지정하지 않는다.
AppSync Function Resolver나 Lambda에서 로그인한 사용자 정보를 기반으로 자동 세팅하는 방식이 더 안전하다.

이 경우 owner 필드에 대한 update 자체를 클라이언트 입력에서 제거함으로써 문제를 원천 차단할 수 있다.

---

### 3. ChatRoom.memberIds 같은 필드는 서버에서만 관리하기

특히 `memberIds`는 방 참여자 관리의 핵심 필드이므로, 클라이언트 측에서 조작할 수 없도록 하고 서버 로직(Resolver)에서만 변경하도록 관리하는 것을 권장한다.

---

## 권장되는 스키마 예시

아래는 위 내용을 반영한 안전한 권한 구성 예시이다.

```ts
MessageContent: a.model({
  messageId: a.id().required(),
  message: a.belongsTo("ChatMessage", "messageId"),
  type: a.ref("MessageContentType").required(),
  text: a.string(),
  downloadUrl: a.string(),
  storagePath: a.string(),

  owner: a.id().authorization((allow) => [
    allow.owner().to(["read"]), // update 차단
  ]),
})
  .identifier(["messageId"])
  .authorization((allow) => [
    allow.owner(), // 레코드 전체에 대한 CRUD 권한
    allow.authenticated().to(["read"]), // 로그인 사용자는 read만 가능
  ]);
```

해당 구성은 다음 조건을 만족한다.

- owner는 레코드를 생성하고 삭제할 수 있다.
- owner 필드 자체는 변경할 수 없다.
- 인증된 모든 사용자는 read만 가능하다.
- Amplify의 소유권 재할당 경고가 사라진다.

---

## 정리

Amplify의 “소유권 재할당 경고”는 update를 통한 민감 필드 조작 위험을 알리는 메시지이며, delete 권한과는 직접적인 연관이 없다.
필드 레벨 authorization을 추가해 update를 제한하면 문제는 해결되며, 더 안전한 데이터 구조를 구현할 수 있다.

---

**참고자료**
[Per-user & owner-based authorization – AWS Amplify Documentation](https://docs.amplify.aws/cli/graphql/authorization-rules/#per-user--owner-based-data-access)
