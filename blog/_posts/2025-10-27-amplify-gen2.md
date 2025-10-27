---
layout: post
title: "Amplify Gen2"
date: 2025-10-27 10:00:00 +0900
description: >
  Pick 데이팅 앱의 인프라로 사용할 Amplify에 대해서 정리합니다.
categories: [pick]
tags: [amplify]
---

# Amplify Gen2의 기본 개념

Amplify Gen2는 AWS Amplify의 차세대 구조로, 모든 백엔드 리소스를 코드로 정의하고 관리하는 인프라 코드(Infra-as-Code) 방식이다.
CLI 명령어 대신 TypeScript 파일(amplify/backend.ts, amplify/functions/...)로 인프라를 구성한다.

## 주요 개념

defineBackend: Amplify 백엔드의 진입점.

sandbox: 로컬 개발 환경에서 AWS 리소스를 임시로 배포.

amplify_outputs: 프런트엔드(Flutter, React 등)에 리소스 정보를 전달.

## 주요 명령어

```bash
npx ampx sandbox --outputs-format dart --outputs-out-dir lib
```

→ Flutter에서 사용할 amplify_outputs.dart 자동 생성