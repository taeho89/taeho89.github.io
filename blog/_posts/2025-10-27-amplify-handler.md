---
layout: post
title: "lambda 핸들러 연결 규칙"
date: 2025-10-27 10:00:00 +0900
description: >
  Pick 데이팅 앱의 인프라로 사용할 Amplify에 대해서 정리합니다.
categories: [pick]
tags: [amplify]
---

# 핸들러 자동 연결 규칙

## 기본 규칙

`defineFunction({ name: "api-function" })` →
자동으로 `amplify/functions/api-function/handler.ts`의 `export const handler`와 매칭된다.

| 구성 요소	| 규칙 |
| 디렉토리명	| 함수명과 동일 |
| 파일명	| handler.ts 또는 handler.js |
| 함수명	| export const handler |

## 커스텀 지정 가능
```ts
defineFunction({
  name: "api-function",
  entry: "./src/customHandler.ts",
  handler: "main",
});
```