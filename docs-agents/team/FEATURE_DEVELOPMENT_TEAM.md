# FEATURE_DEVELOPMENT_TEAM

새로운 기능을 개발할 때 사용하는 Team이다.

---

## 1. 개요

FEATURE_DEVELOPMENT_TEAM은 새로운 기능을 설계부터 구현, 검토까지
전 과정을 수행하는 표준 개발 Team이다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_TECH_LEAD | 요구사항 분석, 작업 분해 |
| 2 | AI_*_ARCHITECT | 아키텍처 설계 |
| 3 | AI_*_ENGINEER | 코드 구현 |
| 4 | AI_REVIEWER | 코드 리뷰 |
| 5 | AI_SECURITY_ENGINEER | 보안 검토 |
| 6 | AI_REFACTOR_ENGINEER | 코드 개선 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- 새 기능 개발 요청
- API 구현 요청
- 서비스 구현 요청
- 도메인 기능 구현 요청

---

## 4. 실행 워크플로우

```
┌─────────────────┐
│  AI_TECH_LEAD   │ 요구사항 분석
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_*_ARCHITECT  │ 아키텍처 설계
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_*_ENGINEER   │ 코드 구현
└────────┬────────┘
         ▼
┌─────────────────┐
│  AI_REVIEWER    │ 코드 리뷰
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_SECURITY_ENG │ 보안 검토
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_REFACTOR_ENG │ 코드 개선
└─────────────────┘
```

---

## 5. 역할 선택 규칙

### 5.1 Architect 선택

| 대상 서버 | 역할 |
|-----------|------|
| Gateway | AI_GATEWAY_ARCHITECT |
| Application | AI_APPLICATION_ARCHITECT |
| System | AI_SYSTEM_ARCHITECT |
### 5.2 Engineer 선택

| 대상 서버 | 역할 |
|-----------|------|
| Gateway | AI_GATEWAY_ENGINEER |
| Application | AI_APPLICATION_ENGINEER |
| System | AI_SYSTEM_ENGINEER |

---

## 6. 핸드오프 규칙

### 6.1 TECH_LEAD → ARCHITECT

전달 항목:
- 요구사항 분석 결과
- 작업 분해 목록
- 개발 전략

### 6.2 ARCHITECT → ENGINEER

전달 항목:
- 아키텍처 설계 문서
- API 스펙
- 도메인 모델

### 6.3 ENGINEER → REVIEWER

전달 항목:
- 생성/수정된 파일 목록
- 구현 요약
- 테스트 결과

---

## 7. 실행 예시

### 요청

"회원 가입 API를 구현해줘"

### 실행

```
[STEP 1] AI_TECH_LEAD
- 요구사항: 회원 가입 기능
- 작업 분해: Controller, Service, Repository, DTO

[STEP 2] AI_APPLICATION_ARCHITECT
- 도메인 구조 설계
- API 스펙 정의
- 검증 규칙 정의

[STEP 3] AI_APPLICATION_ENGINEER
- SignupController 구현
- SignupService 구현
- MemberRepository 구현

[STEP 4] AI_REVIEWER
- 코드 품질 검토
- 아키텍처 준수 검토

[STEP 5] AI_SECURITY_ENGINEER
- 입력 검증 검토
- 비밀번호 처리 검토

[STEP 6] AI_REFACTOR_ENGINEER
- 코드 개선
- 중복 제거
```

---

## 8. 제약 사항

- 서버 경계 준수 필수
- Reviewer 단계 생략 금지
- 보안 민감 기능은 Security 검토 필수

---

## 9. 관련 문서

- [AI_TECH_LEAD](../agent/AI_TECH_LEAD.md)
- [AI_*_ARCHITECT](../agent/) (모듈별 Architect)
- [AI_*_ENGINEER](../agent/) (모듈별 Engineer)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- [AI_REFACTOR_ENGINEER](../agent/AI_REFACTOR_ENGINEER.md)

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|-----------------|------|
| 1 | AI_TECH_LEAD | 01_architecture, 04_backend/CODE_CONVENTION | N |
| 2 | AI_*_ARCHITECT | 역할별 AGENT_DOCS_MAPPING 참조 | N |
| 3 | AI_*_ENGINEER | 역할별 AGENT_DOCS_MAPPING 참조 | N |
| 4 | AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | Y (with 5) |
| 5 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | Y (with 4) |
| 6 | AI_REFACTOR_ENGINEER | 01_architecture, 04_backend/CODE_CONVENTION | N |

### 핸드오프 흐름

```
prompt.md → task_prompt.md → result_AI_TECH_LEAD.md
→ result_AI_*_ARCHITECT.md → result_AI_*_ENGINEER.md
→ result_AI_REVIEWER.md + result_AI_SECURITY_ENGINEER.md (병렬)
→ result_AI_REFACTOR_ENGINEER.md → result_VERIFICATION.md
```

---

END OF FILE
