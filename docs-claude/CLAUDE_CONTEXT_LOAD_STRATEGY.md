# CLAUDE_CONTEXT_LOAD_STRATEGY.md

목적:
Claude Code가 불필요한 문서를 읽지 않도록
상황별 문서 로딩 전략을 정의한다.

---

==================================================
1. 3단계 로딩 원칙
==================================================

[Step 1] 프로젝트 범위 결정  
[Step 2] 해당 프로젝트 CLAUDE.md 로딩  
[Step 3] 작업 유형별 docs-claude 로딩

전역 문서는 항상 마지막에 참조한다.

---

==================================================
2. 작업 유형별 로딩 매트릭스
==================================================

--------------------------------------------------
2.1 API 개발 (Application)
--------------------------------------------------

필수 로딩:

- EF_GLOBAL_POLICY.md
- application/CLAUDE.md
- ARCHITECTURE.md
- CODE_CONVENTION.md

조건부 로딩:

- Validation 수정 → VALIDATION_RULES.md
- Mapper 수정 → DAO_MAPPER_RULES.md
- 응답 수정 → RESPONSE_POLICY.md
- 에러코드 추가 → ERROR_CODE_POLICY.md

읽지 말 것:

- gateway docs
- system docs

--------------------------------------------------
2.2 Gateway 정책 수정
--------------------------------------------------

필수:

- EF_GLOBAL_POLICY.md
- gateway/CLAUDE.md
- SECURITY.md
- ROUTING_POLICY.md

조건부:

- 에러코드 → ERROR_CODE_POLICY.md
- 응답 수정 → RESPONSE_POLICY.md

읽지 말 것:

- application biz 규칙

--------------------------------------------------
2.3 MQ / Batch 수정 (System)
--------------------------------------------------

필수:

- EF_GLOBAL_POLICY.md
- system/CLAUDE.md
- EVENT_PROCESSING_POLICY.md

조건부:

- 응답 수정 → RESPONSE_POLICY.md
- 에러코드 추가 → ERROR_CODE_POLICY.md

읽지 말 것:

- application domain 규칙

--------------------------------------------------
2.4 Database 수정
--------------------------------------------------

필수:

- database/CLAUDE.md
- TABLE_POLICY.md
- NAMING_CONVENTION.md

조건부:

- dml_script 변경 → DML_SCRIPT_POLICY.md
- 변경 후 처리 → CHANGE_PROCESS.md

읽지 말 것:

- application service 규칙

---

==================================================
3. 금지 사항
==================================================

- 모든 문서를 한 번에 읽지 않는다.
- 다른 서버 규칙을 복사하지 않는다.
- 프로젝트 경계를 넘는 수정은 질문 후 진행한다.

---

==================================================
4. 토큰 최소화 전략
==================================================

- 아키텍처 변경이 아닐 경우 ARCHITECTURE.md 재로딩 금지
- 단순 필드 추가 시 GLOBAL_POLICY 재로딩 금지
- 단순 SQL 수정 시 응답/보안 문서 로딩 금지

---

END OF FILE