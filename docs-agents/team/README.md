# Team 정의

이 디렉토리는 AI Agent Team을 정의한다.

---

## Team 개요

Team은 복잡한 작업을 수행하기 위해 여러 Agent가 협업하는 구조이다.

---

## 사전 정의 Team

| 파일 | Team | 용도 |
|------|------|------|
| FEATURE_DEVELOPMENT_TEAM.md | FEATURE_DEVELOPMENT_TEAM | 새로운 기능 개발 |
| EVENTFLOW_TEAM.md | EVENTFLOW_TEAM | Event Driven 시스템 구현 |
| QUALITY_ASSURANCE_TEAM.md | QUALITY_ASSURANCE_TEAM | 코드 품질 보증 |
| ARCHITECTURE_TEAM.md | ARCHITECTURE_TEAM | 시스템 설계 |
| GATEWAY_TEAM.md | GATEWAY_TEAM | Gateway 서버 작업 |
| APPLICATION_TEAM.md | APPLICATION_TEAM | Application 서버 작업 |
| SYSTEM_TEAM.md | SYSTEM_TEAM | System 서버 작업 |
| INFRA_TEAM.md | INFRA_TEAM | 인프라 구성 |

---

## Team 선택 가이드

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

## 실행 전략

### 순차 실행 (Sequential)

```
Role A → Role B → Role C
```

의존성이 있는 작업에 사용한다.

### 병렬 실행 (Parallel)

```
        ┌→ Role B ─┐
Role A ─┤          ├→ Role D
        └→ Role C ─┘
```

독립적인 작업에 사용한다.

### 반복 실행 (Iterative)

```
Developer ↔ Reviewer
```

피드백 기반 수정에 사용한다.

---

## 전략 문서

- [AGENT_TEAM_STRATEGY.md](./AGENT_TEAM_STRATEGY.md) - Team 운영 전략 상세

---

## 관련 문서

- [Agent 역할 정의](../agent/README.md)
- CLAUDE_AI_TEAM_POLICY.md

---

END OF FILE
