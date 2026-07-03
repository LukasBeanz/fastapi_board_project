# FastAPI MySQL 자유게시판 백엔드 가이드

---

## 1. 전체 구조 흐름

```
클라이언트 (Swagger / 프론트엔드)
        ↓ HTTP 요청
    main.py          ← 앱 시작점, 라우터 연결
        ↓
  routers/           ← URL 별로 어떤 로직 실행할지 결정
  auth.py            ← /auth/signup, /auth/login
  boards.py          ← /boards CRUD
        ↓
  schemas.py         ← 요청/응답 데이터 형식 검증
  security.py        ← 비밀번호 해싱, JWT 토큰 발급/검증
        ↓
  models.py          ← DB 테이블 구조 정의
  database.py        ← MySQL 연결 설정
        ↓
    MySQL DB
```

### 파일별 역할

| 파일 | 역할 |
|------|------|
| `main.py` | 앱 시작점. FastAPI 앱 객체 생성, 라우터 등록, 테이블 자동 생성 |
| `database.py` | MySQL 연결 설정. `.env`의 DB 접속 정보를 읽어 엔진과 세션 생성 |
| `models.py` | MySQL 테이블 구조를 파이썬 클래스로 정의. `User` → users 테이블, `Board` → boards 테이블 |
| `schemas.py` | API 요청/응답 데이터 형식 정의 및 자동 검증 |
| `security.py` | 비밀번호 해싱, JWT 토큰 발급/검증, 현재 로그인 사용자 확인 |
| `routers/auth.py` | 회원가입(`/auth/signup`), 로그인(`/auth/login`) API |
| `routers/boards.py` | 게시글 등록/전체조회/상세조회/수정/삭제 API |
| `.env` | DB 접속 정보, JWT 비밀키 등 민감한 설정값 (GitHub 업로드 금지) |
| `mysql_init.sql` | MySQL DB와 사용자 계정 초기 생성 SQL (최초 1회 실행) |
| `requirements.txt` | 프로젝트에 필요한 파이썬 패키지 목록 |

---

## 2. 실행 방법

### 1단계 — MySQL DB 준비 (최초 1회)

MySQL Workbench에서 root 계정으로 접속 후 `mysql_init.sql` 파일을 열고 전체 실행(`Ctrl + Shift + Enter`)합니다.

```sql
-- mysql_init.sql 을 실행하면 아래가 자동으로 생성됩니다.
-- 데이터베이스: fastapi_board_db
-- 사용자 계정: fastapi / fastapi80
```

### 2단계 — 가상환경 활성화 & 패키지 설치 (최초 1회)

```bash
cd fastapi_board_project
.venv\Scripts\activate
pip install -r requirements.txt
```

### 3단계 — `.env` SECRET_KEY 교체 (최초 1회)

터미널에서 랜덤 키를 생성합니다.

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

`.env` 파일의 SECRET_KEY를 생성된 값으로 교체합니다.

```
SECRET_KEY=생성된_랜덤_값_여기에
```

### 4단계 — 서버 실행

```bash
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

또는 `run_server.bat` 더블클릭

### 5단계 — Swagger 접속

브라우저에서 아래 주소로 접속합니다.

```
http://127.0.0.1:8000/docs
```

---

## 3. Swagger 사용법

Swagger는 백엔드 API를 테스트할 수 있는 자동 생성 문서입니다. 프론트엔드 없이도 API 동작을 확인할 수 있습니다.

### 3-1. 회원가입

`POST /auth/signup` → `Try it out` → 아래 내용 입력 후 `Execute`

```json
{
  "username": "gildong",
  "password": "1234",
  "name": "홍길동"
}
```

> 아이디와 비밀번호는 영문+숫자 조합을 권장합니다. 한글 입력 시 Swagger에서 직접 타이핑하면 깨질 수 있으니 붙여넣기를 사용하세요.

### 3-2. 로그인

`POST /auth/login` → `Try it out` → 각 칸에 입력 후 `Execute`

```
username: gildong
password: 1234
```

응답에서 `access_token` 값을 확인합니다.

### 3-3. Swagger Authorize 설정 (게시글 등록/수정/삭제 전 필수)

1. Swagger 우측 상단 `Authorize` 버튼 클릭
2. `username`, `password` 입력
3. `Authorize` 클릭 → `Close`

자물쇠 아이콘이 잠기면 인증 완료입니다.

### 3-4. 게시글 등록

`POST /boards` → `Try it out` → 아래 내용 입력 후 `Execute`

```json
{
  "title": "첫 번째 게시글",
  "content": "내용을 여기에 작성합니다."
}
```

### 3-5. 게시글 전체 목록 조회

`GET /boards` → `Try it out` → `Execute`

로그인 없이도 조회 가능합니다.

### 3-6. 게시글 상세 조회

`GET /boards/{board_id}` → `Try it out` → `board_id`에 번호 입력 후 `Execute`

조회할 때마다 `view_count`가 1씩 증가합니다.

### 3-7. 게시글 수정

`PUT /boards/{board_id}` → `Try it out` → 아래 내용 입력 후 `Execute`

```json
{
  "title": "수정된 제목",
  "content": "수정된 내용입니다."
}
```

본인이 작성한 게시글만 수정 가능합니다.

### 3-8. 게시글 삭제

`DELETE /boards/{board_id}` → `Try it out` → `board_id` 입력 후 `Execute`

본인이 작성한 게시글만 삭제 가능하며, 성공 시 204 응답(본문 없음)을 반환합니다.

### API 흐름 요약

```
회원가입 → 로그인 → Authorize 등록
                          ↓
              게시글 등록 / 수정 / 삭제 (인증 필요)
              게시글 목록 / 상세 조회  (인증 불필요)
```

---

## 4. 백엔드 개발에 필요한 것들

### 이 프로젝트에서 사용한 기술

| 기술 | 역할 |
|------|------|
| **Python** | 백엔드 언어 |
| **FastAPI** | API 서버 프레임워크 |
| **SQLAlchemy** | 파이썬으로 DB 조작 (ORM) |
| **MySQL** | 데이터 저장 |
| **JWT** | 로그인 인증 토큰 |
| **Pydantic** | 요청/응답 데이터 검증 |
| **bcrypt** | 비밀번호 해싱 |
| **Swagger** | 자동 생성 API 문서 및 테스트 도구 |

### Swagger란?

Swagger는 JS 변환 도구가 아닙니다. FastAPI가 코드를 읽고 **자동으로 생성해주는 API 명세서**입니다.

- 백엔드 개발자 → API 완성 후 Swagger URL 공유
- 프론트엔드 개발자 → Swagger 보고 어떤 URL로 어떤 데이터를 주고받는지 파악 후 JS로 연동

```
백엔드 (Python/FastAPI)     프론트엔드 (JS)
  API 완성
      ↓
  Swagger 자동 생성    →    Swagger 보고 API 연동
                            JS 코드로 화면에 표시
```

---

## 5. 다음 단계로 배우면 좋은 것들

| 기술 | 이유 |
|------|------|
| **Git** | 코드 버전 관리, 협업 필수 |
| **Alembic** | DB 테이블 변경사항 관리 (현재는 앱 실행 시 자동 생성) |
| **CORS 설정** | 프론트엔드와 연동할 때 반드시 필요 |
| **Refresh Token** | JWT 보안 강화 (현재는 Access Token만 사용) |
| **Docker** | 서버 배포할 때 필요 |
| **AWS / 클라우드** | 실제 인터넷에 서비스 올리기 |

---

## 6. 백엔드 성장 단계

```
Python 기본
    ↓
FastAPI + MySQL  ← 지금 여기
    ↓
인증 고도화 (Refresh Token, 권한 관리)
    ↓
배포 (Docker + AWS)
    ↓
성능 최적화 (캐싱, 비동기 처리)
```

---

## 7. 주의 사항

- `.env` 파일은 DB 비밀번호와 JWT 비밀키가 포함되어 있으므로 **GitHub에 절대 올리지 않습니다.**
- 아이디(`username`)와 비밀번호(`password`)는 영문+숫자 조합을 사용하세요. 한글은 Swagger에서 입력 시 문제가 생길 수 있습니다.
- `SECRET_KEY`는 반드시 랜덤 문자열로 교체하세요. 기본값 그대로 운영하면 보안에 취약합니다.
- 이 프로젝트는 실습용입니다. 실제 운영 환경에서는 Alembic 마이그레이션, Refresh Token, CORS 설정, 로깅, 예외 처리 구조화를 추가하는 것을 권장합니다.
