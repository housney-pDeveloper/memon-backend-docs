# ARCHITECTURE_TEAM

시스템 설계를 수행할 때 사용하는 Team이다.

---

## 1. 개요

ARCHITECTURE_TEAM은 시스템의 전체 아키텍처를 설계하고
구조적 의사결정을 수행하는 Team이다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_TECH_LEAD | 요구사항 분석 |
| 2 | AI_ARCHITECT | 전체 아키텍처 설계 |
| 3 | AI_EVENT_ARCHITECT | 이벤트 아키텍처 설계 (필요 시) |
| 4 | AI_REVIEWER | 설계 검토 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- 시스템 설계 요청
- 아키텍처 설계 요청
- 도메인 설계 요청
- 구조 변경 요청
- 신규 시스템 기획

---

## 4. 실행 워크플로우

```
┌─────────────────┐
│  AI_TECH_LEAD   │ 요구사항 분석
└────────┬────────┘
         ▼
┌─────────────────┐
│  AI_ARCHITECT   │ 전체 아키텍처 설계
└────────┬────────┘
         ▼
    ┌────┴────┐
    ▼         ▼
┌────────┐  ┌─────────────────┐
│ 완료   │  │AI_EVENT_ARCHITECT│ (필요 시)
└────────┘  └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  AI_REVIEWER    │ 설계 검토
            └─────────────────┘
```

---

## 5. 출력물

| 역할 | 출력물 |
|------|--------|
| AI_TECH_LEAD | task-plan.md, development-plan.md |
| AI_ARCHITECT | architecture.md, service-responsibility.md |
| AI_EVENT_ARCHITECT | event-architecture.md, mq-topology.md |

---

## 6. 서버별 Architect 선택

복잡한 설계의 경우 서버별 Architect를 포함할 수 있다.

| 대상 | 역할 |
|------|------|
| Gateway 설계 포함 | AI_GATEWAY_ARCHITECT 추가 |
| Application 설계 포함 | AI_APPLICATION_ARCHITECT 추가 |
| System 설계 포함 | AI_SYSTEM_ARCHITECT 추가 |

---

## 7. 핸드오프 규칙

### 7.1 TECH_LEAD → ARCHITECT

전달 항목:
- 요구사항 분석 문서
- 기능 목록
- 제약 조건
- 비기능 요구사항

### 7.2 ARCHITECT → EVENT_ARCHITECT

전달 항목:
- 전체 아키텍처 문서
- 이벤트 처리가 필요한 영역
- 서버 간 통신 요구사항

### 7.3 설계 → REVIEWER

전달 항목:
- 설계 문서 전체
- 주요 의사결정 사항
- 트레이드오프 분석

---

## 8. 설계 검토 항목

AI_REVIEWER는 다음을 검토한다:

- 서버 책임 분리 준수
- 확장성 고려
- 유지보수성 고려
- 기존 아키텍처와의 일관성
- 보안 고려사항

---

## 9. 실행 예시

### 요청

"회원 관리 시스템을 설계해줘"

### 실행

```
[STEP 1] AI_TECH_LEAD
- 요구사항 분석
  - 회원 가입
  - 로그인/로그아웃
  - 프로필 관리
  - 알림 발송
- 작업 분해

[STEP 2] AI_ARCHITECT
- 도메인 모델 설계
  - Member 엔티티
  - MemberProfile 엔티티
- API 구조 설계
  - /api/v1/signup
  - /api/v1/login
  - /api/v1/members/{id}
- 서비스 책임 정의

[STEP 3] AI_EVENT_ARCHITECT
- 알림 이벤트 설계
  - Exchange: ef.notification.exchange
  - Queue: ef.notification.welcome.queue
- DLQ 정책 정의

[STEP 4] AI_REVIEWER
- 설계 일관성 검토
- 서버 책임 분리 검토
- 확장성 검토
```

---

## 10. 제약 사항

- 설계 없이 구현 진행 금지
- 서버 책임 분리 원칙 위반 금지
- Reviewer 검토 생략 금지 (복잡한 설계의 경우)

---

## 11. 관련 문서

- [AI_TECH_LEAD](../agent/AI_TECH_LEAD.md)
- [AI_ARCHITECT](../agent/AI_ARCHITECT.md)
- [AI_EVENT_ARCHITECT](../agent/AI_EVENT_ARCHITECT.md)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)

---

END OF FILE
