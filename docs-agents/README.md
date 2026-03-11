# docs-agents

이 디렉토리는 Claude Code AI 팀 역할 및 Team 정의 문서를 포함한다.

---

## 디렉토리 구조

```
docs-agents/
├── README.md                          # 이 파일
├── AGENT_DOCS_MAPPING.md              # 역할별 docs-claude 참조 매핑
├── TEAM_EXECUTION_PROTOCOL.md         # Team 수행 프로토콜
├── agent/                             # 개별 Agent 역할 정의 (15개)
│   ├── AI_TASK_ROUTER.md              # Orchestrator (Pipeline 포함)
│   ├── AI_TECH_LEAD.md                # 기술 리더 + 시스템 아키텍처 총괄
│   ├── AI_EVENT_ARCHITECT.md          # 이벤트/MQ/EventFlow 설계
│   ├── AI_GATEWAY_ARCHITECT.md        # Gateway 설계
│   ├── AI_APPLICATION_ARCHITECT.md    # Application 설계
│   ├── AI_SYSTEM_ARCHITECT.md         # System 설계
│   ├── AI_GATEWAY_ENGINEER.md         # Gateway 구현
│   ├── AI_APPLICATION_ENGINEER.md     # Application 구현
│   ├── AI_SYSTEM_ENGINEER.md          # System 구현
│   ├── AI_DATABASE_ENGINEER.md        # DB 설계/DDL/DML/Mapper
│   ├── AI_API_DOC_ENGINEER.md         # API 문서 생성
│   ├── AI_INFRA_ENGINEER.md           # 인프라 구성
│   ├── AI_REVIEWER.md                 # 코드 품질 검토
│   ├── AI_SECURITY_ENGINEER.md        # 보안 검토
│   └── AI_REFACTOR_ENGINEER.md        # 코드 개선
└── team/                              # Team 정의
    ├── AGENT_TEAM_STRATEGY.md         # Team 전략 총괄
    ├── FEATURE_DEVELOPMENT_TEAM.md
    ├── EVENTFLOW_TEAM.md
    ├── QUALITY_ASSURANCE_TEAM.md
    ├── ARCHITECTURE_TEAM.md
    ├── GATEWAY_TEAM.md
    ├── APPLICATION_TEAM.md
    ├── SYSTEM_TEAM.md
    └── INFRA_TEAM.md
```

---

## 개요

AI Agent 역할과 Team을 개별 문서로 분리하여 관리한다.
각 역할은 AGENT_DOCS_MAPPING.md에 따라 docs-claude 문서를 참조한다.
Team 수행 시 TEAM_EXECUTION_PROTOCOL.md에 따라 프로세스를 진행한다.

---

## Agent 역할 (15개)

### 역할 분류

| 분류 | 역할 | 수 |
|------|------|----|
| Orchestrator | AI_TASK_ROUTER | 1 |
| Architect | AI_TECH_LEAD, AI_EVENT_ARCHITECT, AI_GATEWAY_ARCHITECT, AI_APPLICATION_ARCHITECT, AI_SYSTEM_ARCHITECT | 5 |
| Engineer | AI_GATEWAY_ENGINEER, AI_APPLICATION_ENGINEER, AI_SYSTEM_ENGINEER, AI_DATABASE_ENGINEER, AI_API_DOC_ENGINEER, AI_INFRA_ENGINEER | 6 |
| Quality | AI_REVIEWER, AI_SECURITY_ENGINEER, AI_REFACTOR_ENGINEER | 3 |

---

## Team 정의 (8개)

| Team | 용도 | 핵심 역할 |
|------|------|----------|
| FEATURE_DEVELOPMENT_TEAM | 새로운 기능 개발 | TECH_LEAD → MODULE_ARCHITECT → MODULE_ENGINEER → REVIEWER |
| EVENTFLOW_TEAM | Event Driven 구현 | EVENT_ARCHITECT → APP_ENGINEER → SYS_ENGINEER |
| QUALITY_ASSURANCE_TEAM | 코드 품질 보증 | REVIEWER → SECURITY → REFACTOR |
| ARCHITECTURE_TEAM | 시스템 설계 | TECH_LEAD → EVENT_ARCHITECT (선택) → REVIEWER |
| GATEWAY_TEAM | Gateway 작업 | GATEWAY_ARCHITECT → GATEWAY_ENGINEER |
| APPLICATION_TEAM | Application 작업 | APP_ARCHITECT → APP_ENGINEER |
| SYSTEM_TEAM | System 작업 | SYS_ARCHITECT → SYS_ENGINEER |
| INFRA_TEAM | 인프라 구성 | INFRA_ENGINEER → SECURITY |

---

## 표준 개발 Workflow

```
STEP 1: AI_TECH_LEAD           → 요구사항 분석, 작업 분해, 전체 설계
STEP 2: AI_*_ARCHITECT         → 모듈별 상세 설계
STEP 3: AI_EVENT_ARCHITECT     → Event driven 설계 (필요 시)
STEP 4: AI_*_ENGINEER          → 코드 구현
STEP 5: AI_REVIEWER            → 코드 리뷰
STEP 6: AI_SECURITY_ENGINEER   → 보안 검토
STEP 7: AI_REFACTOR_ENGINEER   → 리팩토링
```

---

## 자동 리뷰 파이프라인

```
Developer → AI_REVIEWER → AI_SECURITY_ENGINEER → AI_REFACTOR_ENGINEER
```

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
| Worker, Consumer | SYSTEM_TEAM |
| Docker, CI/CD | INFRA_TEAM |
| DDL, DML, 스키마 | AI_DATABASE_ENGINEER (단독) |
| API 문서 | AI_API_DOC_ENGINEER (단독) |

---

## 관련 문서

- AGENT_DOCS_MAPPING.md (역할별 docs-claude 매핑)
- TEAM_EXECUTION_PROTOCOL.md (Team 수행 프로토콜)
- CLAUDE.md (각 프로젝트)

---

END OF FILE
