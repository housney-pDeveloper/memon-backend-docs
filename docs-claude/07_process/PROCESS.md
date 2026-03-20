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

## 3. Claude Code 플러그인 활용

Git 작업 수행 시 다음 플러그인을 활용한다.

### 3.1 커밋 플러그인

| 플러그인 | 명령 | 용도 |
|---------|------|------|
| commit-commands | `/commit` | 구조화된 Git 커밋 생성 (섹션 1의 규칙 자동 적용) |
| commit-commands | `/commit-push-pr` | 커밋 + 푸시 + PR 생성 일괄 수행 |
| commit-commands | `/clean_gone` | 원격에서 삭제된 로컬 브랜치 및 연관 worktree 정리 |

### 3.2 GitHub 플러그인

| 플러그인 | 명령 | 용도 |
|---------|------|------|
| github | `gh pr create` | PR 생성 |
| github | `gh pr comment` | PR 코멘트 작성 |
| github | `gh pr review` | PR 리뷰 승인/변경요청 |
| github | `gh issue` | 이슈 관리 |

### 3.3 활용 프로세스

```
작업 완료 후 커밋 시:
    /commit → 섹션 1 규칙에 따른 구조화된 커밋 생성

커밋 + PR 생성 시:
    /commit-push-pr → 커밋, 푸시, PR 생성 일괄 수행

브랜치 정리 시:
    /clean_gone → 원격 삭제된 브랜치 + worktree 정리
```

### 3.4 커밋 규칙과 플러그인 연동

`/commit` 사용 시 다음 규칙이 자동 적용된다:
- 섹션 1.1의 Subject 규칙 (50자 이하, 한글, 동사 원형)
- 섹션 1.2의 Commit 구조 (type + body + footer)
- 섹션 1.3의 type 목록 (feat, fix, docs, style, refactor, test, chore)

---

END OF FILE
