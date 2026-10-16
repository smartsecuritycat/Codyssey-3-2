# Study Mate — 나의 데이터를 아는 AI 학습 비서

일별 학습 시간 `(date, value, memo)`를 저장하고, **서버가 계산한 통계를 근거로 대화하는 웹 서비스**입니다. 학습 시간이 최근 늘었는지, 어느 날 가장 오래 공부했는지, 이번 달에 기록한 시간이 얼마인지를 확인합니다.

> **전달 상태:** 앱 소스, 180개 합성 데이터, 배포 설정, 실행 문서, 로컬 테스트와 스크린샷을 포함합니다. **Render/Vercel 실배포, 실제 Firebase 프로젝트 연결, 실제 GPT 호출은 아직 수행하지 않았습니다.** 아래 순서대로 키를 설정하고 배포해야 제출용 서비스가 됩니다. ZIP만 제출해 배포 요구까지 충족했다고 표기하면 안 됩니다.

## 1. 빠른 시작 — 키 없는 로컬 데모

Python 3.10 이상을 설치합니다. 패키지 설치에는 인터넷이 필요합니다. 파일을 압축 해제한 `study-agent` 폴더에서 터미널을 엽니다.

**Windows PowerShell**

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r backend\requirements.txt
.\.venv\Scripts\python.exe run_demo.py
```

**macOS / Linux**

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r backend/requirements.txt
.venv/bin/python run_demo.py
```

화면: `http://127.0.0.1:5500` / Swagger: `http://127.0.0.1:8000/docs`

가상환경을 활성화하지 않아도 위 명령은 그 안의 Python을 직접 사용합니다. 프론트 HTML을 더블클릭하는 대신 HTTP 주소로 접속합니다. 터미널에서 Ctrl+C를 누르면 두 서버가 종료됩니다. 8000/5500 포트가 이미 사용 중이면 이전 서버를 종료하세요.

**중요한 모드 차이**

| 설정 | 저장소 | 답변 | 용도 |
|---|---|---|---|
| `APP_MODE=demo` | 로컬 SQLite | 규칙 기반 고정 로직, **GPT 아님** | 키 없이 UI·CRUD·분석·저장 흐름 확인 |
| `APP_MODE=production` | **Firebase Firestore** | **OpenAI GPT Responses API** | 실제 과제 제출·배포 |

`run_demo.py`는 의도적으로 demo만 실행합니다. 제출용 production은 뒤의 실행/배포 명령을 사용합니다. Render에서 demo로 시작하면 설정 오류로 중단하도록 했습니다. 데모 DB는 `backend/.local/study_agent.sqlite3`에 생성됩니다. 최초 실행 때만 예제를 넣으며, 기록을 지운 뒤 서버를 다시 켠다고 삭제한 데이터를 자동 복구하지 않습니다.

## 2. 기능 및 요구사항 대응

| 요구사항 | 구현 위치와 동작 |
|---|---|
| 데이터 기반 AI 채팅 | `/api/chat` → 최신 요약 계산 → 시스템 컨텍스트 주입 → GPT → Firestore 자동 저장. 로딩 표시, 빈 입력 방지, 오류 안내 |
| 데이터 CRUD | 추가/목록/전체 수정/삭제 모두 API 및 화면 구현. 수정·삭제 후 요약 갱신 |
| 대화 기록 | 수동 저장 API, 자동 저장, 목록, 삭제. **방식 A**: `/api/conversations/{id}`에서 전체 메시지 불러오기 |
| 데이터 요약 | 기간/개수/합계/평균/최대/최소/추세/월별 통계/결측과 0분 구분 |
| 백엔드 구조 | `routers` / `services` / `repositories` 분리. Pydantic 요청 검증 |
| 프론트엔드 | HTML/CSS/JavaScript. **React/Vue 등 프레임워크 없음**, 차트도 Canvas 직접 구현 |
| 배포 설정 | 루트 `render.yaml`, `frontend/vercel.json`, `frontend/build.mjs` |
| 보너스 UX | 7일 이동평균 그래프, 표준편차·일별 선형 기울기, CSV/JSON 내보내기, 다크 모드, 모바일 화면 |
| 보너스 도구 호출 | **미포함**: Function Calling + MCP/GPT Actions 외부 채널 검증은 하지 않음 |

현재 구조는 **컨텍스트 주입형 AI 비서**입니다. 자율적으로 여러 도구를 고르는 고급 에이전트와는 구분합니다.

## 3. 기술 스택과 프로젝트 구조

백엔드는 Python / FastAPI / Uvicorn / Pydantic, 운영 DB는 `firebase-admin`의 Firestore, AI는 `openai` SDK, 설정은 `python-dotenv`입니다. 프론트는 바닐라 HTML/CSS/JS이며, Node.js는 Vercel에서 공개 API 주소를 파일로 만드는 빌드에만 사용합니다. 로컬 데모에는 Node.js가 필수가 아닙니다.

```text
study-agent/
├── README.md                         # 이 문서
├── START_HERE_KO.md                   # 처음 시작하는 순서
├── run_demo.py                       # 로컬 서버 두 개 실행
├── render.yaml                       # Render Blueprint
├── browser_smoke_test.py              # 브라우저 테스트
├── backend/
│   ├── main.py                       # uvicorn main:app
│   ├── .env.example                  # 로컬 환경 변수 예시
│   ├── .env.production.example       # 운영 환경 변수 예시
│   ├── requirements.txt
│   ├── requirements-dev.txt
│   ├── firestore.rules               # 클라이언트 직접 접근 거부
│   ├── app/
│   │   ├── factory.py                # 앱/의존성 초기화, CORS, 예외 처리
│   │   ├── config.py                 # 환경 변수 검증
│   │   ├── models.py                 # Pydantic 입력/응답 모델
│   │   ├── security.py               # 앱 토큰, 요청 상한, 동시 변경 제한
│   │   ├── routers/                  # HTTP 엔드포인트
│   │   ├── services/                 # 요약, 데이터, 대화, 채팅, GPT 어댑터
│   │   └── repositories/             # Firestore / SQLite 데모
│   ├── sample_data/study_minutes.csv # 180개 합성 학습 기록
│   ├── scripts/seed_data.py          # CSV 적재
│   ├── scripts/live_smoke.py         # 실제 배포 연결 점검
│   └── tests/                        # 단위/API/어댑터 계약 테스트
├── frontend/
│   ├── index.html / styles.css / app.js
│   ├── config.js                     # 로컬 기본 API 주소
│   ├── build.mjs                     # 공개 환경 변수 주입
│   ├── package.json                  # 빌드 명령만 사용, 프레임워크 없음
│   └── vercel.json
└── docs/
    ├── DESIGN_KO.md                   # 설계·학습 목표 설명
    ├── DEPLOY_KO.md                   # Firebase/Render/Vercel 단계별 안내
    ├── SUBMISSION_CHECKLIST.md
    ├── TEST_REPORT.md
    ├── sample_summary.json
    └── screenshots/                   # DEMO임을 표시한 실제 로컬 UI 캡처
```

## 4. 데이터와 분석 기준

기본 데이터는 **2026-04-19~2026-10-15, 180일의 합성 학습 시간**입니다. 실제 본인 기록으로 가장하지 않습니다. 생성 규칙은 `backend/sample_data/README.md`에 적었습니다. 교사가 실제 관측 데이터를 요구한다면 본인의 실제 100개 이상 데이터로 교체해야 합니다.

`value`는 분 단위의 0~1440 숫자이고, 날짜마다 하나의 기록을 둡니다. 같은 날짜를 두 번 추가하면 409를 반환합니다. 날짜를 바꾸는 수정은 새 날짜에 중복이 없는지 확인하고 저장소의 원자적 작업으로 이동합니다.

평균은 **기록된 날짜**의 평균입니다. 0분 기록은 평균에 들어가지만, 아예 없는 날짜는 자동으로 0분이 되지 않습니다. 최근 추세는 **마지막 저장 날짜까지 7일**과 **그 직전 7일**의 평균을 비교합니다. 두 구간이 모두 7개 기록을 포함할 때만 ±5% 기준으로 증가/감소/유지를 판단합니다. 이전 평균이 0분인 상태에서 양수로 바뀌면 증가율을 숫자로 꾸미지 않습니다. ±5%는 이 프로젝트의 표시 규칙이지 통계적 유의성 검정이 아닙니다.

월별 통계는 부분 월과 완전한 월을 표시합니다. “이번 달”은 오늘의 한국 시간 기준이고, 최신 데이터가 지난달에 끝났다고 그것을 이번 달로 바꾸지 않습니다. 변화만 보고 시험·건강·의지 같은 원인을 단정하지 않습니다.

실제 요약 예시는 `docs/sample_summary.json`에 있습니다. 원본 예제 기준 합계는 **20,484분**, 평균 **113.80분**, 최근 7일 평균 **153.57분**, 직전 7일 평균 **170.14분**, 변화율 **-9.74%**입니다. 전체 기간은 상승 기울기를 보이더라도 최근 두 주는 감소할 수 있다는 점을 함께 보여줍니다.

## 5. API 계약

운영에서는 모든 `/api/*` 요청에 `Authorization: Bearer <APP_ACCESS_TOKEN>`이 필요합니다. Swagger 오른쪽 **Authorize**에도 OpenAI 키가 아니라 이 앱 전용 토큰을 입력합니다. `/`, `/health`, `/docs`, `/openapi.json`은 공개입니다.

| 메서드 | 경로 | 응답/특징 |
|---|---|---|
| POST | `/api/data` | 201, `{date,value,memo}` 추가 |
| GET | `/api/data?limit=50&offset=0` | `{items,total,limit,offset}`, 최신 날짜순. limit 최대 500 |
| PUT | `/api/data/{id}` | 200, 전체 수정. 날짜 변경 시 반환 ID도 변경 |
| DELETE | `/api/data/{id}` | 204, 응답 본문 없음 |
| GET | `/api/data/summary` | 전체 기록의 통계·월별 값·시계열 |
| GET | `/api/data/export?format=csv` | CSV 또는 `format=json` 내보내기 |
| POST | `/api/conversations` | 201, `{title,messages}` 수동 저장. 완료된 user/assistant 쌍 |
| GET | `/api/conversations?limit=50&offset=0` | 목록만 반환, **messages 미포함**. limit 최대 100 |
| GET | `/api/conversations/{id}` | 전체 `messages` 반환. 불러오기 UX의 기준 |
| DELETE | `/api/conversations/{id}` | 204 |
| POST | `/api/chat` | `{message,conversation_id?,request_id?}` → 답변/대화 ID/저장 여부/근거 요약 |

`data` 문서 ID는 날짜, `conversations` 문서 ID는 UUID입니다. `request_id`를 재사용한 완료 요청은 저장된 응답을 돌려줍니다. 같은 ID에 다른 질문을 넣으면 409입니다. 단, GPT 과금과 DB 저장을 하나의 분산 트랜잭션으로 묶지는 못하므로 **외부 호출 직후 프로세스가 죽는 상황의 정확히 한 번 과금까지 보장하지 않습니다**. 화면과 SDK는 자동 재전송을 하지 않습니다.

채팅 서비스는 자기 자신의 `/api/data/summary`를 HTTP로 재호출하지 않고, 그 엔드포인트와 **동일한 `DataService.summary()` 함수**를 재사용합니다. 불필요한 네트워크 비용 없이 같은 데이터 요약을 시스템 컨텍스트에 주입합니다.

## 6. 환경 변수

| 변수 | 위치 | 필요 조건 / 설명 |
|---|---|---|
| `APP_MODE` | 백엔드 | `demo` / `production`. 제출은 production |
| `OPENAI_API_KEY` | **백엔드만** | production 필수, 실제 과금 가능한 API 키 |
| `OPENAI_MODEL` | 백엔드 | 기본 `gpt-5.4-mini`. 사용 권한/현재 지원은 본인 프로젝트에서 확인 |
| `OPENAI_MAX_OUTPUT_TOKENS` | 백엔드 | 기본 1000, 허용 128~4000 |
| `OPENAI_TIMEOUT_SECONDS` | 백엔드 | 기본 45초 |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | **백엔드만** | 서비스 계정 JSON 전체 내용. 운영 배포 권장 방식 |
| `FIREBASE_SERVICE_ACCOUNT_PATH` | **백엔드만** | JSON 환경 변수 대신 서버의 키 파일 경로. 상대경로 기준은 backend |
| `ALLOWED_ORIGINS` | 백엔드 | 쉼표로 구분한 **정확한 origin**, `*` 불가. `/` 뒤 경로 제외 |
| `APP_ACCESS_TOKEN` | 백엔드 + 사용자가 UI에 직접 입력 | production에서 최소 32자 무작위 문자열. OpenAI 키와 별개 |
| `CHAT_PER_MINUTE` | 백엔드 | 기본 앱 전체 분당 6회, 단일 프로세스 메모리 제한 |
| `CHAT_DAILY_LIMIT` | 백엔드 | 기본 한국 시간 날짜당 GPT 시도 30회. 실패도 포함, DB에 유지 |
| `MAX_DATA_RECORDS` | 백엔드 | 기본 5000. 초과 시 부분 요약을 사실처럼 내놓지 않고 오류 |
| `DEMO_DB_PATH` | 데모 백엔드 | 기본 `.local/study_agent.sqlite3` |
| `DEMO_AUTO_SEED` | 데모 백엔드 | 기본 true, 최초 실행 때만 예제 180개 추가 |
| `API_BASE_URL` | **Vercel 빌드** | 실제 Render origin. 공개 가능한 서버 주소만 포함 |

`FIREBASE_SERVICE_ACCOUNT_JSON`과 `FIREBASE_SERVICE_ACCOUNT_PATH` 중 하나를 사용합니다. 둘 다 있으면 JSON이 우선입니다. 환경 변수의 날짜 기준은 한국 시간입니다.

**바닐라 프론트 환경 변수 처리:** 브라우저에서 `process.env`를 직접 읽지 않습니다. Vercel 빌드가 `API_BASE_URL`만 읽어 `dist/config.js`를 생성하고, 브라우저는 `window.APP_CONFIG`를 읽습니다. `frontend/.env.example`은 설명용이며 파일을 복사했다고 Node가 자동 로드하는 구조는 아닙니다. Vercel 설정 화면에 넣거나 빌드 프로세스 환경 변수로 전달하세요. 비밀 키·앱 토큰은 이 빌드 파일에 넣지 않습니다.

## 7. 실제 Firestore/GPT로 로컬 실행

자세한 콘솔 단계는 `docs/DEPLOY_KO.md`를 따릅니다.

1. Firebase 프로젝트와 Firestore **Standard / Native 모드** DB를 만들고, 서비스 계정 키를 서버용으로 준비합니다. Realtime Database가 아닙니다.
2. `backend/.env.production.example`을 `backend/.env`로 복사한 뒤 실제 값으로 채웁니다. 키 파일은 예를 들어 `backend/secrets/`에 두고 환경 변수에는 경로만 넣습니다. 이 폴더와 `.env`는 Git에서 제외합니다.
3. 로컬 프론트를 연결하려면 `ALLOWED_ORIGINS=http://localhost:5500,http://127.0.0.1:5500`로 둡니다.
4. 프로젝트 루트의 `.venv`를 활성화하거나, 아래처럼 가상환경 Python을 직접 지정합니다.

```powershell
cd backend
# Firestore에 180개 합성 예제 기록 적재. 기존 날짜는 덮어쓰지 않습니다.
..\.venv\Scripts\python.exe -m scripts.seed_data --confirm-production
# 실제 운영 스택으로 로컬 백엔드 실행
..\.venv\Scripts\python.exe -m uvicorn main:app --reload
```

다른 터미널을 프로젝트 루트에서 열고 프론트를 실행합니다.

```powershell
.\.venv\Scripts\python.exe -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

프론트의 **연결 설정**에 `http://127.0.0.1:8000`과 앱 전용 토큰을 입력합니다. 토큰은 페이지 메모리에만 두므로 새로고침하면 다시 입력합니다. 실제 키를 ChatGPT 대화나 README에 붙이지 마세요.

실제 CSV로 교체할 때는 새 프로젝트/빈 DB를 쓰거나 기존 합성 기록을 명확히 정리한 다음 실행합니다. 적재기는 기존 날짜를 자동 삭제/덮어쓰기하지 않습니다.

```powershell
..\.venv\Scripts\python.exe -m scripts.seed_data --csv "C:/data/my-study.csv" --source user --confirm-production
```

## 8. Render / Vercel 배포

**현재 실제 배포 URL은 없음. 아래 칸을 배포가 성공한 후 본인 URL로 교체합니다.**

| 항목 | 제출 시 기록할 값 |
|---|---|
| 프론트 | `미배포 — 실제 Vercel URL 입력` |
| 백엔드 API | `미배포 — 실제 Render URL 입력` |
| Swagger | `미배포 — 실제 Render URL + /docs 입력` |

GitHub에 비밀값을 제외한 소스를 올린 뒤 다음 값으로 배포합니다.

| 서비스 | Root Directory | Build | Start / Output |
|---|---|---|---|
| Render Web Service | `backend` | `pip install -r requirements.txt` | `uvicorn main:app --host 0.0.0.0 --port $PORT --workers 1` |
| Vercel, Framework Other | `frontend` | `npm run build` | Output: `dist` |

Render에는 운영 환경 변수를 설정하고, Vercel에는 `API_BASE_URL`만 설정합니다. Vercel 주소가 확정되면 Render의 `ALLOWED_ORIGINS`를 해당 주소로 수정합니다. Vercel 환경 변수를 변경한 경우 **다시 배포하여 config.js를 갱신**합니다. Preview 배포 주소는 Production 주소와 다를 수 있으며 필요한 주소만 정확히 추가합니다.

Render 무료 서버는 비활성 상태 후 첫 요청에 지연이 생길 수 있습니다. 화면은 기동 안내와 150초의 네트워크 타임아웃을 두지만, 응답 지연이 항상 그 안에 해결된다고 보장하지는 않습니다. 데이터는 Render 로컬 파일이 아니라 Firestore에 저장합니다. 영구적인 무료 운영을 보장하는 문서는 아닙니다. 최신 플랜/쿼터/과금 조건을 배포 콘솔에서 확인하세요.

## 9. 테스트

```powershell
# 프로젝트 루트
.\.venv\Scripts\python.exe -m pip install -r backend\requirements-dev.txt
cd backend
..\.venv\Scripts\python.exe -m pytest -q
```

제작 환경에서 **67개 테스트 통과**. CRUD, 입력 범위, 미래 날짜, 중복 날짜, 날짜 변경 충돌, 보존성, 대화 불러오기, 컨텍스트 갱신, 빈 데이터, 추세 계산, CORS/인증, 요청 제한, SDK 어댑터의 요청/오류 계약을 포함합니다. 실제 OpenAI/Firestore가 아니라 테스트 저장소/대역을 사용한 결과와 실서버 연결 검증은 구분합니다.

브라우저 UI 통합 검증은 **17개 항목 통과**. 제작 환경 브라우저의 네트워크가 제한되어 **테스트 전송 어댑터로 실제 로컬 FastAPI에 HTTP 요청을 전달**했습니다. 배포 환경의 브라우저 직접 CORS 연결을 검증한 것은 아닙니다. 전체 범위는 `docs/TEST_REPORT.md`와 `docs/browser_test_results.json`을 참고하세요.

실제 사용자 PC에서는 다음으로 일반 브라우저 HTTP 테스트를 실행할 수 있습니다. 테스트는 임시 기록을 추가·수정·삭제하고 테스트 대화를 저장합니다. 기존 본인 기록이 없는 **별도 데모 DB**에서 실행하세요.

```powershell
# 별도 터미널에서 run_demo.py 실행 중
.\.venv\Scripts\python.exe -m pip install playwright httpx
.\.venv\Scripts\python.exe -m playwright install chromium
.\.venv\Scripts\python.exe browser_smoke_test.py
```

실제 배포가 끝난 뒤에는 아래 점검을 사용합니다. 앱 토큰을 환경 변수로 설정해야 합니다. `--allow-paid-chat`을 붙인 명령만 **유료 GPT 요청 1회**를 보냅니다.

```powershell
cd backend
$env:APP_ACCESS_TOKEN="본인 앱 전용 토큰"
..\.venv\Scripts\python.exe -m scripts.live_smoke --api-url "https://YOUR-API.onrender.com" --origin "https://YOUR-FRONT.vercel.app"
# 유료 요청에 동의하고 실행할 때만:
..\.venv\Scripts\python.exe -m scripts.live_smoke --api-url "https://YOUR-API.onrender.com" --origin "https://YOUR-FRONT.vercel.app" --allow-paid-chat
```

## 10. 제출 스크린샷

아래는 **실제 로컬 데모 화면 캡처**입니다. 디자인 목업이나 실제 GPT 호출 증거가 아닙니다. 파일명과 화면에 DEMO / GPT 미사용을 표시했습니다. 최종 제출에는 production으로 연결한 화면에서 같은 동작을 수행하고 다시 캡처하세요.

| 파일 | 확인 내용 |
|---|---|
| `docs/screenshots/01_chat_summary_DEMO.png` | 데이터 요약, 질문+응답, 그래프, 자동 저장 안내 |
| `docs/screenshots/02_data_update_DEMO.png` | 95분→110분 수정, 저장 완료 문구, 갱신된 데이터 목록 |
| `docs/screenshots/03_history_list_DEMO.png` | 이전 대화 목록 |
| `docs/screenshots/04_history_loaded_DEMO.png` | 특정 대화 불러오기 완료와 재표시된 메시지 |
| `docs/screenshots/05_dark_mode_DEMO.png` | 다크 모드 |
| `docs/screenshots/06_mobile_DEMO.png` | 모바일 반응형 화면 |

![로컬 데모: 요약과 채팅](docs/screenshots/01_chat_summary_DEMO.png)

![로컬 데모: CRUD 수정](docs/screenshots/02_data_update_DEMO.png)

![로컬 데모: 대화 불러오기](docs/screenshots/04_history_loaded_DEMO.png)

## 11. 보안·비용·한계

이 앱은 **개인 1명의 실습용 공간**입니다. 회원별 데이터 분리나 다중 사용자 권한 체계가 없습니다. 앱 토큰을 아는 사람은 전체 데이터 조회·변경·삭제가 가능합니다. 여러 사람에게 공개하는 서비스로 확장하려면 Firebase Auth 등 사용자 인증과 소유권 검증이 추가로 필요합니다.

CORS는 인증이 아닙니다. 서버의 앱 토큰 검사로 데이터와 유료 GPT 엔드포인트를 보호합니다. Firebase Admin SDK는 클라이언트 보안 규칙이 아니라 IAM 권한으로 접근하므로, 브라우저 접근 거부 규칙만으로 이 API가 보호된다고 생각하면 안 됩니다. IAM은 필요한 권한만 부여하고 키를 공개 저장소에 넣지 않습니다.

질문은 최대 2000자, 저장 대화는 최대 30회 질문(60개 메시지), GPT로 보내는 과거 대화는 최근 최대 12개 메시지·총 16000자입니다. 출력 토큰과 호출 횟수도 제한하지만 이것이 정확한 원화/달러 지출 상한은 아닙니다. 제공자의 프로젝트 예산·사용량 알림도 확인해야 합니다. `store=False`는 Responses 저장 선택이며 모든 로그나 보존이 무조건 없다는 뜻이 아닙니다.

메모 원문을 프롬프트에서 제외하고 숫자를 서버에서 계산해 위험을 줄였지만, GPT의 환각이나 프롬프트 인젝션을 완전히 제거하지는 못합니다. 중요한 판단은 원본 기록과 대조합니다. 프론트는 메시지/메모를 `textContent`로 표시하고, CSV는 수식으로 해석될 수 있는 메모 앞에 작은따옴표를 붙입니다. 따라서 CSV와 JSON의 메모 문자열이 일부 다를 수 있습니다.

현재 변경 요청은 **단일 프로세스/단일 worker** 기준으로 직렬화합니다. 여러 worker/인스턴스로 늘릴 때는 DB 기반 동시성 제어와 분산 rate limit으로 바꿔야 합니다. 요약·목록은 작은 개인 데이터셋을 가정한 전체 스캔이므로 대규모 운영에는 집계 캐시/커서 페이지네이션을 고려합니다. 외부 제공자의 요청 성공과 로컬 저장 성공을 완벽하게 동시에 보장하지는 않습니다.

## 12. 공식 참고 문서

확인 기준: 2026-10-16. 콘솔 메뉴와 플랜은 변경될 수 있습니다.

- FastAPI CORS: https://fastapi.tiangolo.com/tutorial/cors/
- Pydantic 검증: https://docs.pydantic.dev/latest/concepts/fields/
- Firebase Admin 설정: https://firebase.google.com/docs/admin/setup
- Firestore 시작: https://firebase.google.com/docs/firestore/quickstart
- Firestore 서버 권한과 규칙: https://firebase.google.com/docs/firestore/security/rules-conditions
- OpenAI Responses API: https://developers.openai.com/api/reference/resources/responses/methods/create
- OpenAI 모델: https://developers.openai.com/api/docs/models/gpt-5.4-mini
- Render FastAPI: https://render.com/docs/deploy-fastapi
- Render 무료 인스턴스: https://render.com/docs/free
- Vercel 환경 변수: https://vercel.com/docs/environment-variables
- Vercel 빌드: https://vercel.com/docs/builds/configure-a-build
