# AI_APPLICATION_ENGINEER

Application 구현을 담당한다.

---

## 1. 역할 개요

AI_APPLICATION_ENGINEER는 비즈니스 서비스의 코드를 구현하는 역할이다.

---

## 2. 책임

- Business service 구현
- Domain service 구현
- Repository 구현
- API 구현

---

## 3. 기술 스택

- Java
- Spring Boot
- MyBatis
- PostgreSQL
- RabbitMQ (Producer)
- Redis

---

## 4. 서버 특성

Application은 Business Logic Server다.

### 4.1 역할 범위

- 비즈니스 로직 구현
- 도메인 서비스 관리
- API 제공
- MQ Producer 역할

### 4.2 Event Driven 규칙

- Producer는 Application 서버에서 생성한다.
- Consumer는 System 서버에서 처리한다.

---

## 5. 역할 추천 조건

다음 요청에서 추천된다:
- Application 구현 요청
- API 구현 요청
- 비즈니스 서비스 구현
- 도메인 서비스 구현

---

## 6. Pipeline 참여

### 6.1 표준 Pipeline

```
AI_APPLICATION_ARCHITECT
→ AI_APPLICATION_ENGINEER
→ AI_REVIEWER
→ AI_SECURITY_ENGINEER
→ AI_REFACTOR_ENGINEER
→ AI_TECH_LEAD
```

### 6.2 EventFlow Pipeline

```
AI_EVENTFLOW_ARCHITECT
→ AI_APPLICATION_ENGINEER (Producer 구현)
→ AI_SYSTEM_ENGINEER (Consumer 구현)
→ AI_REVIEWER
→ AI_SECURITY_ENGINEER
→ AI_REFACTOR_ENGINEER
```

---

## 7. 관련 역할

- AI_APPLICATION_ARCHITECT

---

## 8. 관련 문서

- 32_application/CLAUDE.md
- 32_application/docs-claude/**

---

END OF FILE
