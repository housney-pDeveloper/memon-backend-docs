# APPLICATION_TEAM

Application 서버 작업을 수행할 때 사용하는 Team이다.

---

## 1. 개요

APPLICATION_TEAM은 비즈니스 로직을 설계하고 구현하는 Team이다.
API, 도메인 서비스, Repository 등 비즈니스 계층을 담당한다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_APPLICATION_ARCHITECT | 비즈니스 구조 설계 |
| 2 | AI_APPLICATION_ENGINEER | 서비스/API 구현 |
| 3 | AI_REVIEWER | 코드 리뷰 |
| 4 | AI_SECURITY_ENGINEER | 보안 검토 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- 비즈니스 API 요청
- 도메인 서비스 요청
- Application 기능 요청
- CRUD API 요청
- 비즈니스 로직 구현 요청

---

## 4. 실행 워크플로우

```
[A] ┌───────────────────────────┐
    │ AI_APPLICATION_ARCHITECT  │ 비즈니스 구조 설계
    └────────────┬──────────────┘
                 ▼
[B] ┌───────────────────────────┐
    │ AI_APPLICATION_ENGINEER   │ 서비스/API 구현
    └────────────┬──────────────┘
                 ▼
[C] ┌────────────┴────────────┐        ← 병렬 (C-1, C-2)
    ▼                         ▼
┌─────────────┐       ┌─────────────┐
│ AI_REVIEWER │       │AI_SECURITY  │
│ (코드 리뷰)  │       │ (보안 검토)  │
└──────┬──────┘       └──────┬──────┘
       └──────────┬──────────┘
                  ▼  (C-1, C-2 모두 완료 대기)
[D]      result_VERIFICATION.md
```

---

## 5. Application 서버 역할

### 5.1 책임 범위

- 비즈니스 로직 구현
- 도메인 서비스 관리
- API 제공
- MQ Producer 역할
- 트랜잭션 관리

### 5.2 Event Driven 규칙

- **Producer**: Application 서버에서 생성
- Consumer는 System 서버에서 처리 (이 Team 범위 외)

---

## 6. 기술 스택

- Java 21
- Spring Boot
- MyBatis
- PostgreSQL
- RabbitMQ (Producer만)
- Redis

---

## 7. 핸드오프 규칙

### 7.1 APPLICATION_ARCHITECT → APPLICATION_ENGINEER

전달 항목:
- 도메인 모델 설계
- API 스펙
- 서비스 책임 정의
- 검증 규칙

### 7.2 APPLICATION_ENGINEER → REVIEWER

전달 항목:
- Controller 구현체
- Service 구현체
- Repository 구현체
- DTO/VO 정의
- 테스트 코드

---

## 8. 실행 예시

### 요청

"회원 프로필 수정 API를 구현해줘"

### 실행

```
[STEP 1] AI_APPLICATION_ARCHITECT
- API 스펙 정의
  - PUT /api/v1/members/{id}/profile
  - Request: UpdateProfileRequest
  - Response: ProfileResponse
- 도메인 규칙 정의
  - 닉네임 중복 검증
  - 이미지 URL 검증

[STEP 2] AI_APPLICATION_ENGINEER
- MemberController.updateProfile() 구현
- MemberService.updateProfile() 구현
- MemberRepository.updateProfile() 구현
- UpdateProfileRequest DTO 생성
- ProfileResponse DTO 생성

[STEP 3] AI_REVIEWER
- 계층 구조 검토
- 예외 처리 검토
- 코드 품질 검토

[STEP 4] AI_SECURITY_ENGINEER
- 입력 검증 검토
- 권한 검증 검토 (본인만 수정 가능)
- SQL Injection 검토
```

---

## 9. 제약 사항

- MQ Consumer 구현 금지 (System 서버 역할)
- Gateway 역할 침범 금지
- 인증/인가 로직 구현 금지 (Gateway 역할)

---

## 10. 관련 문서

- [AI_APPLICATION_ARCHITECT](../agent/AI_APPLICATION_ARCHITECT.md)
- [AI_APPLICATION_ENGINEER](../agent/AI_APPLICATION_ENGINEER.md)
- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- 32_application/CLAUDE.md

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 단계 | 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|------|-----------------|------|
| A | 1 | AI_APPLICATION_ARCHITECT | 01_architecture, 04_backend/CODE_CONVENTION, 04_backend/APPLICATION | N |
| B | 2 | AI_APPLICATION_ENGINEER | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION, 04_backend/APPLICATION | N |
| C | 3-1 | AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | Y (with 3-2) |
| C | 3-2 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | Y (with 3-1) |

### 핸드오프 흐름

```
prompt.md → task_prompt.md
→ [A] result_AI_APPLICATION_ARCHITECT.md
→ [B] result_AI_APPLICATION_ENGINEER.md
→ [C] result_AI_REVIEWER.md + result_AI_SECURITY_ENGINEER.md  ← 병렬, 모두 완료 대기
→ [D] result_VERIFICATION.md
```

**병렬 입력 규칙:**
- C단계 (3-1, 3-2): prompt.md + task_prompt.md + result_AI_APPLICATION_ARCHITECT.md + result_AI_APPLICATION_ENGINEER.md

---

END OF FILE
