# AGENT_DOCS_MAPPING.md

각 Agent 역할이 단독 수행 시 반드시 참조해야 하는 docs-claude 문서를 정의한다.
문서 경로 기준: `01_docs/docs-claude/`

---

## 1. 역할별 docs-claude 매핑

### 1.1 Orchestrator

| 역할 | 필수 로드 | 선택 로드 |
|------|----------|----------|
| AI_TASK_ROUTER | _INDEX.md | 작업 분석 후 해당 역할의 매핑에 따라 로드 |

### 1.2 Architects

| 역할 | 필수 로드 | 선택 로드 |
|------|----------|----------|
| AI_TASK_ROUTER | _INDEX.md | - |
| AI_TECH_LEAD | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION | 05_infra, 07_process |
| AI_EVENT_ARCHITECT | 01_architecture, 05_infra/INFRASTRUCTURE | 04_backend/SYSTEM |
| AI_GATEWAY_ARCHITECT | 01_architecture, 02_security, 04_backend/GATEWAY | 04_backend/CODE_CONVENTION |
| AI_GATEWAY_ENGINEER | 01_architecture, 04_backend/GATEWAY, 04_backend/CODE_CONVENTION | 02_security |
| AI_APPLICATION_ARCHITECT | 01_architecture, 04_backend/APPLICATION | 03_data |
| AI_APPLICATION_ENGINEER | 01_architecture, 04_backend/APPLICATION, 04_backend/CODE_CONVENTION | 03_data, 05_infra |
| AI_SYSTEM_ARCHITECT | 01_architecture, 04_backend/SYSTEM, 05_infra/INFRASTRUCTURE | 03_data |
| AI_SYSTEM_ENGINEER | 01_architecture, 04_backend/SYSTEM, 05_infra/INFRASTRUCTURE | 03_data |
| AI_DATABASE_ENGINEER | 01_architecture, 03_data, 04_backend/CODE_CONVENTION | - |
| AI_INFRA_ENGINEER | 01_architecture, 05_infra | 07_process |
| AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | 02_security |
| AI_SECURITY_ENGINEER | 01_architecture, 02_security | 04_backend/* |
| AI_REFACTOR_ENGINEER | 04_backend/CODE_CONVENTION | 01_architecture |
| AI_API_DOC_ENGINEER | 01_architecture, docs-api/CLAUDE.md | 04_backend/CODE_CONVENTION |

### 1.3 Engineers

| 역할 | 필수 로드 | 선택 로드 |
|------|----------|----------|
| AI_GATEWAY_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 04_backend/CODE_CONVENTION.md, 04_backend/GATEWAY.md | - |
| AI_APPLICATION_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 03_data/DATABASE.md, 04_backend/CODE_CONVENTION.md, 04_backend/APPLICATION.md | 05_infra/INFRASTRUCTURE.md (MQ Producer 구현 시) |
| AI_SYSTEM_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 03_data/DATABASE.md, 04_backend/CODE_CONVENTION.md, 04_backend/SYSTEM.md, 05_infra/INFRASTRUCTURE.md | - |
| AI_DATABASE_ENGINEER | 03_data/DATABASE.md, 01_architecture/ARCHITECTURE.md, 04_backend/CODE_CONVENTION.md | 작업 대상 모듈 문서 |
| AI_API_DOC_ENGINEER | 06_api-docs/API_DOCUMENTATION.md, 01_architecture/ARCHITECTURE.md | 04_backend/CODE_CONVENTION.md, 작업 대상 모듈 문서 |
| AI_INFRA_ENGINEER | 01_architecture/ARCHITECTURE.md, 05_infra/INFRASTRUCTURE.md | 02_security/SECURITY.md |

---

## 2. 단독 역할 수행 프로세스

```
1. 역할 확인
2. 이 문서(AGENT_DOCS_MAPPING.md)에서 해당 역할의 필수/선택 문서 확인
3. 필수 docs-claude 문서 로딩
4. 작업 대상에 따라 선택 문서 추가 로딩
5. 해당 모듈 CLAUDE.md 로딩
6. 작업 수행
```

---


## 3. 로딩 규칙

### 3.1 순서

1. docs-agents에서 역할 확인
2. 이 문서에서 필수 docs-claude 확인
3. 필수 문서 로딩
4. 작업 내용에 따라 선택 문서 로딩
5. 해당 프로젝트 CLAUDE.md 로딩

### 3.2 금지 사항

- 모든 문서를 한 번에 읽지 않는다
- 다른 서버 문서를 복사하지 않는다
- 역할 범위를 벗어난 문서는 로딩하지 않는다

---

## 4. 문서 경로

```
01_docs/docs-claude/
├── _INDEX.md
├── 01_architecture/ARCHITECTURE.md
├── 02_security/SECURITY.md
├── 03_data/DATABASE.md
├── 04_backend/
│   ├── CODE_CONVENTION.md
│   ├── GATEWAY.md
│   ├── APPLICATION.md
│   └── SYSTEM.md
├── 05_infra/
│   ├── INFRASTRUCTURE.md
│   └── FUTURE_WORK.md
├── 06_api-docs/ (docs-api 참조)
└── 07_process/
    ├── GIT_POLICY.md
    ├── BUILD_AND_PROMPT_POLICY.md
    └── COMMAND_ROUTING.md
```

---

END OF FILE
