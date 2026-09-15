# Customer Churn Prediction Dashboard

고객 CSV를 업로드해 이탈 확률을 계산하고, 지정한 임계값보다 확률이 높은 고객과 집계 통계를 웹 대시보드에서 확인하는 프로젝트입니다. 프론트엔드는 Next.js와 MUI, 백엔드는 FastAPI, 예측 모델은 scikit-learn의 Random Forest로 구성되어 있습니다.

학습 데이터의 `payment=unpaid`를 이탈 레이블 `1`, `paid`를 `0`으로 변환해 이탈 확률을 계산합니다.

## 핵심 기능

- 예측용 또는 재학습용 고객 CSV 업로드
- 사용자가 설정한 임계값에 따른 고위험 고객 분류
- 전체 고객 수, 예상 이탈 고객 수, 그룹별 평균 나이·시청 시간·최근 로그인 경과일 시각화
- 선호 장르 분포와 이탈 현황 표시
- 이탈 위험 고객 상위 10명 조회
- 고위험 고객 목록 Excel 다운로드
- `versioned=true` 예측 시 결과 파일과 실행 이력 저장
- 상위 고객 대상 추천 이메일 생성·발송 실험 API

- 프론트엔드 상태를 이용한 데모 로그인

## 예측 방법

학습과 추론에 사용하는 입력 특성은 세 가지입니다.

| 특성 | 설명 |
| --- | --- |
| `age` | 고객 나이 |
| `watch_time` | 시청 시간 |
| `days_since_login` | 현재 날짜와 `last_login`의 차이로 계산한 경과일 |

학습 과정은 다음과 같습니다.

1. 컬럼명을 소문자와 underscore 형식으로 정규화합니다.
2. `payment_status`가 있으면 `payment`로 변경하고 결제 상태로 `churned` 레이블을 만듭니다.
3. 세 입력 특성을 `MinMaxScaler`로 변환합니다.
4. 데이터를 80:20으로 계층 분할합니다(`random_state=42`).
5. `RandomForestClassifier(n_estimators=100, max_depth=5, random_state=42)`를 학습합니다.
6. 추론 시 `predict_proba`의 이탈 클래스 확률을 사용자가 지정한 임계값과 비교합니다.

모델과 scaler는 `backend/model/model.pkl`, `backend/model/scaler.pkl`에 저장됩니다.

## 실행 환경 설치

### 1. 저장소와 Python 환경

```bash
git clone https://github.com/ChoiSeongyong/Customer_Churn_Prediction.git
cd Customer_Churn_Prediction

python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt matplotlib "openai<1"
```

`matplotlib`과 `openai<1`은 백엔드 실행에 필요한 추가 패키지입니다.

### 2. 백엔드 실행

저장소 루트에서 실행합니다.

```bash
python3 -m uvicorn main:app --app-dir backend --reload --port 8000
```

정상 실행 확인:

```text
http://localhost:8000/
http://localhost:8000/docs
```

### 3. 프론트엔드 실행

새 터미널에서 저장소 루트로 이동합니다.

```bash
npm install
npm run dev
```

브라우저에서 `http://localhost:3000`을 엽니다. 프론트엔드는 백엔드가 `http://localhost:8000`에서 실행된다고 가정합니다.

## 입력 CSV

예측용 CSV는 다음 컬럼을 정확히 사용합니다.

```text
name,age,last_login,watch_time,preferred_category,email
```

재학습용 CSV에는 `payment` 컬럼을 추가합니다.

```text
name,age,last_login,watch_time,preferred_category,email,payment
```

`payment_status`는 재학습 코드에서 `payment`로 변환할 수 있지만, 업로드 API의 컬럼 검사는 현재 `payment` 형식을 기준으로 합니다. 날짜는 pandas가 해석할 수 있는 형식이어야 하며, `watch_time`은 숫자로 변환 가능해야 합니다.

## 사용 흐름

1. 대시보드에서 CSV를 업로드합니다.
2. 이탈 판정 임계값을 입력합니다.
3. 예측을 실행합니다.
4. 생성된 `public/stats.json`을 대시보드에서 확인합니다.
5. 이탈 위험 고객 페이지에서 상위 고객을 조회하거나 Excel 파일을 내려받습니다.
6. 버전 저장을 사용한 경우 과거 예측 기록 페이지에서 결과 파일을 확인합니다.

주요 백엔드 API:

| API | 역할 |
| --- | --- |
| `POST /upload_csv` | CSV 저장, 컬럼 검사, 필요 시 재학습 |
| `POST /set_threshold` | 이탈 판정 임계값 저장 |
| `GET /get_threshold` | 저장된 임계값 조회 |
| `GET /predict` | 이탈 확률 예측 및 통계 생성 |
| `GET /stats` | 생성된 통계 반환 |
| `GET /high-risk-customers` | 확률순 고위험 고객 최대 10명 반환 |
| `GET /download` | 고위험 고객 Excel 다운로드 |
| `GET /list_results` | 버전별 예측 이력 조회 |
| `GET /download_result` | 저장된 예측 결과 다운로드 |
| `POST /send-email` | 추천 이메일 생성·발송 실험 |

## 주요 파일

| 경로 | 역할 |
| --- | --- |
| `backend/main.py` | FastAPI 서버와 업로드·예측·다운로드·이메일 API |
| `backend/scripts/preprocess_and_split.py` | 로그인 경과일과 결제 상태 기반 레이블 생성 |
| `backend/scripts/train_model.py` | scaler와 Random Forest 학습·저장 |
| `backend/scripts/predict_with_model.py` | 이탈 확률 계산, 고위험 고객·통계·이력 파일 생성 |
| `backend/model/` | 학습된 모델과 scaler |
| `src/app/dashboard/page.tsx` | 고객 통계 대시보드 |
| `src/app/dashboard/customers/page.tsx` | 고위험 고객 조회와 Excel 다운로드 |
| `src/app/dashboard/history/page.tsx` | 버전별 예측 기록 조회 |
| `src/components/dashboard/` | 표, 카드와 차트 UI |
| `src/components/auth/` | 데모 로그인 상태와 폼 |
| `public/stats.json` | 프론트엔드가 읽는 최신 예측 통계 |

## 보안 설정

`backend/main.py`의 외부 서비스 인증정보는 환경변수로 분리하고, 기존에 사용한 키와 비밀번호는 폐기·재발급한 뒤 이메일 API를 사용합니다.

## UI 원본과 라이선스

프론트엔드는 [Devias Kit React](https://github.com/devias-io/material-kit-react)를 기반으로 수정했습니다. 포함된 원본 템플릿의 라이선스는 [`LICENSE.md`](LICENSE.md)를 확인하십시오. 사용한 라이브러리와 외부 API에는 각각의 이용 조건이 적용됩니다.
