# 급식배틀 코딩 에이전트 지침

## 프로젝트 개요

이 저장소는 NEIS 공개 API를 사용해 학교를 검색하고, 선택한 학교의 날짜별
중식 급식을 조회하는 웹 애플리케이션이다. 구현은 다음 구성으로 나뉜다.

- `frontend/`: React 19, TypeScript, Vite, Fluent UI 기반의 브라우저 클라이언트
- `backend/`: Python 3.12 이상, FastAPI 기반의 HTTP API
- `src/openapi.json`: 프론트엔드와 백엔드 사이의 내부 API 계약
- `data/openapi.json`: NEIS 외부 API 명세
- `compose.yml`: 백엔드와 프론트엔드를 함께 실행하는 Docker Compose 설정
- `scripts/start.ps1`, `scripts/start.sh`: 로컬 개발 환경에서 두 서비스를 함께 실행하는 스크립트

제품 요구사항은 `PRD.md`, 기술 설계와 데이터 흐름은 `TRD.md`를 따른다.
NEIS 원본 데이터와 워크숍 문서의 의미를 임의로 바꾸지 않는다.

## 구조와 데이터 흐름

- 프론트엔드는 NEIS를 직접 호출하지 않고 `/api/schools`와 `/api/meals`만 사용한다.
- `GET /api/schools?q=...`는 학교명 부분 검색 결과를 정규화해 반환한다.
- `POST /api/meals`는 학교 식별자, ISO 날짜(`YYYY-MM-DD`) 범위를 받아 중식만 반환한다.
- 백엔드는 NEIS의 `ATPT_OFCDC_SC_CODE`, `SD_SCHUL_CODE`, `MMEAL_SC_CODE=2`,
  `MLSV_FROM_YMD`, `MLSV_TO_YMD` 파라미터를 구성하고 외부 응답을 앱 모델로 변환한다.
- NEIS의 중첩 응답과 `YYYYMMDD` 날짜 형식은 백엔드 경계 안에서만 처리한다.
- `backend/app/models.py`의 Pydantic 모델로 요청·응답을 검증한다. 날짜 범위는
  시작일이 종료일보다 늦을 수 없으며, `MealResponse.has_meals`로 빈 결과와 오류를 구분한다.
- 외부 API 오류, 타임아웃, 네트워크 오류는 `NeisError`와 안전한 사용자 메시지로 변환한다.
  인증키, 외부 URL, 스택 트레이스 및 운영 세부정보를 클라이언트에 노출하지 않는다.

## 개발 환경과 실행

- 로컬 실행에는 Python 3.12 이상과 Node.js 22 이상이 필요하다.
- 처음 실행하면 `backend/.env.example`에서 `backend/.env`를 만든다.
  `NEIS_API_KEY`는 백엔드 환경변수에만 저장하며 커밋하거나 로그에 남기지 않는다.
- PowerShell:

  ```powershell
  .\scripts\start.ps1
  ```

- Bash:

  ```bash
  ./scripts/start.sh
  ```

- 스크립트는 백엔드 가상환경과 의존성을 준비하고 FastAPI(`8000`)와 Vite(`5173`)를
  함께 시작한다. 개별 실행이 필요하면 `backend`에서 `python -m uvicorn app.main:app
  --reload --port 8000`, `frontend`에서 `npm run dev`를 사용한다.
- Docker Compose 실행:

  ```bash
  cp backend/.env.example backend/.env
  docker compose up --build
  ```

- 백엔드 상태 확인 주소는 `http://localhost:8000/health`, API 문서는
  `http://localhost:8000/docs`, 프론트엔드는 `http://localhost:5173`이다.

## 변경 규칙

- 변경 전에 `README.md`, `PRD.md`, `TRD.md`, 관련 `docs/`와 변경 대상 디렉터리의
  구현·테스트를 확인한다.
- 기존 컴포넌트, API 클라이언트, Pydantic 모델, `NeisClient`를 우선 재사용한다.
  프론트엔드에 NEIS 호출이나 백엔드 인증키 처리를 추가하지 않는다.
- 외부 API 응답, 사용자 입력, 날짜 및 학교 코드는 신뢰하지 말고 경계에서 검증한다.
  광범위한 `except`, 조용한 기본값, 오류를 성공 응답으로 바꾸는 fallback을 추가하지 않는다.
- UI 변경 시 학교 검색, 학교 선택, 날짜 범위 검증, 로딩·빈 결과·오류 상태와
  키보드 접근성을 함께 보존한다. Fluent UI와 기존 스타일 패턴을 따른다.
- 내부 계약을 변경하면 `src/openapi.json`, 관련 TypeScript 타입·API 클라이언트,
  FastAPI 모델·라우트 및 테스트를 함께 갱신한다. NEIS 외부 명세
  `data/openapi.json`은 실제 계약 변경 없이 수정하지 않는다.
- 의존성은 `frontend/package.json`/`package-lock.json`과
  `backend/pyproject.toml`에서 관리하며, 필요한 경우에만 최소 범위로 변경한다.
- 생성물과 로컬 설정(`backend/.env`, `node_modules`, `.venv`, 캐시)은 커밋하지 않는다.

## 테스트와 검증

변경에 가장 가까운 검사를 먼저 실행하고, 관련 없는 전체 리팩터링이나 도구를
추가하지 않는다.

- 프론트엔드 빌드: `cd frontend && npm run build`
- 프론트엔드 테스트: `cd frontend && npm test`
- 백엔드 테스트: `cd backend && python -m pytest`
- Compose 설정 검증: `docker compose config`

백엔드 테스트에서는 NEIS 실서비스와 실제 인증키를 사용하지 않는다. 외부 HTTP
호출은 모킹하고, 날짜 검증·응답 정규화·중식 필터·오류 매핑·빈 결과를 결정적으로
검증한다. 프론트엔드 테스트에서도 내부 API 응답을 모킹한다.

## 보안과 설정

- NEIS 인증키는 `backend/.env`의 `NEIS_API_KEY`로만 주입한다.
- 기본 설정은 `NEIS_BASE_URL`, `NEIS_TIMEOUT_SECONDS`, `CORS_ORIGINS`이며,
  운영 환경에서는 환경변수 또는 비밀 관리 시스템으로 제공한다.
- 요청 로그와 오류 메시지에 인증키, 전체 외부 요청 URL, 민감한 사용자 입력을
  기록하지 않는다.
- 백엔드는 허용된 엔드포인트와 파라미터만 외부 API로 전달하며 임의 URL 프록시로
  동작하지 않아야 한다.
- CORS 허용 출처와 공개 포트는 환경에 맞게 최소 범위로 설정한다.

## 문서와 Git

- 공개 동작, 실행 방법, API 계약 또는 설정을 바꾸면 관련 `README.md`, `docs/`,
  `PRD.md`, `TRD.md`를 함께 갱신한다.
- 문서는 실제 구현과 실행 명령만 설명하며 아직 구현되지 않은 기능을 사용 가능한
  것처럼 쓰지 않는다.
- 커밋 메시지는 Conventional Commits 형식(`feat:`, `fix:`, `docs:`, `test:`,
  `refactor:`, `chore:`)을 사용한다.
- 하나의 커밋과 Pull Request에는 하나의 논리적 변경만 포함한다. Pull Request를
  만들 때는 `.github/PULL_REQUEST_TEMPLATE.md`의 관련 항목을 모두 작성하고,
  관련 이슈는 `Closes #123` 형식으로 연결한다.
