# QUALITY_ASSURANCE_TEAM

코드 품질 보증을 수행할 때 사용하는 Team이다.

---

## 1. 개요

QUALITY_ASSURANCE_TEAM은 코드의 품질, 보안, 구조를 검토하고
개선하는 품질 보증 전문 Team이다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_REVIEWER | 코드 품질 검토 |
| 2 | AI_SECURITY_ENGINEER | 보안 취약점 검토 |
| 3 | AI_REFACTOR_ENGINEER | 코드 개선 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- 코드 리뷰 요청
- 품질 검토 요청
- PR 리뷰 요청
- 보안 검토 요청
- 리팩토링 요청

---

## 4. 실행 워크플로우

```
┌─────────────────┐
│  AI_REVIEWER    │ 코드 품질 검토
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_SECURITY_ENG │ 보안 취약점 검토
└────────┬────────┘
         ▼
┌─────────────────┐
│ AI_REFACTOR_ENG │ 코드 개선
└─────────────────┘
```

---

## 5. 검토 항목

### 5.1 AI_REVIEWER 검토 항목

- 아키텍처 위반
- Layer 침범
- 예외 처리 누락
- 성능 문제
- 코드 가독성
- 명명 규칙 준수
- 중복 코드

### 5.2 AI_SECURITY_ENGINEER 검토 항목

- Secret 노출
- SQL Injection
- XSS
- CSRF
- 인증 처리 문제
- 인가 처리 문제
- 민감 정보 로깅

### 5.3 AI_REFACTOR_ENGINEER 개선 항목

- 코드 구조 개선
- 중복 제거
- 가독성 향상
- 명명 규칙 적용

---

## 6. 피드백 형식

### 6.1 이슈 분류

| 우선순위 | 설명 |
|----------|------|
| CRITICAL | 즉시 수정 필요 (보안, 장애 위험) |
| HIGH | 배포 전 수정 필요 |
| MEDIUM | 권장 수정 |
| LOW | 개선 제안 |

### 6.2 피드백 형식

```
[ISSUE] {우선순위}
파일: {파일 경로}
라인: {라인 번호}
내용: {이슈 설명}
수정: {수정 제안}
```

---

## 7. 반복 실행

이슈 발견 시 반복 실행될 수 있다.

```
REVIEWER → 이슈 발견
    ↓
ENGINEER ← 수정 요청
    ↓
REVIEWER → 재검토
    ↓
통과 시 다음 단계
```

---

## 8. 실행 예시

### 요청

"SignupService 코드를 리뷰해줘"

### 실행

```
[STEP 1] AI_REVIEWER
이슈 목록:
- [MEDIUM] 예외 처리 누락 (line 45)
- [LOW] 메서드명 개선 필요 (line 23)

[STEP 2] AI_SECURITY_ENGINEER
이슈 목록:
- [HIGH] 비밀번호 로깅 발견 (line 67)
- [MEDIUM] 입력 검증 미흡 (line 34)

[STEP 3] AI_REFACTOR_ENGINEER
개선 사항:
- 중복 코드 추출 (validateInput 메서드)
- 상수 정의 추가
- 주석 정리
```

---

## 9. 제약 사항

- CRITICAL/HIGH 이슈는 반드시 수정 후 재검토
- 보안 이슈는 Security Engineer 검토 필수
- 검토 결과 문서화 필수

---

## 10. 관련 문서

- [AI_REVIEWER](../agent/AI_REVIEWER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- [AI_REFACTOR_ENGINEER](../agent/AI_REFACTOR_ENGINEER.md)

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|-----------------|------|
| 1 | AI_REVIEWER | 01_architecture, 04_backend/CODE_CONVENTION | Y (with 2) |
| 2 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | Y (with 1) |
| 3 | AI_REFACTOR_ENGINEER | 01_architecture, 04_backend/CODE_CONVENTION | N |

### 핸드오프 흐름

```
prompt.md → task_prompt.md
→ result_AI_REVIEWER.md + result_AI_SECURITY_ENGINEER.md (병렬)
→ result_AI_REFACTOR_ENGINEER.md
→ result_VERIFICATION.md
```

---

END OF FILE
