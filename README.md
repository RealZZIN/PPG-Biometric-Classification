# PPG Biometric Classification

PPG(Photoplethysmography) 시계열 신호를 이용해 **개인 식별**과 **연령대(20대 / 50대) 분류**를 수행한 개인 프로젝트입니다.

손가락 PPG 센서로 직접 수집한 신호를 전처리한 뒤, 여러 구간의 특징을 1D CNN으로 추출하고 attention 기반으로 통합하여 파일 단위로 분류했습니다.

---

## 프로젝트 목표

PPG 신호만으로 다음 두 가지 분류가 가능한지 확인했습니다.

1. **개인 식별**
   - 4명의 피험자를 구분하는 4-class classification

2. **연령대 분류**
   - 20대와 50대를 구분하는 binary classification

단순히 짧은 구간 하나를 분류하는 대신, 약 5분 30초 동안 측정한 한 파일 전체의 PPG 패턴을 하나의 분석 단위로 사용했습니다.

---

## 데이터 구성

- 총 피험자: **4명**
- 총 파일 수: **60개**
- 개인당 파일 수: **15개**
- 연령대:
  - 20대: 2명 / 30개 파일
  - 50대: 2명 / 30개 파일
- 파일당 측정 시간: 약 **330초**
- 입력 신호: 손가락에서 측정한 **PPG 시계열**

원본 PPG 데이터는 저장소에 포함하지 않았습니다.

---

## 전체 분석 흐름

```text
PPG raw signal
      │
      ▼
Moving Average Detrending
      │
      ▼
Smoothing
      │
      ▼
Sliding Window
      │
      ▼
Z-score Normalization
      │
      ▼
1D CNN / ResNet-style Encoder
      │
      ▼
Attention Pooling
      │
      ▼
File-level Classification
      │
      ├── Subject Identification
      └── Age-group Classification
```

---

## 전처리

원시 PPG 신호에는 baseline drift와 측정 과정에서 발생한 잡음이 포함되어 있어 다음 과정을 적용했습니다.

### 1. Baseline 제거

이동평균을 이용해 저주파 추세를 구한 뒤 원 신호에서 차감했습니다.

- `BASELINE_MA = 301`

### 2. Smoothing

짧은 이동평균을 적용해 고주파성 잡음을 완화했습니다.

- `SMOOTH_MA = 5`

### 3. Sliding Window

한 파일의 긴 PPG 신호를 여러 개의 짧은 구간으로 나누어 모델 입력을 구성했습니다.

- Window length: `300 samples`
- Hop size: `100 samples`
- 파일당 최대 window 수: `128`

### 4. Z-score 정규화

파일 및 세션마다 발생할 수 있는 신호 크기 차이의 영향을 줄이기 위해 Z-score 정규화를 적용했습니다.

---

## 모델 구조

PPG의 시간적 패턴을 학습하기 위해 **1D CNN 기반 ResNet-style encoder**를 사용했습니다.

각 파일은 여러 개의 window로 나뉘며, 각 window에서 특징을 추출한 뒤 attention pooling을 이용해 파일 전체의 특징으로 통합합니다.

```text
128 Windows / File
        │
        ▼
1D CNN Encoder
        │
        ▼
Window-level Feature Vectors
        │
        ▼
Attention Pooling
        │
        ▼
File-level Feature
        │
        ▼
Classifier
```

주요 설정은 다음과 같습니다.

| 항목 | 설정 |
| --- | --- |
| Base channels | 32 |
| Residual blocks | (2, 2, 2) |
| Pooling | Attention |
| Dropout | 0.3 |
| Optimizer | Adam |
| Learning rate | 1e-3 |
| Weight decay | 1e-4 |
| Loss | CrossEntropyLoss |
| Batch size | 8 files |
| Cross validation | 4-fold StratifiedKFold |

---

## 실험 1. 연령대 분류

20대와 50대 PPG 신호를 구분하는 2-class classification을 수행했습니다.

보고서에 정리한 4-fold 교차검증 결과 기준으로:

- **Accuracy: 약 0.95**
- **Macro F1-score: 약 0.95**

대부분의 fold에서는 높은 성능이 나타났지만 일부 fold에서는 20대 데이터를 50대로 오분류하는 경우가 확인되었습니다.

이 결과는 연령에 따른 공통적인 PPG 차이뿐 아니라 **개인별 생리적 차이**가 함께 반영되었을 가능성을 고려해 해석했습니다.

---

## 실험 2. 개인 식별

4명의 PPG 신호를 구분하는 4-class classification을 수행했습니다.

보고서에 정리한 결과 기준으로:

- **Accuracy: 약 0.98**
- **Macro F1-score: 약 0.98**

개인 식별 실험은 연령대 분류보다 전반적으로 안정적인 성능을 보였으며, 동일 환경에서 반복 측정된 PPG 파형에 개인별로 구분 가능한 특징이 포함되어 있음을 확인했습니다.

---

## 평가 방법

모델 성능은 파일 단위로 평가했습니다.

사용한 지표는 다음과 같습니다.

- Accuracy
- Macro F1-score
- Confusion Matrix
- Classification Report

학습 및 평가 과정에서 클래스 비율을 유지하기 위해 `StratifiedKFold`를 사용했습니다.

---

## 한계

본 프로젝트는 **4명의 소규모 데이터셋**으로 수행했기 때문에 결과를 일반적인 개인 식별 또는 연령 분류 성능으로 바로 해석하기에는 제한이 있습니다.

특히 다음과 같은 한계가 있습니다.

- 각 연령대의 피험자가 2명뿐이므로 연령 특성과 개인 특성을 완전히 분리하기 어려움
- 같은 사람의 반복 측정 데이터가 여러 fold에 포함될 수 있음
- 날짜, 센서 부착 상태, 손가락 압력, 컨디션 등 세션별 차이가 모델에 반영되었을 가능성
- 새로운 사람에 대한 일반화 성능은 별도로 검증하지 않음

따라서 연령대 분류의 높은 성능은 **연령 자체의 일반적인 특징을 학습했다고 단정하기보다, 현재 데이터 구성 안에서 관찰된 분류 결과**로 해석했습니다.

향후에는 피험자 수를 늘리고, 사람 단위 또는 날짜 단위로 학습/평가 데이터를 완전히 분리해 추가 검증할 수 있습니다.

---

## 저장소 구성

```text
PPG-Biometric-Classification/
├── ppg_classification.ipynb
├── README.md
├── requirements.txt
└── .gitignore
```

`ppg_classification.ipynb`에는 프로젝트 수행 당시 사용한 실험 코드와 결과를 그대로 포함했습니다.

---

## 실행

Google Colab 환경을 기준으로 작성했습니다.

노트북에서 데이터 경로를 본인의 환경에 맞게 변경해야 합니다.

```python
DATA_ROOT = "/content/drive/MyDrive/학교(AI헬스케어)"
```

후반부 통합 코드에서는 `TASK` 값을 변경하여 두 실험을 선택할 수 있습니다.

```python
TASK = "age"   # 연령대 분류
```

또는

```python
TASK = "id"    # 개인 식별
```

---

## 사용 기술

- Python
- PyTorch
- NumPy
- scikit-learn
- Matplotlib
- 1D CNN
- ResNet-style architecture
- Attention pooling
