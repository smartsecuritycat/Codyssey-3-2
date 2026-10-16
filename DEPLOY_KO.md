# 실제 서비스 배포 가이드

## 현재 상태

소스/배포 설정은 포함되어 있으나 **계정 생성, 프로젝트 생성, 코드 푸시, 실제 배포는 실행하지 않았습니다.** 이 문서의 `YOUR-*` 문자열은 가짜 배포 링크가 아니라 교체할 자리표시자입니다. 실제 키를 문서에 적지 마세요.

## A. Firebase Firestore 준비

Firebase 콘솔에서 새 프로젝트를 만들고 **Firestore Database**를 선택합니다. Standard edition의 Native Firestore를 사용합니다. Realtime Database나 MongoDB 호환 데이터베이스가 아닙니다. 기본 DB 이름은 `(default)`를 사용합니다. 프로젝트 위치는 본인이 접근하기 적절한 위치로 선택합니다.

외부 브라우저에서 직접 DB에 접근하지 않으므로, 시작 규칙은 production 모드의 차단 규칙을 사용해도 됩니다. `backend/firestore.rules`는 모든 클라이언트 직접 접근을 거부합니다. 서버 Admin SDK의 권한은 별도 IAM으로 관리됩니다.

프로젝트 설정의 서비스 계정에서 Python 서버용 JSON 키를 준비합니다. 서비스 계정이 Firestore 데이터를 읽고 쓸 수 있는 IAM 권한을 가졌는지 확인합니다. 권한 오류가 난다고 데이터베이스를 누구나 읽고 쓰는 규칙으로 열지 마세요.

로컬에서는 `backend/secrets/firebase-service-account.json`과 같이 Git에서 제외되는 위치에 보관하고, `FIREBASE_SERVICE_ACCOUNT_PATH=secrets/firebase-service-account.json`로 지정합니다. 배포 시에는 Render의 `FIREBASE_SERVICE_ACCOUNT_JSON`에 JSON 파일 **전체 내용**을 비밀 환경 변수로 넣습니다. 코드는 `json.loads()`로 읽습니다. private_key의 줄바꿈은 JSON 형식의 `\n`으로 유지됩니다.

처음에는 `data`/`conversations`를 콘솔에서 빈 문서로 만들 필요가 없습니다. 데이터 추가와 대화 저장 시 컬렉션이 생성됩니다. `_usage`는 일일 GPT 시도 수를 유지하는 내부 컬렉션입니다.

## B. OpenAI와 앱 토큰

OpenAI 개발자 플랫폼에서 API 프로젝트/키를 준비하고, 결제 상태와 사용할 모델의 권한을 확인합니다. 앱은 `OPENAI_MODEL=gpt-5.4-mini`를 기본으로 사용하지만 모델의 가용성은 본인 계정에서 확인해야 합니다. 서비스가 오류를 숨기고 데모 응답으로 몰래 대체하지 않도록 했습니다.

앱 접근 토큰은 OpenAI 키와 별도로 직접 만듭니다.

```powershell
.\.venv\Scripts\python.exe -c "import secrets; print(secrets.token_urlsafe(32))"
```

출력한 무작위 문자열을 서버 `APP_ACCESS_TOKEN`에 넣습니다. 이 값은 개인 공간에 접근하는 비밀번호 역할을 하며, 이를 아는 사람은 모든 기록을 변경할 수 있습니다. 프론트 파일에 하드코딩하지 말고 화면의 연결 설정에 직접 입력합니다.

## C. 로컬 production 확인

`backend/.env.production.example`을 `backend/.env`로 복사합니다. JSON 내용 대신 로컬 파일 경로를 사용하는 경우 다음처럼 설정합니다.

```dotenv
APP_MODE=production
OPENAI_API_KEY=실제_키를_로컬_환경_변수에만_입력
OPENAI_MODEL=gpt-5.4-mini
FIREBASE_SERVICE_ACCOUNT_PATH=secrets/firebase-service-account.json
FIREBASE_SERVICE_ACCOUNT_JSON=
APP_ACCESS_TOKEN=직접_생성한_32자_이상_무작위_토큰
ALLOWED_ORIGINS=http://localhost:5500,http://127.0.0.1:5500
OPENAI_MAX_OUTPUT_TOKENS=1000
CHAT_PER_MINUTE=6
CHAT_DAILY_LIMIT=30
```

`.env`에 예시 문구 자체를 넣으면 정상 동작하지 않습니다. Firebase/키/토큰은 실제 값이 필요합니다. 서버에 이미 같은 이름의 OS 환경 변수가 있으면 `.env`보다 우선합니다.

프로젝트 루트에서 다음을 실행합니다.

```powershell
cd backend
..\.venv\Scripts\python.exe -m scripts.seed_data --confirm-production
..\.venv\Scripts\python.exe -m uvicorn main:app --reload
```

별도 터미널에서 프론트를 띄웁니다. `run_demo.py`는 production을 강제로 demo로 바꾸므로 여기서는 사용하지 않습니다.

```powershell
.\.venv\Scripts\python.exe -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

`/docs`에서 Authorize → 앱 토큰 → `/api/data/summary`를 실행해 `count >= 100`인지 확인합니다. 다음으로 웹에서 GPT 질문 한 번을 보내고 Firestore `conversations`에 저장됐는지 확인합니다. 이 질문에는 과금이 생길 수 있습니다.

## D. GitHub에 소스 올리기

개인 GitHub에서 새 저장소를 만든 뒤 프로젝트 루트를 푸시합니다. 초기에 README를 자동 생성하지 않은 빈 저장소를 사용하는 편이 간단합니다.

```bash
git init
git add .
git status
# git diff --cached 로 실제 키가 포함되지 않았는지 확인한 다음 실행
git commit -m "Build Study Mate AI assistant"
git branch -M main
git remote add origin https://github.com/YOUR-USER/YOUR-REPO.git
git push -u origin main
```

`git status`에 `.env`, `secrets/`, 개인 CSV/스크린샷, 키 JSON이 올라오는지 반드시 확인합니다. 기본 예제 CSV는 합성 데이터라 포함할 수 있습니다. `.gitignore`는 이미 커밋된 비밀을 지워 주지 않습니다. 키를 올린 적이 있다면 단순 삭제만 하지 말고 해당 키를 폐기·재발급하세요.

## E. Render 백엔드 배포

Render에서 **New → Web Service**, GitHub 저장소 연결 후 다음으로 설정합니다.

| 설정 | 값 |
|---|---|
| Runtime | Python |
| Root Directory | `backend` |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn main:app --host 0.0.0.0 --port $PORT --workers 1` |
| Health Check Path | `/health` |
| Instance | 실습 목적의 Free 선택 가능, 최신 조건 확인 |

필수 환경 변수는 `APP_MODE=production`, `OPENAI_API_KEY`, `OPENAI_MODEL`, `FIREBASE_SERVICE_ACCOUNT_JSON`, `APP_ACCESS_TOKEN`, `ALLOWED_ORIGINS`입니다. 출력·호출 상한도 README 기본값을 사용합니다. Blueprint를 사용하면 저장소 루트의 `render.yaml`을 이용할 수 있습니다. `sync: false` 항목에는 사용자가 실제 값을 입력해야 합니다. Blueprint가 생성한 앱 토큰을 Render 설정에서 확인해 브라우저에 직접 입력합니다.

Vercel 주소를 아직 모르면 먼저 프로젝트를 생성해 도메인을 확인하거나, 로컬 origin으로 백엔드를 배포한 뒤 다음 단계에서 정확한 Vercel origin을 추가합니다. 무조건 `*`로 열지 마세요.

배포가 성공하면 실제 Render URL과 `/docs`를 확인합니다. `/health`의 `external_services_verified: false`는 의도적인 표시입니다. 이 경로는 **프로세스 기동 상태**만 확인합니다. Firestore 연결 성공은 `/api/data/summary`, GPT 연결 성공은 `/api/chat`으로 별도 확인합니다.

Render 무료 Web Service는 오랫동안 요청이 없으면 쉬었다가 다시 기동할 수 있습니다. 처음 접속해도 바로 계속 새로고침하지 말고 안내 문구를 확인합니다. Render의 로컬 디스크는 영구 저장소가 아니므로 실제 데이터는 Firestore에 둡니다.

## F. Vercel 프론트 배포

Vercel의 새 프로젝트에서 같은 GitHub 저장소를 가져옵니다.

| 설정 | 값 |
|---|---|
| Framework Preset | **Other** |
| Root Directory | `frontend` |
| Build Command | `npm run build` |
| Output Directory | `dist` |
| 환경 변수 | `API_BASE_URL=https://실제-Render주소.onrender.com` |

`frontend/vercel.json`에 빌드/출력 설정이 있습니다. 프레임워크를 새로 설치할 필요는 없습니다. Node.js 20 이상이 필요하며, 프론트 런타임 의존성은 없습니다. API_BASE_URL을 넣을 환경(Production, 필요하면 Preview)을 구분합니다. 비밀 키를 Vercel 프론트에 넣지 않습니다.

Vercel의 최종 production 주소를 확인한 다음 Render의 허용 origin에 등록합니다.

```dotenv
ALLOWED_ORIGINS=https://실제-프로젝트.vercel.app
```

여러 origin을 허용해야 하면 쉼표로 구분합니다. URL 경로·query는 넣지 않고 프로토콜+호스트+필요 시 포트까지만 넣습니다. Vercel에서 API 주소를 바꾼 뒤에는 재배포해야 합니다. 브라우저 Network 탭에서 `config.js`가 올바른 Render origin을 포함하는지 확인합니다.

## G. 실제 동작 확인과 제출

Vercel 화면에 접속해 앱 토큰을 입력하고 데이터 요약을 확인합니다. 질문을 보내 실제 GPT 답변과 저장을 확인하고, 기록 한 개의 값을 수정한 후 요약이 갱신되는지 봅니다. 대화 기록 화면에서 이전 대화를 불러와 메시지가 다시 표시되는지 확인합니다.

백엔드 테스트 스크립트는 README의 `scripts.live_smoke` 명령을 사용합니다. 비용이 드는 GPT 검증은 `--allow-paid-chat`을 명시한 경우만 실행됩니다. 이 스크립트는 스스로 새로 만든 임시 기록/대화를 삭제하며, 기존 기록은 덮어쓰지 않습니다. 동시에 다른 사용자가 조작하는 동안에는 검증을 실행하지 마세요.

그다음 README의 세 가지 URL 칸을 실제 값으로 채우고 production 화면을 캡처합니다. 준비된 데모 캡처를 실제 GPT/배포 증거로 제출하지 않습니다. 평가자가 앱을 확인할 수 있도록 토큰 전달 방법은 과목 담당자와 별도로 정하되, 공개 README에 비밀 토큰을 적지 않습니다.

## 문제 해결

| 현상 | 먼저 확인할 것 |
|---|---|
| 401 | OpenAI 키가 아니라 **앱 토큰**을 입력했는지. 새로고침 후 재입력했는지 |
| 409 날짜 충돌 | 이미 같은 날짜가 있는지. 추가 대신 기존 날짜 수정 |
| 409 처리 중 | 요청이 끝나기 전에 다른 변경 요청을 보냈는지 |
| 422 | YYYY-MM-DD, 오늘 이하, 숫자 0~1440, 메모 300자 이하 |
| GPT 429 | 앱 요청 상한 또는 OpenAI의 잔액/요청 한도 |
| 채팅 502/503/504 | 모델 권한·키·제공자 상태·시간 초과. 저장 기록 먼저 확인 |
| DB 503 | 서비스 계정 권한, Firestore 생성 여부, JSON 형식, 네트워크 |
| 브라우저 CORS 오류 | 실제 Vercel 주소가 Render `ALLOWED_ORIGINS`와 정확히 같은지 |
| Vercel 화면이 로컬 API를 호출 | API_BASE_URL 설정 뒤 **재배포**했는지 |
| Render 배포 즉시 실패 | 키/토큰/APP_MODE, rootDir, requirements 설치 로그 |
| 서버 상태는 ok인데 대화 안 됨 | /health는 외부 연결 시험이 아님. /summary와 /chat을 각각 검증 |
| 키 없는 데모가 안 됨 | Python/패키지 설치와 8000/5500 포트 사용 여부 확인 |
