# AI-based Lecture Review Analysis and Recommendation

강의평 데이터를 정형 및 텍스트 feature로 변환하고, 평균 평점 예측과 사용자 선호 기반 Top-K 강의 추천을 실험한 프로젝트입니다.

## Project Overview

프로젝트의 주요 흐름은 다음과 같습니다.

1. 강의평 raw JSON 정규화
2. 리뷰 수가 적은 강의 제외
3. 강의별 31차원 feature vector 생성
4. 평균 평점 예측 모델 비교
5. 5-fold cross-validation 기반 하이퍼파라미터 탐색
6. 사용자 persona와 강의 vector의 유사도를 이용한 Top-K 추천

원본 강의평과 사용자 식별 가능성이 있는 데이터는 저장소에 포함하지 않습니다.

## Lecture Vector

각 강의는 총 31개의 feature로 표현됩니다.

- 정형 feature 10개
  - 과제 적음/많음
  - 팀플 적음/많음
  - 학점 후함/엄격함
  - 출석 부담 낮음/높음
  - 시험 부담 낮음/높음
- 텍스트 TF-IDF feature 18개
  - 과제, 시험, 팀플, 출석, 학점, 강의력 카테고리
  - 카테고리별 전체/긍정/부정 TF-IDF
- 감성 비율 feature 3개
  - 긍정, 중립, 부정 리뷰 비율

감성 라벨은 기존 리뷰 별점을 이용한 weak labeling입니다.

```text
4~5점: positive
3점: neutral
1~2점: negative
```

## Rating Prediction

예측 target은 5점 만점 평균 별점을 0~1로 정규화한 값입니다.

```text
rating_average_norm = rating_average / 5
```

비교 모델:

- Train mean baseline
- Ridge regression
- Content-based KNN
- Graph-augmented MLP

전체 753개 강의에 대해 5-fold cross-validation을 수행한 결과, Ridge regression이 가장 좋은 성능을 보였습니다.

```text
Best model: Ridge Regression, alpha=10
CV MSE:  0.00319527
CV RMSE: 0.05641645
CV MAE:  0.04178956
```

정규화 값을 5점 척도로 환산하면 RMSE는 약 0.28점, MAE는 약 0.21점입니다.

## Graph-augmented MLP

- Node: 강의
- Node feature: 강의별 31차원 vector
- Edge: cosine similarity가 높은 k개의 강의를 이웃으로 연결
- Input: 자기 feature, 이웃 평균 feature, 두 feature의 차이

실제 학생-강의 interaction이 없는 상태에서는 Graph-augmented MLP가 Ridge와 KNN보다 낮은 성능을 보였습니다. 따라서 현재 데이터에서는 복잡한 그래프 구조의 이점이 확인되지 않았습니다.

## Top-K Recommendation

학생 persona는 강의와 동일한 feature space의 preference vector로 표현됩니다. 학생을 그래프 노드로 학습하는 대신 추천 시점의 query vector로 사용합니다.

```text
recommendation_score
  = 0.7 * preference_similarity
  + 0.3 * predicted_quality
```

- `preference_similarity`: 학생 선호와 강의 vector의 cosine similarity를 0~1로 변환
- `predicted_quality`: Ridge regression이 예측한 강의 품질
- K: 기본값 10

지원하는 persona preset:

- `low_workload`
- `learning_quality`
- `exam_light`
- `no_team_project`
- `challenging_but_good`

## Run

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

전처리된 feature 데이터로 전체 CV 실험:

```powershell
python scripts\run_full_cv_experiment.py
python scripts\make_cv_visuals.py
```

Top-10 추천:

```powershell
python scripts\recommend_topk.py --preset low_workload --top-k 10
```

직접 persona 가중치 입력:

```powershell
python scripts\recommend_topk.py --weights-json '{ "assignment_low_score": 1.0, "exam_light_score": 0.8, "text_positive_ratio": 0.6 }'
```

## Repository Structure

```text
scripts/                  preprocessing, experiments, recommendation
data/model/               anonymized lecture feature vectors
data/experiments/full_cv/ cross-validation results and figures
data/recommendations/     recommendation descriptions and summaries
```

## Limitations

- 과목명, 교수명, 학과 metadata가 없어 결과가 `lecture_id` 중심으로 표시됩니다.
- 사용자 ID와 학생별 수강/평가 이력이 없어 진정한 collaborative filtering은 수행하지 못했습니다.
- 현재 평가는 평균 평점 예측 성능을 중심으로 하며, 개인화 추천 성능 자체를 검증한 것은 아닙니다.
- 실제 개인화 평가에는 학생 interaction 데이터와 Hit@K, Recall@K, NDCG@K 등의 ranking metric이 필요합니다.

## Data Notice

원본 강의평 JSON과 리뷰 텍스트는 개인정보, 서비스 약관 및 저작권 문제를 고려해 공개 저장소에서 제외합니다. 저장소에는 실험 재현에 필요한 코드와 익명화된 파생 feature 및 결과만 포함합니다.
