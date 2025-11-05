---
layout: post
title: "Chat 관련 스키마 정리"
date: 2025-11-06 06:00:00 +0900
description: >
  채팅 관련 데이터베이스 스키마의 설계에 대한 설명입니다.
categories: [pick]
tags: [amplify, graphql]
---

## AWS Amplify 기반 채팅 모델 구조 설계 패턴

ChatMessage의 식별자는 id로, ChatMessage를 구성하는 하위 모델은 messageId라는 이름을 설계하였다.

Amplify 코드 환경 기준.

---

### 1. 핵심 모델 구조 개요

```ts
// ChatMessage: 채팅방 내 실제 메시지(1행 = 1개 메시지)
ChatMessage: a.model({
  id: a.id().required(),                // PK, 고유 메시지 ID
  roomId: a.id().required(),            // 채팅방 FK
  room: a.belongsTo("ChatRoom", "roomId"),
  sentFrom: a.id().required(),          // 보낸 유저 id
  content: a.hasOne("MessageContent", "messageId"),  // 메시지 컨텐츠와 1:1 연결
  reads: a.hasMany("ReadBy", "messageId"),     // 메시지별 읽음 기록(복수)
  owner: a.string().authorization((allow) => [allow.owner().to(["read", "delete"])]),
})
.identifier(["id"])
.authorization((allow) => [allow.owner()]),

// MessageContent: 메시지 실제 데이터(텍스트, 파일, 타입 등)
MessageContent: a.model({
  messageId: a.id().required(),               // PK, hasOne/hasMany시 messageId 기준 통일
  message: a.belongsTo("ChatMessage", "id"),  // ChatMessage 1:1 연결 (참조)
  type: a.enum(["TEXT", "IMAGE", "AUDIO", "VIDEO"]),
  text: a.string(),
  downloadUrl: a.string(),
  storagePath: a.string(),
  owner: a.string().authorization((allow) => [allow.owner().to(["read", "delete"])]),
})
.identifier(["messageId"])
.authorization((allow) => allow.owner()),

// ReadBy: 누가 이 메시지를 언제 읽었는지 기록
ReadBy: a.model({
  readAt: a.datetime().required(),      // 읽은 시각(타임스탬프)
  readFrom: a.id().required(),          // 읽은 유저 id
  messageId: a.id().required(),         // 메시지 FK
  message: a.belongsTo("ChatMessage", "messageId"), // ChatMessage와 N:1 (복수 읽음)
  owner: a.string().authorization((allow) => [allow.owner().to(["read", "delete"])]),
})
.authorization((allow) => [allow.owner().to(["read", "delete"])]),
```

---

### 1. ChatMessage

- 하나의 채팅방 메시지를 표현하는 메인 테이블.
- 필드 설명:
  - `id`: PK, 고유 메시지 식별자 (서버에서 자동 생성 or 명시 입력)
  - `roomId`: FK, 채팅방(채팅룸)과 연결
  - `room`: 채팅방 객체 참조 (belongsTo)
  - `sentFrom`: 메시지를 보낸 유저의 id
  - `content`: MessageContent와 1:1 연결 → 해당 메시지의 실제 컨텐츠 데이터와 연결
  - `reads`: 여러 ReadBy와 1:N (hasMany) 관계 → 누가 이 메시지를 읽었는지 추적
  - `owner`: 인증/권한용, 보통 메시지 소유자

---

### 2. MessageContent

- 메시지 본문의 실제 데이터(텍스트, 파일 경로 등)를 분리 저장
- 필드 설명:
  - `messageId`: PK, ChatMessage의 id와 연결되는 1:1 관계 (즉, 한 메시지당 하나의 컨텐츠)
  - `message`: ChatMessage 객체 참조 (belongsTo)
  - `type`: 메시지 컨텐츠 타입 (TEXT, IMAGE 등)
  - `text`, `downloadUrl`, `storagePath`: 실제 컨텐츠 데이터
  - `owner`: 인증/권한용

---

### 3. ReadBy

- 어떤 유저가 어느 메시지를 언제 읽었는지 기록하는 로그성 테이블
- 필드 설명:
  - `readAt`: 읽은 시각(타임스탬프)
  - `readFrom`: 읽은 유저 id
  - `messageId`: FK, 읽은 메시지의 id (ChatMessage와 연결)
  - `message`: ChatMessage 객체 참조 (belongsTo)
  - `owner`: 인증/권한용

---

### 관계 요약

- **ChatMessage : MessageContent**는 1:1 구조 - messageId(=ChatMessage.id)로 연결, 각각 독립적으로 생성
- **ChatMessage : ReadBy**는 1:N 구조 - 여러 ReadBy가 하나의 메시지를 참조. 메시지 읽을 때마다 새로운 ReadBy를 생성
- FK(외래키)는 각 모델의 id나 messageId로 통일
- 모든 주요 참조에는 belongsTo/hasOne/hasMany로 관계를 명시

---

### 실사용 흐름

1. 메시지 전송:
   - ChatMessage를 생성하고, 이어서 MessageContent도 별도 생성 (자동으로 둘 다 만들어지지 않음!)
   - 트랜잭션성 요구(동시 insert 등)는 custom mutation으로 처리
2. 메시지 읽음:
   - 해당 메시지를 읽은 유저마다 별도의 ReadBy를 생성해준다.
3. 메시지/읽음/컨텐츠 연결은 FK로 일관되게 관리, 권한은 owner 또는 ownersDefinedIn을 이용

### 해당 설계의 장점

1.  데이터 무결성과 일관성, 확장성의 확보
    메시지, 메시지 컨텐츠, 읽음 기록을 모두 별도 모델로 분리

        각 개체(메시지 자체, 실제 컨텐츠, 누가 읽었는지 등을 기록하는 로그)를 분리하면, 데이터의 의미와 책임이 명확하게 구분된다. 모델별로 책임이 분리되어 유지보수, 확장, 트래픽 최적화가 쉽다.

    PK와 FK 관계를 명확하게 정의

         각 모델이 고유 식별자를 기준(id/messageId)으로 연결되도록 설계한다. 이는 참조 무결성, 쿼리 명확화, 관계형 모델의 강점을 살리기 위함이다.

2.  GraphQL 기반 실시간/관계형 쿼리 최적화
    관계형 패턴(hasOne, hasMany, belongsTo) 적극 활용

         관계를 모델 단에서 명확하게 선언해두면, GraphQL 레이어에서 내부적으로 연관 데이터를 효율적으로 조인·조회할 수 있다.

        실제로 메시지 조회 시 content/reads를 함께 fetch하거나, 관계형 subscription 구현 등에서도 구조가 매우 자연스럽다.

    hasMany/hasOne/PK 기반 관계 필드 통일

        FK명이 PK명과 맞아 떨어져야 쿼리 자동화, type-safe한 코드 생성, amplify helper 등에서 에러 없는 쿼리 합성이 가능하다.

3.  API 활용성과 비즈니스 요구 충족
    읽음(누가, 언제 읽었는지) 정보의 세분화 저장

        ReadBy 모델을 별도로 두어 다수 유저의 읽음기록(메시지 1 : 다수 읽음 로그)을 유연하게 관리한다.

        메시지별 '누가 읽었는가' 뿐 아니라, 통계/알림/구체적 UX 개선까지 유연하게 확장할 수 있다.

    컨텐츠 분리(멀티미디어 등 다양한 메시지 속성 저장)

        MessageContent 별도 분리로, 텍스트/이미지/오디오/다운로드URL 등 확장성 있게, 메시지와 분리된 상태로 다양한 컨텐츠 저장/관리 가능.

4.  보안, 권한, 데이터 관리의 최적화
    권한 필드(owner 등)와 인증 전략 명확화

        모델별로 owner/ownersDefinedIn을 명확하게 지정해, 각 객체의 읽기/삭제/통합 권한을 세분화 관리하면서도 보안 표준을 유지.

    PK(id) 통일 및 입력 required 필드 분리

        모든 모델 PK를 id로 표준화해 혼동을 원천 차단하고, 누락/중복 없이 쿼리에서 일관성 확보.

[1](https://docs.amplify.aws/react/build-a-backend/data/data-modeling/)
