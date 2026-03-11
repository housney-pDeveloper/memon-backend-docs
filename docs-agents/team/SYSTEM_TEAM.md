# SYSTEM_TEAM

System 서버 작업을 수행할 때 사용하는 Team이다.

---

## 1. 개요

SYSTEM_TEAM은 Worker, MQ Consumer, Background Job을 설계하고 구현하는 Team이다.
비동기 처리와 이벤트 소비를 담당한다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_SYSTEM_ARCHITECT | Worker/Consumer 설계 |
| 2 | AI_SYSTEM_ENGINEER | Worker/Consumer 구현 |
| 3 | AI_REVIEWER | 코드 리뷰 |
| 4 | AI_SECURITY_ENGINEER | 보안 검토 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- Worker 구현 요청
- Consumer 구현 요청
- Background job 요청
- 알림 처리 요청
- 비동기 처리 요청

---

## 4. 실행 워크플로우

```
┌─────────────────────┐
│ AI_SYSTEM_ARCHITECT │ Worker/Consumer 설계
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ AI_SYSTEM_ENGINEER  │ Worker/Consumer 구현
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│    AI_REVIEWER      │ 코드 리뷰
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ AI_SECURITY_ENGINEER│ 보안 검토
└─────────────────────┘
```

---

## 5. System 서버 역할

### 5.1 책임 범위

- MQ Consumer 처리
- Worker 운영
- 알림 발송
- Background processing
- SSE (Server-Sent Events)

### 5.2 Event Driven 규칙

- Producer는 Application 서버에서 생성 (이 Team 범위 외)
- **Consumer**: System 서버에서 처리

---

## 6. 기술 스택

- Java 21
- Spring Boot
- RabbitMQ (Consumer)
- Redis
- SSE (Server-Sent Events)

---

## 7. 핸드오프 규칙

### 7.1 SYSTEM_ARCHITECT → SYSTEM_ENGINEER

전달 항목:
- Consumer 설계
- Worker 설계
- 처리 흐름 정의
- 에러 처리 정책
- DLQ 처리 정책

### 7.2 SYSTEM_ENGINEER → REVIEWER

전달 항목:
- Consumer 구현체
- Worker 구현체
- 에러 처리 로직
- 재시도 로직

---

## 8. Consumer 설계 규칙

### 8.1 필수 요소

- 멱등성 보장
- DLQ 처리
- Retry 정책
- 에러 로깅

### 8.2 네이밍 규칙

```java
@Component
public class {Domain}{Action}Consumer {
    @RabbitListener(queues = "ef.{domain}.{action}.queue")
    public void handle{Action}Event({Event} event) {
        // 처리 로직
    }
}
```

---

## 9. 실행 예시

### 요청

"회원 가입 시 환영 알림을 발송하는 Consumer를 구현해줘"

### 실행

```
[STEP 1] AI_SYSTEM_ARCHITECT
- Consumer 설계
  - 큐: ef.notification.welcome.queue
  - 이벤트: WelcomeNotificationEvent
- 처리 흐름 정의
  - 이벤트 수신
  - 알림 템플릿 조회
  - 알림 발송
- DLQ 정책
  - 3회 재시도 후 DLQ 이동

[STEP 2] AI_SYSTEM_ENGINEER
- WelcomeNotificationConsumer 구현
- NotificationService 구현
- 에러 처리 로직 구현

[STEP 3] AI_REVIEWER
- 멱등성 검토
- 에러 처리 검토
- DLQ 처리 검토

[STEP 4] AI_SECURITY_ENGINEER
- 민감 정보 처리 검토
- 로깅 검토
```

---

## 10. 제약 사항

- MQ Producer 구현 금지 (Application 서버 역할)
- 비즈니스 API 제공 금지 (Application 서버 역할)
- Gateway 역할 침범 금지

---

## 11. 관련 문서

- [AI_SYSTEM_ARCHITECT](../agent/AI_SYSTEM_ARCHITECT.md)
- [AI_SYSTEM_ENGINEER](../agent/AI_SYSTEM_ENGINEER.md)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- 33_system/CLAUDE.md
- 01_docs/docs-spec/mq/*.md

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|-----------------|------|
| 1 | AI_SYSTEM_ARCHITECT | 01_architecture, 04_backend/SYSTEM, 05_infra | N |
| 2 | AI_SYSTEM_ENGINEER | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION, 04_backend/SYSTEM, 05_infra | N |
| 3 | AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | Y (with 4) |
| 4 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | Y (with 3) |

### 핸드오프 흐름

```
prompt.md → task_prompt.md
→ result_AI_SYSTEM_ARCHITECT.md → result_AI_SYSTEM_ENGINEER.md
→ result_AI_REVIEWER.md + result_AI_SECURITY_ENGINEER.md (병렬)
→ result_VERIFICATION.md
```

---

END OF FILE
