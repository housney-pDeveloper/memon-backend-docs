# PROCESS.md

Git 정책, 명령 라우팅 등 개발 프로세스 규칙을 통합 정의한다.

---

## 1. Git Commit 규칙

### 1.1 Subject

- 50자 이하
- 마침표 및 특수기호 금지
- 동사 원형 + 대문자 시작
- 완전한 문장 금지
- 한글로 작성

### 1.2 Commit 구조

```
type

body

footer
```

### 1.3 type 목록

- feat
- fix
- docs
- style
- refactor
- test
- chore

### 1.4 body

- 한 줄 72자 이내
- 무엇/왜 변경했는지 설명
- 한글로 작성

### 1.5 footer

- 유형: #이슈번호
- Fixes, Resolves, Ref, Related to

---

## 2. 명령 라우팅 (Prefix)

| Prefix | 대상 디렉토리 |
|--------|----------------|
| [app]  | ./application |
| [gate] | ./gateway |
| [data] | ./database |
| [comm] | ./hs-common |
| [sys]  | ./system |
| [all]  | ./** |
| [README] | ./**/README.md |

### 2.1 작업 범위 규칙

- 해당 디렉토리만 수정
- 다른 디렉토리 수정이 필요하면 작업 중단 후 질문
- 읽기 상호작용은 허용

---

END OF FILE
