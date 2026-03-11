# EVENTFLOW_TEAM

Event Driven 시스템을 구현할 때 사용하는 Team이다.

---

## 1. 개요

EVENTFLOW_TEAM은 MQ 기반의 이벤트 처리 시스템을 설계하고 구현하는 Team이다.
Producer(Application)와 Consumer(System) 간의 협업을 관리한다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_EVENT_ARCHITECT | MQ topology 설계 |
| 2 | AI_APPLICATION_ENGINEER | Producer 구현 |
| 3 | AI_SYSTEM_ENGINEER | Consumer 구현 |
| 4 | AI_REVIEWER | 코드 리뷰 |
| 5 | AI_SECURITY_ENGINEER | 보안 검토 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- MQ 시스템 구현 요청
- Event 처리 시스템 요청
- Producer/Consumer 구현 요청
- 알림 시스템 구현 요청
- 비동기 처리 요청

---

## 4. 실행 워크플로우

```
┌──────────────────────┐
│ AI_EVENT_ARCHITECT│ MQ topology 설계
└──────────┬───────────┘
           ▼
    ┌──────┴──────┐
    ▼             ▼
┌────────┐   ┌────────┐
│  APP   │   │ SYSTEM │
│ENGINEER│   │ENGINEER│
└───┬────┘   └───┬────┘
    │ Producer   │ Consumer
    └──────┬─────┘
           ▼
┌─────────────────┐
│   AI_REVIEWER   │ 전체 흐름 검토
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_SECURITY_ENG │ 보안 검토
└─────────────────┘
```

---

## 5. 서버 책임 분리

| 역할 | 서버 | 책임 |
|------|------|------|
| Producer | Application | 이벤트 발행 |
| Consumer | System | 이벤트 처리 |

**중요**: 이 경계는 절대 침범하지 않는다.

---

## 6. MQ 설계 규칙

### 6.1 필수 정책

- Queue는 역할별로 분리
- Dead Letter Queue 정책 필수
- Retry 정책 정의 필수

### 6.2 네이밍 규칙

```
Exchange: ef.{domain}.exchange
Queue: ef.{domain}.{action}.queue
DLQ: ef.{domain}.{action}.dlq
```

---

## 7. 핸드오프 규칙

### 7.1 EVENTFLOW_ARCHITECT → ENGINEERS

전달 항목:
- MQ topology 설계서
- Exchange/Queue 정의
- 메시지 스키마
- DLQ/Retry 정책

### 7.2 병렬 실행

AI_APPLICATION_ENGINEER와 AI_SYSTEM_ENGINEER는 병렬로 실행 가능하다.

```
EVENTFLOW_ARCHITECT
        │
        ├──→ APPLICATION_ENGINEER (Producer)
        │
        └──→ SYSTEM_ENGINEER (Consumer)
```

---

## 8. 실행 예시

### 요청

"회원 가입 시 환영 알림을 발송하는 시스템을 구현해줘"

### 실행

```
[STEP 1] AI_EVENT_ARCHITECT
- Exchange: ef.notification.exchange
- Queue: ef.notification.welcome.queue
- DLQ: ef.notification.welcome.dlq
- 메시지 스키마 정의

[STEP 2-A] AI_APPLICATION_ENGINEER (Application 서버)
- NotificationProducer 구현
- SignupService에서 이벤트 발행

[STEP 2-B] AI_SYSTEM_ENGINEER (System 서버)
- WelcomeNotificationConsumer 구현
- 알림 발송 처리

[STEP 3] AI_REVIEWER
- Producer/Consumer 흐름 검토
- 에러 처리 검토
- DLQ 처리 검토

[STEP 4] AI_SECURITY_ENGINEER
- 메시지 보안 검토
- 민감 정보 처리 검토
```

---

## 9. 제약 사항

- Application에서 Consumer 구현 금지
- System에서 Producer 구현 금지
- DLQ 정책 없이 구현 금지

---

## 10. 관련 문서

- [AI_EVENT_ARCHITECT](../agent/AI_EVENT_ARCHITECT.md)
- [AI_APPLICATION_ENGINEER](../agent/AI_APPLICATION_ENGINEER.md)
- [AI_SYSTEM_ENGINEER](../agent/AI_SYSTEM_ENGINEER.md)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- 01_docs/docs-spec/mq/*.md

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|-----------------|------|
| 1 | AI_EVENT_ARCHITECT | 01_architecture, 05_infra, 04_backend/SYSTEM | N |
| 2 | AI_APPLICATION_ENGINEER | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION, 04_backend/APPLICATION, 05_infra | N |
| 3 | AI_SYSTEM_ENGINEER | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION, 04_backend/SYSTEM, 05_infra | N |
| 4 | AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | Y (with 5) |
| 5 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | Y (with 4) |

### 핸드오프 흐름

```
prompt.md → task_prompt.md
→ result_AI_EVENT_ARCHITECT.md
→ result_AI_APPLICATION_ENGINEER.md (Producer)
→ result_AI_SYSTEM_ENGINEER.md (Consumer)
→ result_AI_REVIEWER.md + result_AI_SECURITY_ENGINEER.md (병렬)
→ result_VERIFICATION.md
```

### 특수 규칙

- AI_APPLICATION_ENGINEER는 Producer만 구현 (Consumer 금지)
- AI_SYSTEM_ENGINEER는 Consumer만 구현 (Producer 금지)
- MQ 리소스 변경 시 05_infra/INFRASTRUCTURE.md 업데이트 필수

---

END OF FILE
