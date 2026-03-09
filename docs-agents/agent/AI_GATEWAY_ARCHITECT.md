# AI_GATEWAY_ARCHITECT

Gateway 서버 설계를 담당한다.

---

## 1. 역할 개요

AI_GATEWAY_ARCHITECT는 API Gateway의 구조와 정책을 설계하는 역할이다.

---

## 2. 책임

- API Gateway 구조 설계
- OAuth2 인증 흐름 설계
- Rate limit 설계
- Routing 정책 설계
- Header 전달 정책 설계

---

## 3. 출력물

- gateway-architecture.md

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

---

## 5. 역할 추천 조건

다음 요청에서 추천된다:
- Gateway 관련 요청
- 라우팅 설계 요청
- 인증 흐름 설계
- Rate limit 설계

---

## 6. 관련 역할

- AI_GATEWAY_ENGINEER

---

## 7. 관련 문서

- 31_gateway/CLAUDE.md
- 31_gateway/docs-claude/ARCHITECTURE.md
- 31_gateway/docs-claude/ROUTING_POLICY.md
- 31_gateway/docs-claude/SECURITY.md

---

END OF FILE
