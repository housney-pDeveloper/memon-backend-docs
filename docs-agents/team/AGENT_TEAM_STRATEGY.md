# AGENT_TEAM_STRATEGY.md

이 문서는 복잡한 작업 수행을 위한 Agent Team/Group 전략을 정의한다.

---

## 1. 개요

단일 역할로 수행하기 어려운 복잡한 작업은 여러 Agent가 협업하여 처리한다.
이를 위해 Agent Team과 Agent Group을 정의한다.

---

## 2. 개념 정의

### 2.1 Agent Team

특정 목표를 달성하기 위해 구성된 고정 역할 집합이다.
Team은 사전 정의된 워크플로우를 따른다.

### 2.2 Agent Group

동적으로 구성되는 역할 집합이다.
작업 특성에 따라 AI_TASK_ROUTER가 구성한다.

---

## 3. 사전 정의 Team

### 3.1 FEATURE_DEVELOPMENT_TEAM

새로운 기능을 개발할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_TECH_LEAD | 요구사항 분석, 작업 분해 |
| 2 | AI_*_ARCHITECT | 아키텍처 설계 |
| 3 | AI_*_ENGINEER | 코드 구현 |
| 4 | AI_REVIEWER | 코드 리뷰 |
| 5 | AI_SECURITY_ENGINEER | 보안 검토 |
| 6 | AI_REFACTOR_ENGINEER | 코드 개선 |

트리거 조건:
- 새 기능 개발 요청
- API 구현 요청
- 서비스 구현 요청

---

### 3.2 EVENTFLOW_TEAM

Event Driven 시스템을 구현할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_EVENTFLOW_ARCHITECT | MQ topology 설계 |
| 2 | AI_APPLICATION_ENGINEER | Producer 구현 |
| 3 | AI_SYSTEM_ENGINEER | Consumer 구현 |
| 4 | AI_REVIEWER | 코드 리뷰 |
| 5 | AI_SECURITY_ENGINEER | 보안 검토 |

트리거 조건:
- MQ 시스템 구현 요청
- Event 처리 시스템 요청
- Producer/Consumer 구현 요청

---

### 3.3 QUALITY_ASSURANCE_TEAM

코드 품질 보증을 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_REVIEWER | 코드 품질 검토 |
| 2 | AI_SECURITY_ENGINEER | 보안 취약점 검토 |
| 3 | AI_REFACTOR_ENGINEER | 코드 개선 |

트리거 조건:
- 코드 리뷰 요청
- 품질 검토 요청
- PR 리뷰 요청

---

### 3.4 ARCHITECTURE_TEAM

시스템 설계를 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_TECH_LEAD | 요구사항 분석 |
| 2 | AI_ARCHITECT | 전체 아키텍처 설계 |
| 3 | AI_EVENT_ARCHITECT | 이벤트 아키텍처 설계 (필요 시) |
| 4 | AI_REVIEWER | 설계 검토 |

트리거 조건:
- 시스템 설계 요청
- 아키텍처 설계 요청
- 도메인 설계 요청

---

### 3.5 GATEWAY_TEAM

Gateway 서버 작업을 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_GATEWAY_ARCHITECT | Gateway 구조 설계 |
| 2 | AI_GATEWAY_ENGINEER | Filter/Routing 구현 |
| 3 | AI_SECURITY_ENGINEER | 인증/인가 검토 |
| 4 | AI_REVIEWER | 코드 리뷰 |

트리거 조건:
- Gateway 기능 요청
- 인증 시스템 요청
- 라우팅 정책 요청

---

### 3.6 APPLICATION_TEAM

Application 서버 작업을 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_APPLICATION_ARCHITECT | 비즈니스 구조 설계 |
| 2 | AI_APPLICATION_ENGINEER | 서비스/API 구현 |
| 3 | AI_REVIEWER | 코드 리뷰 |
| 4 | AI_SECURITY_ENGINEER | 보안 검토 |

트리거 조건:
- 비즈니스 API 요청
- 도메인 서비스 요청
- Application 기능 요청

---

### 3.7 SYSTEM_TEAM

System 서버 작업을 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_SYSTEM_ARCHITECT | Worker/Consumer 설계 |
| 2 | AI_SYSTEM_ENGINEER | Worker/Consumer 구현 |
| 3 | AI_REVIEWER | 코드 리뷰 |
| 4 | AI_SECURITY_ENGINEER | 보안 검토 |

트리거 조건:
- Worker 구현 요청
- Consumer 구현 요청
- Background job 요청

---

### 3.8 INFRA_TEAM

인프라 구성 작업을 수행할 때 사용한다.

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_INFRA_ENGINEER | 인프라 구성 |
| 2 | AI_SECURITY_ENGINEER | 보안 설정 검토 |

트리거 조건:
- Docker 구성 요청
- CI/CD 구성 요청
- AWS 구성 요청

---

## 4. Team 실행 전략

### 4.1 순차 실행 (Sequential)

기본 실행 모드이다.
각 역할이 순서대로 실행된다.

```
Role A → Role B → Role C
```

사용 조건:
- 다음 단계가 이전 단계 결과에 의존하는 경우
- 설계 → 구현 → 검토 순서

---

### 4.2 병렬 실행 (Parallel)

독립적인 작업을 동시에 수행한다.

```
        ┌→ Role B ─┐
Role A ─┤          ├→ Role D
        └→ Role C ─┘
```

사용 조건:
- 작업 간 의존성이 없는 경우
- 여러 서버에 동시 작업이 필요한 경우

예시:
- Gateway + Application 동시 수정
- Producer + Consumer 동시 구현

---

### 4.3 반복 실행 (Iterative)

피드백 기반으로 반복 수행한다.

```
Developer → Reviewer → Developer → Reviewer → ...
```

사용 조건:
- 코드 리뷰 후 수정이 필요한 경우
- 설계 검토 후 재설계가 필요한 경우

---

## 5. Team 조합 규칙

### 5.1 필수 역할 포함

| Team 유형 | 필수 역할 |
|-----------|-----------|
| 개발 Team | AI_*_ENGINEER, AI_REVIEWER |
| 설계 Team | AI_*_ARCHITECT, AI_REVIEWER |
| 품질 Team | AI_REVIEWER, AI_SECURITY_ENGINEER |

---

### 5.2 역할 제약

다음 역할 조합은 금지된다:

| 금지 조합 | 이유 |
|-----------|------|
| AI_GATEWAY_ENGINEER + DB 작업 | Gateway는 DB 접근 금지 |
| AI_APPLICATION_ENGINEER + MQ Consumer | Consumer는 System 전용 |
| AI_SYSTEM_ENGINEER + MQ Producer | Producer는 Application 전용 |

---

### 5.3 서버 경계 준수

Team 구성 시 서버 책임을 반드시 준수한다.

| 서버 | 역할 제한 |
|------|-----------|
| Gateway | AI_GATEWAY_* 만 구현 수행 |
| Application | AI_APPLICATION_* 만 구현 수행 |
| System | AI_SYSTEM_* 만 구현 수행 |

---

## 6. Team 커뮤니케이션

### 6.1 핸드오프 프로토콜

역할 간 작업 인계 시 다음을 전달한다:

1. 수행한 작업 요약
2. 생성/수정된 파일 목록
3. 다음 역할에 대한 지시사항
4. 주의 사항

형식:
```
[HANDOFF: Role A → Role B]

수행 작업:
- {작업 내용}

생성 파일:
- {파일 목록}

다음 단계:
- {지시사항}

주의:
- {주의 사항}
```

---

### 6.2 피드백 루프

Reviewer → Developer 피드백 형식:

```
[FEEDBACK: AI_REVIEWER → AI_*_ENGINEER]

이슈:
- {이슈 목록}

수정 요청:
- {수정 사항}

우선순위:
- CRITICAL / HIGH / MEDIUM / LOW
```

---

### 6.3 충돌 해결

역할 간 의견 충돌 시 다음 우선순위를 따른다:

1. AI_TECH_LEAD (최종 결정권)
2. AI_*_ARCHITECT (설계 관련)
3. AI_SECURITY_ENGINEER (보안 관련)
4. AI_REVIEWER (품질 관련)

---

## 7. 동적 Group 구성

### 7.1 AI_TASK_ROUTER 분석

AI_TASK_ROUTER는 요청을 분석하여 적절한 Group을 구성한다.

분석 항목:
- 작업 유형
- 대상 서버
- 복잡도
- 의존성

---

### 7.2 Group 구성 규칙

```
IF 작업이 단일 서버 대상 THEN
    해당 서버 Team 선택
ELSE IF 작업이 여러 서버 대상 THEN
    Cross-Server Group 구성
END IF

IF 복잡도 HIGH THEN
    AI_TECH_LEAD 포함
END IF

IF Event Driven 작업 THEN
    EVENTFLOW_TEAM 사용
END IF
```

---

### 7.3 Cross-Server Group

여러 서버에 걸친 작업 시 구성한다.

예시: 회원 가입 API + 알림 발송

| 순서 | 역할 | 서버 | 작업 |
|------|------|------|------|
| 1 | AI_APPLICATION_ARCHITECT | Application | API 설계 |
| 2 | AI_EVENTFLOW_ARCHITECT | - | MQ 설계 |
| 3 | AI_APPLICATION_ENGINEER | Application | API + Producer 구현 |
| 4 | AI_SYSTEM_ENGINEER | System | Consumer 구현 |
| 5 | AI_REVIEWER | - | 전체 코드 리뷰 |
| 6 | AI_SECURITY_ENGINEER | - | 보안 검토 |

---

## 8. Team 선택 가이드

### 8.1 요청 분석 → Team 매핑

| 요청 키워드 | 추천 Team |
|-------------|-----------|
| 새 기능, 새 API | FEATURE_DEVELOPMENT_TEAM |
| MQ, Event, 알림 | EVENTFLOW_TEAM |
| 코드 리뷰, PR 검토 | QUALITY_ASSURANCE_TEAM |
| 아키텍처, 설계 | ARCHITECTURE_TEAM |
| Gateway, 인증, 라우팅 | GATEWAY_TEAM |
| 비즈니스 API, 도메인 | APPLICATION_TEAM |
| Worker, Consumer, Background | SYSTEM_TEAM |
| Docker, CI/CD, 배포 | INFRA_TEAM |

---

### 8.2 복잡도 기반 선택

| 복잡도 | 권장 방식 |
|--------|-----------|
| LOW | 단일 역할 |
| MEDIUM | 단일 Team |
| HIGH | Team + AI_TECH_LEAD |
| VERY HIGH | Cross-Server Group + Pipeline |

---

## 9. Team 실행 예시

### 9.1 회원 가입 API 구현

요청: "회원 가입 API를 구현해줘"

Team 선택: APPLICATION_TEAM

실행 순서:
```
1. AI_APPLICATION_ARCHITECT
   - 회원 도메인 구조 설계
   - API 스펙 정의

2. AI_APPLICATION_ENGINEER
   - Controller 구현
   - Service 구현
   - Repository 구현

3. AI_REVIEWER
   - 코드 품질 검토
   - 아키텍처 준수 검토

4. AI_SECURITY_ENGINEER
   - 입력 검증 검토
   - 인증 처리 검토
```

---

### 9.2 알림 시스템 구현

요청: "회원 가입 시 알림을 발송하는 시스템을 구현해줘"

Team 선택: EVENTFLOW_TEAM

실행 순서:
```
1. AI_EVENTFLOW_ARCHITECT
   - MQ topology 설계
   - Exchange/Queue 정의
   - DLQ 정책 설계

2. AI_APPLICATION_ENGINEER
   - Producer 구현 (Application 서버)
   - 이벤트 발행 로직

3. AI_SYSTEM_ENGINEER
   - Consumer 구현 (System 서버)
   - 알림 발송 처리

4. AI_REVIEWER
   - 전체 흐름 검토
   - 에러 처리 검토

5. AI_SECURITY_ENGINEER
   - 메시지 보안 검토
   - 민감 정보 처리 검토
```

---

### 9.3 Gateway 인증 수정

요청: "OAuth2 인증 흐름을 수정해줘"

Team 선택: GATEWAY_TEAM

실행 순서:
```
1. AI_GATEWAY_ARCHITECT
   - 인증 흐름 재설계
   - 정책 정의

2. AI_GATEWAY_ENGINEER
   - Filter 수정
   - 인증 로직 구현

3. AI_SECURITY_ENGINEER
   - 보안 취약점 검토
   - 인증 흐름 검증

4. AI_REVIEWER
   - 코드 리뷰
   - Gateway 원칙 준수 검토
```

---

## 10. Result 출력

Team 실행 완료 시 결과 보고서를 작성한다.

### 10.1 출력 위치

```
prompt-result/{team-name}-{timestamp}.html
```

### 10.2 포함 내용

- Team 구성 정보
- 각 역할별 수행 내역
- 핸드오프 이력
- 이슈 목록 및 조치 상태
- 생성/수정 파일 목록
- 종합 평가

---

## 11. 금지 사항

다음은 Team 운영 시 금지된다:

1. 서버 경계 침범
   - Gateway Team이 DB 작업 수행
   - Application Team이 Consumer 구현

2. 역할 생략
   - 개발 작업 후 Reviewer 생략
   - 보안 민감 작업 후 Security 검토 생략

3. 병렬 실행 오용
   - 의존성 있는 작업을 병렬 실행

4. 피드백 무시
   - Reviewer 피드백 반영 없이 진행

---

END OF FILE
