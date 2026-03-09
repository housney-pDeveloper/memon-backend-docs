# CONTENT_POLICY.md

API 문서 내용 작성 규칙을 정의한다.

---

## 1. 필수 작성 항목

모든 API는 아래 규격에 따라 작성한다.

### 1-1. API URL

- 외부 유입 URL 기준으로 작성
- Gateway 진입점 기준

### 1-2. API Method

- HTTP Method 명시 (GET, POST, PUT, DELETE 등)

### 1-3. API Parameter

- 요청 파라미터 목록
- parameter description은 field의 `@Info`를 참조

### 1-4. API Response

- 정상 응답 구조
- 응답 필드 설명

### 1-5. TypeScript Interface

- 프론트엔드용 TypeScript interface 정의
- Request/Response 타입 제공
- 프론트 개발자가 바로 복사하여 사용할 수 있도록 구현

#### 1-5-1. Interface 변환 규칙

| Java Type | TypeScript Type | 비고 |
|-----------|-----------------|------|
| String | string | |
| Integer, Long, int, long | number | |
| Boolean, boolean | boolean | |
| LocalDateTime, LocalDate | string | ISO 8601 형식 |
| List<T> | T[] | |
| Map<K, V> | Record<K, V> | |
| BigDecimal | number | |
| Object | unknown | 명확한 타입 권장 |

#### 1-5-2. 필수값 판단 기준

필수값 여부는 다음 순서로 판단한다.

1. **Database 스키마 참조**
   - `database/**/ddl/*.sql` 내 NOT NULL 제약조건 확인
   - 해당 컬럼이 NOT NULL이면 필수값

2. **쿼리 참조**
   - Mapper XML 내 SELECT 쿼리 확인
   - COALESCE, IFNULL 등 null 처리 여부 확인

3. **소스 코드 참조**
   - VO/DTO 클래스의 `@NotNull`, `@NotBlank` 어노테이션 확인
   - 비즈니스 로직에서 null 체크 여부 확인

4. **판단 불가 시**
   - optional(?)로 표기
   - 주석으로 "확인 필요" 명시

#### 1-5-3. Interface 작성 형식

```typescript
/** API 설명 주석 */
interface {MethodName}Response {
  /** 필드 설명 (@Info 참조) */
  fieldName: string;           // 필수값
  optionalField?: number;      // 선택값
  nullableField: string | null; // null 허용
}
```

#### 1-5-4. 복사 기능 구현

- 각 API별 interface에 "복사" 버튼 제공
- 클릭 시 클립보드에 interface 코드 복사
- HTML 구현 예시:

```html
<div class="typescript-interface">
  <div class="interface-header">
    <span>TypeScript Interface</span>
    <button onclick="copyToClipboard('interface-{methodName}')" class="copy-btn">복사</button>
  </div>
  <pre id="interface-{methodName}">
interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}
  </pre>
</div>
```

### 1-6. Controller 전체 Interface 통합 영역

Controller 단위로 모든 API의 TypeScript Interface를 한 곳에 모아서 제공한다.

#### 1-6-1. 통합 영역 목적

- 프론트 개발자가 Controller 전체 타입을 한 번에 복사 가능
- API별 개별 interface를 자동으로 수집하여 통합
- 파일 최하단에 위치

#### 1-6-2. 통합 영역 구성

```html
<section id="all-interfaces">
  <h2>전체 TypeScript Interface</h2>
  <div class="interface-header">
    <span>모든 API Interface 통합</span>
    <button onclick="copyToClipboard('all-interfaces-code')" class="copy-btn">전체 복사</button>
  </div>
  <pre id="all-interfaces-code">
// ============================================
// {ControllerName} API Interfaces
// Generated from: {ControllerPath}
// ============================================

// ---------- Request Interfaces ----------

interface LoginRequest {
  userId: string;
  password: string;
}

interface LogoutRequest {
  refreshToken: string;
}

// ---------- Response Interfaces ----------

interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

interface LogoutResponse {
  success: boolean;
}

// ---------- Common Types ----------

type TokenType = 'ACCESS' | 'REFRESH' | 'TEMP';
  </pre>
</section>
```

#### 1-6-3. 통합 규칙

- 각 API method별 작성된 interface를 수집
- Request / Response 그룹으로 분류
- 중복 타입은 Common Types로 분리
- Controller 파일 경로를 주석으로 명시

---

### 1-7. 인증 및 권한 정보

각 API별 필요한 토큰 종류와 권한을 명시한다.

#### 1-7-1. 토큰 종류

| 토큰 종류 | 설명 | Header Key |
|----------|------|------------|
| ACCESS_TOKEN | 일반 API 접근용 | Authorization: Bearer {token} |
| REFRESH_TOKEN | 토큰 갱신용 | X-Refresh-Token |
| TEMP_TOKEN | 임시 인증용 (회원가입 등) | X-Temp-Token |
| NONE | 토큰 불필요 (public API) | - |

#### 1-7-2. 권한 종류

| 권한 | 설명 |
|------|------|
| PUBLIC | 인증 불필요 |
| USER | 일반 사용자 |
| ADMIN | 관리자 |
| SYSTEM | 시스템 내부 호출 |

#### 1-7-3. 작성 형식

```html
<h3>인증 및 권한</h3>
<table>
  <tr>
    <th>토큰</th>
    <td>ACCESS_TOKEN</td>
  </tr>
  <tr>
    <th>권한</th>
    <td>USER</td>
  </tr>
  <tr>
    <th>Header</th>
    <td><code>Authorization: Bearer {accessToken}</code></td>
  </tr>
</table>
```

#### 1-7-4. 토큰/권한 판단 기준

1. **Gateway Filter 참조**
   - `SecurityFilter`, `JwtAuthFilter` 등에서 경로별 인증 설정 확인
   - `permitAll()` 경로는 NONE/PUBLIC

2. **Controller 어노테이션 참조**
   - `@PreAuthorize`, `@Secured` 어노테이션 확인
   - `@TempTokenRequired` 등 커스텀 어노테이션 확인

3. **라우팅 정책 참조**
   - `31_gateway/docs-claude/ROUTING_POLICY.md` 참조
   - `31_gateway/docs-claude/SECURITY.md` 참조

---

### 1-8. API Error Response

- API 호출 결과 Exception 발생 시
- Exception과 BizException을 구분하여 기술

### 1-9. API Description

- API의 목적
- 선행 조건 (login 여부, header의 token 값 등)
- API의 기대효과

### 1-10. API Exception Case

- case별 어떤 상황일 때 발생하는지 기술
- ResponseCode의 message도 함께 기술

---

## 2. 작성 예시

### 2-1. 개별 API 문서 예시

```html
<h2>POST /app/v1/login</h2>

<h3>Description</h3>
<p>사용자 로그인을 수행합니다.</p>
<ul>
  <li>목적: 사용자 인증 및 토큰 발급</li>
  <li>선행 조건: 없음</li>
  <li>기대효과: JWT 토큰 발급</li>
</ul>

<h3>인증 및 권한</h3>
<table>
  <tr><th>토큰</th><td>NONE</td></tr>
  <tr><th>권한</th><td>PUBLIC</td></tr>
  <tr><th>Header</th><td>-</td></tr>
</table>

<h3>Parameters</h3>
<table>
  <tr><th>Name</th><th>Type</th><th>Required</th><th>Description</th></tr>
  <tr><td>userId</td><td>String</td><td>Y</td><td>사용자 ID</td></tr>
  <tr><td>password</td><td>String</td><td>Y</td><td>비밀번호</td></tr>
</table>

<h3>Response</h3>
<table>
  <tr><th>Field</th><th>Type</th><th>Required</th><th>Description</th></tr>
  <tr><td>accessToken</td><td>String</td><td>Y</td><td>액세스 토큰</td></tr>
  <tr><td>refreshToken</td><td>String</td><td>Y</td><td>리프레시 토큰</td></tr>
  <tr><td>expiresIn</td><td>Long</td><td>Y</td><td>만료 시간(초)</td></tr>
</table>

<h3>TypeScript Interface</h3>
<div class="typescript-interface">
  <div class="interface-header">
    <span>TypeScript Interface</span>
    <button onclick="copyToClipboard('interface-login')" class="copy-btn">복사</button>
  </div>
  <pre id="interface-login">
/** 로그인 요청 */
interface LoginRequest {
  /** 사용자 ID */
  userId: string;
  /** 비밀번호 */
  password: string;
}

/** 로그인 응답 */
interface LoginResponse {
  /** 액세스 토큰 */
  accessToken: string;
  /** 리프레시 토큰 */
  refreshToken: string;
  /** 만료 시간(초) */
  expiresIn: number;
}
  </pre>
</div>

<h3>Error Cases</h3>
<table>
  <tr><th>Code</th><th>Message</th><th>Description</th></tr>
  <tr><td>30001</td><td>사용자를 찾을 수 없습니다</td><td>존재하지 않는 사용자 ID</td></tr>
  <tr><td>30002</td><td>비밀번호가 일치하지 않습니다</td><td>잘못된 비밀번호 입력</td></tr>
</table>
```

### 2-2. Controller 전체 Interface 통합 영역 예시

```html
<!-- 파일 최하단에 위치 -->
<section id="all-interfaces">
  <h2>전체 TypeScript Interface</h2>
  <div class="interface-header">
    <span>LoginController 전체 Interface</span>
    <button onclick="copyToClipboard('all-interfaces-code')" class="copy-btn">전체 복사</button>
  </div>
  <pre id="all-interfaces-code">
// ============================================
// LoginController API Interfaces
// Generated from: biz/login/controller/LoginController.java
// ============================================

// ---------- Request Interfaces ----------

/** 로그인 요청 */
interface LoginRequest {
  /** 사용자 ID */
  userId: string;
  /** 비밀번호 */
  password: string;
}

/** 로그아웃 요청 */
interface LogoutRequest {
  /** 리프레시 토큰 */
  refreshToken: string;
}

/** 토큰 갱신 요청 */
interface RefreshTokenRequest {
  /** 리프레시 토큰 */
  refreshToken: string;
}

// ---------- Response Interfaces ----------

/** 로그인 응답 */
interface LoginResponse {
  /** 액세스 토큰 */
  accessToken: string;
  /** 리프레시 토큰 */
  refreshToken: string;
  /** 만료 시간(초) */
  expiresIn: number;
}

/** 로그아웃 응답 */
interface LogoutResponse {
  /** 성공 여부 */
  success: boolean;
}

/** 토큰 갱신 응답 */
interface RefreshTokenResponse {
  /** 새 액세스 토큰 */
  accessToken: string;
  /** 만료 시간(초) */
  expiresIn: number;
}
  </pre>
</section>

<script>
function copyToClipboard(elementId) {
  const element = document.getElementById(elementId);
  const text = element.innerText;
  navigator.clipboard.writeText(text).then(() => {
    alert('클립보드에 복사되었습니다.');
  });
}
</script>
```

---

END OF FILE
