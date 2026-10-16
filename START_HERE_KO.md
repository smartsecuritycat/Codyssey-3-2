# 먼저 읽기 — 이 과제 시작 순서

## 1. 무엇을 받았나요?

`Study Mate`라는 학습 시간 분석 AI 비서의 전체 소스입니다. FastAPI 백엔드, 바닐라 웹 화면, Firestore 저장 코드, GPT 호출 코드, 180개 합성 예제, 배포 설정, 테스트와 설명 문서를 포함합니다.

**아직 실제 배포 URL은 없습니다. 실제 Firebase/OpenAI 키를 넣은 연결 시험도 하지 않았습니다.** 로컬 테스트가 통과한 것과 배포가 완료된 것은 다릅니다.

## 2. 먼저 내 컴퓨터에서 화면 보기

압축을 풀고 `study-agent` 폴더 안에서 PowerShell을 엽니다. Python 3.10 이상이 필요합니다.

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r backend\requirements.txt
.\.venv\Scripts\python.exe run_demo.py
```

브라우저에서 `http://127.0.0.1:5500`에 접속합니다. API 문서는 `http://127.0.0.1:8000/docs`입니다. 프로그램을 끄려면 실행한 터미널에서 Ctrl+C를 누릅니다.

이 단계는 **GPT를 쓰지 않는 로컬 데모**라 OpenAI 사용료가 없습니다. 화면에서 최근 추세 질문, 데이터 수정/삭제, 대화 불러오기를 확인할 수 있습니다. 임의 질문을 해석하는 진짜 GPT 기능은 다음 단계가 필요합니다.

## 3. 과제 제출용으로 바꾸기

`docs/DEPLOY_KO.md`를 순서대로 따라갑니다.

1. Firebase Firestore 프로젝트와 서비스 계정 키를 준비합니다.
2. OpenAI API 키와 별도의 앱 접근 토큰을 준비합니다.
3. `backend/.env`에 넣고 production으로 실제 연결을 확인합니다.
4. 예제 180개 또는 실제 본인 CSV를 Firestore에 적재합니다.
5. GitHub에 키를 제외한 코드를 올리고, 백엔드는 Render, 프론트는 Vercel에 배포합니다.
6. 실제 배포 화면에서 GPT 질문, 데이터 변경, 대화 불러오기를 실행하고 URL과 스크린샷을 기록합니다.

키를 이 대화에 붙여 넣지 마세요. 프론트 환경 변수에는 공개 가능한 `API_BASE_URL`만 넣습니다.

## 4. 제출 전 알아야 할 것

180개 기록은 **실제 공부 기록이 아니라 합성 예제**입니다. README에도 그렇게 표시했습니다. 실제 기록만 허용되는 평가라면 본인 데이터로 교체하세요.

필수 API와 화면은 구현했고, 그래프·추가 통계·내보내기·다크 모드를 추가했습니다. 선택 과제인 Function Calling/MCP/GPT Actions는 포함하지 않았습니다.

제작 환경 자동 테스트: 67개 통과. 브라우저 UI 통합 점검: 17개 통과. 후자는 테스트용 HTTP 전송 어댑터를 거쳐 실제 로컬 FastAPI를 호출했습니다. 실배포 검증 증거와는 다릅니다.

제출 목록은 `docs/SUBMISSION_CHECKLIST.md`, 설명 준비는 `docs/DESIGN_KO.md`, 실제 테스트 범위는 `docs/TEST_REPORT.md`를 참고하세요.
