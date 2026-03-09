# GATEWAY_TEAM

Gateway 서버 작업을 수행할 때 사용하는 Team이다.

---

## 1. 개요

GATEWAY_TEAM은 API Gateway의 설계와 구현을 담당하는 Team이다.
인증, 인가, 라우팅 등 Edge 계층의 작업을 수행한다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_GATEWAY_ARCHITECT | Gateway 구조 설계 |
| 2 | AI_GATEWAY_ENGINEER | Filter/Routing 구현 |
| 3 | AI_SECURITY_ENGINEER | 인증/인가 검토 |
| 4 | AI_REVIEWER | 코드 리뷰 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- Gateway 기능 요청
- 인증 시스템 요청
- 라우팅 정책 요청
- Filter 구현 요청
- Rate Limit 요청

---

## 4. 실행 워크플로우

```
┌──────────────────────┐
│ AI_GATEWAY_ARCHITECT │ Gateway 구조 설계
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ AI_GATEWAY_ENGINEER  │ Filter/Routing 구현
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ AI_SECURITY_ENGINEER │ 인증/인가 검토
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│    AI_REVIEWER       │ 코드 리뷰
└──────────────────────┘
```

---

## 5. Gateway 절대 원칙

이 Team은 다음 원칙을 반드시 준수한다:

### 5.1 금지 사항

- Spring MVC 도입 금지
- DB 접근 금지
- 비즈니스 로직 구현 금지
- 트랜잭션 관리 금지
- DTO 변환 금지
- 응답 Body 가공 금지
- 상태 저장 금지
- JWT 재검증 또는 재파싱 금지

### 5.2 허용 범위

- Authentication 처리
- Authorization 처리
- Routing 처리
- Header 전달
- Rate Limiting

---

## 6. 기술 스택

- Spring Boot 3.5.x
- Java 21 (LTS)
- Spring WebFlux (Reactive)
- Spring Cloud Gateway

---

## 7. 핸드오프 규칙

### 7.1 GATEWAY_ARCHITECT → GATEWAY_ENGINEER

전달 항목:
- Gateway 구조 설계서
- 라우팅 정책
- Filter 체인 설계
- 인증 흐름 설계

### 7.2 GATEWAY_ENGINEER → SECURITY_ENGINEER

전달 항목:
- 구현된 Filter 목록
- 인증/인가 처리 코드
- 토큰 처리 로직

---

## 8. 실행 예시

### 요청

"OAuth2 인증 흐름을 수정해줘"

### 실행

```
[STEP 1] AI_GATEWAY_ARCHITECT
- 인증 흐름 재설계
- Token 검증 정책 정의
- Filter 체인 설계

[STEP 2] AI_GATEWAY_ENGINEER
- AuthenticationFilter 수정
- TokenValidationFilter 구현
- 라우팅 설정 수정

[STEP 3] AI_SECURITY_ENGINEER
- 토큰 검증 로직 검토
- 인증 흐름 취약점 검토
- 인가 처리 검토

[STEP 4] AI_REVIEWER
- Gateway 원칙 준수 검토
- Reactive 패턴 준수 검토
- 코드 품질 검토
```

---

## 9. 제약 사항

- Application/System 서버 코드 수정 금지
- 비즈니스 로직 포함 금지
- DB 연결 금지
- 동기 블로킹 코드 금지

---

## 10. 관련 문서

- [AI_GATEWAY_ARCHITECT](../agent/AI_GATEWAY_ARCHITECT.md)
- [AI_GATEWAY_ENGINEER](../agent/AI_GATEWAY_ENGINEER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- 31_gateway/CLAUDE.md
- 31_gateway/docs-claude/SECURITY.md
- 31_gateway/docs-claude/ROUTING_POLICY.md

---

END OF FILE
