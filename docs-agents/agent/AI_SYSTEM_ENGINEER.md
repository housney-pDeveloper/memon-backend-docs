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

### 4.3 절대 금지

- MQ Producer 구현 금지 (Application 서버 역할)
- 비즈니스 API 제공 금지 (Application 서버 역할)
- Gateway 역할 침범 금지

### 4.4 MEMON 필수 패턴

- Profile 기반 배포: api(8082), worker-log(8083), worker-batch(8084), worker-msg(8085)
- Consumer 멱등성: 중복 메시지 처리 보장
- Retry: 3회 재시도 + Exponential Backoff 후 DLQ 이동
- SSE: Last-Event-ID 기반 복구, Polling fallback
- DI: 생성자 주입만 허용 (필드 주입 금지)

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
AI_EVENT_ARCHITECT
→ AI_APPLICATION_ENGINEER (Producer 구현)
→ AI_SYSTEM_ENGINEER (Consumer 구현)
→ AI_REVIEWER
→ AI_SECURITY_ENGINEER
→ AI_REFACTOR_ENGINEER
```

---

## 7. 관련 역할

- AI_SYSTEM_ARCHITECT
- AI_EVENT_ARCHITECT

---

## docs-claude 참조

단독 수행 시 다음 문서를 로드한다.
- 01_architecture/ARCHITECTURE.md (필수)
- 02_security/SECURITY.md (필수)
- 03_data/DATABASE.md (필수)
- 04_backend/CODE_CONVENTION.md (필수)
- 04_backend/SYSTEM.md (필수)
- 05_infra/INFRASTRUCTURE.md (필수)

---

## Team 수행 시 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.
- prompt.md + task_prompt.md + 이전 역할 result 파일을 읽고 수행한다
- 수행 완료 후 result_AI_SYSTEM_ENGINEER.md를 생성한다

---

END OF FILE
