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
| AI_TECH_LEAD | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 03_data/DATABASE.md, 04_backend/CODE_CONVENTION.md | 07_process/PROCESS.md, 작업 대상 모듈 문서 |
| AI_EVENT_ARCHITECT | 01_architecture/ARCHITECTURE.md, 05_infra/INFRASTRUCTURE.md, 04_backend/SYSTEM.md | 04_backend/APPLICATION.md |
| AI_GATEWAY_ARCHITECT | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 04_backend/GATEWAY.md | - |
| AI_APPLICATION_ARCHITECT | 01_architecture/ARCHITECTURE.md, 04_backend/CODE_CONVENTION.md, 04_backend/APPLICATION.md | 03_data/DATABASE.md |
| AI_SYSTEM_ARCHITECT | 01_architecture/ARCHITECTURE.md, 04_backend/SYSTEM.md, 05_infra/INFRASTRUCTURE.md | 03_data/DATABASE.md |

### 1.3 Engineers

| 역할 | 필수 로드 | 선택 로드 |
|------|----------|----------|
| AI_GATEWAY_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 04_backend/CODE_CONVENTION.md, 04_backend/GATEWAY.md | - |
| AI_APPLICATION_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 03_data/DATABASE.md, 04_backend/CODE_CONVENTION.md, 04_backend/APPLICATION.md | 05_infra/INFRASTRUCTURE.md (MQ Producer 구현 시) |
| AI_SYSTEM_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md, 03_data/DATABASE.md, 04_backend/CODE_CONVENTION.md, 04_backend/SYSTEM.md, 05_infra/INFRASTRUCTURE.md | - |
| AI_DATABASE_ENGINEER | 03_data/DATABASE.md, 01_architecture/ARCHITECTURE.md, 04_backend/CODE_CONVENTION.md | 작업 대상 모듈 문서 |
| AI_API_DOC_ENGINEER | 06_api-docs/API_DOCUMENTATION.md, 01_architecture/ARCHITECTURE.md | 04_backend/CODE_CONVENTION.md, 작업 대상 모듈 문서 |
| AI_INFRA_ENGINEER | 01_architecture/ARCHITECTURE.md, 05_infra/INFRASTRUCTURE.md | 02_security/SECURITY.md |

### 1.4 Quality

| 역할 | 필수 로드 | 선택 로드 |
|------|----------|----------|
| AI_REVIEWER | 01_architecture/ARCHITECTURE.md, 04_backend/CODE_CONVENTION.md | 리뷰 대상 모듈 문서 |
| AI_SECURITY_ENGINEER | 01_architecture/ARCHITECTURE.md, 02_security/SECURITY.md | 검증 대상 모듈 문서 |
| AI_REFACTOR_ENGINEER | 01_architecture/ARCHITECTURE.md, 04_backend/CODE_CONVENTION.md | 리팩토링 대상 모듈 문서 |

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

## 3. 모듈 판별 기준

작업 대상에 따라 추가 로드할 모듈 문서:

| 작업 대상 | 추가 로드 |
|-----------|----------|
| Gateway | 04_backend/GATEWAY.md |
| Application | 04_backend/APPLICATION.md |
| System | 04_backend/SYSTEM.md |
| Database | 03_data/DATABASE.md |
| MQ/이벤트 | 05_infra/INFRASTRUCTURE.md |
| API 문서 | 06_api-docs/API_DOCUMENTATION.md |

---

END OF FILE
