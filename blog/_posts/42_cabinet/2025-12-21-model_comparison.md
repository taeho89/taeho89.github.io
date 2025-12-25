---
layout: post
title: "사물함 점유 판별 모델 성능 비교 분석"
date: 2025-12-21 00:32:06 +0900
description: >
  HOG+SVM, MobileNetV3, EfficientNetV2 등 6가지 모델의 정확도, 
  Recall, Precision, 추론 시간, 모델 크기를 비교 분석합니다.
  실전 배포를 위한 모델 선택 가이드를 제공합니다.
categories: [42_cabinet]
tags: [model-comparison, performance-analysis, efficientnet, mobilenet, benchmark]
---

# 모델 성능 비교 리포트

**생성일시**: 2025-12-21 00:32:06  
**평가 데이터**: `data/train`  
**CNN 결정 임계값**: 0.6

---

## 요약

| 순위 | 모델                          | 정확도 | Empty Recall | Occupied Recall | 모델 크기 |  추론 시간  |
| :--: | ----------------------------- | :----: | :----------: | :-------------: | :-------: | :---------: |
|  1   | **EfficientNetV2-S (v1.1.0)** | 98.7%  |    100.0%    |      97.4%      |  96.0 MB  | 91.1 ms/img |
|  2   | **MobileNetV3Large (v1.1.0)** | 96.8%  |    97.5%     |      96.1%      |  13.5 MB  | 15.6 ms/img |
|  3   | **HOG + SVM (v1.1.0)**        | 89.2%  |    85.0%     |      93.5%      |  0.7 MB   | 3.3 ms/img  |
|  4   | **EfficientNetV2-S (v1.0.0)** | 86.0%  |    95.0%     |      76.6%      |  81.1 MB  | 91.4 ms/img |
|  5   | **MobileNetV3Small (v1.0.0)** | 62.4%  |    35.0%     |      90.9%      |  22.5 MB  | 15.8 ms/img |
|  6   | **MobileNetV3Large (v1.0.0)** | 58.0%  |    22.5%     |      94.8%      |  13.5 MB  | 14.5 ms/img |

---

## 상세 결과

### EfficientNetV2-S (v1.1.0)

- **모델 파일**: `models/efficientNetV2-S-v1.1.0.keras`
- **모델 타입**: EfficientNetV2-S
- **입력 크기**: 384×384
- **모델 크기**: 96.00 MB
- **추론 시간**: 14.31s (전체) / 91.1ms (이미지당)

#### 성능 지표

| 지표          | Empty  | Occupied |
| ------------- | :----: | :------: |
| **Recall**    | 100.0% |  97.4%   |
| **Precision** | 97.6%  |  100.0%  |
| **F1-Score**  | 0.988  |  0.987   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       80        0
Actual occupied     2       75
```

---

### MobileNetV3Large (v1.1.0)

- **모델 파일**: `models/mobileNetV3Large-v1.1.0.keras`
- **모델 타입**: MobileNetV3Large
- **입력 크기**: 224×224
- **모델 크기**: 13.49 MB
- **추론 시간**: 2.44s (전체) / 15.6ms (이미지당)

#### 성능 지표

| 지표          | Empty | Occupied |
| ------------- | :---: | :------: |
| **Recall**    | 97.5% |  96.1%   |
| **Precision** | 96.3% |  97.4%   |
| **F1-Score**  | 0.969 |  0.967   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       78        2
Actual occupied     3       74
```

---

### HOG + SVM (v1.1.0)

- **모델 파일**: `models/locker-detector-v1.1.0.pkl`
- **모델 타입**: HOG+SVM
- **입력 크기**: 128×128
- **모델 크기**: 0.74 MB
- **추론 시간**: 0.52s (전체) / 3.3ms (이미지당)

#### 성능 지표

| 지표          | Empty | Occupied |
| ------------- | :---: | :------: |
| **Recall**    | 85.0% |  93.5%   |
| **Precision** | 93.2% |  85.7%   |
| **F1-Score**  | 0.889 |  0.894   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       68       12
Actual occupied     5       72
```

---

### EfficientNetV2-S (v1.0.0)

- **모델 파일**: `models/efficientNetV2-S-v1.0.0.keras`
- **모델 타입**: EfficientNetV2-S
- **입력 크기**: 384×384
- **모델 크기**: 81.13 MB
- **추론 시간**: 14.35s (전체) / 91.4ms (이미지당)

#### 성능 지표

| 지표          | Empty | Occupied |
| ------------- | :---: | :------: |
| **Recall**    | 95.0% |  76.6%   |
| **Precision** | 80.9% |  93.7%   |
| **F1-Score**  | 0.874 |  0.843   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       76        4
Actual occupied    18       59
```

---

### MobileNetV3Small (v1.0.0)

- **모델 파일**: `models/locker-classifier-cnn-v1.0.0.h5`
- **모델 타입**: MobileNetV3Small
- **입력 크기**: 224×224
- **모델 크기**: 22.52 MB
- **추론 시간**: 2.49s (전체) / 15.8ms (이미지당)

#### 성능 지표

| 지표          | Empty | Occupied |
| ------------- | :---: | :------: |
| **Recall**    | 35.0% |  90.9%   |
| **Precision** | 80.0% |  57.4%   |
| **F1-Score**  | 0.487 |  0.704   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       28       52
Actual occupied     7       70
```

---

### MobileNetV3Large (v1.0.0)

- **모델 파일**: `models/mobileNetV3Large-v1.0.0.keras`
- **모델 타입**: MobileNetV3Large
- **입력 크기**: 224×224
- **모델 크기**: 13.50 MB
- **추론 시간**: 2.27s (전체) / 14.5ms (이미지당)

#### 성능 지표

| 지표          | Empty | Occupied |
| ------------- | :---: | :------: |
| **Recall**    | 22.5% |  94.8%   |
| **Precision** | 81.8% |  54.1%   |
| **F1-Score**  | 0.353 |  0.689   |

#### Confusion Matrix

```
                 Predicted
                 empty  occupied
Actual empty       18       62
Actual occupied     4       73
```

---

## 결론

**최고 성능 모델**: **EfficientNetV2-S (v1.1.0)**

- 정확도: 98.7%
- Empty Recall: 100.0% (반납된 사물함 탐지율)
- Occupied Recall: 97.4%
- 모델 크기: 96.00 MB
- 추론 시간: 91.1 ms/image

### 모델 선택 가이드

| 우선순위            | 추천 모델                 | 이유                |
| ------------------- | ------------------------- | ------------------- |
| Empty Recall 최우선 | EfficientNetV2-S (v1.1.0) | Empty Recall 100.0% |
| 전체 정확도 최우선  | EfficientNetV2-S (v1.1.0) | Accuracy 98.7%      |
| 경량 모델           | HOG + SVM (v1.1.0)        | 0.7 MB              |
| 빠른 추론           | HOG + SVM (v1.1.0)        | 3.3 ms/img          |
