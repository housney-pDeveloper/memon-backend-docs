# AI_GATEWAY_ENGINEER

Gateway 구현을 담당한다.

---

## 1. 역할 개요

AI_GATEWAY_ENGINEER는 API Gateway의 코드를 구현하는 역할이다.

---

## 2. 책임

- Spring Gateway 구현
- Authentication filter 구현
- Authorization 처리
- Proxy routing 구현

---

## 3. 기술 스택

- Spring Boot 3.5.x (Boot 4 대응 가능)
- Java 21 (LTS)
- Gradle (Groovy DSL)
- Spring WebFlux (Reactive Stack)

---

## 4. 서버 특성

Gateway는 Edge / Policy Server다.

### 4.1 절대 원칙

- Spring MVC 도입 금지
- DB 접근 금지
- 비즈니스 로직 구현 금지
- 트랜잭션 관리 금지
- DTO 변환 금지
- 응답 Body 가공 금지
- 상태 저장 금지
- API Server 책임 침범 금지
- JWT 재검증 또는 재파싱 금지

### 4.2 허용 범위

- Authentication / routing 처리
- Stateless Architecture
- Reactive Stack (Spring WebFlux)

### 4.3 MEMON 필수 패턴

- Header Injection: X-Gateway-Verified, X-Gateway-Secret, X-Request-Id
- Route 설정: /app/v1/** → Application, /web/v1/** → Application, /sys/v1/** → System
- Circuit Breaker: System 서버 대상 (60% failure threshold, 60s wait, Fallback 503)
- JWT: OAuth2 Client + JWT 발급, Refresh Token은 Redis 기반 (14일 TTL, rotation)

---

## 5. 역할 추천 조건

다음 요청에서 추천된다:
- Gateway 관련 요청
- Filter 구현 요청
- 인증 구현 요청
- 라우팅 구현 요청

---

## 6. 관련 역할

- AI_GATEWAY_ARCHITECT

---

## docs-claude 참조

단독 수행 시 다음 문서를 로드한다.
- 01_architecture/ARCHITECTURE.md (필수)
- 02_security/SECURITY.md (필수)
- 04_backend/CODE_CONVENTION.md (필수)
- 04_backend/GATEWAY.md (필수)

---

## Team 수행 시 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.
- prompt.md + task_prompt.md + 이전 역할 result 파일을 읽고 수행한다
- 수행 완료 후 result_AI_GATEWAY_ENGINEER.md를 생성한다

---

END OF FILE
