# API Documentation

## 1. 개요

Docs 프로젝트는 **프론트엔드 개발자를 위한 API 레퍼런스 문서**를 관리합니다.

### 1.1 프로젝트 목표

- API 명세 문서 제공
- Gateway 오류 응답 가이드 제공
- 프론트엔드-백엔드 간 계약 문서화
- 버전별 API 변경 이력 관리

### 1.2 대상 독자

- 프론트엔드 개발자 (Web/App)
- QA 엔지니어
- 외부 연동 개발자

---

## 2. 아키텍처

### 2.1 문서 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Documentation                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                       index.html                             │  │
│   │                    (API 목록 통합 문서)                       │  │
│   │                                                              │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │  │
│   │  │ Gateway 오류  │  │ API 목록      │  │ 응답 구조     │   │  │
│   │  │ 응답 가이드   │  │ 네비게이션    │  │ 설명          │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────┘   │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│              ┌───────────────┼───────────────┐                     │
│              ▼               ▼               ▼                     │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
│   │   biz/        │  │   biz/        │  │   guide/      │         │
│   │   auth/       │  │   login/      │  │               │         │
│   │   *.html      │  │   *.html      │  │   *.html      │         │
│   └───────────────┘  └───────────────┘  └───────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 URL 표기 기준

문서의 모든 API URL은 **외부 유입 URL (Gateway 진입점)** 기준으로 작성합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       URL 표기 기준                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   [문서에 표기하는 URL - 외부 URL]                                   │
│   /{clientType}/{version}/{endpoint}                                │
│                                                                     │
│   예시:                                                              │
│   POST /app/v1/signup/getBizRegNo                                   │
│   POST /web/v1/login                                                │
│                                                                     │
│   ─────────────────────────────────────────────────────────────    │
│                                                                     │
│   [문서에 표기하지 않는 URL - 내부 URL]                              │
│   /api/{version}/{endpoint}                                         │
│                                                                     │
│   틀린 예시:                                                         │
│   ❌ POST /api/v1/signup/getBizRegNo                                │
│   ❌ POST /app/api/v1/signup/getBizRegNo                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 디렉토리 구조

```
docs/
├── CLAUDE.md                    # 문서 작성 규칙
├── README.md                    # 이 파일
├── index.html                   # API 목록 통합 문서
│
├── biz/                         # 비즈니스 API 문서
│   ├── auth/
│   │   └── authController.html  # 인증 API
│   ├── login/
│   │   └── loginController.html # 로그인 API
│   ├── signup/
│   │   └── signupController.html # 회원가입 API
│   └── mypage/
│       └── mypageController.html # 마이페이지 API
│
└── guide/                       # 가이드 문서
    └── front-security-api-guide.html
```

---

## 4. 문서 작성 규칙

### 4.1 API 문서 필수 항목

각 API 문서는 다음 항목을 포함해야 합니다:

| 항목 | 설명 |
|------|------|
| API URL | 외부 유입 URL 기준 |
| API Method | POST (모든 API는 POST) |
| API Parameter | @Info 어노테이션 참조 |
| API Response | 성공 응답 예시 |
| API Error Response | 실패 응답 예시 |
| API Description | 목적, 선행 조건, 기대 효과 |
| API Exception Case | ResponseCode별 발생 상황 |

### 4.2 응답 구조

```json
{
    "code": [응답코드],
    "message": "[응답메시지]",
    "data": [응답데이터 또는 null]
}
```

**성공 응답**
```json
{
    "code": 1,
    "message": "OK",
    "data": { ... }
}
```

**실패 응답**
```json
{
    "code": 2001,
    "message": "사용자를 찾을 수 없습니다.",
    "data": null
}
```

### 4.3 Frontend 처리 규칙

```javascript
// 프론트엔드 응답 처리 예시
if (response.code === 1) {
    // 성공: 다음 단계 진행
    processSuccess(response.data);
} else {
    // 실패: message를 사용자에게 표시
    showError(response.message);
}
```

---

## 5. 오류 코드 체계

### 5.1 코드 범위

| 발생 위치 | 코드 범위 | 설명 |
|-----------|-----------|------|
| Gateway | 8xxx | JWT 오류, 버전 정책, 라우팅 오류 |
| Application | 1xxx-7xxx | 비즈니스 오류 |
| Application | 9xxx | 시스템 오류, 파라미터 오류 |

### 5.2 Gateway 오류 코드

| 코드 | 이름 | 설명 |
|------|------|------|
| 8001 | JWT_TOKEN_MISSING | 인증 토큰 없음 |
| 8002 | JWT_TOKEN_INVALID_FORMAT | 토큰 형식 오류 |
| 8003 | JWT_TOKEN_EXPIRED | 토큰 만료 |
| 8004 | JWT_SIGNATURE_INVALID | 서명 검증 실패 |
| 8101 | API_VERSION_BLOCKED | API 버전 차단됨 |
| 8102 | API_VERSION_READ_ONLY | API 버전 읽기 전용 |
| 8103 | API_VERSION_SUNSET | API 버전 서비스 종료 |
| 8201 | ROUTE_NOT_FOUND | 경로를 찾을 수 없음 |
| 8901 | BACKEND_CONNECTION_FAILED | 백엔드 서버 연결 실패 |
| 8902 | SYSTEM_SERVICE_UNAVAILABLE | 시스템 서비스 일시 불가 |

---

## 6. 네비게이션 규칙

### 6.1 index.html → 상세 문서 링크

```html
<!-- index.html -->
<li>
    <a href="biz/signup/signupController.html#getBizRegNo">
        <span class="method post">POST</span>
        <span class="api-url">/{clientType}/{version}/signup/getBizRegNo</span>
        <span class="api-desc">사업자등록번호 중복 체크</span>
    </a>
</li>
```

### 6.2 상세 문서의 앵커 ID

```html
<!-- signupController.html -->
<div class="api-section" id="getBizRegNo">
    <h2>1. 사업자등록번호 중복 체크 (getBizRegNo)</h2>
    <!-- API 상세 내용 -->
</div>
```

---

## 7. 문서 템플릿

### 7.1 API 섹션 템플릿

```html
<div class="api-section" id="{methodName}">
    <h2>{순번}. {API 설명} ({methodName})</h2>

    <!-- API 기본 정보 -->
    <div class="api-info">
        <p><strong>URL:</strong> POST /{clientType}/{version}/{endpoint}</p>
        <p><strong>설명:</strong> {API 상세 설명}</p>
    </div>

    <!-- Request -->
    <h3>Request</h3>
    <table class="param-table">
        <tr>
            <th>파라미터</th>
            <th>타입</th>
            <th>필수</th>
            <th>설명</th>
        </tr>
        <tr>
            <td>{paramName}</td>
            <td>{type}</td>
            <td>{required}</td>
            <td>{description}</td>
        </tr>
    </table>

    <!-- Response -->
    <h3>Response (성공)</h3>
    <pre class="code-block">
{
    "code": 1,
    "message": "OK",
    "data": { ... }
}
    </pre>

    <!-- Error Response -->
    <h3>Response (실패)</h3>
    <table class="error-table">
        <tr>
            <th>코드</th>
            <th>메시지</th>
            <th>발생 상황</th>
        </tr>
        <tr>
            <td>{errorCode}</td>
            <td>{errorMessage}</td>
            <td>{situation}</td>
        </tr>
    </table>
</div>
```

---

## 8. 문서 배포

### 8.1 GitLab Pages

```yaml
# .gitlab-ci.yml
pages:
  stage: deploy
  script:
    - mkdir .public
    - cp -r * .public
    - mv .public public
  artifacts:
    paths:
      - public
  only:
    - main
```

### 8.2 접근 URL

| 환경 | URL |
|------|-----|
| Develop | https://docs-dev.xxx.com |
| Production | https://docs.xxx.com |
