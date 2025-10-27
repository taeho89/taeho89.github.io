---
layout: post
title: "타입 오류 및 리소스 연결 에러"
date: 2025-10-27 10:00:00 +0900
description: >
  Pick 데이팅 앱의 인프라로 사용할 Amplify에 대해서 정리합니다.
categories: [pick]
tags: [amplify]
---

# 타입 오류 및 리소스 연결 에러 해결

## 문제 상황
```ts
new HttpLambdaIntegration("HelloIntegration", myApiFunction);
```

에서 다음 오류 발생:
```ts
'ConstructFactory<ResourceProvider<...>>' 형식의 인수는 
'IFunction' 형식의 매개 변수에 할당될 수 없습니다.
```

## 원인

myApiFunction은 단순한 선언체(ConstructFactory)이고,
HttpLambdaIntegration은 CDK의 실제 Lambda 객체(IFunction)를 요구한다.

- 해결 방법
```ts
const integration = new HttpLambdaIntegration(
  "HelloIntegration",
  backend.myApiFunction.resources.lambda
);
```

## 핵심 개념

`myApiFunction`: 선언용 Amplify 리소스

`backend.myApiFunction.resources.lambda`: 실제 AWS Lambda 리소스