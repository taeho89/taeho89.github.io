---
layout: post
title: "GitHub Action에서 AWS OIDC를 활용한 보안 강화 및 자동 생성 파일 관리"
date: 2025-12-03 21:00:00 +0900
description: >
categories: [pick]
tags: [github-action, aws, oidc, security, ci/cd]
---

# GitHub Action과 AWS OIDC

프로젝트를 진행하다 보면 `amplify_outputs.dart`나 GraphQL 모델 파일(`modelgen`)과 같이 빌드 시점에 자동으로 생성되는 파일들을 마주하게 된다.
이러한 파일들은 보통 `.gitignore`에 포함시켜 관리한다. 그 이유는 다음과 같다.

1.  **Merge Conflict 방지**: 여러 개발자가 작업할 때 자동 생성 파일의 변경으로 인한 불필요한 충돌을 막기 위함이다.
2.  **보안**: `amplify_outputs.dart`와 같은 파일에는 민감한 설정 정보가 포함될 수 있어 저장소에 커밋하지 않는 것이 좋다.

하지만, 이러한 파일들을 git으로 관리하지 않으면 **CI/CD 파이프라인(GitHub Actions)에서 빌드 실패**라는 문제에 직면하게 된다.
"Build 테스트가 통과되어야 Pull Request가 승인된다"는 팀 규칙을 지키기 위해서는 Action 실행 환경에서도 이 파일들이 필요하기 때문이다.

이 문제를 해결하기 위해 저희는 **AWS OIDC(OpenID Connect)**를 도입하여, GitHub Action 내에서 안전하게 AWS 리소스에 접근하고 필요한 파일들을 즉석에서 생성하도록 구성했다.

## OIDC(OpenID Connect)란?

OIDC는 GitHub Actions 워크플로우가 AWS와 같은 클라우드 공급자에게 인증할 수 있도록 해주는 표준 인증 프로토콜이다.
기존에는 AWS 리소스에 접근하기 위해 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY` 같은 장기(Long-lived) 자격 증명을 GitHub Secrets에 저장해서 사용했다.

하지만 OIDC를 사용하면 다음과 같은 장점이 있다:

*   **장기 자격 증명 불필요**: 액세스 키를 직접 관리하거나 저장할 필요가 없습니다.
*   **보안 강화**: GitHub OIDC Provider가 발급한 토큰을 AWS STS(Security Token Service)가 검증하고, 짧은 시간 동안만 유효한 임시 보안 자격 증명을 발급해준다.
*   **세밀한 제어**: AWS IAM Role의 신뢰 관계(Trust Relationship)를 통해 특정 리포지토리나 브랜치에서만 해당 역할을 수행할 수 있도록 제한할 수 있다.

더 자세한 내용은 GitHub 공식 문서의 [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/ko/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)를 참고하면 좋다.

## 적용 방법

우선 OpenId Conenct ID 제공업체를 추가하여야 한다.

1. AWS IAM 콘솔 -> ID 제공업체 -> 공급자 추가

![alt text](/assets/img/blog/image-12.png)

2. 실제 수행할 작업의 권한을 가진 Role(역할) 생성
   신뢰 관계에 해당 내용 복사 후 알맞게 수정

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<계정 번호>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": "repo:taeho89(github_id)/<repo_name>:ref:refs/heads/<branch_name>"
                }
            }
        }
    ]
}
```

![alt text](/assets/img/blog/image-13.png)

이후 필요한 권한을 추가하고 github_action_dev와 같이 적절한 이름을 부여한 후 역할을 생성하고, 그 역할의 ARN을 복사해서 GitHub Secrets에 저장한다. (Github -> Repository -> Settings -> Secrets and Variables -> Actions -> New repository secret)

저희 프로젝트에서는 다음과 같이 GitHub Action을 설정하여 빌드 과정에서 `amplify_outputs.dart`와 모델 코드를 생성하고 있다.

### GitHub Action Workflow (`build-ios` 예시)

```yaml
build-ios:
    runs-on: macos-latest
    # OIDC 사용을 위해 id-token 권한 필요
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Clone repository
        uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8
      
      - name: Set up Flutter
        uses: subosito/flutter-action@fd55f4c5af5b953cc57a2be44cb082c8f6635e8e
        with:
          channel: stable
          cache: true
      
      - run: flutter pub get
      
      # AWS OIDC 인증 단계
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@main
        with:
          role-to-assume: ${{ secrets.AWS_ACTION_ROLE_ARN }} # 미리 생성한 IAM Role ARN
          aws-region: ${{ secrets.AWS_REGION }}
      
      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 10
      
      - run: pnpm install
      - run: flutter pub get
      
      # 인증된 권한으로 파일 생성
      - name: Generate amplify outputs
        run: |
          pnpm ampx generate outputs \
            --app-id ${{ secrets.APP_ID }} \
            --branch dev \
            --format dart \
            --out-dir lib
            
      - name: Generate GraphQL client code
        run: |
          pnpm ampx generate graphql-client-code \
            --app-id ${{ secrets.APP_ID }} \
            --branch dev \
            --format modelgen \
            --model-target dart \
            --out lib/models/generated
            
      - run: dart run build_runner build
      
      # 이후 빌드 과정 진행
      - run: flutter build ios --debug --no-codesign
      - run: flutter build ipa --debug --no-codesign
      
      - name: Upload iOS Build Artifact
        uses: actions/upload-artifact@330a01c490aca151604b8cf639adc76d48f6c5d4
        with:
          name: ios_build
          path: |
            build/ios/iphoneos/Runner.app
            build/ios/archive/Runner.xcarchive
          if-no-files-found: error
```

위 설정에서 `aws-actions/configure-aws-credentials` 액션이 OIDC를 통해 임시 자격 증명을 획득하고, 이후 `ampx generate` 명령어들이 이 자격 증명을 사용하여 AWS Amplify 백엔드 정보를 가져와 필요한 파일들을 생성한다.

## 그 외 민감한 파일 관리 (firebase_options, google-services)

`amplify_outputs.dart` 외에도 `firebase_options.dart`나 `google-services.json`과 같은 파일들도 보안상 git에 올리지 않는 경우가 많다.
이러한 파일들은 내용이 고정되어 있거나 AWS 리소스와 직접적인 상호작용이 필요 없는 경우, **GitHub Secrets에 파일 내용을 Base64로 인코딩하여 저장**하거나, **Large Secrets** 관리 방법을 사용할 수 있다.

이에 대한 자세한 내용은 GitHub 공식 문서의 [Storing large secrets](https://docs.github.com/ko/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets#storing-large-secrets)를 참고하면 도움이 된다.

---

이렇게 OIDC를 도입함으로써 보안을 유지하면서도 CI/CD 파이프라인의 안정성을 확보할 수 있었다.
