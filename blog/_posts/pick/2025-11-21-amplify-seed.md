---
layout: post
title: "How to use Amplify Seed"
date: 2025-11-21 00:21:00 +0900
description: >
  Amplify Sandbox Seed를 이용하여 시드를 생성하는 법을 배워봅시다.
categories: [pick]
tags: [amplify, sandbox, seed]
---

Amplify Sandbox는 Seed를 이용하여 시드 데이터를 추가하는 기능을 지원한다.
우선 seed 기능을 이용하기 위한 seed 디렉토리 및 seed.ts 파일을 생성하고, 권한 정책 설정을 진행해주어야 한다.

### 필수 디렉토리 및 파일 생성 후 sandbox 실행

이 seed.ts 파일이 sandbox 환경에서 `ampx sandbox seed` CLI 명령 시 실행되는 파일이 될 것이다.

```shell
mkdir amplify/seed && touch amplify/seed/seed.ts

pnpm ampx sandbox
```

### 정책 구성 파일 생성 및 role에 추가

`ampx sandbox seed generate-policy` 명령을 통해 seed 이용에 필요한 권한이 포함된 구성 파일을 생성할 수 있다.

```shell
pnpm ampx sandbox seed generate-policy > seed-policy.json
```

정책 구성 파일을 만들었으니, 해당 정책을 사용할 role을 생성하여야 한다.
이미 사용하고 있는 role이 있다면 해당 role에 AmplifyBackendDeployFullAccess 권한이 존재하는지 확인 후 그곳에 추가해도 무방하다.

```shell
aws iam put-role-policy --role-name <your-role-name> --policy-name <policy-name> --policy-document file://seed-policy.json
```

또는, role을 생성하기 전에 해당 파일의 내용으로 정책을 만들어 IAM 사용자에게 추가해도 되지만, 이는 최소 권한 원칙에 위배될 수 있는 행위임으로 잘 판단하여야 한다.

새로 Role을 생성했다면, 아래의 게시글을 참고하여 role에 대해서 알아보고 오자~
[Amplify IAM Role](2025-11-21-amplify-iam-role.md)

---

## Congito 유저 생성 및 로그인

그럼 이제 Amplify Sandbox Seed의 기능을 사용할 준비가 완료되었으니, 어떻게 코드를 작성해야 하는지 알아보자.

### auth 설정

```ts
// amplify/auth/resource.ts
import { defineAuth } from "@aws-amplify/backend";

export const auth = defineAuth({
  loginWith: {
    email: true,
  },
  userAttributes: {
    address: {
      required: true,
      mutable: true,
    },
    birthdate: {
      required: true,
      mutable: true,
    },
    // ...
  },
  groups: ["FREE, ADMIN"],
});
```

### Secret 설정

username과 password 등 민감한 정보는 ampx sandbox secret 명령으로 관리할 수 있다.
위에서 email로 로그인 설정을 하였기에 username은 email의 형식을 가져야한다. 또한, 그 username이 email속성으로 추가된다.

```shell
pnpm ampx sandbox secret set username
pnpm ampx sandbox secret set password
```

## Cognito 유저 생성 및 로그인

seed.ts에 아래 내용을 추가하여 cognito 유저를 생성할 수 있다.

```ts
// amplify/seed/seed.ts

import { readFile } from "node:fs/promises";
import { createAndSignUpUser, getSecret, signInUser, addToUserGroup } from "@aws-amplify/seed";
import { Amplify } from "aws-amplify";
import type { Schema } from "../data/resource";
import * as auth from "aws-amplify/auth";

async function configureAmplify() {
  // amplify_outputs.json 은 sandbox가 생성된 이후에 생기므로 URL로 읽어옴
  const url = new URL("../../amplify_outputs.json", import.meta.url);
  const outputs = JSON.parse(await readFile(url, { encoding: "utf8" }));
  Amplify.configure(outputs);
}

// 위에서 설정한 secret을 불러옴
const username = await getSecret("username");
const password = await getSecret("password");

try {
  const user = await createAndSignUpUser({
    username,
    password,
    signInAfterCreation: true, // 생성, 회원가입과 동시에 로그인
    signInFlow: "Password",
    userAttributes: {
      // 현재 Auth에 required: true인 속성이 있다면 추가해주어야 한다.
      address: "123 Main St, Springfield, USA",
      birthdate: "1990-01-01",
      // username이 email로 설정되어 email 속성은 자동으로 추가되니 아래에서는 제외하여야 한다.
      // 아니면 email 속성은 수정할 수 없다는 오류가 발생한다. (NotAuthorizedException)
      // email: example@example.com
      // ...
    },
  });

  await addToUserGroup(user, "ADMIN"); // GROUP에 추가하려면 이렇게
} catch (err) {
  const error = err as Error;

  // 이미 생성된 User라면 아이디 생성을 건너뛰고 로그인
  if (error.name === "UsernameExistsError") {
    console.log("Username already exists, signing in instead...");
    await signInUser({
      username,
      password,
      signInFlow: "Password",
    });
  } else {
    throw err;
  }
}
```

## Seed 생성

`aws-amplify/cli`를 이용하여 GraphQL문을 이용하여 원하는 데이터를 생성하면 된다.

```ts
import { generateClient } from "aws-amplify/api";

let dataClient: ReturnType<typeof generateClient<Schema>>; // Client 전역 변수로 등록

async function configureAmplify() {
  const url = new URL("../../amplify_outputs.json", import.meta.url);
  const outputs = JSON.parse(await readFile(url, { encoding: "utf8" }));
  Amplify.configure(outputs);
  dataClient = generateClient<Schema>(); // 해당 줄 추가 (초기화)
}

// 채팅방 시드 (ChatRoomWithMembers 생성)
async function seedChatRoom(ownerUsername: string): Promise<string | null> {
  try {
    const response = await dataClient.mutations.createChatRoomWithMembers(
      {
        memberIds: [ownerUsername, "user-id-2"],
      },
      {
        authMode: "userPool",
      }
    );

    if (response.errors && response.errors.length > 0) {
      throw response.errors;
    }

    const roomId = response.data?.id ?? null;
    if (!roomId) {
      console.error("Chat room created but no id returned.");
      return null;
    }

    console.log("Chat room created:", roomId);
    return roomId;
  } catch (err) {
    console.error("Error creating chat room:", err);
    return null;
  }
}
```

**참고자료**
[Sandbox Seed - AWS Amplify Gen 2 Documentation](https://docs.amplify.aws/javascript/deploy-and-host/sandbox-environments/seed/#setting-up-your-seed-script)
