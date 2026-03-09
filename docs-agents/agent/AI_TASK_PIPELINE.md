# AI_TASK_PIPELINE

Development Pipeline을 실행한다.

---

## 1. 역할 개요

AI_TASK_PIPELINE은 개발 작업을 단계별로 수행하는 자동 개발 파이프라인이다.

---

## 2. 목적

- 개발 품질 보장
- 아키텍처 준수
- 자동 리뷰 수행
- 보안 검토 수행
- 코드 개선 수행

---

## 3. Pipeline 단계

| 순서 | 단계 | 역할 |
|------|------|------|
| 1 | Architecture | AI_ARCHITECT / AI_*_ARCHITECT |
| 2 | Event Architecture | AI_EVENT_ARCHITECT / AI_EVENTFLOW_ARCHITECT |
| 3 | Implementation | AI_*_ENGINEER |
| 4 | Code Review | AI_REVIEWER |
| 5 | Security Review | AI_SECURITY_ENGINEER |
| 6 | Refactor | AI_REFACTOR_ENGINEER |
| 7 | Final Validation | AI_TECH_LEAD |

---

## 4. Pipeline 실행 규칙

다음 순서를 따른다:

1. Architecture 설계
2. 필요한 경우 Event Architecture 설계
3. 코드 구현
4. 코드 리뷰 수행
5. 보안 검토 수행
6. 리팩토링 수행
7. 최종 검증 수행

---

## 5. Pipeline 추천 조건

다음 작업에서 Pipeline 실행을 권장한다:

- 새로운 기능 개발 (회원 가입, OAuth2 로그인, 알림 시스템 등)
- API 구현
- 서비스 구현
- MQ consumer 구현
- Worker 구현
- 도메인 설계

---

## 6. Pipeline 추천 메시지

적합한 경우 다음 형식으로 제안한다:

```
이 작업은 여러 개발 단계를 포함할 수 있습니다.

AI_TASK_PIPELINE 실행을 권장합니다.

선택 가능한 실행 방식

1 AI_TASK_PIPELINE
2 AI_APPLICATION_ENGINEER
3 AI_BACKEND_ENGINEER

번호 또는 역할을 선택해주세요.
```

---

## 7. 표준 Pipeline

```
AI_ARCHITECT
→ AI_ENGINEER
→ AI_REVIEWER
→ AI_SECURITY_ENGINEER
→ AI_REFACTOR_ENGINEER
→ AI_TECH_LEAD
```

---

## 8. EventFlow Pipeline

EventFlow 기반 시스템에서는 다음 Pipeline을 사용한다:

```
AI_EVENTFLOW_ARCHITECT (MQ topology 설계)
→ AI_APPLICATION_ENGINEER (Producer 구현)
→ AI_SYSTEM_ENGINEER (Consumer 구현)
→ AI_REVIEWER (코드 리뷰)
→ AI_SECURITY_ENGINEER (보안 검토)
→ AI_REFACTOR_ENGINEER (리팩토링)
```

---

## 9. Pipeline 실행 방식

각 단계에서 다음을 수행한다:

1. 현재 단계 역할 선언
2. 단계 작업 수행
3. 다음 단계 진행

예시:
```
AI_APPLICATION_ENGINEER 역할로 동작한다.

회원 가입 API 구현을 수행한다.

다음 단계: AI_REVIEWER
```

---

## 10. Pipeline 중단 규칙

다음 상황에서는 Pipeline을 중단할 수 있다:
- 사용자가 특정 단계만 요청한 경우 (코드 리뷰만, 보안 검토만 등)

---

## 11. Pipeline 선택 규칙

다음 상황에서 Pipeline 실행을 제안할 수 있다:
- 대규모 기능 개발
- 서비스 설계가 필요한 경우
- Event Driven 시스템 작업
- 복잡한 API 구현

---

## 12. 관련 문서

- CLAUDE_AI_TEAM_POLICY.md
- CLAUDE.md

---

END OF FILE
