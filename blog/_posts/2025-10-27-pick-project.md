---
layout: post
title: "Pick 프로젝트 개요"
date: 2025-10-27 10:00:00 +0900
description: >
  Pick 데이팅 앱 개발을 시작하며 핵심 목표, 기술 스택, 초기 설정을 정리했습니다.
categories: [pick]
tags: [roadmap, amplify]
---

## 프로젝트 목표
- 성격 기반 매칭 로직을 빠르게 실험할 수 있는 MVP 출시
- 사용자 유입을 고려한 온보딩 플로우 설계와 A/B 테스트 기반 개선

## 기술 스택
- 프런트엔드: Flutter (iOS/Android 동시 배포)
- 백엔드: AWS Amplify + Lambda + DynamoDB
- 인증: AWS Cognito

## 초기 진행 상황
1. Amplify 프로젝트 생성 및 환경 분리(dev/prod) 설정
2. Cognito User Pool을 이용한 이메일 기반 회원가입 플로우 구축
3. Flutter에서 Amplify Auth 플러그인 연동 완료

## 다음 단계
- 매칭 관련 도메인 모델 정의 및 DynamoDB 스키마 초안 작성
- 실시간 알림을 위한 AppSync(GraphQL) 채널 구성 검토
- 기본 온보딩 화면(프로필 입력, 관심사 선택) 와이어프레임 확정
