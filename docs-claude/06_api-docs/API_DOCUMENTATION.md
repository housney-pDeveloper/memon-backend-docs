# API_DOCUMENTATION.md

API 문서 생성 규칙을 통합 정의한다. (콘텐츠, URL, 디렉토리 구조)

---

## 1. 기본 원칙

- 한글로 작성
- 프론트엔드 개발자를 위한 문서
- 모든 API는 Gateway를 통한 접근만 허용
- 문서의 URL은 외부 유입 URL 기준으로 작성 (Gateway 진입점 기준)

---

## 2. URL 패턴

```
/{clientType}/{version}/{endpoint}
```

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| {clientType} | app, web | 클라이언트 유형 |
| {version} | v1, v2, ... | API 버전 |
| {endpoint} | 각 API 경로 | Controller의 RequestMapping 경로 (/api 제외) |

예시:
- `POST /app/v1/login`
- `POST /web/v1/signup/insClient`

Gateway URL 변환 참고:

| 외부 URL (문서 기준) | 내부 URL (Application) |
|---------------------|----------------------|
| `/app/v1/signup/getBizRegNo` | `/api/v1/signup/getBizRegNo` |
| `/web/v1/login` | `/api/v1/login` |

주의: 문서에는 항상 외부 URL을 기재. 내부 URL (/api/...)은 문서에 기재하지 않음.

---

## 3. 디렉토리 구조

```
docs
├─ biz
│   └─ login
│   │   └─ loginController.html
│   └─ signup
│       └─ signupController.html
├─ system
│   └─ code
│       └─ codeController.html
└─ index.html
```

### 3.1 규칙

- **biz**: application의 biz 패키지별 구분, controller당 html 파일 1개
- **system**: system 프로젝트 API, 구조는 biz와 동일
- **index.html**: 전체 API 목록 및 링크

파일명: `{controllerName}.html`

---

## 4. 필수 작성 항목

### 4.1 API URL / Method / Parameter / Response

기본 API 명세 (표 형식)

### 4.2 TypeScript Interface

프론트엔드용 TypeScript interface 정의. "복사" 버튼 제공.

**타입 변환 규칙:**

| Java Type | TypeScript Type |
|-----------|-----------------|
| String | string |
| Integer, Long, int, long | number |
| Boolean, boolean | boolean |
| LocalDateTime, LocalDate | string (ISO 8601) |
| List<T> | T[] |
| Map<K, V> | Record<K, V> |
| BigDecimal | number |
| Object | unknown |

**필수값 판단 기준 (순서대로):**

1. Database DDL의 NOT NULL 제약조건
2. Mapper XML의 COALESCE/IFNULL 처리
3. VO/DTO의 @NotNull, @NotBlank 어노테이션
4. 판단 불가 시 optional(?)로 표기 + "확인 필요" 주석

### 4.3 Controller 전체 Interface 통합 영역

- 파일 최하단에 위치
- Request / Response 그룹으로 분류
- 중복 타입은 Common Types로 분리
- "전체 복사" 버튼 제공

### 4.4 인증 및 권한 정보

**토큰 종류:**

| 토큰 | Header Key |
|------|------------|
| ACCESS_TOKEN | Authorization: Bearer {token} |
| REFRESH_TOKEN | X-Refresh-Token |
| TEMP_TOKEN | X-Temp-Token |
| NONE | - |

**권한:** PUBLIC, USER, ADMIN, SYSTEM

**판단 기준:**
1. Gateway Filter (SecurityFilter, JwtAuthFilter) 경로별 설정
2. Controller 어노테이션 (@PreAuthorize, @Secured)
3. 라우팅/보안 정책 문서

### 4.5 Error Response / Description / Exception Case

- Exception과 BizException 구분 기술
- API 목적, 선행 조건, 기대효과
- case별 발생 상황 + ResponseCode message

---

END OF FILE
