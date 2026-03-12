# AGENT_TEAM_STRATEGY.md

Team 운영 전략 및 협업 규칙을 정의한다.

---

## 1. 사전 정의 Team

### 1.1 Team 목록

| Team | 용도 | 구성 역할 |
|------|------|----------|
| FEATURE_DEVELOPMENT_TEAM | 새 기능 개발 | TECH_LEAD → ARCHITECT → ENGINEER → REVIEWER → SECURITY → REFACTOR |
| EVENTFLOW_TEAM | Event Driven 구현 | TECH_LEAD → EVENT_ARCHITECT → APP_ENGINEER → SYS_ENGINEER → REVIEWER |
| QUALITY_ASSURANCE_TEAM | 코드 품질 보증 | REVIEWER → SECURITY → REFACTOR |
| ARCHITECTURE_TEAM | 시스템 설계 | TECH_LEAD → GATEWAY_ARCH → APP_ARCH → SYSTEM_ARCH |
| GATEWAY_TEAM | Gateway 작업 | GATEWAY_ARCHITECT → GATEWAY_ENGINEER → REVIEWER |
| APPLICATION_TEAM | Application 작업 | APP_ARCHITECT → APP_ENGINEER → REVIEWER |
| SYSTEM_TEAM | System 작업 | SYSTEM_ARCHITECT → SYSTEM_ENGINEER → REVIEWER |
| INFRA_TEAM | 인프라 구성 | INFRA_ENGINEER → REVIEWER |

---

## 2. 실행 전략

### 2.1 순차 실행 (기본)

대부분의 작업은 순차 실행:
```
역할1 → 역할2 → 역할3
```

### 2.2 병렬 실행 (독립 작업)

서버가 다른 독립 작업:
```
Gateway 작업 ─┬─ AI_GATEWAY_ENGINEER
              └─ AI_APPLICATION_ENGINEER (병렬)
```

### 2.3 반복 실행 (피드백)

검증 실패 시 반복:
```
ENGINEER → REVIEWER → (문제 발견) → ENGINEER → REVIEWER
```

---

## 3. 서버 경계 준수

### 3.1 절대 금지

| 역할 | 금지 행위 |
|------|----------|
| AI_GATEWAY_* | DB 접근, 비즈니스 로직 |
| AI_APPLICATION_* | @RabbitListener, JWT 검증 |
| AI_SYSTEM_* | MQ Producer, 사용자 API |

### 3.2 책임 경계

```
Gateway     ← 인증/라우팅/정책
Application ← 비즈니스/트랜잭션/Producer
System      ← Consumer/Worker/Batch
```

---

## 4. 역할 간 충돌 해결

### 4.1 우선순위

```
AI_TECH_LEAD (최고)
    ↓
AI_*_ARCHITECT
    ↓
AI_SECURITY_ENGINEER
    ↓
AI_REVIEWER
    ↓
AI_*_ENGINEER (최저)
```

### 4.2 충돌 시 처리

1. 상위 역할 결정 우선
2. 보안 관련은 AI_SECURITY_ENGINEER 결정 우선
3. 해결 불가 시 AI_TECH_LEAD 판단

---

## 5. Team 선택 기준

### 5.1 트리거 조건

| 조건 | 추천 Team |
|------|----------|
| 새 API 추가 | FEATURE_DEVELOPMENT_TEAM |
| MQ 기반 기능 | EVENTFLOW_TEAM |
| 코드 리뷰만 | QUALITY_ASSURANCE_TEAM |
| 아키텍처 변경 | ARCHITECTURE_TEAM |
| Gateway 정책 수정 | GATEWAY_TEAM |
| Application 기능 추가 | APPLICATION_TEAM |
| Consumer/Batch 추가 | SYSTEM_TEAM |
| Docker/CI 수정 | INFRA_TEAM |

### 5.2 복합 조건

여러 서버에 걸친 작업:
- FEATURE_DEVELOPMENT_TEAM 사용
- 또는 서버별 Team 병렬 실행

---

## 6. docs-claude 로딩 전략

### 6.1 Team 시작 시

1. AGENT_DOCS_MAPPING.md 참조
2. Team 첫 역할의 필수 문서 로딩
3. task_prompt.md에 역할별 문서 명시

### 6.2 역할 전환 시

1. 이전 역할 result 파일 읽기
2. 현재 역할 필수 문서 로딩
3. 선택 문서는 필요 시만 로딩

---

## 7. 품질 보증

### 7.1 자동 리뷰 파이프라인

코드 생성 후 자동 실행:
```
AI_REVIEWER → AI_SECURITY_ENGINEER → AI_REFACTOR_ENGINEER
```

### 7.2 검증 기준

| 항목 | 검증 역할 |
|------|----------|
| 아키텍처 위반 | AI_REVIEWER |
| 보안 취약점 | AI_SECURITY_ENGINEER |
| 코드 품질 | AI_REFACTOR_ENGINEER |

### 7.3 통과 조건

- 모든 검토 역할에서 "이상없음"
- 연속 2회 이상 "이상없음" 도출

---

## 8. 예외 처리

### 8.1 단일 역할 요청

사용자가 특정 역할만 요청:
- Team 구성하지 않고 단독 실행
- 자동 리뷰 파이프라인은 생략 가능

### 8.2 긴급 수정

- QUALITY_ASSURANCE_TEAM 생략 가능
- 단, 추후 반드시 검토 수행

---

END OF FILE
