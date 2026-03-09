# Agent 역할 정의

이 디렉토리는 개별 AI Agent 역할을 정의한다.

---

## 역할 분류

### Orchestrator 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_TASK_ROUTER.md | AI_TASK_ROUTER | 명령 분석 및 역할 추천 |
| AI_TASK_PIPELINE.md | AI_TASK_PIPELINE | 자동 개발 파이프라인 |

### Architecture 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_TECH_LEAD.md | AI_TECH_LEAD | 기술 리더 (기본 역할) |
| AI_ARCHITECT.md | AI_ARCHITECT | 시스템 아키텍처 설계 |
| AI_EVENT_ARCHITECT.md | AI_EVENT_ARCHITECT | Event Driven 설계 |
| AI_EVENTFLOW_ARCHITECT.md | AI_EVENTFLOW_ARCHITECT | EventFlow 아키텍처 설계 |

### Engineering 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_BACKEND_ENGINEER.md | AI_BACKEND_ENGINEER | 백엔드 코드 구현 |
| AI_INFRA_ENGINEER.md | AI_INFRA_ENGINEER | 인프라 구성 |

### Quality 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_REVIEWER.md | AI_REVIEWER | 코드 품질 검토 |
| AI_SECURITY_ENGINEER.md | AI_SECURITY_ENGINEER | 보안 검토 |
| AI_REFACTOR_ENGINEER.md | AI_REFACTOR_ENGINEER | 코드 개선 |

### Gateway 전용 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_GATEWAY_ARCHITECT.md | AI_GATEWAY_ARCHITECT | Gateway 서버 설계 |
| AI_GATEWAY_ENGINEER.md | AI_GATEWAY_ENGINEER | Gateway 구현 |

### Application 전용 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_APPLICATION_ARCHITECT.md | AI_APPLICATION_ARCHITECT | Application 서버 설계 |
| AI_APPLICATION_ENGINEER.md | AI_APPLICATION_ENGINEER | Application 구현 |

### System 전용 역할

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_SYSTEM_ARCHITECT.md | AI_SYSTEM_ARCHITECT | System 서버 설계 |
| AI_SYSTEM_ENGINEER.md | AI_SYSTEM_ENGINEER | System 서버 구현 |

---

## 역할 선택 규칙

1. 사용자가 역할을 명시하면 해당 역할로 동작
2. 명시하지 않으면 AI_TASK_ROUTER가 역할 추천
3. 기본 fallback 역할은 AI_TECH_LEAD

---

## 관련 문서

- [Team 정의](../team/README.md)
- CLAUDE_AI_TEAM_POLICY.md

---

END OF FILE
