---
layout: post
title: "REST API Lambda 핸들러 연결"
date: 2025-10-27 10:00:00 +0900
description: >
  Pick 데이팅 앱의 인프라로 사용할 Amplify에 대해서 정리합니다.
categories: [pick]
tags: [amplify]
---

# REST API 설계 및 Lambda 핸들러 연결

## 함수 정의

```ts
// amplify/functions/api-function/resource.ts
import { defineFunction } from "@aws-amplify/backend";

export const myApiFunction = defineFunction({
  name: "api-function",
});
```

## 핸들러 코드

```ts
// amplify/functions/api-function/handler.ts
export const handler = async (event) => {
  return {
    statusCode: 200,
    headers: { "Access-Control-Allow-Origin": "*" },
    body: JSON.stringify({ message: "Hello from Lambda!" }),
  };
};
```

## API Gateway 연결

```ts
// amplify/backend.ts
import { defineBackend } from "@aws-amplify/backend";
import { HttpApi, HttpMethod } from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";
import { myApiFunction } from "./functions/api-function/resource";

const backend = defineBackend({ myApiFunction });

const httpApi = new HttpApi(backend, "HttpApi");
const integration = new HttpLambdaIntegration(
  "LambdaIntegration",
  backend.myApiFunction.resources.lambda // 실제 Lambda 리소스 연결
);

httpApi.addRoutes({
  path: "/hello",
  methods: [HttpMethod.GET],
  integration,
});

backend.addOutput({
  custom: {
    rest_api: {
      myHttpApi: {
        url: httpApi.apiEndpoint,
      },
    },
  },
});
```

# Flutter와 Amplify 연동

## Amplify 초기화

```dart
import 'package:amplify_flutter/amplify_flutter.dart';
import 'amplify_outputs.dart';

Future<void> configureAmplify() async {
  await Amplify.configure(amplifyOutputs);
}
```

## API 호출 예시

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'amplify_outputs.dart';

Future<void> callHello() async {
  final baseUrl = amplifyOutputs['custom']['rest_api']['myHttpApi']['url'];
  final response = await http.get(Uri.parse('$baseUrl/hello'));

  if (response.statusCode == 200) {
    print(jsonDecode(response.body));
  } else {
    throw Exception('Request failed: ${response.statusCode}');
  }
}
```

## 인증이 필요한 경우

로그인한 사용자의 Cognito 토큰을 헤더에 포함한다.

```dart
final session = await Amplify.Auth.fetchAuthSession();
final token = (session as CognitoAuthSession).userPoolTokens?.idToken;

final response = await http.get(
  Uri.parse('$baseUrl/hello'),
  headers: { 'Authorization': token ?? '' },
);
```

# 전체 동작 흐름 요약

## 요청 → 응답 흐름

```scss
[Flutter App]
     │ (GET /hello)
     ▼
[API Gateway (HttpApi)]
     │
     ▼
[HttpLambdaIntegration]
     │
     ▼
[Lambda Function (defineFunction)]
     │
     ▼
[handler.ts → JSON 응답]
```

간단한 호출 구조

```text
Flutter → API Gateway → Lambda(handler.ts) → JSON Response
```

---

**참고 자료**

[AWS Amplify Documentation][AWS Amplify Documentation]: https://docs.amplify.aws/gen2/
[AWS CDK v2 API Reference][AWS CDK v2 API Reference]: https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html
[AWS Amplify Flutter Developer Guide][AWS Amplify Flutter Developer Guide]: https://docs.amplify.aws/flutter/
