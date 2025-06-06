# 📋 시계열 분류 프로젝트: Robot Execution Failures

본 프로젝트는 로봇 작업 중 발생하는 실패 여부를 시계열 센서 데이터 기반으로 분류하는 것을 목표로 합니다.  

시간의 흐름에 따라 수집된 다변량 센서 데이터를 분석하고 이를 기반으로 분류 모델을 구축하였습니다. 모델 해석성과 분류 정확도를 동시에 확보하기 위해 시계열에서 [추출된 주요 특징을 해석](#3-eda-feature-exploration)하는 것에 중점을 두었습니다.

> 🛠️ **Tech Stack**
> 
>Language: &nbsp;Python  
Data Analysis & EDA: &nbsp;pandas, numpy, Jupyter Notebook  
Time Series Feature Extraction: &nbsp;tsfresh  
Visualization: &nbsp;matplotlib, seaborn  
Machine Learning:<br>- Modeling: &nbsp;`scikit-learn` (LogisticRegression, RandomForestClassifier), `XGBoost`<br>- Evaluation: &nbsp;`scikit-learn` (accuracy_score, classification_report, confusion_matrix)

<br>
<br>

## 프로젝트 개요

- 데이터: `tsfresh.examples.robot_execution_failures`

- 목표: &nbsp;시계열 내 구조적 패턴을 파악하고 로봇의 작업 실패 여부를 분류

- 구성:  
  - 총 88개의 시계열 (id 단위)  
  - 각 시계열은 15개의 시간 구간을 가지며, 총 6개의 센서 값 포함  
  - 센서: F_x, F_y, F_z (Force), T_x, T_y, T_z (Torque)  
  - 타겟: 작업 실패 여부 (`True`: 실패, `False`: 정상)

<br>

## 목차

1. [데이터 로드 및 확인](#1-데이터-로드-및-확인)  
2. [데이터 전처리](#2-데이터-전처리)  
3. [EDA: Feature Exploration](#3-eda-feature-exploration)  
4. [모델 학습 및 성능 비교](#4-모델-학습-및-성능-비교)  
5. [결론 도출](#5-결론-도출)  

<br>
<br>

## 분석 프로세스

### 1. 데이터 로드 및 확인

#### 1.1 &nbsp;데이터 로드 및 구조 확인
- tsfresh에서 제공하는 시계열 예제 데이터 로드
- id 기준의 그룹 시계열 구조 확인

#### 1.2 &nbsp;타겟 분포 확인
- `True`/`False` 분포 확인 (작업 실패 여부)
- 클래스 불균형 여부 점검

#### 1.3 &nbsp;시계열 길이 및 센서 분포 확인
- 각 id의 시계열 길이 = 15
- 센서 값 전체 분포 및 값 범위 시각화

<br>

### 2. 데이터 전처리

#### 2.1 &nbsp;데이터 분할 (Custom Split)
- `True`, `False` 균형 유지하며 train/test 3:1 비율 분할

#### 2.2 &nbsp;특징 추출 (Tsfresh)
- `EfficientFCParameters` 기반 자동 추출
- 4662개의 시계열 파생 특징 생성

#### 2.3 &nbsp;결측치 처리
- 일부 복잡한 파생 특징에서 NaN 발생
- `impute()`로 일괄 보간 처리 후 모델링에 활용

<br>

### 3. EDA: Feature Exploration

>시계열 분석의 해석력을 높이기 위해 중점적으로 수행한 파트입니다.
>
>4662개의 자동 생성 특징 중 정량 평가를 통해 의미 있는 상위 특징을 선별하고,  
>클래스 간 분포 차이 및 패턴을 시각화하여 해석하였습니다.

#### 3.1 &nbsp;추출된 상위 특징 선정
- 기준: 분산(`variance`), 상호정보량(`mutual_info`), 클래스 간 평균 차이(`mean_diff`)
- 세 기준을 rank로 정규화 후 종합 점수(`combined_score`) 계산
- Top 10 특징 자동 선정

#### 3.2 &nbsp;특징 해석

- `F_z__c3__lag_3`, `F_z__abs_energy`, `T_y__variance` 등은 타겟 구분에 높은 분별력을 보임
- 일부 특징은 특정 샘플에서만 급격한 변화 → 예측 타겟과 직접적 연결 가능성


| 순위 | Feature Name                                    | Variance          | Mutual<br>Info     | Mean<br>Diff         | Combined<br>Score | 분석 요약                                        |
| -- | ----------------------------------------------- | ----------------- | --------------- | ----------------- | -------------- | -------------------------------------------- |
| 1  | F_z__c3__<br>lag_3                                | 9.43e+16      | 0.4728          | 1.41e+08      | 0.9949     | 매우 높은 분산과 클래스 간 평균 차이<br>→ 타겟 구분에 강한 신호         |
| 2  | T_y__time_<br>reversal_<br>asymmetry_<br>statistic__lag_3 | 8.17e+13          | 0.4836     | 3.75e+05          | 0.9941         | Mutual Info 기준 2위<br>→ 시계열의 비대칭성이 타겟 구분에 도움  |
| 3  | F_z__variance                                 | 4.77e+09          | 0.5133<br>(최고) | 6.42e+04          | 0.9931         | Mutual Info 최고<br>→ 타겟과의 정보적 연결성이 가장 강함         |
| 4  | F_z__c3__lag_2                                | 1.02e+17<br>(최고) | 0.4494          | 1.55e+08      | 0.9918         | 최고 분산과 높은 평균 차이<br>→ 가장 폭발적인 분산형 신호             |
| 5  | F_z__abs_energy                               | 1.98e+13          | 0.4545          | 3.44e+06<br>(최고) | 0.9909         | 에너지 총량 기준 클래스 간 차이 극대화<br>→ 강력한 타겟 구분 가능성       |
| 6  | F_y__abs_energy                               | 9.23e+09          | 0.4867      | 5.53e+04          | 0.9908         | Mutual Info 높고,<br>에너지 수준의 변화가 유의미함             |
| 7  | F_y__c3__lag_3                                | 9.73e+11          | 0.4551          | 2.55e+05          | 0.9898         | 지연 자기상관 기반 특징<br>클래스 간 변화 양상이 큼                |
| 8  | T_x__abs_energy                               | 7.32e+11          | 0.4540          | 5.84e+05          | 0.9898         | 에너지 기반 특징<br>T_x 축 기준 클래스 간 차이 존재            |
| 9  | T_y__variance                                 | 5.15e+07<br>(최소)  | 0.5175<br>(최고) | 3.75e+03          | 0.9897         | Mutual Info는 높지만 실제 값 차이는 작음<br>→ 비선형 패턴 반영 가능성 |
| 10  | T_y__abs_energy                               | 2.55e+11          | 0.4547          | 2.51e+05          | 0.9893         | T_y 축 에너지 수준에서 구분력 존재                      |

<br>

#### 3.3 &nbsp;특징 시각화
- 상위 특징들의 전체 샘플 분포를 선형 그래프로 시각화
- 클래스에 따른 분포 패턴 차이 확인 → 예측 가능성 해석

![Top Feature Variance Visulization](https://github.com/user-attachments/assets/11d594d4-9e4d-45fa-9f99-01cf671c5cfe)

시각화된 상위 특징들은 대부분 전체 샘플에 대해 고르게 분포되지 않고, 일부 특정 샘플에서만 큰 값의 급격한 변화를 보였다.
이는 해당 특징들이 특정 조건 또는 타겟 클래스에 민감하게 반응한다고 생각해 볼 수 있다. 모델이 클래스 간 차이를 학습하는 데 있어 충분히 의미 있는 정보가 존재한다고 해석할 수 있다.

>자세한 결과 해석은 노트북 파일에서 확인하실 수 있습니다.


<br>

### 4. 모델 학습 및 성능 비교

#### 4.1 &nbsp;적용 모델
- Logistic Regression
- Random Forest
- XGBoost (최종 모델)

#### 4.2 &nbsp;성능 비교 테이블

| Model               | Accuracy | Precision | Recall | F1-score |
|--------------------|----------|-----------|--------|----------|
| Logistic Regression | 0.5714   | 0.8469    | 0.5714 | 0.5891   |
| Random Forest       | 0.9524   | 0.9603    | 0.9524 | 0.9538   |
| XGBoost             | **1.0000**   | **1.0000**    | **1.0000** | **1.0000**   |

XGBoost 성능 지표에서 과적합을 의심해 볼 수도 있지만  
해당 데이터셋의 크기가 매우 작기 때문에 발생한 상황이라고 해석하였다.


#### 4.3 &nbsp;특징 중요도 시각화: XGBoost
- `F_z__abs_energy` 특징만이 실제 분류 의사결정에 사용되었음을 확인
- 해당 특징의 분류력이 매우 강력함을 의미하지만, 데이터셋이 작기 때문에 모델이 과도하게 의존했을 가능성도 고려해야 함

<br>

### 5. 결론 도출

모델 비교 결과, XGBoost가 모든 평가 지표에서 가장 우수한 성능을 기록하며 최종 선택되었다.  
정량적으로 선별한 주요 특징이 실제 모델 학습에도 유의미하게 반영된 점에서 해석 가능성을 확인할 수 있었다.

다만, 데이터의 크기가 매우 작아 모델이 소수 특징에 과도하게 의존한 결과일 수 있음을 고려해야 한다.  

일반화를 위해 추가적인 데이터 확보와 검증이 필요하다.

<br>
<br>

## 인사이트 및 회고

시계열 데이터를 다룰 때도 마찬가지로 특징을 어떻게 추출하고 해석하느냐가 분석에서 가장 어렵고 중요한 부분이라는 것을 알 수 있었다.

tsfresh를 통해 자동으로 생성된 수많은 특징 중 실제로 의미 있는 특징을 정량적으로 선별하고 시각화해보는 과정에서 시계열과 모델의 이해도를 높이는 데 도움이 되었다.

데이터 수가 매우 적은 상황에서도 하나의 특징만으로 분류가 가능했던 점은 흥미로웠지만, 동시에 데이터의 일반화 가능성에 대한 한계를 확인하였다.  

이번 분석을 통해 시계열 분류 문제에서도 데이터의 구조를 드러내는 특징 설계와 해석이 실제 모델의 성능만큼이나 중요하다는 점을 배웠다.

<br>

>본 프로젝트는 시계열 데이터의 분류 문제에서 구조적 해석과 특징 기반 설명 가능성 확보에 중점을 두었습니다.



