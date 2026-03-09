# ERROR_CODE_GLOBAL_POLICY.md

## 1. 기본 규칙

- OK = 1
- FAIL = -1
- 에러 코드는 5자리 숫자

형식:
{프로젝트번호}{비즈니스1레벨}{비즈니스2레벨}{시퀀스}

---

## 2. 프로젝트 번호

| 프로젝트 | 번호 |
|----------|------|
| Gateway | 1 |
| System | 2 |
| Application | 3 |
| 공통 | 9 |

---

## 3. 신규 프로젝트 추가

1. 번호 4-8 중 할당
2. ef-common/ErrorCode.java 주석 추가
3. 프로젝트 CLAUDE.md 업데이트