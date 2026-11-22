# sentrix-ml

SentriX의 **장애 탐지**와 **원인 후보 순위(서비스 / 서비스 간 호출 경로)** 모델을 개발·고정·평가하는 저장소입니다.

> SentriX: 쿠버네티스 MSA 환경의 관측성 기반 장애 탐지 및 RCA 시스템 (충북대학교 소프트웨어학부 졸업작품)
> 수집 환경과 원본 기록: [`seNNTRIX/sentrix-collection`](https://github.com/seNNTRIX/sentrix-collection)

## 한눈에 보기

| 질문 | 방법 | 학습 |
|---|---|---|
| 지금 이상한가? | **LightGBM** — 30초 구간마다 이상 점수, 정상 점수 상위 5%를 2번 연속 넘으면 경보 | 정상·장애 라벨 |
| 어느 서비스가 원인인가? | **Random Forest** — 서비스마다 **평소 대비 변화량** 18개 특징으로 원인 확률 | 원인 서비스 라벨 |
| 어느 호출 경로가 원인인가? | **규칙 기반** — 호출 지연 p90·오류율의 평소 대비 변화 | 학습 없음 |

**핵심 아이디어 — 서비스마다 다른 평소 상태를 기준으로 비교 (Baseline Delta)**
CPU·지연 같은 값의 절대 수준 대신 `(현재 − 그 서비스의 평소 중앙값) / 평소 IQR`을 씁니다. 서비스 이름과 절대 수준을 입력에서 빼서, 모델이 "어느 서비스가 자주 원인인가"를 외우지 않고 **학습에 없던 서비스**에도 적용되게 했습니다.

## 결과 (공개 데이터 RCAEval RE2-OB, 장애 90건)

**원인 서비스 순위 — 학습에 없던 원인 서비스로 평가 (leave-one-service-out)**

| 방법 | AC@1 | AC@3 | Avg@5 | MRR |
|---|---:|---:|---:|---:|
| Random | 0.091 | 0.273 | 0.273 | 0.275 |
| BARO (RCAEval 공개 방법) | 0.144 | 0.878 | 0.742 | 0.526 |
| 통계 기준선 (학습 없음) | 0.794 | 0.978 | 0.943 | 0.886 |
| Random Forest + 원본 관측값 | 0.756 | 0.833 | 0.844 | 0.820 |
| **SentriX (RF + Baseline Delta)** | **0.856** | **0.978** | **0.958** | **0.919** |

- 같은 Random Forest에서 입력만 바꿨을 때 Avg@5 **0.844 → 0.958 (+0.114)**
- 장애 탐지 AUROC **0.996** (통계 기준선 0.951), 반복 회차 3-fold
- 신뢰구간과 모든 비교: [`docs/METHOD-FREEZE-v2.md`](docs/METHOD-FREEZE-v2.md) 3절
- RE2-OB 점수는 **방법을 고를 때 쓴 데이터**의 교차검증 결과라 낙관적일 수 있습니다. 공정한 시험은 아래 자체 데이터가 맡습니다.

## 자체 수집 데이터 (linux-v1)

- Linux PC 2대, k3s 단일 노드, Online Boutique v0.9.0 (서비스 11개), Chaos Mesh로 장애 주입, k6로 사용자 부하
- 장애 5종: CPU 과부하 · Pod 장애 · 네트워크 지연 · 패킷 손실 · 트래픽 급증 (네트워크 장애는 **특정 호출 경로**에 주입 → 호출 경로 단위 정답)
- **장애 실험 69건 + 같은 조건의 대조 실험 69건**, 조건별 3회 반복 (수집 완료: `sentrix-collection` 태그 `linux-v1-collected-20261116`)
- 평가 계획: 방법을 고정한 뒤 (1) RE2-OB 모델을 그대로 적용, (2) 반복 회차 3-fold로 자체 데이터 일부를 학습에 추가. **진행 중**

## 방법 고정 (결과 보기 전에 정함)

| 문서 | 내용 |
|---|---|
| [`docs/METHOD-FREEZE-v2.md`](docs/METHOD-FREEZE-v2.md) | 최종 방법과 하이퍼파라미터, RE2-OB 성능, 자체 데이터 평가 계획 (고정 커밋 `399d3d0`) |
| [`docs/LINUX-V1-DATA-TRANSLATION.md`](docs/LINUX-V1-DATA-TRANSLATION.md) | 자체 데이터를 읽는 규칙 (수집하지 않은 신호, 장애 시작 시각 정의 등) |
| [`experiments/20261116-log-semantic-r8/PREREG.md`](experiments/20261116-log-semantic-r8/PREREG.md) | **로그 의미 변화** 추가 실험 사전 등록 (MiniLM 임베딩, 서비스별 평소 로그 대비 의미 거리, 비교군 A·A′·B·C·D) |

고정 모델 파일: [`experiments/20261028-re2ob-v2-freeze/models/`](experiments/20261028-re2ob-v2-freeze/models) (`ranker_rf.joblib`, `detector_lgbm.txt`, 해시는 `frozen.json`)

## 폴더 구조

```
features/      특징 계산 (compute.py는 고정), 30초 window·라벨, RE2-OB·linux-v1 어댑터,
               R8 로그 의미 특징(log_semantic.py), 근거용 로그 군집(log_cluster.py)
training/      실험 R4~R7 실행, 방법 고정 스크립트(freeze_r5.py, freeze_v2.py)
evaluation/    AC@k·Avg@5·MRR·AUROC, case 단위 bootstrap, 호출 경로 점수, BARO 기준선
experiments/   실험별 기록과 결과 (R3 기준선 ~ R7, v2 고정, R8 사전 등록)
docs/          방법 고정 문서, RE2-OB 결정·목록, 데이터 번역 결정, feature schema
tests/         특징·라벨·지표·교차검증·로그 특징 테스트
```

## 실행

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python -m unittest discover -s tests                 # 43 tests

# RE2-OB 원본은 포함하지 않습니다 (RCAEval에서 받아 data/RE2/RE2-OB에 둠)
python -m features.build_re2ob --root data/RE2/RE2-OB --out data/features/re2ob-v0
python -m training.freeze_v2 --out experiments/20261028-re2ob-v2-freeze
```

## 출처

이 저장소의 코드·문서·결과는 `sentrix-collection`의 연구 브랜치(`log-semantic-r8`, 커밋 `42ffbea`, `research-re2ob` 포함)에서 옮겼습니다. 수집 데이터(run 폴더, 로그 원본)는 용량과 환경 정보 때문에 포함하지 않습니다.

참고 연구: BARO (FSE 2024), LogRobust (FSE 2019), CHASE (arXiv:2406.19711). 로그 분석 아이디어: (주)케파스 제공 자료 참고.
