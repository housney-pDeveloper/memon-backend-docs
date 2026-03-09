# docs-agents

이 디렉토리는 Claude Code AI 팀 역할 및 Team 정의 문서를 포함한다.

---

## 디렉토리 구조

```
docs-agents/
├── README.md              # 이 파일
├── agent/                 # 개별 Agent 역할 정의
│   ├── README.md
│   ├── AI_TECH_LEAD.md
│   ├── AI_ARCHITECT.md
│   ├── AI_EVENT_ARCHITECT.md
│   ├── AI_EVENTFLOW_ARCHITECT.md
│   ├── AI_BACKEND_ENGINEER.md
│   ├── AI_INFRA_ENGINEER.md
│   ├── AI_REVIEWER.md
│   ├── AI_SECURITY_ENGINEER.md
│   ├── AI_REFACTOR_ENGINEER.md
│   ├── AI_GATEWAY_ARCHITECT.md
│   ├── AI_GATEWAY_ENGINEER.md
│   ├── AI_APPLICATION_ARCHITECT.md
│   ├── AI_APPLICATION_ENGINEER.md
│   ├── AI_SYSTEM_ARCHITECT.md
│   ├── AI_SYSTEM_ENGINEER.md
│   ├── AI_TASK_ROUTER.md
│   └── AI_TASK_PIPELINE.md
└── team/                  # Team 정의
    ├── README.md
    ├── AGENT_TEAM_STRATEGY.md
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

CLAUDE_AI_TEAM_POLICY.md에서 정의된 AI 역할과 Team을 개별 문서로 분리하여 관리한다.

---

## Agent 역할

개별 Agent 역할은 [agent/](./agent/README.md) 디렉토리에서 관리한다.

### 역할 분류

| 분류 | 역할 |
|------|------|
| Orchestrator | AI_TASK_ROUTER, AI_TASK_PIPELINE |
| Architecture | AI_TECH_LEAD, AI_ARCHITECT, AI_EVENT_ARCHITECT, AI_EVENTFLOW_ARCHITECT |
| Engineering | AI_BACKEND_ENGINEER, AI_INFRA_ENGINEER |
| Quality | AI_REVIEWER, AI_SECURITY_ENGINEER, AI_REFACTOR_ENGINEER |
| Gateway | AI_GATEWAY_ARCHITECT, AI_GATEWAY_ENGINEER |
| Application | AI_APPLICATION_ARCHITECT, AI_APPLICATION_ENGINEER |
| System | AI_SYSTEM_ARCHITECT, AI_SYSTEM_ENGINEER |

---

## Team 정의

Team 정의는 [team/](./team/README.md) 디렉토리에서 관리한다.

### 사전 정의 Team

| Team | 용도 | 핵심 역할 |
|------|------|----------|
| FEATURE_DEVELOPMENT_TEAM | 새로운 기능 개발 | TECH_LEAD → ARCHITECT → ENGINEER → REVIEWER |
| EVENTFLOW_TEAM | Event Driven 구현 | EVENTFLOW_ARCHITECT → APP_ENGINEER → SYS_ENGINEER |
| QUALITY_ASSURANCE_TEAM | 코드 품질 보증 | REVIEWER → SECURITY → REFACTOR |
| ARCHITECTURE_TEAM | 시스템 설계 | TECH_LEAD → ARCHITECT → EVENT_ARCHITECT |
| GATEWAY_TEAM | Gateway 작업 | GATEWAY_ARCHITECT → GATEWAY_ENGINEER |
| APPLICATION_TEAM | Application 작업 | APP_ARCHITECT → APP_ENGINEER |
| SYSTEM_TEAM | System 작업 | SYS_ARCHITECT → SYS_ENGINEER |
| INFRA_TEAM | 인프라 구성 | INFRA_ENGINEER → SECURITY |

---

## 표준 개발 Workflow

```
STEP 1: AI_TECH_LEAD           → 요구사항 분석, 작업 분해
STEP 2: AI_ARCHITECT           → 아키텍처 설계
STEP 3: AI_EVENTFLOW_ARCHITECT → Event driven 설계 (필요 시)
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

---

## 관련 문서

- CLAUDE_AI_TEAM_POLICY.md
- CLAUDE.md (각 프로젝트)

---

END OF FILE
