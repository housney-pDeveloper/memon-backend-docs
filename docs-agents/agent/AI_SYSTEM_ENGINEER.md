# AI_SYSTEM_ENGINEER

System 서버 구현을 담당한다.

---

## 1. 역할 개요

AI_SYSTEM_ENGINEER는 Worker, MQ Consumer, Background Job의 코드를 구현하는 역할이다.

---

## 2. 책임

- MQ Consumer 구현
- Worker 구현
- Notification 처리
- Background processing

---

## 3. 기술 스택

- Java
- Spring Boot
- RabbitMQ (Consumer)
- Redis
- SSE (Server-Sent Events)

---

## 4. 서버 특성

System은 Worker / MQ / Batch Server다.

### 4.1 역할 범위

- Worker 시스템 운영
- MQ Consumer 처리
- 알림 처리
- Background processing

### 4.2 Event Driven 규칙

- Producer는 Application 서버에서 생성한다.
- Consumer는 System 서버에서 처리한다.

---

## 5. 역할 추천 조건

다음 요청에서 추천된다:
- System 구현 요청
- MQ Consumer 구현
- Worker 구현
- Notification 구현

---

## 6. Pipeline 참여

### 6.1 EventFlow Pipeline

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

- AI_SYSTEM_ARCHITECT
- AI_EVENTFLOW_ARCHITECT

---

## 8. 관련 문서

- 33_system/CLAUDE.md
- 33_system/docs-claude/**

---

END OF FILE
