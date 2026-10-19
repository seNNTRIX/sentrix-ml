# sentrix-ml

SentriX 모델 연구·학습·평가.

> 프로젝트 기준 문서: `sentrix-collection` 저장소의 `sentrix-project-context.md`, 진행 상황: `docs/MILESTONES.md` (2026-10-20 Transfer 예정).

SentriX의 장애 탐지와 원인 위치 순위(node/edge ranking) 모델을 연구·학습·평가한다.

## 범위

- RE2-OB(RCAEval) baseline: 통계 threshold, 빈도 prior, random, RF/LightGBM, joint classification
- 공유 가중치 node/edge 스코어러 (서비스 이름을 feature로 쓰지 않음)
- linux-v1 데이터셋 3-fold CV(반복 기준), leave-one-service/edge-out, RE2-OB ↔ linux-v1 교차 평가
- 평가기: detection(AUROC, AUPRC, 시간당 오탐), ranking(Hit@1·3, MRR, AC@k, Avg@5), fault type(macro-F1), case 단위 bootstrap
- 실험 기록: dataset version, feature version, window, model, hyperparameter, seed, split, 결과

## 데이터

- 입력은 `sentrix-collection`의 데이터셋 빌더가 만든 **버전·해시가 붙은 데이터셋**만 사용한다. run 원본을 직접 읽지 않는다.
- feature 계산은 `sentrix-features` 패키지를 버전 고정해서 쓴다.
- window 수가 아니라 독립 case 수로 결과를 보고한다. scaler·vocabulary·threshold는 fold별 train에서만 fit한다.

## 예정 구조

```
experiments/   설정과 결과 기록
evaluation/    지표, bootstrap
models/        학습 코드, model card
notebooks/     탐색용 (결과 근거로 쓰지 않음)
```
