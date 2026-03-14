# TEAM_EXECUTION_PROTOCOL.md

Team 단위 수행 시 역할 간 핸드오프, 결과물 관리, 검증 프로세스를 정의한다.

---

## 1. Team 수행 프로세스

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Team 수행 프로세스                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [사용자 프롬프트]                                                     │
│       │                                                              │
│       ▼                                                              │
│  STEP 1. prompt.md 저장                                              │
│       │  result-docs/{prompt-id}/prompt.md                           │
│       ▼                                                              │
│  STEP 2. AI_TECH_LEAD → task_prompt.md 생성                          │
│       │  result-docs/{prompt-id}/task_prompt.md                      │
│       ▼                                                              │
│  STEP 3. 역할 순차/병렬 수행                                          │
│       │  각 역할 → result_{역할명}.md 생성                             │
│       │  다음 역할은 prompt.md + task_prompt.md + 이전 result.md 전체 + 역할별 md 읽기      │
│       ▼                                                              │
│  STEP 4. 검증 (연속 2회 이상 이상없음 도출 시까지)                       │
│       │  task_prompt.md + prompt.md 기준 검증                         │
│       ▼                                                              │
│  STEP 5. Build & Compile & Test                                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. prompt-id 규칙

- 형식: `{YYYYMMDD}_{순번}_{키워드}`
- 예시: `20260311_001_dashboard_api`
- 키워드는 프롬프트 핵심 내용을 3단어 이내로 요약

---

## 3. result-docs 디렉토리 구조

```
01_docs/docs-agents/result-docs/{prompt-id}/
├── prompt.md                    ← 사용자 프롬프트 원문
├── task_prompt.md               ← AI_TECH_LEAD가 생성한 수행 프로세스
├── result_** ← 각 step별 결과 md
```

---

## 4. 각 파일 작성 규칙

### 4.1 prompt.md

사용자 프롬프트 원문을 그대로 저장한다. 수정하지 않는다.

```markdown
# Prompt

{사용자 프롬프트 원문}

---
저장일시: {YYYY-MM-DD HH:mm}
```

### 4.2 task_prompt.md

AI_TECH_LEAD 역할로 수행 프로세스를 생성한다.

```markdown
# Task Prompt

## 1. 프롬프트 분석
{요구사항 분석 내용}

## 2. 적용 Team
{Team 이름}

## 3. 수행 순서
| 순서 | 역할 | 수행 내용 | 참조 docs-claude | 병렬 가능 |
|------|------|----------|-----------------|----------|
| 1 | {역할} | {내용} | {문서 목록} | N |
| 2 | {역할} | {내용} | {문서 목록} | N |
| ... | ... | ... | ... | Y/N |

## 4. 병렬 처리 계획
{병렬 가능한 단계 식별 및 그룹핑}

## 5. 검증 기준
{task_prompt 기준 검증 항목 목록}

## 6. 빌드/테스트 계획
{빌드 명령어, 테스트 범위}

---
생성 역할: AI_TECH_LEAD
생성일시: {YYYY-MM-DD HH:mm}
```

### 4.3 result_{역할명}.md

각 역할이 수행 완료 후 결과를 기록한다.

```markdown
# Result: {역할명}

## 1. 수행 내용
{수행한 작업 요약}

## 2. 변경 파일 목록
| 파일 경로 | 변경 유형 | 설명 |
|----------|----------|------|
| {경로} | 생성/수정/삭제 | {설명} |

## 3. 주요 결정 사항
{아키텍처/설계/구현 결정 사항}

## 4. 다음 역할 참고 사항
{다음 역할이 반드시 알아야 할 내용}

## 5. 이슈/리스크
{발견된 이슈 또는 리스크}

---
수행 역할: {역할명}
수행일시: {YYYY-MM-DD HH:mm}
```

### 4.4 result_VERIFICATION.md

검증 결과를 기록한다.

```markdown
# Verification Result

## 검증 회차 기록

### Round {N}
- 검증일시: {YYYY-MM-DD HH:mm}
- 검증 기준: task_prompt.md + prompt.md
- 결과: 이상있음 / 이상없음
- 발견 사항:
  | 항목 | 상태 | 설명 |
  |------|------|------|
  | {항목} | PASS/FAIL | {설명} |
- 조치 사항: {수정 내용}

### Round {N+1}
...

## 최종 결과
- 연속 이상없음 횟수: {N}회
- 최종 판정: PASS / FAIL
- 빌드 결과: SUCCESS / FAIL
- 테스트 결과: SUCCESS / FAIL

---
검증일시: {YYYY-MM-DD HH:mm}
```

---

## 5. 역할 수행 규칙

### 5.1 각 역할의 입력

모든 역할은 수행 전 다음을 읽는다:

1. `prompt.md` (원문)
2. `task_prompt.md` (수행 프로세스)
3. 이전 역할의 `result_{역할명}.md` (해당하는 경우)
4. AGENT_DOCS_MAPPING.md에 따른 docs-claude 문서

### 5.2 각 역할의 출력

모든 역할은 수행 완료 후 다음을 생성한다:

1. `result_{역할명}.md` (결과 기록)
2. 실제 코드/설정 변경 (해당하는 경우)

### 5.3 역할 간 핸드오프 (순차)

```
역할 A 완료 → result_A.md 생성
    ↓
역할 B 시작 → prompt.md + task_prompt.md + result_A.md 읽기
    ↓
역할 B 완료 → result_B.md 생성
    ↓
역할 C 시작 → prompt.md + task_prompt.md + result_A.md + result_B.md 읽기
```

### 5.4 역할 간 핸드오프 (병렬 포함)

전체 프로세스가 A → B → C → D 단계로 이루어지고, B·C 단계가 분석 결과 병렬 가능하다고 판단될 때 다음을 따른다.

```
[STEP A] 순차 수행
    │
    ▼
[STEP B] B-1, B-2 병렬 수행 (C는 대기)
    │  각 agent가 독립 수행 후 산출물(result md) 각각 생성
    │  B-1 → result_B1.md
    │  B-2 → result_B2.md
    │
    ▼  (B-1, B-2 모두 완료될 때까지 대기)
[STEP C] C-1, C-2 병렬 수행 (D는 대기)
    │  각 agent가 독립 수행 후 산출물(result md) 각각 생성
    │  C-1 → result_C1.md  (입력: prompt.md + task_prompt.md + result_A + result_B1 + result_B2)
    │  C-2 → result_C2.md  (입력: prompt.md + task_prompt.md + result_A + result_B1 + result_B2)
    │
    ▼  (C-1, C-2 모두 완료될 때까지 대기)
[STEP D] 순차 수행
    │  입력: prompt.md + task_prompt.md + 이전 모든 result
```

**핵심 규칙:**

1. 순차 단계는 이전 단계 완료 후 수행한다
2. 병렬 단계의 각 agent는 독립 수행하며, 산출물(result md)을 각각 생성한다
3. 병렬 단계의 모든 agent가 완료되어 산출물이 생성된 이후에만 다음 단계로 진행한다
4. 다음 단계의 입력에는 이전 병렬 단계의 모든 산출물이 포함된다

---

## 6. 병렬 처리 규칙

### 6.1 병렬 가능 조건

다음 조건을 모두 만족할 때 병렬 수행 가능:

- 두 역할 간 입력/출력 의존성이 없음
- 서로 다른 모듈/파일을 대상으로 함
- task_prompt.md에서 "병렬 가능: Y"로 표기됨

### 6.2 병렬 실행 프로세스

전체 프로세스가 A → B → C → D 단계일 때:

| 단계 | 수행 방식 | 다음 단계 진행 조건 |
|------|----------|-------------------|
| A | 순차 수행 | A 완료 |
| B (B-1, B-2) | 병렬 수행 | B-1, B-2 모두 완료 + 산출물 생성 |
| C (C-1, C-2) | 병렬 수행 | C-1, C-2 모두 완료 + 산출물 생성 |
| D | 순차 수행 | - |

**병렬 단계 입력 파일:**
- 병렬 agent는 prompt.md + task_prompt.md + 이전 단계까지의 모든 result를 읽는다
- 같은 병렬 그룹 내 다른 agent의 result는 읽지 않는다 (아직 미생성)

### 6.3 일반적인 병렬 패턴

| 병렬 그룹 | 역할 조합 | 조건 |
|-----------|----------|------|
| 설계 후 구현 | AI_APPLICATION_ENGINEER + AI_SYSTEM_ENGINEER | 서로 다른 서버 구현 |
| 품질 검증 | AI_REVIEWER + AI_SECURITY_ENGINEER | 코드 리뷰와 보안 검증 독립 |
| 빌드 검증 | 각 모듈 빌드 | 독립 빌드 구조 |

### 6.4 병렬 불가 패턴

- Architect → Engineer (설계 완료 후 구현)
- Engineer → Reviewer (구현 완료 후 리뷰)
- 검증 Round N → Round N+1 (이전 검증 완료 후)

---

## 7. 검증 프로세스

### 7.1 검증 수행 조건

모든 역할 수행이 완료된 후 검증을 시작한다.

### 7.2 검증 기준

1. `prompt.md`의 요구사항이 모두 구현되었는가
2. `task_prompt.md`의 수행 프로세스가 모두 완료되었는가
3. 각 `result_{역할명}.md`의 이슈가 모두 해결되었는가
4. 아키텍처/보안/코드품질 기준 충족

### 7.3 검증 종료 조건

**연속 2회 이상 "이상없음"이 도출될 때까지 반복한다.**

```
Round 1: 이상있음 → 수정 후 재검증
Round 2: 이상없음 → 1회 연속
Round 3: 이상없음 → 2회 연속 → 검증 완료
```

### 7.4 검증 후 빌드/테스트

검증 완료 후 다음을 수행한다:

```bash
# 각 모듈 독립 빌드
cd hs-common && ./gradlew clean build -x test
cd gateway && ./gradlew clean build -x test
cd application && ./gradlew clean build -x test
cd system && ./gradlew clean build -x test

# 테스트 (구현된 경우)
cd {module} && ./gradlew test
```

---

## 8. Team별 수행 순서 참조

각 Team의 역할 순서는 해당 Team 문서를 참조한다:

| Team | 문서 | 역할 순서 |
|------|------|----------|
| FEATURE_DEVELOPMENT_TEAM | team/FEATURE_DEVELOPMENT_TEAM.md | TECH_LEAD → MODULE_ARCHITECT → MODULE_ENGINEER → REVIEWER → SECURITY → REFACTOR |
| EVENTFLOW_TEAM | team/EVENTFLOW_TEAM.md | EVENT_ARCHITECT → APP_ENGINEER → SYS_ENGINEER → REVIEWER → SECURITY |
| QUALITY_ASSURANCE_TEAM | team/QUALITY_ASSURANCE_TEAM.md | REVIEWER → SECURITY → REFACTOR |
| ARCHITECTURE_TEAM | team/ARCHITECTURE_TEAM.md | TECH_LEAD → MODULE_ARCHITECT(선택) → EVENT_ARCHITECT(선택) → REVIEWER |
| GATEWAY_TEAM | team/GATEWAY_TEAM.md | GW_ARCHITECT → GW_ENGINEER → SECURITY → REVIEWER |
| APPLICATION_TEAM | team/APPLICATION_TEAM.md | APP_ARCHITECT → APP_ENGINEER → REVIEWER → SECURITY |
| SYSTEM_TEAM | team/SYSTEM_TEAM.md | SYS_ARCHITECT → SYS_ENGINEER → REVIEWER → SECURITY |
| INFRA_TEAM | team/INFRA_TEAM.md | INFRA_ENGINEER → SECURITY |

---

## 9. 결과물 보존 정책

- `result-docs/**` 하위의 모든 파일은 삭제하지 않는다
- `result-docs/**`는 .gitignore에 등록하여 Git 추적에서 제외한다
- 결과물은 향후 참조 및 감사 용도로 보존한다

---

END OF FILE
