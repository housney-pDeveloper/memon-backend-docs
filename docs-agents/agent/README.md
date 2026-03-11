# Agent 역할 정의

이 디렉토리는 개별 AI Agent 역할을 정의한다. (총 15개)

---

## 역할 분류

### Orchestrator (1개)

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_TASK_ROUTER.md | AI_TASK_ROUTER | 명령 분석, 역할 추천, Pipeline 오케스트레이션 |

### Architect (5개)

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_TECH_LEAD.md | AI_TECH_LEAD | 기술 리더 + 시스템 아키텍처 총괄 (기본 역할) |
| AI_EVENT_ARCHITECT.md | AI_EVENT_ARCHITECT | Event Driven / EventFlow 설계 |
| AI_GATEWAY_ARCHITECT.md | AI_GATEWAY_ARCHITECT | Gateway 서버 설계 |
| AI_APPLICATION_ARCHITECT.md | AI_APPLICATION_ARCHITECT | Application 서버 설계 |
| AI_SYSTEM_ARCHITECT.md | AI_SYSTEM_ARCHITECT | System 서버 설계 |

### Engineer (6개)

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_GATEWAY_ENGINEER.md | AI_GATEWAY_ENGINEER | Gateway 구현 |
| AI_APPLICATION_ENGINEER.md | AI_APPLICATION_ENGINEER | Application 구현 |
| AI_SYSTEM_ENGINEER.md | AI_SYSTEM_ENGINEER | System 서버 구현 |
| AI_DATABASE_ENGINEER.md | AI_DATABASE_ENGINEER | DB 설계, DDL/DML, Mapper |
| AI_API_DOC_ENGINEER.md | AI_API_DOC_ENGINEER | API 문서 생성 |
| AI_INFRA_ENGINEER.md | AI_INFRA_ENGINEER | 인프라 구성 |

### Quality (3개)

| 파일 | 역할 | 설명 |
|------|------|------|
| AI_REVIEWER.md | AI_REVIEWER | 코드 품질 검토 |
| AI_SECURITY_ENGINEER.md | AI_SECURITY_ENGINEER | 보안 검토 |
| AI_REFACTOR_ENGINEER.md | AI_REFACTOR_ENGINEER | 코드 개선 |

---

## 역할 선택 규칙

1. 사용자가 역할을 명시하면 해당 역할로 동작
2. 명시하지 않으면 AI_TASK_ROUTER가 역할 추천
3. 기본 fallback 역할은 AI_TECH_LEAD

---

## 관련 문서

- [Team 정의](../team/README.md)
- [역할별 docs-claude 매핑](../AGENT_DOCS_MAPPING.md)
- [Team 수행 프로토콜](../TEAM_EXECUTION_PROTOCOL.md)

---

END OF FILE
